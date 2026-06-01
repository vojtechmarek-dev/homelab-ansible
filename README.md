
# Ansible Playbooks for Home Server Setup

This repository contains personal Ansible playbooks designed for setting up and managing a home server. These playbooks document each step to easily replicate the setup on managed new servers currently in use.

## Documentation

- **[INFRASTRUCTURE.md](INFRASTRUCTURE.md)** — the homelab handbook: system architecture
  (dual-site topology, storage tiering, split-horizon DNS, 3-2-1 backups), a worked example
  for deploying and publicly exposing a new service, and a maintenance/operations cheat sheet.
- **This README** — repository layout, how to run the playbooks, and the day-to-day workflow
  for adding services.

## Introduction to Ansible

Ansible automates the management of remote systems, ensuring they maintain a desired state. While this README provides a basic overview, it is recommended to consult the [official Ansible documentation](https://docs.ansible.com/) for a more comprehensive understanding. 

### Key Components of Ansible

- **Control Node**: The computer with Ansible installed, used to manage all remote hosts.
- **Managed Node**: A remote computer (host) managed by Ansible.

Although there are various tools that automate host management via Ansible, in its purest form, Ansible is a set of command-line tools that require manual execution.

## Getting Started

### Prerequisites

Ensure the following are installed on both the Control and Managed nodes:

- Python 3

### Installation

1. Install Ansible on the Control node:

    ```bash
    sudo apt update
    sudo apt upgrade -y
    sudo apt install -y python3
    sudo apt install python3-pip -y
    python3 -m pip -V  # Verify pip installation
    ```

    :bulb: **Note**: As of Python 3.11, it is strongly recommended to use `pipx` for Ansible. See the [Ansible installation guide](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installation-guide) for more details.

    If Ansible commands are not accessible, try adding your Python user bin directory to your PATH. For macOS, this can be configured as follows:

    ```bash
    echo "export PATH="$(python3 -m site --user-base)/bin:\$PATH"" >> ~/.bash_profile
    ```

## Purpose of This Repository

The main goal of this repository is to learn Ansible and server management from the ground up, without relying on external tools or "crutches." Each step in the server setup process is meticulously documented in Ansible playbooks, ensuring the setup can be easily replicated on new (or replacement) servers.

## Core Concepts Used
### Secrets Management and Templating

To avoid committing sensitive data like API keys or passwords to Git, this project uses a combination of Ansible Vault and the ansible.builtin.template module.

1. Ansible Vault: All secrets (e.g., secrets.github.ghcr_token) are stored in an encrypted file (e.g., ../ansible-secrets/secrets.yml), which is kept outside of this repository.

2. Templating: Instead of static configuration files, we use **Jinja2** templates (e.g., `docker-compose.yml.j2`). These templates contain placeholders for variables.

3. Deployment: When a playbook is run, Ansible reads the template, injects the decrypted secrets from the vault into the placeholders, and generates the final configuration file on the target server.

This approach ensures that our Git repository is secure, while deployments are automated and repeatable.

## Repository Structure

```
ansible.cfg        # auto-loads inventory.yml + ../ansible-secrets/secrets.yml, sets roles_path
inventory.yml      # hosts + functional groups
site.yml           # orchestrator: runs base config -> networking -> apps -> bespoke services
apps.yml           # all simple docker-compose apps, one play each, via the docker_stack role
server_*.yml       # bespoke playbooks for services/host-setup that need custom logic
provisioning/      # one-time, destructive host/disk setup (NOT run by site.yml)
roles/             # base, user, docker, ssh_security, docker_stack + per-service roles
templates/         # Jinja2 docker-compose / config templates, one dir per service
```

### Provisioning vs. convergence

`provisioning/` holds **one-time, destructive** disk/storage setup (partitioning, mkfs,
LVM, mounts) — run by hand once when a host or disk is added, e.g.:

```bash
ansible-playbook provisioning/kuzelovlab-fast-storage.yml --ask-vault-pass -K
```

These are deliberately **excluded from `site.yml`**: re-running them is dangerous, and they
describe initial state rather than the converged state `site.yml`/`apps.yml` maintain.
(`server_tailscale_baremetal.yml` and `server_prepare_remote_borg_backup.yml` are similar
host-bootstrap one-offs.)

### How to run

`ansible.cfg` wires the inventory and secrets automatically, so no `-i` flags are needed.
Because the repo may live on a world-writable filesystem (e.g. WSL `/mnt/d`), Ansible
silently ignores `./ansible.cfg` unless `ANSIBLE_CONFIG` points at it explicitly.

The repo ships a `.envrc` that exports `ANSIBLE_CONFIG` only inside this directory, via
[**direnv**](https://direnv.net/) — recommended so the variable doesn't leak into other
Ansible repos (e.g. a work one):

```bash
# one-time control-node setup
sudo apt install -y direnv
echo 'eval "$(direnv hook bash)"' >> ~/.bashrc
source ~/.bashrc

cd /path/to/this/repo
direnv allow .                                 # trust the .envrc
ansible-galaxy collection install -r requirements.yml
```

After that, `cd` into the repo auto-exports `ANSIBLE_CONFIG`; `cd` out unsets it.

```bash
ansible-playbook site.yml --ask-vault-pass -K           # everything, safe order
ansible-playbook apps.yml --ask-vault-pass -K           # just the docker apps
ansible-playbook apps.yml --limit immich --ask-vault-pass -K   # one host group
ansible-playbook apps.yml --tags homepage_zen132 --ask-vault-pass -K   # one tagged play
```

Add `--check --diff` to preview rendered files without applying changes.

**Without direnv** — export manually before each session (won't auto-unload):

```bash
export ANSIBLE_CONFIG=/absolute/path/to/repo/ansible.cfg
```

**Faster auth** — load the SSH key into the agent once to skip per-task passphrase prompts:

```bash
eval "$(ssh-agent -s)"; ssh-add ~/.ssh/ansible_manager_homelab_rsa
```

### The `docker_stack` role

Most apps follow the same pattern: make dirs -> render compose (+ optional env) ->
`docker compose up` -> restart on change. That pattern lives once in `roles/docker_stack`.
Each app is just a play in `apps.yml` supplying `stack_*` vars (and any vars its template
references). See [Adding a New Service](#adding-a-new-service-to-the-homelab) below.

## Playbooks Overview

The playbooks are divided into two main categories: initial server setup and application deployment.

### Server Setup Playbooks

- `server_basic_config.yml`: Basic configuration and common packages for all servers.
- `server_user.yml`: Creates a new user with sudo privileges for ongoing management. This playbook should be run with secrets.
- `server_docker.yml`: Docker setup, with tasks including sensitive data management (e.g., passwords, authorized keys) stored externally (not in this repository). 


### Application Deployment Playbooks

These playbooks deploy and manage specific services using Docker. They leverage the templating method described above.

1. `server_discord_bot_lafayette.yml`: Deploys a custom Discord bot. It logs into a private container registry, pulls the latest image, and generates a docker-compose.yml from a template with the necessary API keys from Ansible Vault.

2. `server_tailscale.yml`: Configures the server as a Tailscale subnet router. It sets the required kernel parameters and generates a docker-compose.yml with the correct subnet configuration.


### Running playbooks with secrets
Secrets live in the external vaulted inventory `../ansible-secrets/secrets.yml`, which
`ansible.cfg` loads automatically (see [How to run](#how-to-run)). Pass `--ask-vault-pass`
to decrypt it:

```bash
ansible-playbook server_user.yml --ask-vault-pass
```

### Tailscale preauth key

The `tailscale` role consumes `secrets.tsauth_key` (preauth key) to register Docker-Tailscale
hosts (`kuzelovlab`, `medialab`, `homelab-pi`) without a browser-based login. Generate
it once in Tailscale admin (**Settings → Keys → Generate auth key**):

- **Reusable** (same key serves all three hosts)
- **Tagged** (e.g. `tag:server`) — avoids per-machine admin approval
- **Expiry** 90 days for hygiene, or `never` for set-and-forget

Add it to the vault file:

```bash
ansible-vault edit ../ansible-secrets/secrets.yml
# add under all.vars.secrets:
#   tsauth_key: tskey-auth-...
```

The key is only consumed on **first registration** (or when `/srv/tailscale/state` is
wiped); once a host has a valid node identity, container recreates and VM reboots use the
persisted state and ignore the key. With this set, deploying or reprovisioning a new
Docker-Tailscale host is fully unattended — no browser click, no SIGTERM after 60s.

> `homelab-thinkcentre` runs Tailscale **bare-metal** (`server_tailscale_baremetal.yml`),
> not via this role, so it isn't covered by this key.

### Playbook Execution Order

## Bootstraping the server

If your ssh user is not sudoer (or you cannot ssh as root), and you have no sudo user to connect as - ssh into server and add your user to sudoers manually.
Before adding ssh key you need to connect to ssh with password add `-k -K` to your playbook call

```bash 
usermod -aG sudo username
```

ansible-playbook -i inventory.yml -i ../ansible-secrets/secrets.yml server_basic_config.yml -k -K
ansible-playbook -i inventory.yml -i ../ansible-secrets/secrets.yml server_user.yml -k -K

1. **Run `server_basic_config.yml` as root** with SSH access.
2. **Run `server_user.yml` as root**.
3. Verify that the user created in the previous playbook has sudo privileges.

then run harden to have your server only accessible via ...
then docker and tailscale, then the rest

:warning: **Important**: After the initial setup, all subsequent playbook runs should be executed under the new user (root access will not be allowed). Use `--ask-vault-pass` for playbooks using Ansible Vault, and `--ask-become-pass` for those requiring sudo privileges.

## Roles Included

The playbooks currently include the following roles:

- **base**: Basic configuration of a Linux Debian server.
- **user**: Adding `ansible-manager` superuser to the server with ssh access to the server.
- **docker**: Configuration of Docker service including installing needed packages.
- **ssh_security**: SSH hardening (used by `server_harden.yml`).
- **docker_stack**: Generic docker-compose stack deployer used by every play in `apps.yml`.
- **caddy / nextcloud / tailscale / borgbackup / borgmatic**: services with bespoke logic
  (image builds, post-deploy `occ`, sysctl, cron) that the generic role can't cover; each
  is driven by a thin `server_<name>.yml`.


# Adding a New Service to the Homelab

The full, current step-by-step procedure — covering local/public DNS, the Caddy
site block, the `docker_stack` play, and exposing the service through the Cloudflare
Tunnel — lives in **[INFRASTRUCTURE.md](INFRASTRUCTURE.md)**:

- **Section 3** — the concise 5-step recipe.
- **Section 4** — a complete worked example (deploying and publicly exposing Uptime Kuma).

For the quick "just add a compose app" pattern, see
[The `docker_stack` role](#the-docker_stack-role) above.


## Client-Side Troubleshooting Checklist

If you get a `NS_ERROR_UNKNOWN_HOST` or similar DNS error after deployment:

-   [ ] Did you add the DNS rewrite in **AdGuard Home**?
-   [ ] Did you add the MagicDNS entry in **Tailscale**?
-   [ ] Is your device configured to use AdGuard Home as its DNS server (check router DHCP settings)?
-   [ ] Have you disabled **"Private DNS" / "Secure DNS"** in your phone's or browser's settings?
-   [ ] Have you tried flushing your computer's DNS cache?

# Home Server Disaster Recovery Plan

Procedure for restoring the homelab when a host (or its data) is lost. Numbers and paths
below reflect the **current** playbooks — older revisions of this section referenced
playbooks (`deploy_*.yml`, `inventory.ini`) that no longer exist.

## Backup Topology

Two independent encrypted Borg repositories cover the two writer hosts. Both repositories
use the same passphrase, stored in the vault as `secrets.borgbackup_password`.

| Writer | Driver | Schedule | Sources | Destination |
|---|---|---|---|---|
| `kuzelovlab` | Borgmatic (Docker, `b3vis/borgmatic`) — `server_borgmatic.yml` | daily 04:00 | `/srv`, `/srv_fast`, `/mnt/samba_share` + pre-backup DB dumps (Authentik, Mealie, Seafile) to `/srv/database_dumps` | `ssh://borg@<thinkcentre TS IP>/mnt/backup_tank/kuzelovlab` (pushed over Tailscale) |
| `homelab-thinkcentre` (a) | Borgmatic (Docker) — `server_borgmatic.yml` | daily 05:00 | `/srv`, `/etc` + pre-backup DB dump (Journiv) to `/srv/database_dumps` | local `/mnt/backup_tank/local_borg_repo` |
| `homelab-thinkcentre` (b) | System `borg` + cron + bash script — `server_borgbackup.yml` (`roles/borgbackup`) | daily 02:30 | `/srv` | same local repo `/mnt/backup_tank/local_borg_repo` |

Notes:

- `homelab-thinkcentre` is also the **receiver** for `kuzelovlab`'s remote pushes — that side is configured by `server_prepare_remote_borg_backup.yml` (creates `borg` user + authorized key + target dir).
- Both layers on `homelab-thinkcentre` write to the same repo path; pick whichever archive name is freshest at restore time.
- Retention on both Borgmatic configs: `7 daily / 4 weekly / 6 monthly`.
- Healthchecks.io pings on success: `secrets.kuzelovlab_borg_healtcheck_url` (kuzelovlab) and `secrets.healthcheck_borg_url` (homelab-thinkcentre + the system script).

## Prerequisites

Before starting any restore, have:

1. A control node with this repo cloned, `direnv` activated (`ANSIBLE_CONFIG` exported), and `community.docker` installed (`ansible-galaxy collection install -r requirements.yml`).
2. The vault passphrase for `../ansible-secrets/secrets.yml`.
3. The Borg passphrase = `secrets.borgbackup_password` from the vault.
4. SSH access (LAN initially; Tailscale once it's deployed) to the target host.
5. **For Scenario A** — `homelab-thinkcentre` reachable and `/mnt/backup_tank` mounted.
6. **For Scenario B** — the external backup-tank disk physically attached to the new `homelab-thinkcentre`; if the disk is gone, the only recoverable data is whatever is still live on `kuzelovlab`.

## Scenario A — `kuzelovlab` VM lost (Proxmox VM corruption / disk failure)

The remote Borg repo on `homelab-thinkcentre:/mnt/backup_tank/kuzelovlab` is the authoritative source.

```bash
# 1. Re-create the VM in Proxmox with the original hostname + LAN IP from inventory.yml.
#    Boot a minimal Debian (Trixie). Create the initial admin user xmarek + sshd.

# 2. Bootstrap from the control node (SSH password + sudo password the first time)
ansible-playbook server_basic_config.yml --limit kuzelovlab -k -K --ask-vault-pass
ansible-playbook server_user.yml         --limit kuzelovlab -k -K --ask-vault-pass
ansible-playbook server_harden.yml       --limit kuzelovlab     -K --ask-vault-pass

# 3. Storage provisioning (destructive — only run on a fresh VM)
ansible-playbook provisioning/kuzelovlab-vm-storage.yml     --limit kuzelovlab -K --ask-vault-pass
ansible-playbook provisioning/kuzelovlab-fast-storage.yml   --limit kuzelovlab -K --ask-vault-pass

# 4. Docker + Tailscale (TS_AUTHKEY from vault makes this silent)
ansible-playbook server_docker.yml    --limit kuzelovlab -K --ask-vault-pass
ansible-playbook server_tailscale.yml --limit kuzelovlab -K --ask-vault-pass

# 5. Restore /srv, /srv_fast, /mnt/samba_share from the remote Borg repo over Tailscale.
#    Stop docker first so containers do not fight the restore.
ssh ansible-manager@kuzelovlab
sudo systemctl stop docker
sudo apt install -y borgbackup
export BORG_REPO=ssh://borg@100.99.5.102/mnt/backup_tank/kuzelovlab
export BORG_PASSPHRASE='<secrets.borgbackup_password>'
borg list                                     # pick the newest archive name
sudo -E borg list ::<archive>                 # optional sanity peek
cd /
sudo -E borg extract --verbose --list ::<archive>     # restores srv, srv_fast, mnt/samba_share
sudo systemctl start docker
exit

# 6. Re-deploy all services (data already on disk; compose just starts containers)
ansible-playbook server_caddy.yml             --limit kuzelovlab -K --ask-vault-pass
ansible-playbook server_cloudflared.yml                          -K --ask-vault-pass
ansible-playbook apps.yml                     --limit kuzelovlab -K --ask-vault-pass
ansible-playbook server_borgmatic.yml         --limit kuzelovlab -K --ask-vault-pass
```

If a database container starts on an empty volume (it should not, because `/srv*` was restored), import the pre-backup dumps from `/srv/database_dumps/` — see **Database recovery from dumps** below.

## Scenario B — `homelab-thinkcentre` lost (bare-metal Lenovo)

This is the harder case: the box is also the backup target for `kuzelovlab`. As long as the **external backup-tank disk survives**, both layers are recoverable.

```bash
# 1. Reinstall Debian on the internal SSD with the original partitioning:
#    EFI (512MB) + /boot (1GB, ext4) + / (50GB, ext4). Leave the rest unallocated.
#    Reattach the external backup-tank drive (do NOT format it).

# 2. Bootstrap
ansible-playbook server_basic_config.yml --limit homelab-thinkcentre -k -K --ask-vault-pass
ansible-playbook server_user.yml         --limit homelab-thinkcentre -k -K --ask-vault-pass
ansible-playbook server_harden.yml       --limit homelab-thinkcentre     -K --ask-vault-pass

# 3. Recreate the internal LVM layout (var/home/srv) — destructive
ansible-playbook provisioning/homelab-lvm.yml --limit homelab-thinkcentre -K --ask-vault-pass

# 4. Bring the external backup-tank disk back online (LVM activation; do NOT mkfs)
ssh ansible-manager@homelab-thinkcentre
sudo apt install -y lvm2 borgbackup
sudo vgscan && sudo vgchange -ay
sudo mkdir -p /mnt/backup_tank
# replace <vg>/<lv> with what `lvs` shows for the backup tank
sudo mount /dev/<vg>/<lv> /mnt/backup_tank
# add it to /etc/fstab so it persists across reboots
exit

# 5. Restore /srv + /etc from the LOCAL Borg repo
ssh ansible-manager@homelab-thinkcentre
export BORG_PASSPHRASE='<secrets.borgbackup_password>'
sudo -E borg list /mnt/backup_tank/local_borg_repo
cd /
sudo -E borg extract --verbose --list /mnt/backup_tank/local_borg_repo::<archive>   # restores srv, etc
exit

# 6. Networking (this host runs Tailscale BARE-METAL, not via the docker role)
ansible-playbook server_tailscale_baremetal.yml --limit homelab-thinkcentre -K

# 7. Docker + services
ansible-playbook server_docker.yml              --limit homelab-thinkcentre -K --ask-vault-pass
ansible-playbook apps.yml                       --limit homelab-thinkcentre -K --ask-vault-pass
ansible-playbook server_caddy.yml               --limit homelab-thinkcentre -K --ask-vault-pass
ansible-playbook server_nextcloud.yml                                       -K --ask-vault-pass

# 8. Re-arm both backup layers + the remote-receiver side
ansible-playbook server_prepare_remote_borg_backup.yml -K --ask-vault-pass   # restores borg user + authorized_keys + target dir
ansible-playbook server_borgmatic.yml  --limit homelab-thinkcentre -K --ask-vault-pass
ansible-playbook server_borgbackup.yml                              -K --ask-vault-pass

sudo reboot
```

If the external disk is gone too, restart `kuzelovlab`'s backup chain from scratch (`borg init` will run automatically via the `borgbackup` role; the first `kuzelovlab` push will reinitialize the remote repo). All historical archives are lost.

## Database recovery from dumps

Borgmatic captures fresh dumps to `/srv/database_dumps/<service>.sql` immediately before every backup. After service containers come up but **before** users hit them, import as needed:

```bash
# on kuzelovlab
docker exec -i authentik-postgresql-1 psql -U authentik authentik   < /srv/database_dumps/authentik.sql
docker exec -i mealie-postgres        sh -c 'psql -U $POSTGRES_USER $POSTGRES_DB' < /srv/database_dumps/mealie.sql
docker exec -i seafile-mysql          sh -c 'mariadb -u root -p"$MYSQL_ROOT_PASSWORD"'         < /srv/database_dumps/seafile.sql

# on homelab-thinkcentre
docker exec -i journiv-postgres-db    psql -U journiv                                          < /srv/database_dumps/journiv.sql
```

In normal restores the database container's data dir (`/srv/<service>/postgres` or `/srv_fast/<service>/postgres`) is already restored from Borg, so import is only a fallback.

## What the backups do NOT cover

- **Docker images** — re-pulled by the playbooks.
- **Tailscale node identity** — wiped state means a re-registration; `secrets.tsauth_key` makes that silent (no browser).
- **Cloudflare Tunnel state** — tunnel token lives in the vault; the tunnel registration persists on Cloudflare's side.
- **SSH host keys** of restored hosts — other clients' `known_hosts` will mismatch on first reconnect; clear and re-accept.
- **Borg encryption key file** if you ever switch a repo from `repokey` to `keyfile` mode — currently all repos use `repokey`, so the key is embedded in the repo and protected by the passphrase only. Keep `secrets.borgbackup_password` safe.

## Validation checklist

- [ ] `/srv`, `/srv_fast`, `/mnt/samba_share` trees present at expected sizes (compare against `borg info ::<archive>`).
- [ ] `docker ps` on each host matches the pre-disaster service set.
- [ ] Caddy serves the LAN sites; Cloudflare Tunnel still routes public sites to `caddy:443`.
- [ ] AdGuard rewrites resolve the LAN URLs to the host IP (split-horizon DNS).
- [ ] Tailscale: node visible in admin, advertised routes (kuzelovlab) re-accepted.
- [ ] First scheduled Borgmatic run pings Healthchecks.io successfully (next cron tick after restore).