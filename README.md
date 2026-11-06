# raspi-ansible

Ansible playbooks for provisioning and managing my Raspberry Pi fleet and Strato VPS.

## Overview & Inventory

All managed hosts are defined in [`inventory/hosts`](inventory/hosts):

| Host / Group | IP Address | User | Description |
| :--- | :--- | :--- | :--- |
| `raspberrypi` | `192.168.178.10` | `pi` | Home server running Docker Compose services and an external USB drive |
| `homeassistant_pi` | `192.168.0.20` | `pi` | Home Assistant Raspberry Pi |
| `parents_pi` | `192.168.178.46` | `pi` | Off-site Raspberry Pi at parents' house with an external USB drive |
| `strato_vps` | `217.154.166.48` | `nico` | Rented Strato VPS (Ubuntu 26.04 LTS) |

### Host Groups
- **`drive_hosts`**: `raspberrypi`, `parents_pi` — Pis with an external drive attached (used for `smartctl_exporter`).
- **`monitoring_targets`**: `raspberrypi`, `homeassistant_pi`, `parents_pi` — All Pis scraped by the Prometheus/Grafana monitoring stack.

*(Note: All Raspberry Pis are reachable from the control machine via Twingate.)*

---

## Repository Structure

```text
.
├── ansible.cfg                        # Default Ansible configuration (inventory, remote_user=pi)
├── group_vars/
│   ├── raspberrypi/
│   │   └── vault.yml                  # Encrypted secrets for raspberrypi (Twingate, Samba)
│   └── strato_vps/
│       └── vault.yml                  # Encrypted secrets for strato_vps (sudo password)
├── inventory/
│   └── hosts                          # Inventory hosts, groups, and connection variables
└── playbooks/
    ├── main.yml                       # Base OS updates & packages on raspberrypi
    ├── drive.yml                      # External drive fstab mount on raspberrypi
    ├── docker.yml                     # Docker CE & weekly prune cron on raspberrypi
    ├── docker-compose.yml             # Pulls NicoGartmann/docker-config & starts services
    ├── monitoring.yml                 # Installs node_exporter & smartctl_exporter on Pis
    └── vps-setup.yml                  # Base OS setup, user creation, and SSH hardening on strato_vps
```

---

## Playbooks & Usage

Run all commands from the repository root:

### 1. Raspberry Pi — Base Setup (`playbooks/main.yml`)
Updates apt cache, runs `dist-upgrade` and `autoremove`, and installs base utilities (`htop`, `git`, `curl`) on `raspberrypi`.
```bash
ansible-playbook playbooks/main.yml
```

### 2. Raspberry Pi — External Drive Mount (`playbooks/drive.yml`)
Creates `/mnt/drive` and configures `/etc/fstab` to mount the external `ext4` drive by UUID on `raspberrypi`.
```bash
ansible-playbook playbooks/drive.yml
```

### 3. Raspberry Pi — Docker Installation (`playbooks/docker.yml`)
Installs `docker-ce`, `docker-ce-cli`, `containerd.io`, and `docker-compose-plugin` from the official Docker Debian `arm64` repository, adds `pi` to the `docker` group, and configures a weekly `docker system prune -af` cron job (Sundays at 03:00).
```bash
ansible-playbook playbooks/docker.yml
```

### 4. Raspberry Pi — Docker Compose Services (`playbooks/docker-compose.yml`)
Clones/updates [`NicoGartmann/docker-config`](https://github.com/NicoGartmann/docker-config) in `/opt/services`, creates `/mnt/drive/nas` and `/opt/rclone`, writes secrets from `group_vars/raspberrypi/vault.yml` to `/opt/services/.env`, and runs `docker compose pull` + `docker compose up -d`.
```bash
ansible-playbook playbooks/docker-compose.yml --ask-vault-pass
```

### 5. Raspberry Pis — Monitoring Exporters (`playbooks/monitoring.yml`)
Installs Prometheus exporters as native binaries and `systemd` services across the Pi fleet:
- `node_exporter` on all hosts in `monitoring_targets` (`raspberrypi`, `homeassistant_pi`, `parents_pi`).
- `smartctl_exporter` (plus `smartmontools`) on all hosts in `drive_hosts` (`raspberrypi`, `parents_pi`).
```bash
ansible-playbook playbooks/monitoring.yml
```

### 6. Strato VPS — Base Setup & SSH Hardening (`playbooks/vps-setup.yml`)
Provisions `strato_vps`:
- Updates packages and installs `sudo`, `htop`, `git`, and `curl`.
- Ensures non-root user `nico` exists with `sudo` group membership and SSH `authorized_keys` copied from `root`.
- Hardens OpenSSH (`/etc/ssh/sshd_config` and `/etc/ssh/sshd_config.d/00-hardening.conf`) with `PermitRootLogin no`, `PasswordAuthentication no`, and `PubkeyAuthentication yes`.

*(Note: For first-time bootstrap on a brand-new VPS where `nico` does not exist yet, temporarily pass `-e ansible_user=root` on the first run.)*
```bash
ansible-playbook playbooks/vps-setup.yml --ask-vault-pass
```

---

## Dry Run & Syntax Check

Before applying changes to physical Pis or the VPS, you can validate syntax or run in check mode:

```bash
# Check playbook syntax
ansible-playbook playbooks/<playbook>.yml --syntax-check

# Dry run with diff output
ansible-playbook playbooks/<playbook>.yml --check --diff
```

---

## Secrets Management (Ansible Vault)

Sensitive variables are encrypted using `ansible-vault`:

- **`group_vars/raspberrypi/vault.yml`**:
  - `vault_twingate_network`
  - `vault_twingate_access`
  - `vault_twingate_refresh`
  - `vault_samba_password`
- **`group_vars/strato_vps/vault.yml`**:
  - `vault_vps_sudo_password`

To view or edit encrypted vault files:
```bash
ansible-vault edit group_vars/raspberrypi/vault.yml
ansible-vault edit group_vars/strato_vps/vault.yml
```
