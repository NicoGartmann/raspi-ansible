# Code-Review: `raspi-ansible` Refactoring (SUP-143)

- **Branch**: `feat/SUP-143-Ansible-Repo-mit-KI-bisherige-Struktur-grillen-und-reviewen` → `main`
- **Geprüfte Commits**: `7d4f25b` bis `baddab3` (7 Commits, 24 geänderte Dateien)
- **Referenz-Architektur**: [`docs/SUP-143 Ansible Repo: mit KI über bisherige Struktur grillen und reviewen.md`](./SUP-143%20Ansible%20Repo:%20mit%20KI%20über%20bisherige%20Struktur%20grillen%20und%20reviewen.md)
- **Syntax-Check**: Alle 7 Playbooks unter `playbooks/*.yml` bestehen `ansible-playbook --syntax-check` fehlerfrei.

---

## 1. Gesamtfazit (TL;DR)

Das Refactoring setzt die in **SUP-143** getroffenen Architektur-Entscheidungen sehr sauber, strukturiert und vollständig um. Die Trennung in **Inventory (`inventory/`)**, **wiederverwendbare Rollen (`roles/`)**, **On-Demand-Vaults (`vaults/`)** und **schlanke Einstiegs-Playbooks (`playbooks/`)** macht das Repository deutlich wartbarer und direkt bereit für zukünftige Erweiterungen (weitere Pis, Themen-basierte Docker-Compose-Stacks, Monitoring).

**Empfehlung**: **Approved / Merge-ready** (die in Abschnitt 3 identifizierten Punkte zu `state: mounted`, NTFS-Support für `parents_pi` sowie `changed_when` bei `docker compose up -d` wurden direkt im Anschluss behoben).

---

## 2. Abgleich mit den Architektur-Entscheidungen (SUP-143)

| Bereich | Anforderung laut Architektur-Plan | Umsetzung im Branch | Status |
| :--- | :--- | :--- | :---: |
| **Inventory & Gruppen** | Logische Hostnamen (`raspberrypi`, `parents_pi`, `strato_vps`), Funktionsgruppen (`pis`, `vps`, `drive_hosts`, `docker_hosts`, `monitoring_targets`), `homeassistant_pi` ausgeklammert | Exakt so in [`inventory/hosts.yml`](../inventory/hosts.yml) umgesetzt | ✅ |
| **Variablen & Vaults** | Klartext-Variablen in `inventory/group_vars/` & `inventory/host_vars/<host>/vars.yml`; Vaults nach `vaults/<host>.yml` ausgelagert (On-Demand Load) | Sauber getrennt; Alltags-Playbooks laufen ohne `--ask-vault-pass` | ✅ |
| **Rolle `base`** | Einheitlicher User `nico`, `NOPASSWD: ALL` in `/etc/sudoers.d/nico` (mit `visudo`-Validierung), SSH-Keys aus `all.yml` + Bootstrapping-Fallback + Lockout-Gate + SSH-Härtung | Vollständig in [`roles/base/tasks/main.yml`](../roles/base/tasks/main.yml) implementiert; 3 echte SSH-Keys in Commit `829ea78` hinterlegt | ✅ |
| **Rolle `drive`** | Multi-Disk-Support über Liste `host_drives` pro Host | In [`roles/drive/tasks/main.yml`](../roles/drive/tasks/main.yml) umgesetzt; in Commit `baddab3` bereits mit den 2 echten Laufwerken für `parents_pi` befüllt | ✅ |
| **Rolle `docker`** | Dynamische Erkennung von Debian/Ubuntu & `arm64`/`amd64`/`armhf`, moderner GPG-Keyring (`/etc/apt/keyrings/docker.asc`), Docker-Gruppe & Prune-Cron | Sauber in [`roles/docker/tasks/main.yml`](../roles/docker/tasks/main.yml) gelöst (altes `apt_key` abgelöst) | ✅ |
| **Rolle `docker_compose`** | Git-Clone/Pull, Host-Verzeichnisse, `.env` aus Vault (`no_log: true`), Loop über `compose_files` für Einzel- & Themen-Stacks | Vollständig in [`roles/docker_compose/tasks/main.yml`](../roles/docker_compose/tasks/main.yml) umgesetzt | ✅ |
| **Playbooks & Cleanup** | `site.yml`, `base-setup.yml`, `update.yml` neu; `main.yml` und `vps-setup.yml` entfernt; `monitoring.yml` vorerst unverändert | Alle Playbooks migriert, Altlasten entfernt | ✅ |
| **Dokumentation** | `README.md` und `CLAUDE.md` auf neue Struktur und Workflows aktualisiert | Umfassend und verständlich dokumentiert | ✅ |

---

## 3. Wichtige Findings (im Rahmen des Reviews direkt behoben)

### 3.1 ✅ [Behoben] `roles/drive`: `state: present` schreibt nur `/etc/fstab`, mountet aber nicht sofort
* **Datei**: [`roles/drive/tasks/main.yml`](../roles/drive/tasks/main.yml)
* **Beobachtung**: Im Task `fstab-Einträge für konfigurierte Festplatten sicherstellen` wurde `state: present` verwendet (wie schon im alten `playbooks/drive.yml`).
* **Auswirkung**: `state: present` trägt die Festplatte **nur** in `/etc/fstab` ein, hängt sie aber im laufenden Betrieb **nicht** sofort ein. Wenn [`playbooks/site.yml`](../playbooks/site.yml) auf einem frischen System am Stück durchläuft (`base` → `drive` → `docker` → `docker_compose`), war `/mnt/drive` beim anschließenden Lauf von `docker_compose` noch gar nicht gemountet. Der Task `Benötigte Host-Verzeichnisse für Container erstellen` (`/mnt/drive/nas`) hätte den Ordner dann auf der SD-Karte im leeren Mountpoint-Verzeichnis angelegt statt auf der externen Festplatte.
* **Umgesetzte Lösung**: Auf `state: "{{ item.state | default('mounted') }}"` umgestellt, sodass konfigurierte Laufwerke sowohl in `/etc/fstab` eingetragen als auch sofort gemountet werden.

### 3.2 ✅ [Behoben] `parents_pi`: NTFS-Festplatte mit `passno: "2"` in `/etc/fstab`
* **Datei**: [`inventory/host_vars/parents_pi/vars.yml`](../inventory/host_vars/parents_pi/vars.yml) (Zeile 4–9) & [`roles/drive/tasks/main.yml`](../roles/drive/tasks/main.yml)
* **Beobachtung**: In Commit `baddab3` wurde für `/mnt/oldhdd` eine NTFS-Festplatte (`fstype: ntfs`, `passno: "2"`) konfiguriert.
* **Auswirkung**:
  1. **`passno: "2"` bei NTFS**: Ein `passno`-Wert `> 0` weist `systemd-fsck` beim Booten an, `fsck.ntfs` auszuführen. Unter Linux/Debian existiert standardmäßig kein vollwertiges `fsck.ntfs` für den automatischen Boot-Check, was beim Systemstart zu Fehlermeldungen oder Verzögerungen führen kann. Für NTFS-Laufwerke in `/etc/fstab` sollte `passno: "0"` gesetzt werden.
  2. **NTFS-Treiberpaket**: Je nach Raspberry Pi OS Version wird für `fstype: ntfs` in `/etc/fstab` das Paket `ntfs-3g` benötigt.
* **Umgesetzte Lösung**: In [`inventory/host_vars/parents_pi/vars.yml`](../inventory/host_vars/parents_pi/vars.yml) wurde `passno: "0"` für `/mnt/oldhdd` gesetzt und [`roles/drive/tasks/main.yml`](../roles/drive/tasks/main.yml) installiert automatisch `ntfs-3g`, sobald ein Laufwerk mit `fstype: ntfs` konfiguriert ist.

### 3.3 ✅ [Behoben] `docker compose up -d`: `changed_when`-Erkennung bei Container-Recreate
* **Dateien**: [`roles/docker_compose/tasks/main.yml`](../roles/docker_compose/tasks/main.yml) & [`playbooks/update.yml`](../playbooks/update.yml)
* **Beobachtung**: Das moderne `docker compose` (v2 Plugin) gibt Statusmeldungen auf `stderr` im Past-Tense aus (`Container <name> Recreated`, `Container <name> Started`, `Container <name> Created`). Wenn ein bereits laufender Container nach einem `docker compose pull` mit neuem Image neu erstellt wird, fehlte `'Recreated' in ...stderr` in der Bedingung.
* **Umgesetzte Lösung**: `'Recreated' in compose_up.stderr` (bzw. `update_compose_up.stderr`) ergänzt:
  ```yaml
  changed_when: >-
    'Creating' in compose_up.stdout or 'Recreating' in compose_up.stdout or
    'Created' in compose_up.stderr or 'Recreated' in compose_up.stderr or
    'Started' in compose_up.stderr
  ```

---

## 4. Kleinere Hinweise & Optimierungspotenziale (Nice-to-have)

### 4.1 Code-Duplizierung zwischen `roles/docker_compose` und `playbooks/update.yml`
* **Beobachtung**: Die 3 Tasks (`git` update, `docker compose pull`, `docker compose up -d`) in [`playbooks/update.yml`](../playbooks/update.yml) (Zeile 25–48) sind identisch mit den Tasks in [`roles/docker_compose/tasks/main.yml`](../roles/docker_compose/tasks/main.yml).
* **Einordnung**: Das funktioniert einwandfrei und hält `update.yml` frei von Vault-Abhängigkeiten. Perspektivisch könnte man in `roles/docker_compose` eine Variable wie `compose_deploy_env: true` einführen und die Rolle in `update.yml` mit `compose_deploy_env: false` wiederverwenden, um die Compose-Logik nur an einer einzigen Stelle pflegen zu müssen.

### 4.2 `roles/base`: Verhalten im `--check`-Modus beim Erst-Bootstrapping
* **Datei**: [`roles/base/tasks/main.yml`](../roles/base/tasks/main.yml) (Zeile 26–43)
* **Beobachtung**: Wenn der Benutzer `{{ admin_user }}` (`nico`) auf einem Zielsystem noch **gar nicht** existiert und man `ansible-playbook playbooks/base-setup.yml --check --diff` ausführt, wird `/home/nico/.ssh` im Check-Mode nur simuliert. Der nachfolgende `lineinfile`-Task auf `/home/nico/.ssh/authorized_keys` kann im `--check`-Modus dann melden, dass das Elternverzeichnis `/home/nico/.ssh` nicht existiert. Sobald `nico` einmal angelegt ist, läuft `--check` problemlos durch.

### 4.3 `roles/base`: `lineinfile` vs. `ansible.posix.authorized_key`
* **Datei**: [`roles/base/tasks/main.yml`](../roles/base/tasks/main.yml) (Zeile 34–43)
* **Einordnung**: Die Nutzung von `ansible.builtin.lineinfile` vermeidet externe Ansible-Galaxy-Collection-Abhängigkeiten (`ansible.posix`) und funktioniert zuverlässig. Man sollte lediglich im Hinterkopf behalten, dass `lineinfile` alte Schlüssel nicht automatisch aus `~/.ssh/authorized_keys` löscht, falls später ein Key aus `admin_ssh_public_keys` entfernt wird.

### 4.4 ✅ [Behoben] Kleine Doku-Aktualisierung nach Commit `baddab3`
* **Datei**: [`README.md`](../README.md) (Zeile 33)
* **Beobachtung**: Im Verzeichnisbaum der `README.md` stand bei `inventory/host_vars/parents_pi/vars.yml` noch `# Placeholders for the 2 external drives & compose stacks`, obwohl die 2 Festplatten in Commit `baddab3` bereits aktiv eingetragen wurden. Wurde angepasst.

---

## 5. Commit-Historie im Überblick

Die Commit-Historie ist sauber nach Conventional Commits (`docs`, `refactor`, `feat`, `chore`) mit Ticket-Referenz (`SUP-143`) gegliedert:

1. `7d4f25b` — `docs(SUP-143): add architecture and refactoring plan`
2. `7713e64` — `refactor(SUP-143): restructure inventory, host_vars, group_vars, and vaults`
3. `754919d` — `feat(SUP-143): add reusable roles for base, drive, docker, and docker_compose`
4. `183e491` — `refactor(SUP-143): migrate playbooks to role-based structure and add update.yml`
5. `883845e` — `docs(SUP-143): update README.md and CLAUDE.md for role-based architecture`
6. `829ea78` — `chore(SUP-143): added public keys`
7. `baddab3` — `feat(SUP-143): configure external drive mounts for parents_pi`
