# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Ansible playbooks that provision a single Raspberry Pi (host `raspberrypi` in `inventory/hosts`, currently `192.168.178.10`). There is no application code here — every change is a change to server configuration/state on that physical device.

## Commands

Run all commands from the repo root (`ansible.cfg` sets `inventory = ./inventory/hosts` and `remote_user = pi`).

```bash
# Base OS setup: apt update/upgrade/autoremove + install htop/git/curl
ansible-playbook playbooks/main.yml

# Install Docker, add `pi` to the docker group, set up weekly `docker system prune -af` cron
ansible-playbook playbooks/docker.yml

# Deploy/update docker-compose services from the separate NicoGartmann/docker-config repo,
# writes Twingate secrets into /opt/services/.env, then `docker compose pull` + `up -d`
ansible-playbook playbooks/docker-compose.yml

# Mount the external drive (by UUID) at /mnt/drive via fstab
ansible-playbook playbooks/drive.yml

# Dry run / check mode before applying changes
ansible-playbook playbooks/<file>.yml --check --diff

# Limit to a single playbook's tasks by tag or name if tags are added later
ansible-playbook playbooks/<file>.yml --syntax-check
```

There are no lint or test tooling configured (no `ansible-lint` config, no CI). If invoked, `ansible-playbook --syntax-check` and `--check --diff` are the practical substitutes for "testing" a change before it touches the real Pi.

## Secrets

`group_vars/raspberrypi/vault.yml` is an `ansible-vault`-encrypted file holding Twingate credentials (`vault_twingate_network`, `vault_twingate_access`, `vault_twingate_refresh`) consumed by `playbooks/docker-compose.yml`. Edit it with:

```bash
ansible-vault edit group_vars/raspberrypi/vault.yml
```

Never print or write out its decrypted contents in plaintext, and never commit a decrypted version.

## Architecture notes

- Each file under `playbooks/` is a standalone playbook targeting the `raspberrypi` host group — they are not imported by `playbooks/main.yml` despite its name; run each one independently for its concern (base packages, Docker install, compose deployment, disk mount).
- `playbooks/docker-compose.yml` does not contain application/service definitions itself — the actual `docker-compose.yml` and service configs live in the external `NicoGartmann/docker-config` repo, which this playbook clones/pulls to `/opt/services` on the Pi and then runs `docker compose pull && docker compose up -d` against.
- Task names and comments are written in German; keep new tasks consistent with that convention unless told otherwise.
- `host_key_checking = False` and `deprecation_warnings = False` are set deliberately in `ansible.cfg`.
