# 🚀 Ansible Initial Server Setup Playbook

Automates the first-time configuration of new servers — creates a dedicated Ansible user, installs your SSH public key, and grants passwordless sudo — so subsequent playbooks can connect without manual intervention.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Authentication Methods](#authentication-methods)
- [Inventory Configuration](#inventory-configuration)
- [Usage](#usage)
- [After Running This Playbook](#after-running-this-playbook)
- [Security Notes](#security-notes)
- [Troubleshooting](#troubleshooting)

---

## Overview

### What this playbook does

1. **Verifies connectivity** — pings each host with the provided credentials.
2. **Creates an admin user** — adds the configured user to the `sudo` group.
3. **Installs an SSH public key** — copies your local `~/.ssh/id_rsa.pub` to the new user's `authorized_keys`.
4. **Configures passwordless sudo** — writes a validated sudoers drop-in file so Ansible never needs a `become` password on subsequent runs.

After this playbook succeeds, all future playbooks connect as the admin user using key-based authentication with `NOPASSWD` sudo.

---

## Prerequisites

| Requirement | Details |
|---|---|
| **Ansible** | 2.9 or later |
| **`ansible.posix` collection** | For the `authorized_key` module |
| **SSH public key** | `~/.ssh/id_rsa.pub` on the control node (will be pushed to servers) |
| **`sshpass`** | Required only for Method 2 (password auth) |

Install the POSIX collection:

```bash
ansible-galaxy collection install ansible.posix
```

Install `sshpass` (Method 2 only):

```bash
# Ubuntu/Debian
sudo apt install sshpass

# RHEL/CentOS
sudo yum install sshpass
```

---

## Project Structure

```
.
├── inventory.ini        # Host definitions — edit this to match your setup
├── site.yml             # The playbook
└── passwords.yml        # Ansible Vault file — required for Method 2 (password auth)
```

---

## Authentication Methods

Two methods are supported. You can use either or mix both in the same inventory.

---

### Method 1 — SSH Private Key

Use this when each server already has an SSH key configured (e.g., Vagrant boxes, cloud instances with injected keys).

Each host entry points to its own private key file. No vault file is needed.

**Inventory:**

```ini
[servers]
node1 ansible_host=<IP_ADDRESS> ansible_user=<INITIAL_USER> ansible_ssh_private_key_file=~/.ssh/<NODE1_KEY>
node2 ansible_host=<IP_ADDRESS> ansible_user=<INITIAL_USER> ansible_ssh_private_key_file=~/.ssh/<NODE2_KEY>


[servers:vars]
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
new_ansible_user=<ADMIN_USERNAME>
```

**Run:**

```bash
ansible-playbook -i inventory.ini setup_servers.yml
```

---

### Method 2 — Username & Password via Ansible Vault

Use this when servers have no SSH key configured and you connect with a username and password. Instead of passing credentials on the command line, they are stored securely in an encrypted `passwords.yml` vault file and referenced directly in the inventory.

#### Step 1 — Create the vault file

```bash
ansible-vault create passwords.yml
```

Add your credentials inside:

```yaml
vault_node1_pass: "your_password"
vault_node2_pass: "your_password"
```

> If the SSH password and sudo password are the same, set both variables to the same value.

#### Step 2 — Reference the vault variables in your inventory

```ini
[servers]
node1 ansible_host=<IP_ADDRESS> ansible_user=<USERNAME> ansible_ssh_pass="{{ vault_node1_pass }}" ansible_become_pass="{{ vault_node1_pass }}"
node2 ansible_host=<IP_ADDRESS> ansible_user=<USERNAME> ansible_ssh_pass="{{ vault_node2_pass }}" ansible_become_pass="{{ vault_node2_pass }}"

[servers:vars]
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
new_ansible_user=<ADMIN_USERNAME>
```

Ansible reads the SSH and become passwords directly from the vault — nothing is passed on the command line.

#### Step 3 — Run with the vault password

```bash
ansible-playbook -i inventory.ini setup_servers.yml --ask-vault-pass
```

You will be prompted once for the vault password that decrypts `passwords.yml`.

---


## Inventory Configuration

Full variable reference:

| Variable | Method   | Description |
|---|----   |---|
| `ansible_host` | Both | IP address of the target server |
| `ansible_user` | Both | User Ansible connects as for the initial bootstrap |
| `ansible_ssh_private_key_file` | Method 1 | Path to the SSH private key for this host |
| `ansible_ssh_pass` | Method 2 | SSH login password, sourced from vault |
| `ansible_become_pass` | Method 2 | Sudo password, sourced from vault |
| `new_ansible_user` | Both | Name of the admin user to create (set in `[group:vars]`) |
| `ansible_ssh_common_args` | Both | Extra SSH flags — `StrictHostKeyChecking=no` skips host-key prompts for new machines |

---

## Usage

### Run against all servers

```bash
# Method 1 — SSH key
ansible-playbook -i inventory.ini setup_servers.yml

# Method 2 — Vault passwords
ansible-playbook -i inventory.ini setup_servers.yml --ask-vault-pass
```

### Target a single server

```bash
ansible-playbook -i inventory.ini setup_servers.yml --limit node1
```

### Dry run (no changes applied)

```bash
ansible-playbook -i inventory.ini setup_servers.yml --check
```

### Verbose output (for debugging)

```bash
ansible-playbook -i inventory.ini setup_servers.yml -v
```

### Edit the vault file later

```bash
ansible-vault edit passwords.yml
```

---

## After Running This Playbook

Once the playbook completes, update your future inventories to use the newly created admin user with your SSH key — no password ever needed again:

```ini
[all:vars]
ansible_user=<ADMIN_USERNAME>
ansible_ssh_private_key_file=~/.ssh/id_rsa
```

No `ansible_become_pass` is required because passwordless sudo is already configured.

---


## Troubleshooting

### `UNREACHABLE` on the ping task

- Confirm the host is reachable: `ping <IP_ADDRESS>`
- Test SSH manually:
  ```bash
  # Key auth
  ssh -i ~/.ssh/<YOUR_KEY> <USER>@<IP_ADDRESS>

  # Password auth
  ssh <USER>@<IP_ADDRESS>
  ```
- Make sure `sshpass` is installed if using Method 2

### `Permission denied` when creating the user

- The bootstrap user must have sudo privileges on the target server
- Confirm `ansible_become_pass` in the vault matches the actual sudo password

### `Vault secret not found` or decryption error

- Make sure you are using the same vault password used when the file was created
- To use a password file instead of a prompt: `--vault-password-file ~/.vault_pass`

### `authorized_key` module not found

```bash
ansible-galaxy collection install ansible.posix
```

### Host key verification failed

- The `StrictHostKeyChecking=no` flag in the inventory should prevent this
- If it persists, clear the old entry: `ssh-keygen -R <IP_ADDRESS>`