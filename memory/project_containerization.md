---
name: project-containerization
description: Containerization-strategie OLVP. Podman als default-runtime (vervangt Docker) sinds ADR 0004 op 2026-05-23. Odoo draait nu in Docker-containers (niet bare-metal-processes), migratie naar Podman gepland. Portainer uit te faseren, cockpit-podman als per-host UI.
metadata:
  type: project
---

OLVP containerization-strategie (vastgelegd 2026-05-23):

**Belangrijk feit dat eerst niet in docs stond**: Odoo-instances draaien al als **Docker-containers** (multi-instance per VM, niet bare-metal-processes zoals onze oude docs suggereerden). Memory + vm-inventory bijgewerkt om dit te reflecteren.

**Beslissing**: Podman wordt default container-runtime. Geen big-bang migratie maar gefaseerd.

**Why** (kort):
- `dockerd` als permanente root-daemon = blast-radius bij compromise
- `/var/run/docker.sock` = "één regel mounten = volledige host-controle"
- Podman rootless first-class + geen daemon + systemd Quadlet-integratie
- OCI-image-compatibel, geen lock-in, geen verlies van features die we gebruiken
- Bus-factor: Linux+systemd-skills vertalen zich direct naar Podman; geen Docker-specifieke patterns te leren

**Niet-keuze**: K8s/K3s (disproportioneel voor onze schaal), LXC (voor systeem-containers — doet Proxmox al), all-native (verliest container-isolation-laag).

**Migratiepath** (drie stappen, oplopende complexiteit):
1. **Semaphore-greenfield (pilot Podman-deploy)** — Semaphore op `10.35.0.10` (VLAN 35) live op Podman sinds 2026-05-23 als eerste OLVP-Podman-deploy. Leerwerf voor Quadlet + journald + UID-mapping. Setup-procedure: `management-tools/semaphore.md`.
2. **Odoo test cutover** — **Uitgevoerd 2026-05-29 op `SRVV-TST-ODOO-02` (10.200.0.41), niet -01** zoals oorspronkelijk gepland, omdat -02 een vrije test-bak was met identieke Docker-stack maar geen actieve dev-data. Rootful Podman, zelfde `docker-compose.yml` + `Dockerfile` na in-place edit naar FQDN-image-namen (`docker.io/library/...`), `podman-compose -f docker-compose.yml up -d` werkte direct. Patroon bewezen voor prod-migratie.
3. **Odoo prod** — `SRVV-ODOO-01` in schoolvakantie, **gecombineerd met IP-shift naar VLAN 36** (R-25). Twee transitions in één maintenance-window. -01-pilot kan parallel of overgeslagen worden gezien -02 het pattern al bewezen heeft.

**Daarna greenfield op Podman**: Wazuh, Bacularis, MCP, Forgejo, monitoring-stack.

**Cleanup**: Docker + Portainer uitfaseren na ~2 weken stabiele draaitijd op Podman over alle services.

**Beheer-UI**: Cockpit + cockpit-podman per host (geen central-management). Past bij schaal.

**Runtime-conventies (te formaliseren bij eerste echte deploy)**:
- Rootless waar mogelijk; root waar privileged-ports of UID-specifieke vereisten zijn
- systemd Quadlet voor productie (`.container`-files in `/etc/containers/systemd/`)
- Image-pinning altijd op sha256-digest of expliciete versie-tag, nooit `latest`
- Labels: `org.opencontainers.image.source`, `org.olvp.werf`, `org.olvp.fase`
- Naming: `<service>-<rol>.container`
- Logging via journald → Loki via promtail (Fase 3)

**Risk-entries**:
- R-29: Docker→Podman migratie-risico (volume-eigenaarrechten rootless, compose-file-incompat, custom-modules-compat, gecombineerd met R-25). Score 8. Mitigatie: pilot eerst, vzdump-restore-point, gedetailleerde rollback. Lager risico-druk omdat Odoo nog niet in productie is.

**Bevestigde reality-gaps tijdens TST-ODOO-02 cutover (2026-05-29):**
- `unqualified-search-registries` op Debian 13 default leeg → óf compose-file + Dockerfile FQDN-pinnen (`docker.io/library/...`), óf system-wide config (niet aanbevolen — zie [[containers/README]] gotcha). Wij kozen FQDN-pin.
- Docker volledig stoppen + service-disable + **reboot** = schoonste pad voor iptables. `iptables -X DOCKER-*` direct na docker-stop faalt op "Device or resource busy" door referenties in FORWARD. Reboot vermijdt het manuele forwardingchain-onthechting.
- Bestaande bind-mounts onder rootful Podman werken zonder UID-rechown — Docker-UIDs (101/999) blijven valid.
- `podman-compose` 1.x is compose-v1-compatibel, neemt `build: .` correct over (kiest Containerfile dan Dockerfile).
- Portainer-edge-agent stopt impliciet bij `systemctl stop docker.service` zonder compose-down.

**Te checken vóór prod-migratie**:
- Pilot op SRVV-TST-ODOO-01 succesvol afgerond
- Quadlet-templates voor Odoo + DB + Caddy bevestigd
- Ansible-role `roles/odoo-podman/` geschreven
- Wazuh-agent-monitoring van Podman-events gevalideerd

**Docs**:
- ADR: `platform-handbook/hosting/decisions/0004-podman-vs-docker.md`
- Werf: `platform-handbook/containers/README.md` (folder hernoemd van `docker/`)
- Tool-doc: `platform-handbook/management-tools/podman.md`
- Migratierunbook: `platform-handbook/hosting/operations/migrate-odoo-to-podman.md`

## Status na alle Odoo-VMs gemigreerd (2026-06-01)

Alle 4 Odoo-VMs draaien Podman+Quadlet — Docker uit alle VMs gepurged + iptables-reset via reboot:

| VM | Hostname | Status |
|---|---|---|
| TST-ODOO-02 | SRVV-TST-ODOO-02 | Pilot 2026-05-29, herzien 2026-06-01 |
| TST-ODOO-01 | SRVV-TST-ODOO-01 | Migratie 2026-06-01 |
| SRVV-ODOO-01 (prod) | SRVV-ODOO-01 | Migratie 2026-06-01 (vakantie-window niet nodig, server niet in gebruik) |
| Template | SRVV-ODOO-TEMPLATE | Tier 1 baseline 2026-06-01 (gereed voor clone, zie [[project-template-strategy]]) |

**Dockerfile canoniek-pattern**: gebruik **`FROM docker.io/library/odoo:19.0`** (fully-qualified) i.p.v. `FROM odoo:19.0`. Werkt zonder `unqualified-search-registries`-dependency. Versie-divergentie gedetecteerd 2026-06-01: TST-ODOO-02 + TEMPLATE waren canoniek; TST-ODOO-01 + SRVV-ODOO-01 hadden nog short-name versie. **Gesynced 2026-06-01**: alle 4 VMs nu identiek md5 (`2be40e7a26219bb6122c86fa46308ddd`). Image-rebuild nog niet getriggerd — wordt automatisch consistent bij eerstvolgende `podman build` (bv. tijdens addons-update werf #29 indien rebuild meegenomen).

**Cert/ldap3-baseline** (geverifieerd consistent over alle 4 Odoo-VMs op 2026-06-01):
- `/opt/odoo-multi-instance/certs/` = `olvp-ca.crt` (legacy OLVP CA) + `SRVV-INFRA002.{cer,crt,pem}` (AD-DC LDAP-cert in 3 formaten — voor `ldap3` bind tegen AD)
- `ldap3` versie **2.9.1** in container-image (via `pip3 install --break-system-packages ldap3` in Dockerfile)
- Bij elke image-rebuild blijven beide consistent zolang Dockerfile + certs/-dir unchanged

**Cockpit + cockpit-podman**: bevestigd als standaard web-UI per host (was al genoemd in ADR 0004 als Portainer-vervanger). Sinds 2026-06-01 onderdeel van Tier 1 baseline → automatisch op alle nieuwe clones. Default port 9090 met self-signed cert; step-ca cert later via Ansible.

Gerelateerd: [[project-architecture-haproxy]] (HAProxy blijft native, geen container), [[project-security-layers]] (containerization = extra defense-in-depth-laag), [[project-phasing]] (Odoo-migratie samen met R-25 IP-shift in maintenance-window), [[project-security-testing]] (Wazuh-rules voor Podman-events bij Wazuh-deploy), [[project-template-strategy]] (Tier 1/2 baseline), [[feedback-docker-to-podman-migration]] (gotchas).
