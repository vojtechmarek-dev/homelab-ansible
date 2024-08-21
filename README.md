
# Ansible Playbooks for Home Server Setup

This repository contains personal Ansible playbooks designed primarily for setting up a home server. These playbooks document each step to easily replicate the setup on new servers, such as a Raspberry Pi 4, which is currently being used as a "training" server.

## Introduction to Ansible

Ansible automates the management of remote systems, ensuring they maintain a desired state. While this README provides a basic overview, it is recommended to consult the [official Ansible documentation](https://docs.ansible.com/) for a more comprehensive understanding. 

### Key Components of Ansible

- **Control Node**: The computer with Ansible installed, used to manage all remote hosts.
- **Managed Node**: A remote computer (host) managed by Ansible.

Although there are various tools that automate host management via Ansible, in its purest form, Ansible is a set of command-line tools that require manual execution.

## Purpose of This Repository

The main goal of this repository is to learn Ansible and server management from the ground up, without relying on external tools or "crutches." Each step in the server setup process is meticulously documented in Ansible playbooks, ensuring the setup can be easily replicated on new servers.

## Playbooks Overview

This repository includes several playbooks, each designed for a specific aspect of server configuration:

- `server_basic_config.yml`: Basic configuration and common packages for all servers.
- `server_user.yml`: User access playbook that creates a user and adds them to the sudoers group.
- `server_docker.yml`: Docker setup, with tasks including sensitive data management (e.g., passwords, authorized keys) stored externally (not in this repository). This playbook should be run using Ansible Vault:

    ```bash
    ansible-playbook -i inventory.yml -i ../ansible-secrets/secrets.yml server_user.yml --ask-vault-pass
    ```
  
### Playbook Execution Order

1. **Run `server_basic_config.yml` as root** with SSH access.
2. **Run `server_user.yml` as root**.
3. Verify that the user created in the previous playbook has sudo privileges.

:warning: **Important**: After the initial setup, all subsequent playbook runs should be executed under the new user (root access will not be allowed). Use `--ask-vault-pass` for playbooks using Ansible Vault, and `--ask-become-pass` for those requiring sudo privileges.

## Roles Included

The playbooks currently include the following roles:

- **base**: Basic configuration of a Linux Debian server.

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
