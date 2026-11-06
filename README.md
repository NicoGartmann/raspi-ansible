# raspi-ansible

Role-based Ansible playbooks for provisioning, configuring, and updating my Raspberry Pi fleet and Strato VPS.

## Overview & Inventory

All managed hosts and functional groups are defined in [`inventory/hosts.yml`](inventory/hosts.yml):

| Host | IP Address | OS | Groups | Description |
| :--- | :--- | :--- | :--- | :--- |
| `raspberrypi` | `192.168.178.10` | Raspberry Pi OS (Debian) | `pis`, `drive_hosts`, `docker_hosts`, `monitoring_targets` | Home server running Docker Compose services and an external USB drive |
| `parents_pi` | `192.168.178.46` | Raspberry Pi OS (Debian) | `pis`, `drive_hosts`, `docker_hosts`, `monitoring_targets` | Off-site Raspberry Pi at parents' house with two external USB drives |
| `strato_vps` | `217.154.166.48` | Ubuntu 26.04 LTS | `vps`, `docker_hosts` | Public Strato VPS (Docker host & future monitoring server) |
| *(homeassistant_pi)* | `192.168.0.20` | Home Assistant OS | *(commented out)* | Excluded from Debian/Ubuntu playbooks |

*(Note: All Raspberry Pis are reachable via Twingate. All hosts share the unified admin user `nico` with passwordless `sudo` and SSH key authentication.)*

---

## Repository Structure

```text
.
├── ansible.cfg                              # Default Ansible config (inventory, remote_user=nico, roles_path)
├── inventory/
│   ├── hosts.yml                            # Logical hostnames and functional groups
│   ├── group_vars/
│   │   └── all.yml                          # Global settings (admin_user, admin_ssh_public_keys, base_packages)
│   └── host_vars/
│       ├── raspberrypi/
│       │   └── vars.yml                     # Drive UUID, compose repo/files, directories, .env mappings
│       ├── parents_pi/
│       │   └── vars.yml                     # External drive UUIDs, compose repo/files, directories, .env mappings
│       └── strato_vps/
│           └── vars.yml                     # VPS-specific settings (sudo.ws compatibility & compose config)
├── vaults/
│   ├── raspberrypi.yml                      # Encrypted secrets for raspberrypi (Twingate, Samba)
│   ├── parents_pi.yml                       # Encrypted secrets for parents_pi (Twingate, Samba)
│   └── strato_vps.yml                       # Encrypted secrets for strato_vps (initial sudo password, Twingate)
├── roles/
│   ├── base/tasks/main.yml                  # Base packages, user 'nico', NOPASSWD sudo, SSH keys & hardening
│   ├── drive/tasks/main.yml                 # Multi-drive /etc/fstab mounting via 'host_drives' list
│   ├── docker/tasks/main.yml                # Docker CE & Compose plugin (Debian/Ubuntu, arm64/amd64) + prune cron
│   └── docker_compose/tasks/main.yml        # Clones docker-config repo, creates dirs/.env, runs topic compose files
└── playbooks/
    ├── site.yml                             # Full provisioning run across all groups (base, drive, docker, compose)
    ├── base-setup.yml                       # Applies 'base' role to all (or --limit selected) hosts
    ├── update.yml                           # Routine maintenance: apt dist-upgrade + compose pull/up + reboot check
    ├── drive.yml                            # Applies 'drive' role to 'drive_hosts'
    ├── docker.yml                           # Applies 'docker' role to 'docker_hosts'
    ├── docker-compose.yml                   # Applies 'docker_compose' role to 'docker_hosts'
    └── monitoring.yml                       # Prometheus exporters (node_exporter & smartctl_exporter)
```

---

## Common Workflows & Commands

Run all commands from the repository root. You can target all applicable hosts or restrict any playbook to a single device using `--limit <hostname>`.

### 1. Routine System & Container Updates (`playbooks/update.yml`)
Updates APT package cache, performs `dist-upgrade` and `autoremove`, pulls and restarts configured Docker Compose stacks, and reports if a reboot is required (`/var/run/reboot-required`) — **no vault password required**:
```bash
# Update all hosts
ansible-playbook playbooks/update.yml

# Update only a specific host (e.g. strato_vps or raspberrypi)
ansible-playbook playbooks/update.yml --limit strato_vps
```

### 2. Base Setup & Bootstrapping (`playbooks/base-setup.yml`)
Installs standard utilities (`sudo`, `htop`, `git`, `curl`, `ca-certificates`), creates user `nico` with `NOPASSWD` sudo, configures SSH public keys (with lockout protection), and hardens OpenSSH (`PermitRootLogin no`, `PasswordAuthentication no`):
```bash
# Run base setup on all hosts
ansible-playbook playbooks/base-setup.yml

# First-time bootstrap on a Raspberry Pi that still uses the default 'pi' user
ansible-playbook playbooks/base-setup.yml --limit raspberrypi -e ansible_user=pi

# First-time bootstrap on strato_vps (if nico still requires the sudo password from vault)
ansible-playbook playbooks/base-setup.yml --limit strato_vps -e @vaults/strato_vps.yml -e 'ansible_become_password={{ vault_vps_sudo_password }}' --ask-vault-pass
```
*(Tip: Add your SSH public keys from 1Password to `admin_ssh_public_keys` in [`inventory/group_vars/all.yml`](inventory/group_vars/all.yml).)*

### 3. External Drive Mounts (`playbooks/drive.yml`)
Mounts all drives configured under `host_drives` in `inventory/host_vars/<host>/vars.yml` via `/etc/fstab`:
```bash
ansible-playbook playbooks/drive.yml
```

### 4. Docker Engine Installation (`playbooks/docker.yml`)
Installs `docker-ce`, `docker-ce-cli`, `containerd.io`, and `docker-compose-plugin` (automatically matching Debian/Ubuntu and `arm64`/`amd64`), adds `nico` to the `docker` group, and configures a weekly `docker system prune -af` cron job:
```bash
ansible-playbook playbooks/docker.yml
```

### 5. Docker Compose Services (`playbooks/docker-compose.yml`)
Clones/updates [`NicoGartmann/docker-config`](https://github.com/NicoGartmann/docker-config) in `/opt/services`, creates required host directories, writes paths and secrets from `vaults/<host>.yml` to `/opt/services/.env`, and runs `docker compose --env-file /opt/services/.env -f <file> pull` + `up -d` for all configured topic-based `compose_files` (e.g. `connectivity/adguard.yml`, `documents/docker-compose.yml`, `media/docker-compose.yml`):
```bash
ansible-playbook playbooks/docker-compose.yml --ask-vault-pass
```

### 6. Full Provisioning (`playbooks/site.yml`)
Runs `base`, `drive`, `docker`, and `docker_compose` across the entire fleet (or a single host via `--limit`):
```bash
ansible-playbook playbooks/site.yml --ask-vault-pass
```

---

## Dry Run & Syntax Check

Before applying changes to physical Pis or the VPS, validate syntax or run in check mode:

```bash
# Check playbook syntax
ansible-playbook playbooks/site.yml --syntax-check

# Dry run with diff output
ansible-playbook playbooks/base-setup.yml --check --diff
```

---

## Secrets Management (Ansible Vault)

Sensitive variables are encrypted using `ansible-vault` inside the [`vaults/`](vaults/) directory (loaded on-demand by `roles/docker_compose` so that routine commands do not prompt for a vault password):

- **[`vaults/raspberrypi.yml`](vaults/raspberrypi.yml)**:
  - `vault_twingate_network`
  - `vault_twingate_access`
  - `vault_twingate_refresh`
  - `vault_samba_password`
- **`vaults/parents_pi.yml`**:
  - `vault_twingate_network`
  - `vault_twingate_access`
  - `vault_twingate_refresh`
  - `vault_samba_password`
- **[`vaults/strato_vps.yml`](vaults/strato_vps.yml)**:
  - `vault_vps_sudo_password`
  - *(künftig)* `vault_twingate_network`, `vault_twingate_access`, `vault_twingate_refresh`

To view, create, or edit encrypted vault files:
```bash
ansible-vault edit vaults/raspberrypi.yml
ansible-vault create vaults/parents_pi.yml
ansible-vault edit vaults/strato_vps.yml
```
