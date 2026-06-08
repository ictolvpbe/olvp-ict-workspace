---
name: project-template-strategy
description: "Tier 1/Tier 2 baseline-strategie voor OLVP-VMs: één golden-image Tier 1 (cleane Debian + Ansible-managed deps) wordt gecloned naar elke nieuwe VM (Odoo, Forgejo, MCP, Wazuh, Bacula, ...). Tier 2 = applicatie-specifieke overlay via Ansible-playbook. Live sinds 2026-06-01."
metadata:
  type: project
---

**Beslissing 2026-06-01**: alle nieuwe OLVP-VMs (mgmt + webapps) clones van **één** Tier 1 baseline-template, daarna app-specifieke overlay via Ansible. Geen tweede template per applicatie.

**Why:** zonder gedeelde baseline kruipt configuratie-drift in tussen VMs (verschillende Podman-versies, certs niet up-to-date, ansible-user-config-edge-cases). Eén template = één onderhouds-pad. Plus: Forgejo, MCP, en toekomstige mgmt-VMs hebben **dezelfde** basis-stack (Podman + step + Ansible) — duplicatie van templates voor Odoo vs niet-Odoo loont niet voor onze schaal.

## Tier 1 — universele baseline

Wat staat in de golden-image (`SRVV-ODOO-TEMPLATE` op 10.200.0.40, Proxmox-snapshot `Tier1-baseline-2026-06-01`):

| Component | Status |
|---|---|
| Debian 13 minimal | OS-baseline |
| `ansible`-user met NOPASSWD sudo + SSH-key in authorized_keys | Service-account voor automation (zie [[provision-ansible-account]] runbook) |
| Podman 5.4.2 | Container-runtime (per [[ADR 0004]]) |
| step CLI 0.30.2 | Smallstep CLI voor cert-management |
| step-ca-root cert in `/usr/local/share/ca-certificates/` + `update-ca-certificates` | OLVP Internal CA trust voor alle TLS-verbindingen |
| `/etc/containers/registries.conf.d/unqualified.conf` met `unqualified-search-registries = ["docker.io"]` | Voor short-name image-resolutie (anders moet elke `FROM` fully-qualified — zie [[feedback-docker-to-podman-migration]] gotcha 1) |
| Cockpit + cockpit-podman op port 9090 (HTTPS, self-signed default) | Web-UI per host voor system + container management (vervangt Portainer per [[project-containerization]]) |
| Hostname placeholder (`SRVV-ODOO-TEMPLATE`) | Clones overschrijven via `hostnamectl set-hostname` + `/etc/hosts` edit |
| Geen Docker, geen iptables-residu | `apt purge docker.io + autoremove + rm -rf /var/lib/docker /var/lib/containerd + reboot` (per [[feedback-docker-to-podman-migration]]) |

**Bonus voor Odoo-clones** (Odoo-overlay-skeleton, gekozen 2026-06-01 als pragmatisch compromis vs pure-Tier-1):
- `/opt/odoo-multi-instance/` met `Dockerfile`, `certs/` (4 files: `olvp-ca.crt` + `SRVV-INFRA002.{cer,crt,pem}`), `addons1/2/3` (template-content), `config1/2/3`, lege `db1-3-data` + `odoo1-3-data` dirs
- Forgejo-clones moeten deze `/opt/odoo-multi-instance/` zelf opruimen (kleine extra stap in toekomstige `forgejo.yml` Ansible-role)

## Tier 2 — applicatie-overlay via Ansible

Per applicatie eigen Ansible-playbook in `platform-ansible`:

| App | Playbook | Status |
|---|---|---|
| HAProxy + keepalived | `site.yml` | ✅ Live (TST-PROD x2) |
| Caddy reverse-proxy (Odoo-VMs) | `caddy.yml` met target `webapps` | ✅ Live (3 Odoo-VMs) |
| Addons-update (Odoo) | `addons-update.yml` met target `webapps` | ⏳ Klaar voor Semaphore-test (#29) |
| Forgejo | (toekomstig) `forgejo.yml` | ⏳ #30 |
| MCP-Odoo | (toekomstig) onderdeel van `addons-update.yml` of aparte | ⏳ #31 |
| ACC-01 (account-admin) | hergebruik `caddy.yml` (template moet `expose: internal` ondersteunen) | ⏳ Aparte werf |

## Clone-conventie

Bij elke nieuwe VM:

1. **Proxmox**: clone van `SRVV-ODOO-TEMPLATE` (full clone, niet linked — voor onafhankelijke lifecycle).
2. **Eerste boot**: hostname-edit via `hostnamectl` + `/etc/hosts` + reboot. IP-config via Proxmox cloud-init of handmatig.
3. **Tier 2 overlay**: Ansible-playbook(s) runnen op de nieuwe host (`inventory.yml` uitbreiden + run).
4. **Cert-installatie** (per FQDN): handmatige `step ca certificate` met admin-provisioner password (eenmalig per FQDN) — alle subsequente renewals daarna automatic via systemd-timer.

## Snapshot-conventie Proxmox

| Wanneer | Naam-pattern |
|---|---|
| Tier 1 baseline na grote upgrades (Podman, step, OS) | `Tier1-baseline-<YYYY-MM-DD>` |
| Vóór risico-volle migratie op een live VM | `<vmname>-pre-<change>-<YYYY-MM-DD>` |
| Routine-recoverypoint | Automatisch via Proxmox Backup Server (Bacula komt later voor file-level binnen VM) |

## How to apply

- **Nieuwe VM = clone van template, NIET fresh install** — sneller, gegarandeerd consistent met huidige Tier 1.
- **Template-updates** (Podman versie, step CLI versie, etc.): clone-template → apply update → Proxmox-snapshot met nieuwe datum. Oude snapshot houden voor rollback.
- **Forgejo-clones**: zelfde Tier 1, daarna in `forgejo.yml`: `rm -rf /opt/odoo-multi-instance/` als pre-task om Odoo-residu weg te halen.
- **Cockpit-cert vervangen**: standaard self-signed → kan vervangen door step-ca cert via `/etc/cockpit/ws-certs.d/0-self-signed.cert` overschrijven. Later op te zetten via Ansible-role.

Gerelateerd: [[project-containerization]] (Podman + ADR 0004), [[provision-ansible-account]] (Tier 1 ansible-user), [[project-internal-pki-coverage]] (step-ca trust everywhere), [[feedback-docker-to-podman-migration]] (gotchas tijdens migratie).
