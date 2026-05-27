# KuzelovLab & Lenovo Remote Homelab Infrastructure

A fully automated, highly secure, dual-site home server infrastructure managed entirely as **Infrastructure as Code (IaC)** via Ansible. This setup features containerized services, split-horizon DNS, hybrid tiered storage, and an encrypted off-site backup pipeline bypassing CGNAT.

---

## Table of Contents

1. [System Architecture](#1-system-architecture)
2. [Core Pillars](#2-core-pillars)
3. [The 5-Step Recipe: Deploying a New Service](#3-the-5-step-recipe-deploying-a-new-service)
4. [Worked Example: Deploying & Exposing Uptime Kuma](#4-worked-example-deploying--exposing-uptime-kuma)
5. [Maintenance & Operations Cheat Sheet](#5-maintenance--operations-cheat-sheet)

---

## 1. System Architecture

```text
       SITE A (Production - Proxmox Host)                  SITE B (Backup Target - Lenovo SFF)
┌──────────────────────────────────────────────────┐     ┌──────────────────────────────────────────────────┐
│ rpool (NVMe SSD)                                 │     │ 512GB NVMe SSD                                   │
│ ├─ Proxmox VE / local-zfs                        │     │ ├─ Debian 13 (Trixie) Host OS                    │
│ ├─ VM 100: kuzelovlab (Core/SSO)                 │     │ └─ /srv (Docker configs / Journiv DB)            │
│ │  └─ /srv_fast (100GB NVMe LVD for DBs)         │     │                                                  │
│ └─ VM 101: medialab (Media / Arrs)               │     │ 1TB Crucial SATA SSD                             │
│    └─ OS Disk (64GB NVMe for Metadata)           │     │ └─ /mnt/backup_tank                              │
│                                                  │     │    ├─ /local_lenovo (Borg local backup)          │
│ tank (6TB HDD ZFS Pool RAID-Z1)                  │     │    └─ /remote_kuzelovlab (Borg WAN backup)       │
│ ├─ VM 100: /srv (2TB LVD for Photo/File Bulk)    │     │                                                  │
│ └─ VM 101: /data (4TB LVD for Movies/Torrents)   │     │ Networking & Security                            │
│                                                  │     │ ├─ Tailscale (Bare Metal - Subnet Router)        │
│ Networking & Security                            │     │ ├─ AdGuard Home (Local DNS on Port 3000)         │
│ ├─ Tailscale (LVM Subnet Router on Host PVE)     │     │ └─ Caddy Reverse Proxy (Port 80/443)             │
│ ├─ Caddy Reverse Proxy (Port 80/443)             │     │                                                  │
│ ├─ Cloudflare Tunnel (CGNAT Bypass)              │     │ Services                                         │
│ └─ AdGuard Home (Split-Horizon DNS)              │     │ └─ Journiv (PostgreSQL)                          │
└──────────────────────────────────────────────────┘     └──────────────────────────────────────────────────┘
                          ▲                                                  ▲
                          └────────────────────── Tailscale VPN ────────────┘
```

---

## 2. Core Pillars

### I. Storage Tiering (Performance vs. Capacity)

To optimize performance, we separate databases (high IOPS) from bulk storage (high capacity):

- **Hot Tier (NVMe SSD):** All databases (Postgres, MariaDB, Redis) and application configs live on NVMe storage. (Migrated from `/srv` to `/srv_fast` via `rsync -avH`.)
- **Cold Tier (HDD Pool):** Bulk files (Immich libraries, Seafile sync blocks, movies, and torrents) live on the HDD ZFS pool.

### II. Secure Remote Access & CGNAT Bypass

- **Cloudflare Tunnel (`cloudflared`):** Bypasses home CGNAT without opening router ports. Runs outbound on the proxy network, routing securely to `https://caddy:443` (using *No TLS Verify* and explicit `originServerName` / `httpHostHeader` configurations).
- **Cloudflare WAF:** Protects public subdomains using custom firewall rules (e.g., blocking all non-Czechia traffic at the DNS edge).
- **Tailscale:** A peer-to-peer WireGuard mesh VPN.
  - **Lenovo Host:** Runs on bare metal, acting as a Subnet Router (`192.168.66.0/24`) with kernel-level IP forwarding and UDP GRO offload optimizations.
  - **Firewall Fix:** Explicit `iptables` FORWARD and MASQUERADE rules applied to allow subnet routing alongside Docker.
- **Split-Horizon DNS:**
  - **Away from home:** Public subdomains route safely through the Cloudflare Tunnel.
  - **At home / on VPN:** AdGuard Home rewrites DNS queries (e.g., `photos.zen132.cz` -> `192.168.0.20`), routing traffic directly over the local network (1Gbps+ speed, zero internet overhead).

### III. Identity Management (SSO)

- **Authentik:** Centralized Identity Provider (OIDC/OAuth2) running on `kuzelovlab` behind Caddy. It protects Immich (`fotky.zen132.cz`), providing seamless SSO for web and mobile clients.

### IV. Automated 3-2-1 Backups (Borgmatic)

- **Borgmatic (Docker-based on Kuzelovlab):**
  - Runs on a schedule at **03:00 AM**.
  - Triggers native database dumps (`mariadb-dump`, `pg_dumpall`) into `/srv/database_dumps` right before backup.
  - Compresses, encrypts, and deduplicates the entirety of `/srv` and `/srv_fast`.
  - Ships backups over Tailscale to the remote Lenovo server.
  - Integrates with Healthchecks.io for active success/failure monitoring on the dashboard.
- **Borgmatic (Docker-based on Lenovo):**
  - Runs locally at **05:00 AM**.
  - Performs database dumps for Journiv.
  - Backs up `/srv` to `/mnt/backup_tank/local_lenovo` (the internal SATA SSD).
- **The Hardware Quirk:** The remote 1TB SSD uses the `usb-storage` kernel quirk (disabling buggy UAS) to ensure 100% write stability on the physical disk without silent data corruption.

---

## 3. The 5-Step Recipe: Deploying a New Service

To add a new service (e.g., `jellyfin`) to your architecture, follow this exact workflow:

1. **DNS (Cloudflare):** Create a CNAME for the subdomain (e.g., `jellyfin`) pointing to your placeholder/wildcard. Keep it **Grey Cloud (DNS Only)**.
2. **Local DNS (AdGuard Home):** Add a DNS Rewrite for `jellyfin.zen132.cz` -> pointing to your Caddy/Main VM IP (`192.168.0.20`).
3. **Caddyfile:** Add the site block to the per-host Caddy template `roles/caddy/templates/Caddyfile.<host>.private.j2`:

   ```caddy
   https://jellyfin.{{ domain }} {
       tls { dns cloudflare {env.CLOUDFLARE_API_TOKEN} }
       reverse_proxy 192.168.0.30:8096 # Point to the target VM
   }
   ```

4. **Ansible:** Add a play to `apps.yml` using the `docker_stack` role (plus a compose template under `templates/`). Ensure the container is on the `proxy` network if on the same host, or expose the correct port if on the Media VM.
5. **Deploy:** Run the app play, then run the Caddy playbook to reload Caddy's configuration.

> See section 4 for a complete, copy-paste worked example.

---

## 4. Worked Example: Deploying & Exposing Uptime Kuma

This walks through deploying a brand-new app and exposing it safely to the public internet
via the **Cloudflare Tunnel + Caddy** architecture. Example: **Uptime Kuma** (a lightweight
monitoring dashboard) hosted at `status.zen132.cz` on the `kuzelovlab` VM.

### Step 1 — Local DNS Configuration (AdGuard Home)

Give your local network fast, direct access before the public internet sees it.

1. Log in to your AdGuard Home dashboard (`http://192.168.0.20:3000`).
2. Go to **Filters -> DNS rewrites**.
3. Click **Add DNS rewrite**:
   - **Domain:** `status.zen132.cz`
   - **IP Address:** `192.168.0.20` (the local IP of the Main VM where Caddy lives).
4. Click **Save**.

> If you use Tailscale, also add `status.zen132.cz` -> your server's `100.x.x.x` IP in the
> Tailscale MagicDNS panel.

### Step 2 — Update the Caddyfile Template

Tell Caddy how to route traffic for the new domain to the container. Edit the per-host
template for `kuzelovlab`: `roles/caddy/templates/Caddyfile.kuzelovlab.private.j2`, and add
the block at the bottom:

```caddy
# --- Uptime Kuma (Monitoring) ---
https://status.{{ domain }} {
    tls {
        dns cloudflare {env.CLOUDFLARE_API_TOKEN}
    }
    # Point directly to the container name and its internal port
    reverse_proxy uptime-kuma:3001
}
```

### Step 3 — Deploy the Application with Ansible

The app is a standard single-container compose stack, so it slots straight into the generic
`docker_stack` role. **Crucially, the container must join the `proxy` network** so Caddy can
reach it.

**3a.** Create the compose template `templates/uptime-kuma/docker-compose.yml.j2`:

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    container_name: uptime-kuma
    restart: unless-stopped
    volumes:
      - "{{ stack_path }}/data:/app/data"
    # No host ports (e.g. 3001:3001) are published — Caddy and the Tunnel
    # reach the container internally over the shared `proxy` network.
    networks:
      - proxy

networks:
  proxy:
    external: true
```

**3b.** Add a play to `apps.yml`:

```yaml
- name: Deploy Uptime Kuma
  hosts: kuzelovlab
  become: true
  vars:
    stack_name: uptime-kuma
    stack_path: /srv/uptime-kuma
    stack_compose_template: uptime-kuma/docker-compose.yml.j2
    stack_extra_dirs:
      - /srv/uptime-kuma/data
  roles:
    - docker_stack
```

**3c.** Deploy the app, then push the new Caddyfile and reload Caddy:

```bash
export ANSIBLE_CONFIG=$(pwd)/ansible.cfg   # see README "How to run"

# Deploy just this app
ansible-playbook apps.yml --limit kuzelovlab --ask-vault-pass -K

# Reload Caddy so it picks up the new site block
ansible-playbook server_caddy.yml --ask-vault-pass -K
```

### Step 4 — Expose via Cloudflare Tunnel (in the browser)

With the app running and Caddy ready to route it, open the secure public bridge.

1. Log in to the **Cloudflare Zero Trust Dashboard**.
2. Go to **Networks -> Tunnels -> select `homelab-tunnel` -> Configure**.
3. Open the **Public Hostname** tab -> **Add a public hostname**.
4. Fill in the fields exactly:
   - **Subdomain:** `status`
   - **Domain:** `zen132.cz`
   - **Service:** `HTTPS` -> `caddy:443`
5. **Additional Application Settings:**
   - Expand **TLS** -> turn **No TLS Verify** to **ON**.
   - Expand **HTTP Settings** -> in **HTTP Host Header**, type `status.zen132.cz` (this tells Caddy which site block to use).
6. Click **Save hostname**.

### Step 5 — Test the Public Access

1. Disconnect your phone from home Wi-Fi **and** Tailscale (switch to mobile data).
2. Go to `https://status.zen132.cz`.

---

## 5. Maintenance & Operations Cheat Sheet

### General Health Check

```bash
# Check disk usage
df -h

# Check RAM and swap
free -h

# Monitor real-time processes
htop
```

### Borgmatic Backup Operations (on Kuzelovlab)

```bash
# Force a manual backup run (with stats and progress)
sudo docker exec -it borgmatic borgmatic create --verbosity 1 --stats --progress

# List all completed backups
sudo docker exec -it borgmatic borgmatic list

# Show storage and deduplication statistics
sudo docker exec -it borgmatic borgmatic info

# Run a deep integrity check on the remote repository
sudo docker exec -it borgmatic borgmatic check --verbosity 1 --force
```

### Mount Recovery (Browse Backups as a Folder)

```bash
# Mount the remote repository as a local filesystem on kuzelovlab
sudo docker exec -it borgmatic borgmatic mount --mountpoint /mnt/recovery

# Browse it on the host OS
cd /mnt/recovery/srv/

# Unmount when finished
sudo docker exec -it borgmatic borgmatic umount --mountpoint /mnt/recovery
```

### Docker Cleanup (run monthly to reclaim SSD space)

```bash
# Delete all unused images, stopped containers, and dangling layers
docker system prune -a --volumes
```
