
# Ansible Playbooks for Home Server Setup

This repository contains personal Ansible playbooks designed for setting up and managing a home server. These playbooks document each step to easily replicate the setup on new servers, such as the Raspberry Pi 4 currently in use.

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
As stated before some playbooks require secrets variables and shold be run with Ansible Vault secrets:

```bash
ansible-playbook -i inventory.yml -i ../ansible-secrets/secrets.yml server_user.yml --ask-vault-pass
```

### Playbook Execution Order

## Bootstraping the server

If your ssh user is not sudoer (or you cannot ssh as root), and you have no sudo user to connect as - ssh into server and add your user to sudoers manually.
Before adding ssh key you need to connect to ssh with password add `-k` to your playbook call

```bash 
usermod -aG sudo username
```

1. **Run `server_basic_config.yml` as root** with SSH access.
2. **Run `server_user.yml` as root**.
3. Verify that the user created in the previous playbook has sudo privileges.

:warning: **Important**: After the initial setup, all subsequent playbook runs should be executed under the new user (root access will not be allowed). Use `--ask-vault-pass` for playbooks using Ansible Vault, and `--ask-become-pass` for those requiring sudo privileges.

## Roles Included

The playbooks currently include the following roles:

- **base**: Basic configuration of a Linux Debian server.
- **user**: Adding `xmarek` superuser to the server with ssh access to the server.
- **docker**: Configuration of Docker service including installing needed packages.

    
# Adding a New Service to the Homelab

This document is the Standard Operating Procedure (SOP) for deploying a new containerized service to the `homelab-thinkcentre` server. Following these steps ensures that the service is properly configured, networked, secured, and accessible.

## Prerequisites

-   A working Ansible control machine with access to the server.
-   The service you want to deploy must be available as a Docker image.
-   Access to the Cloudflare, AdGuard Home, and Tailscale admin panels.

---

## Step 1: Planning & Variable Definition

Before writing any code, define the key attributes of the new service.

-   **Service Name:** A short, lowercase name (e.g., `jellyfin`).
-   **Docker Image:** The full image name from Docker Hub (e.g., `jellyfin/jellyfin:latest`).
-   **Hostname:** The desired internal URL (e.g., `jellyfin.vojtechmarek.dev`).
-   **Internal Port:** The port the application listens on *inside* the container (e.g., `8096`).
-   **Persistent Data:** Identify which directories inside the container need to be saved. These will be mapped to `/srv/<service-name>/...` on the host.

## Step 2: Public DNS Configuration (Cloudflare)

This step makes the domain "real" so Caddy can get an SSL certificate for it.

-   [ ] Log in to the **Cloudflare Dashboard**.
-   [ ] Go to the DNS settings for `vojtechmarek.dev`.
-   [ ] Create a new **CNAME** record:
    -   **Type:** `CNAME`
    -   **Name:** The new service name (e.g., `jellyfin`).
    -   **Content / Target:** `placeholder.vojtechmarek.dev` (your dummy hostname).
    -   **Proxy status:** **DNS only (Grey Cloud)**. This is mandatory.

## Step 3: Local & Remote DNS Configuration

This step points your private networks to the server, keeping the service off the public internet.

-   [ ] **AdGuard Home (for LAN access):**
    -   Log in to your AdGuard dashboard (`https://adguardhome.vojtechmarek.dev`).
    -   Go to **Filters -> DNS Rewrites**.
    -   Add a new rewrite:
        -   **Domain:** `jellyfin.vojtechmarek.dev`
        -   **Answer:** `192.168.0.146` (your server's local IP).

-   [ ] **Tailscale (for remote access):**
    -   Log in to your Tailscale admin console.
    -   Go to the **DNS** page.
    -   Add a new MagicDNS entry:
        -   **Name:** `jellyfin.vojtechmarek.dev`
        -   **IP Address:** `homelab-thinkcentre` (your server's Tailscale name).

## Step 4: Ansible Deployment Playbook

Create the new Ansible playbook and associated template files for the service.

1.  **Create the Docker Compose Template:**
    -   Create a new file: `templates/<service-name>.docker-compose.yml.j2`.
    -   Define the service, making sure to:
        -   Use a `container_name`.
        -   Set `restart: unless-stopped`.
        -   Map all necessary persistent data volumes to `/srv/<service-name>/...` on the host.
        -   **Do not** expose ports to the host unless absolutely necessary.
        -   Connect the container to the shared `proxy` network.

    ```yaml
    # Example for templates/jellyfin.docker-compose.yml.j2
    services:
      jellyfin:
        image: jellyfin/jellyfin:latest
        container_name: jellyfin
        restart: unless-stopped
        volumes:
          - "{{ app_config_path }}:/config"
          - "{{ app_media_path }}:/media"
        networks:
          - proxy

    networks:
      proxy:
        external: true
    ```

2.  **Create the Main Playbook:**
    -   Create a new file: `deploy_<service-name>.yml`.
    -   Use the standard pattern:
        -   Define variables for paths (`project_path`, `app_config_path`, etc.).
        -   Create the necessary directories under `/srv/<service-name>/` with an `ansible.builtin.file` task.
        -   Copy the `docker-compose.yml.j2` template to the server.
        -   Run the `community.docker.docker_compose_v2` module.
        -   Include a handler for restarting the service.

## Step 5: Configure the Reverse Proxy (Caddy)

Tell Caddy how to route traffic to the new service.

-   [ ] Open the Caddyfile template: `templates/Caddyfile.public.j2`.
-   [ ] Add a new site block for the service:

    ```
    # In templates/Caddyfile.public.j2

    https://jellyfin.vojtechmarek.dev {
        tls {
            dns cloudflare {env.CLOUDFLARE_API_TOKEN}
        }
        # Proxy to the new container on its internal port
        reverse_proxy jellyfin:8096
    }
    ```

## Step 6: Deploy and Verify

Run the Ansible playbooks to apply all your changes.

1.  **Run the new service playbook:**
    ```bash
    ansible-playbook -i inventory.ini deploy_jellyfin.yml --user xmarek -K
    ```
2.  **Run the Caddy playbook** to update its configuration:
    ```bash
    ansible-playbook -i inventory.ini deploy_caddy.yml --user xmarek -K
    ```
3.  **Verify DNS on your client machine:**
    -   Flush your DNS cache (`ipconfig /flushdns` on Windows).
    -   Run `ping jellyfin.vojtechmarek.dev`. It should resolve to `192.168.0.146`.
4.  **Access the service:**
    -   Open your browser and navigate to `https://jellyfin.vojtechmarek.dev`.
    -   You should see the service's setup page or main interface with a valid SSL certificate.

---

## Client-Side Troubleshooting Checklist

If you get a `NS_ERROR_UNKNOWN_HOST` or similar DNS error after deployment:

-   [ ] Did you add the DNS rewrite in **AdGuard Home**?
-   [ ] Did you add the MagicDNS entry in **Tailscale**?
-   [ ] Is your device configured to use AdGuard Home as its DNS server (check router DHCP settings)?
-   [ ] Have you disabled **"Private DNS" / "Secure DNS"** in your phone's or browser's settings?
-   [ ] Have you tried flushing your computer's DNS cache?

# Home Server Disaster Recovery Plan

This part outlines the procedure to perform a full restore of the `managed` server from scratch in the event of a total system failure (e.g., OS disk corruption, hardware failure).

The recovery process relies on two key components:

1.  **Ansible Playbooks:** Our "Infrastructure as Code" which rebuilds the server's configuration, services, and applications.
2.  **Borg Backups:** Our data backups, stored on the external drive, which contain all user data, service configurations, and system settings.

## Prerequisites

Before you begin, you must have:

1.  **This Ansible Project:** The complete directory with all playbooks, templates, and the encrypted `secrets.yml` vault file.
2.  **Ansible Vault Passphrase:** The password to decrypt `secrets.yml`.
3.  **The External Backup Drive:** The physical hard drive containing the Borg backup repository.
4.  **Borg Repository Passphrase:** The password you created to encrypt the Borg backups.

## Recovery Procedure

The process is divided into two main phases: **System Rebuild** and **Data Restore**.

### Phase 1: System Rebuild with Ansible

This phase rebuilds the server to the exact state it was in before the failure, but with empty data directories.

1.  **Prepare New Hardware:**
    *   If the hardware failed, replace it.
    *   Ensure the internal SSDs and the external backup drive are connected.

2.  **Minimal Debian Installation:**
    *   Install a fresh, minimal Debian system on the new internal SSD.
    *   Follow the **"Minimal Manual Install"** procedure used during the initial setup:
        *   Boot the installer in **UEFI mode**.
        *   Choose **Manual Partitioning**.
        *   Create only three partitions:
            1.  `512 MB` - EFI System Partition
            2.  `1 GB` - `/boot` (ext4)
            3.  `50 GB` - `/` (ext4)
        *   Leave the rest of the internal SSD as **unallocated free space**.
    *   During installation, create the initial administrative user (e.g., `xmarek`) and install the SSH server.

3.  **Prepare the Ansible Control Machine:**
    *   Ensure your Ansible control machine (e.g., your laptop) has Ansible and SSH access to the newly installed server.
    *   Update your `inventory.ini` file with the new server's IP address if it has changed.

4.  **Run All Ansible Playbooks:**
    *   Execute your playbooks in a logical order to rebuild the entire system configuration. The order is important.

    ```bash
    # Make sure your user is in the sudo group on the new server first!
    # You may need to do this manually by logging in as root at the console:
    # usermod -aG sudo xmarek

    # 1. Configure the base system (users, security, etc.)
    ansible-playbook -i inventory.ini server_basic_config.yml --ask-become-pass

    # 2. Configure LVM on the remaining internal disk space
    ansible-playbook -i inventory.ini configure_lvm.yml --ask-become-pass

    # 3. Install Docker
    ansible-playbook -i inventory.ini install_docker.yml --ask-become-pass

    # 4. Deploy all your containerized services
    ansible-playbook -i inventory.ini deploy_tailscale.yml --ask-become-pass
    ansible-playbook -i inventory.ini deploy_adguard_home.yml --ask-become-pass
    ansible-playbook -i inventory.ini deploy_caddy.yml --ask-become-pass
    ansible-playbook -i inventory.ini deploy_stirling_pdf.yml --ask-become-pass
    ansible-playbook -i inventory.ini deploy_nextcloud.yml --ask-become-pass
    ansible-playbook -i inventory.ini deploy_samba.yml --ask-become-pass

    # 5. Install the backup software (but don't run a backup yet)
    ansible-playbook -i inventory.ini deploy_borg_backup.yml --ask-become-pass
    ```

At the end of this phase, your server is fully configured and all services are running, but all their data directories (like `/srv/nextcloud/app`) are empty.

### Phase 2: Data Restore with BorgBackup

This phase populates the newly built server with your backed-up data.

1.  **Prepare the External Drive:**
    *   SSH into the newly rebuilt server.
    *   Follow the manual steps to prepare the external drive for LVM and mount the backup volume. **DO NOT FORMAT OR WIPE IT.**

    ```bash
    # Detect the LVM volumes on the external drive
    sudo vgscan
    sudo vgchange -ay external_vg

    # Create the mount point and mount the backup volume
    sudo mkdir -p /mnt/backups
    sudo mount /dev/external_vg/backup_lv /mnt/backups
    ```

2.  **Stop All Services:**
    *   Before restoring data, it's crucial to stop the applications that use it to prevent conflicts.

    ```bash
    # Stop all your application containers
    sudo docker stop nextcloud-app nextcloud-db nextcloud-redis stirling-pdf caddy samba
    ```

3.  **Perform the Borg Restore:**
    *   We will use `borg extract` to restore the files. This command is run as `root` to preserve all file permissions.
    *   First, find the name of the backup you want to restore from (usually the latest one).

    ```bash
    # List all available backups
    sudo BORG_PASSPHRASE='YOUR_BORG_PASSPHRASE' borg list /mnt/backups/borg-repo
    ```

    *   Now, run the extract command. This restores the contents of the backup *into* the specified directories, overwriting the empty ones.

    ```bash
    # Set the name of the backup you want to restore from
    BACKUP_NAME="backup-2025-06-30_02-30-00" # <-- Change this to your chosen backup

    # Restore the data
    sudo BORG_PASSPHRASE='YOUR_BORG_PASSPHRASE' borg extract \
        --verbose \
        --list \
        /mnt/backups/borg-repo::${BACKUP_NAME} \
        etc srv home
    ```

    **Important:** This command must be run from the `/` directory. It will restore the `etc`, `srv`, and `home` folders from the backup directly into their correct locations on your root filesystem.

4.  **Reboot the Server:**
    *   A full reboot is the cleanest way to ensure all services start up correctly with their newly restored data and configuration.

    ```bash
    sudo reboot
    ```

## Verification

After the reboot, all your services should be running exactly as they were at the time of the last backup.
*   Log in to Nextcloud and verify your files and photos are there.
*   Check that Caddy is serving your domains correctly.
*   Verify your Samba share is accessible.

Your server is now fully recovered.