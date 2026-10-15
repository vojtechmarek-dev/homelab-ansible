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

### IV. Automated 3-2-1 Backups (Borgmatic + config sync)

- **Borgmatic (Docker-based on Kuzelovlab):**
  - Runs on a schedule at **03:00 AM**.
  - Triggers native database dumps (`mariadb-dump`, `pg_dumpall`) into `/srv/database_dumps` right before backup.
  - CouchDB (Obsidian LiveSync, `/srv_fast/obsidian-livesync`) has no dump hook on purpose: its `.couch` files are append-only, so a live file copy is consistent. Every synced device also holds a full copy of the vault.
  - Compresses, encrypts, and deduplicates the entirety of `/srv` and `/srv_fast`.
  - Ships backups over Tailscale to the remote Lenovo server.
  - Integrates with Healthchecks.io for active success/failure monitoring on the dashboard.
- **Borgmatic (Docker-based on Lenovo):**
  - Runs locally at **05:00 AM**.
  - Performs database dumps for Journiv.
  - Backs up `/srv` to `/mnt/backup_tank/local_lenovo` (the internal SATA SSD).
- **Config sync (MediaLab):**
  - Runs at **02:30 AM**, ahead of Kuzelovlab's Borg run so the night's config is already in place.
  - `rsync` of `/srv/medialab/config` (~1.4GB) to `kuzelovlab:/srv/medialab-backup`, which sits inside Kuzelovlab's Borg source — so it inherits encryption, deduplication, off-site shipping and 7/4/6 retention without a second repository.
  - MediaLab has **no Borg repo of its own on purpose**: `/data` (~2TB of media and torrents) is re-downloadable and deliberately unprotected, leaving only config worth keeping. A second repo, container and key exchange was not worth 1.4GB.
  - Jellyfin's metadata is excluded — it is re-fetched from providers and churns nightly, which would defeat deduplication.
- **The Hardware Quirk:** The remote 1TB SSD uses the `usb-storage` kernel quirk (disabling buggy UAS) to ensure 100% write stability on the physical disk without silent data corruption.

### V. Storage Lifecycle on MediaLab

`/data` is a fixed 2.4TB volume with no expansion path, so the media stack manages its own reclamation rather than relying on headroom.

**Hardlinks are the foundation.** Radarr and Sonarr import from `/data/torrents` into `/data/media` by hardlink, so both names share one inode and a seeding torrent costs no extra disk. Anything that *replaces* a media file — a Tdarr transcode, an \*arr quality upgrade, a manual delete — breaks that link, and the torrent copy silently becomes a full-size duplicate.

**qbit_manage** (every 30 min) is what notices. Its `tag_nohardlinks` check tags un-linked torrents `noHL`, and the `share_limits` groups then delete them:

| group | matches | seeding floor | deletes |
|---|---|---|---|
| `noHL-sktorrent` | `noHL` + private tracker | `qbit_private_seeding_time` | yes |
| `noHL` | `noHL`, public trackers | none | yes, immediately |
| `default` | still hardlinked | `qbit_default_max_seeding_time` | no — stops only |

Deletions route through `/data/torrents/.recycle_bin` for a short window before the space actually returns.

**Two non-obvious interactions, both intentional:**

1. **The `default` group stops torrents; Radarr/Sonarr then remove them.** With *Remove Completed Downloads* enabled, an \*arr sees a completed download that is no longer seeding and deletes it from the client. So `qbit_default_max_seeding_time` is effectively "how long before a torrent is discarded", not just "stopped". The media copy survives — only the torrent-side hardlink goes.
2. **Transcoding after the torrent is gone is strictly cheaper.** A still-linked file costs `new + stranded original` until cleanup runs; a file whose torrent has already been removed is link-count-1, so the old data frees the instant Tdarr replaces it. No tag, no wait, no recycle bin.

**Tdarr** transcodes h264 → h265 on CPU only (the VM has emulated VGA, no render node, so every hardware encoder reports *enabled but not working*). Plugin filters exclude h265, 2160p and HDR/DV; `Migz1FFMPEG_CPU` is used rather than the `075*` plugins because it preserves subtitle and audio streams.

**Cleanuparr** removes downloads the \*arrs refuse to import — notably padded `.exe` malware from public trackers, which otherwise sits in the queue until cleared by hand.

> **Not captured by Ansible:** qBittorrent's own settings — torrent queueing limits, excluded file names, WebUI options — live only in `qBittorrent.conf` on the host. They *are* covered by the config sync, but a from-scratch rebuild needs them set by hand. qbit_manage owns all share limits (`disable_qbt_default_share_limits: true`), so setting those in the qBittorrent UI is pointless: they are overwritten within 30 minutes. Change them in `apps.yml`.

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

### Media Stack Storage (on MediaLab)

```bash
# Force a reclaim pass instead of waiting for the 30-minute cycle
sudo docker restart qbit_manage && sleep 30 && sudo docker logs --since 40s qbit_manage | tail -20

# What was deleted, and by which share-limit group
sudo docker logs qbit_manage 2>&1 | grep -E 'Cleaning up torrents|Contents Deleted' | tail

# Torrent data with no remaining media hardlink (i.e. reclaimable)
find /data/torrents/radarr /data/torrents/tv-sonarr -type f -links 1 -size +100M -printf '%s\n' \
  | awk '{s+=$1} END {printf "%.1f GiB in %d files\n", s/1073741824, NR}'

# Pending reclaim still parked on disk
sudo du -sh /data/torrents/.recycle_bin /data/torrents/.orphans

# Codec census of the library (filename-based; Tdarr itself reads real metadata)
find /data/media -type f \( -name '*.mkv' -o -name '*.mp4' \) -printf '%s\t%p\n' \
  | awk -F'\t' '{c=tolower($2)
      if (c ~ /2160p/) b="2160p"; else if (c ~ /x265|h265|hevc/) b="h265"
      else if (c ~ /x264|h264|avc/) b="h264 (transcodable)"; else b="unknown"
      n[b]++; s[b]+=$1} END {for (k in n) printf "%-20s %5d files %8.1f GiB\n", k, n[k], s[k]/1073741824}'

# Test the disk alert path end to end (sends, then clears on the next normal run)
ansible-playbook server_disk_alert.yml --ask-vault-pass -K -e disk_alert_free_percent=99
sudo /usr/local/bin/disk-alert.sh

# Force a config sync rather than waiting for 02:30
sudo /usr/local/bin/medialab-config-sync.sh && echo OK
```

### Docker Cleanup (run monthly to reclaim SSD space)

```bash
# Delete all unused images, stopped containers, and dangling layers
docker system prune -a --volumes
```
