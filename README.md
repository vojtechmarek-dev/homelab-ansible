
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