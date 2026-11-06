# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) and AI coding assistants when working with code in this repository.

## What this repo is

Role-based Ansible playbooks that provision and maintain a fleet of Raspberry Pis and a rented Strato VPS. `inventory/hosts.yml` defines logical hostnames and functional groups:
- **Raspberry Pis (`pis`)** reachable over Twingate:
  - `raspberrypi` (`192.168.178.10` — Home server running Docker Compose services and an external USB drive)
  - `parents_pi` (`192.168.178.46` — Off-site Pi at parents' house with two external drives and Docker Compose)
  - *(Commented out / excluded)*: `homeassistant_pi` (`192.168.0.20` — Home Assistant OS, not a Debian/Ubuntu target)
- **Public VPS (`vps`)**:
  - `strato_vps` (`217.154.166.48` — Ubuntu 26.04 LTS, future Prometheus/Grafana monitoring server)
- **Functional Groups**:
  - `drive_hosts`: `raspberrypi`, `parents_pi`
  - `docker_hosts`: `raspberrypi`, `parents_pi`, `strato_vps`
  - `monitoring_targets`: `raspberrypi`, `parents_pi`

There is no application code here — every change is a change to server configuration/state on those devices.

## Commands

Run all commands from the repo root (`ansible.cfg` sets `inventory = ./inventory/hosts.yml`, `remote_user = nico`, and `roles_path = ./roles`).

```bash
# Daily/regular maintenance: apt update/dist-upgrade/autoremove + docker compose pull/up + reboot check
ansible-playbook playbooks/update.yml

# Update only a single host
ansible-playbook playbooks/update.yml --limit strato_vps

# Base OS setup on all hosts (or a single host with --limit): base packages, user 'nico', NOPASSWD sudo, SSH keys & hardening
ansible-playbook playbooks/base-setup.yml

# First-time bootstrap on an existing Pi that still only has the 'pi' user
ansible-playbook playbooks/base-setup.yml --limit raspberrypi -e ansible_user=pi

# Mount external drives (by UUID) via /etc/fstab based on 'host_drives' in host_vars
ansible-playbook playbooks/drive.yml

# Install Docker CE + Compose plugin (auto-detects Debian/Ubuntu and arm64/amd64) + weekly prune cron
ansible-playbook playbooks/docker.yml

# Deploy/update docker-compose services + write secrets from Ansible Vault to .env
ansible-playbook playbooks/docker-compose.yml --ask-vault-pass

# Run complete setup across the fleet (base + drive + docker + docker_compose)
ansible-playbook playbooks/site.yml --ask-vault-pass

# Install node_exporter and smartctl_exporter (currently legacy/to-be-refactored)
ansible-playbook playbooks/monitoring.yml

# Dry run / check mode before applying changes
ansible-playbook playbooks/<file>.yml --check --diff

# Syntax check a playbook
ansible-playbook playbooks/<file>.yml --syntax-check
```

There is no CI configured. Always run `ansible-playbook --syntax-check` (and `--check --diff` against live hosts when applicable) to validate changes.

## Secrets (Ansible Vault)

Because `ansible.cfg` points to `./inventory/hosts.yml`, Ansible automatically loads plaintext variables from `inventory/group_vars/` and `inventory/host_vars/<hostname>/`.
Encrypted `ansible-vault` files live in `vaults/<hostname>.yml` (outside `host_vars/`) so that daily commands like `playbooks/update.yml`, `playbooks/base-setup.yml`, `playbooks/drive.yml`, and `playbooks/docker.yml` run **without** requiring `--ask-vault-pass`. `roles/docker_compose` dynamically loads `vaults/{{ inventory_hostname }}.yml` via `include_vars` when writing `.env` secrets:
- `vaults/raspberrypi.yml`: `ansible-vault`-encrypted file holding Twingate and Samba credentials (`vault_twingate_network`, `vault_twingate_access`, `vault_twingate_refresh`, `vault_samba_password`) referenced by `inventory/host_vars/raspberrypi/vars.yml`.
- `vaults/parents_pi.yml`: `ansible-vault`-encrypted file holding host-specific Twingate Connector and Samba credentials (`vault_twingate_network`, `vault_twingate_access`, `vault_twingate_refresh`, `vault_samba_password`) referenced by `inventory/host_vars/parents_pi/vars.yml`.
- `vaults/strato_vps.yml`: `ansible-vault`-encrypted file holding `vault_vps_sudo_password` (only needed if bootstrapping before passwordless sudo is active via `-e @vaults/strato_vps.yml --ask-vault-pass`) and future Twingate Connector credentials for `strato_vps`.

Edit vault files with:

```bash
ansible-vault edit vaults/raspberrypi.yml
ansible-vault edit vaults/parents_pi.yml
ansible-vault edit vaults/strato_vps.yml
```

Never print or write out decrypted contents in plaintext, and never commit a decrypted version.

## Architecture notes

- **Unified Admin User (`nico`)**: Managed via `admin_user` in `inventory/group_vars/all.yml`. `roles/base` creates `nico` with `/etc/sudoers.d/nico` (`NOPASSWD: ALL`) and configures SSH hardening (`PermitRootLogin no`, `PasswordAuthentication no`).
- **SSH Lockout Protection**: `roles/base` installs public keys from `admin_ssh_public_keys` in `inventory/group_vars/all.yml`. If that list is empty during initial bootstrapping, it falls back to copying `authorized_keys` from `/root/.ssh/` or `/home/pi/.ssh/` on the remote host and asserts that `/home/nico/.ssh/authorized_keys` is non-empty before restarting `sshd`.
- **Ubuntu 26.04 `sudo-rs` Compatibility**: `strato_vps` runs Ubuntu 26.04 LTS, which defaults to `sudo-rs` (`/usr/bin/sudo`). Because `sudo-rs` does not support custom `-p` prompts used by Ansible's `sudo` become plugin, `inventory/host_vars/strato_vps/vars.yml` sets `ansible_become_exe: /usr/bin/sudo.ws`.
- **Multi-Disk & Multi-Compose Support**:
  - `roles/drive` loops over `host_drives` defined in `inventory/host_vars/<host>/vars.yml`.
  - `roles/docker_compose` clones `compose_repo_url` to `compose_repo_dest` and loops over `compose_files` (passing `--env-file {{ compose_repo_dest }}/.env` so topic-based compose files in subdirectories like `connectivity/adguard.yml`, `documents/docker-compose.yml`, and `media/docker-compose.yml` read the central `.env` file).
  - `playbooks/update.yml` updates APT packages and pulls/restarts `compose_files` without rewriting `.env`, so daily updates run without requiring `--ask-vault-pass` (unless a host still has encrypted variables loaded that require vault decryption; note that if `vault.yml` is in `host_vars/<host>/`, Ansible attempts to decrypt all files in `host_vars/<host>/` when targeting that host unless `--ask-vault-pass` is passed).
- Task names and comments are written in German; keep new tasks consistent with that convention unless told otherwise.
