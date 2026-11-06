# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Ansible playbooks that provision Raspberry Pis and a rented Strato VPS. `inventory/hosts` defines:
- Three Raspberry Pis reachable over Twingate:
  - `raspberrypi` (home, Docker services, `192.168.178.10`)
  - `homeassistant_pi` (home, Home Assistant, `192.168.0.20`)
  - `parents_pi` (parents' house, `192.168.178.46`)
  - Derived groups: `drive_hosts` (Pis with an external drive: `raspberrypi` + `parents_pi`) and `monitoring_targets` (all three Pis).
- One public-facing VPS:
  - `strato_vps` (`217.154.166.48`, Ubuntu 26.04 LTS, user `nico`).

There is no application code here — every change is a change to server configuration/state on those devices.

## Commands

Run all commands from the repo root (`ansible.cfg` sets `inventory = ./inventory/hosts` and `remote_user = pi`).

```bash
# Base OS setup on raspberrypi: apt update/upgrade/autoremove + install htop/git/curl
ansible-playbook playbooks/main.yml

# Install Docker, add `pi` to the docker group, set up weekly `docker system prune -af` cron
ansible-playbook playbooks/docker.yml

# Deploy/update docker-compose services from the separate NicoGartmann/docker-config repo,
# writes Twingate/Samba secrets into /opt/services/.env, then `docker compose pull` + `up -d`
ansible-playbook playbooks/docker-compose.yml --ask-vault-pass

# Mount the external drive (by UUID) at /mnt/drive via fstab
ansible-playbook playbooks/drive.yml

# Install node_exporter (all 3 Pis) and smartctl_exporter (Pis with a drive) as systemd
# services, scraped by the Prometheus/Grafana stack on the monitoring VPS (separate repo)
ansible-playbook playbooks/monitoring.yml

# Base OS setup, user creation (`nico` + sudo + SSH keys), and SSH hardening on strato_vps
ansible-playbook playbooks/vps-setup.yml --ask-vault-pass

# Dry run / check mode before applying changes
ansible-playbook playbooks/<file>.yml --check --diff

# Syntax check a playbook
ansible-playbook playbooks/<file>.yml --syntax-check
```

There are no lint or test tooling configured (no `ansible-lint` config, no CI). If invoked, `ansible-playbook --syntax-check` and `--check --diff` are the practical substitutes for "testing" a change before it touches the real servers.

## Secrets

- `group_vars/raspberrypi/vault.yml` is an `ansible-vault`-encrypted file holding Twingate and Samba credentials (`vault_twingate_network`, `vault_twingate_access`, `vault_twingate_refresh`, `vault_samba_password`) consumed by `playbooks/docker-compose.yml`.
- `group_vars/strato_vps/vault.yml` is an `ansible-vault`-encrypted file holding the VPS sudo password (`vault_vps_sudo_password`) consumed by `playbooks/vps-setup.yml` and `strato_vps` privilege escalation.

Edit vault files with:

```bash
ansible-vault edit group_vars/raspberrypi/vault.yml
ansible-vault edit group_vars/strato_vps/vault.yml
```

Never print or write out decrypted contents in plaintext, and never commit a decrypted version.

## Architecture notes

- Each file under `playbooks/` is a standalone playbook — they are not imported by `playbooks/main.yml` despite its name; run each one independently for its concern:
  - `main.yml`, `docker.yml`, `docker-compose.yml`, and `drive.yml` target only `raspberrypi`.
  - `monitoring.yml` targets the `monitoring_targets`/`drive_hosts` groups across all three Pis.
  - `vps-setup.yml` targets `strato_vps`.
- Note on `group_vars/`: Because `ansible.cfg` points to `./inventory/hosts`, files under `./group_vars/` at the repo root are not auto-loaded by inventory path discovery and are explicitly included via `vars_files` in playbooks that need them (`docker-compose.yml`, `vps-setup.yml`).
- `strato_vps` runs Ubuntu 26.04 LTS, which defaults to `sudo-rs` (`/usr/bin/sudo`). Because `sudo-rs` does not support custom `-p` prompts used by Ansible's `sudo` become plugin, `[strato_vps:vars]` in `inventory/hosts` sets `ansible_become_exe=/usr/bin/sudo.ws` (traditional C `sudo`) along with `ansible_become_password={{ vault_vps_sudo_password }}`.
- `playbooks/monitoring.yml` installs `node_exporter`/`smartctl_exporter` as binaries + systemd units (not Docker containers), since `homeassistant_pi` and `parents_pi` aren't assumed to run Docker. Container-level metrics (`cadvisor`) instead live as a service in the `docker-config` repo, next to the other containers on `raspberrypi`.
- `playbooks/docker-compose.yml` does not contain application/service definitions itself — the actual `docker-compose.yml` and service configs live in the external `NicoGartmann/docker-config` repo, which this playbook clones/pulls to `/opt/services` on the Pi and then runs `docker compose pull && docker compose up -d` against.
- Task names and comments are written in German; keep new tasks consistent with that convention unless told otherwise.
- `host_key_checking = False` and `deprecation_warnings = False` are set deliberately in `ansible.cfg`.
