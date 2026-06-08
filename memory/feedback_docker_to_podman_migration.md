---
name: feedback-docker-to-podman-migration
description: "Concrete gotchas bij Docker→Podman migratie op een server: unqualified-search-registries config, --network=host tijdens build om Docker-iptables-residu te omzeilen, Podman Quadlet-units zijn transient (geen systemctl enable). Trigger: nieuwe Odoo-VM die van Docker naar Podman migreert."
metadata:
  type: feedback
---

**Regel:** bij elke Docker→Podman migratie op een server (zoals TST-ODOO-01 op 2026-06-01, SRVV-ODOO-01 later in schoolvakantie) zijn er drie gotchas die elk ~5min verlies veroorzaakten. Documenteer ze hier om herhaling te voorkomen.

**Gotcha 1 — `unqualified-search-registries` ontbreekt in default Debian-config:**
```
Error: short-name "odoo:19.0" did not resolve to an alias and no unqualified-search registries are defined
```
Fix vooraf, eenmaal per VM:
```bash
echo 'unqualified-search-registries = ["docker.io"]' \
  | sudo tee /etc/containers/registries.conf.d/unqualified.conf > /dev/null
```
Eventueel ook van TST-ODOO-02 kopiëren — `shortnames.conf` heeft veel aliases (alpine, debian, ubuntu, ...) maar `odoo` zit daar standaard niet in.

**Gotcha 2 — DNS-resolution faalt tijdens `podman build` na Docker-disable:**
```
Temporary failure in name resolution
WARNING: Retrying after connection broken by NewConnectionError
ERROR: Could not find a version that satisfies the requirement ldap3
```
Docker laat na `systemctl disable docker` nog **iptables-chains** rondhangen die Podman's bridge-network DNS interrupteren. Twee fixes:
- **Quick** (eenmaal per build): `sudo podman build --network=host -t <tag> .` — bypass de bridge, gebruik host-DNS direct.
- **Structureel**: na alle Podman-containers up zijn, `reboot` om Docker iptables-state volledig te resetten. Daarna `podman` werkt normaal.

`--network=host` tijdens build is acceptabel security-trade-off omdat build-fase tijdelijk is en de container niet runtime in host-mode komt.

**Gotcha 3 — Podman Quadlet-units zijn transient/generated:**
```
Failed to enable unit: Unit /run/systemd/generator/caddy.service is transient or generated
```
Quadlet-generator genereert units in `/run/systemd/generator/`, niet `/etc/systemd/system/`. Dat betekent:
- `systemctl enable <name>.service` → **faalt** voor Quadlet-units
- `systemctl start <name>.service` → **werkt** (na `daemon-reload`)
- Boot-time start gebeurt via `[Install] WantedBy=...` in de `.container` file, niet via systemctl enable

In Ansible-playbook: gebruik `systemd: state=started, daemon_reload=yes`, NIET `enabled: yes`. Voor normale (non-Quadlet) systemd-units (zoals `caddy-cert-renew.timer`) wel `enabled: yes`.

Gefixt in [`platform-ansible/caddy.yml`](platform-ansible/caddy.yml) op 2026-06-01 na TST-ODOO-01 migratie.

**How to apply:**
- Volg [`platform-handbook/hosting/operations/migrate-odoo-to-podman.md`](platform-handbook/hosting/operations/migrate-odoo-to-podman.md) runbook.
- Bij gotcha 1: registries.conf eerst toevoegen vóór `podman build` proberen.
- Bij gotcha 2: gebruik `--network=host` voor de eerste build, niet voor runtime.
- Bij gotcha 3: nooit `systemctl enable` voor Quadlet-generated units; gebruik `daemon_reload` na elke template-deploy.

Gerelateerd: [[project-containerization]] (Podman-first strategie + Docker iptables-reset via reboot), [[feedback-yaml-colon-space]] (eveneens config-syntax-eenduidigheid), TST-ODOO-02 migratie-precedent (2026-05-29).
