---
name: project-infrastructure-params
description: "Concrete infrastructuurparameters voor [[project-odoo-public-access]]: VLANs, IPs, VM-inventaris met rol+expose. Tier-scheiding tussen webapps (VLAN 36), services (VLAN 33), mgmt-appliances (VLAN 35), admin-werkstations (VLAN 34), DMZ (VLAN 21/22), test-webapps (VLAN 207). Bijgewerkt 2026-05-31."
metadata: 
  node_type: memory
  type: project
  originSessionId: 59a419f3-43f1-477b-bb2b-196d0c16926a
---

Bevestigde parameters voor [[project-odoo-public-access]] (laatste herziening 2026-05-21 na volledige VLAN-architectuur-review):

**Hypervisor:**
- Proxmox VE — user provisioned VMs zelf op basis van specs uit het actieplan.

**Standaard VM-OS (sinds 2026-05-28):**
- **Debian 13 minimal** voor alle nieuwe Linux-VMs (jump-01, stack-VMs in DMZ + mgmt + webapps). Bevestigd na step-ca-bringup waar de runbook nog Debian 12 noemde maar de werkelijke install Debian 13 was. Sommige aandachtspunten t.o.v. Debian 12: systemd 256+ (cosmetische `systemd-ssh-generator` AF_VSOCK-error op niet-vsock-VMs is harmless), OpenSSH 10.x, deb822 `.sources`-formaat verplicht voor nette repos. Proxmox-host-OS (`network-physical/physical/servers.md`) staat los hiervan: dat is Debian onder Proxmox VE en volgt Proxmox' versie-cyclus, niet deze keuze.

**VLAN-tier-scheiding (sinds 2026-05-21):**
- **VLAN 33** `VL-SD-SERVICE` `10.33.0.0/23` — schoolservers (AD, printserver, fileserver). **Géén web-apps** meer.
- **VLAN 34** `VL-SD-ADMIN` `10.34.0.0/27` — ICT-beheerder werkstations. Géén appliances.
- **VLAN 35** `VL-SD-MGMTSVC` `10.35.0.0/24` — ICT-mgmt-appliances (jump-01, Semaphore, Forgejo, Wazuh, Bacula-controle, monitoring). Nieuw.
- **VLAN 36** `VL-SD-WEBAPPS` `10.36.0.0/24` — publieke web-apps (Odoo). Nieuw.
- **VLAN 19** `VL-SD-STORAGE` `10.19.0.0/24` — NAS, backup-data, Proxmox backup-targets.
- **VLAN 21** `VL-SD-DMZ-RPROXY` `10.21.0.0/24` — HAProxy HA-paar.
- **VLAN 22** `VL-SD-DMZ-CA` `10.21.1.0/24` — step-ca.
- **VLAN 20** `VL-SD-DMZ-OUD` `10.20.0.0/23` — bestaande DMZ (sunset).

Volledige VLAN-tabel (30+ VLANs over hoofdcampus + remote BAWA + test): zie `platform-handbook/network-physical/network/vlans.md`.

**DMZ IP-toekenning HAProxy-VLAN (VLAN 21, IPs akkoord 2026-05-21; VLAN-tag verhuisd 210 → 21 op 2026-05-31):**
- 10.21.0.1 — gateway
- 10.21.0.5 — VIP (VRRP shared tussen haproxy-1 en haproxy-2)
- 10.21.0.6 — haproxy-1
- 10.21.0.7 — haproxy-2

**DMZ IP-toekenning step-ca-VLAN (VLAN 22; VLAN-tag verhuisd 211 → 22 op 2026-05-31, IP-subnet ongewijzigd, service uptime onaangetast):**
- 10.21.1.1 — gateway
- 10.21.1.10 — `SRVV-STEPCA-01` (interne CA, native Debian-install — beslist 2026-05-28)

**Mgmt-appliances IP-toekenning (VLAN 35):**
- 10.35.0.1 — gateway
- 10.35.0.2 — **jump-01** (bastion, verhuisd uit eerder veronderstelde 10.34.0.2)
- 10.35.0.x — overige appliances (Semaphore, Forgejo, Wazuh, Bacula Director, NetXMS, Grafana, Loki, Prometheus, MCP, Bacularis), exacte IPs bij VM-provisioning.

**Webapps IP-toekenning (VLAN 36):**
- 10.36.0.1 — gateway
- 10.36.0.40 — SRVV-ODOO-01 (**verhuist** van 10.33.0.40 — aparte werf in [[project-phasing]])
- 10.36.0.41 — SRVV-ID-01 (identity, nieuw)
- 10.36.0.42 — SRVV-ACC-01 (account-admin, nieuw)

**Test-webapps IP-toekenning (VLAN 207):**
- 10.200.14.1 — gateway
- 10.200.14.41 — SRVV-TST-ODOO-02 (**verhuist** van 10.200.0.41 bij Caddy-Quadlet-deploy)
- 10.200.14.x — toekomstige test-webapp-VMs (id-test-VM, acc-test-VM bij splitsing van TST-ODOO-01)

**TST-WEBAPPS-VLAN aangemaakt (2026-05-31):**
Tier-discipline-gap (2026-05-29) opgelost: **VLAN 207 `VL-TST-WEBAPPS` `10.200.14.0/23`** ingericht als test-equivalent van VLAN 36 `VL-SD-WEBAPPS`. Plaats in zone "Services" (samen met prod-Webapps; intra-zone deny houdt scheiding). Firewall-rules T-003 t/m T-007 in firewall-rules-matrix gespiegeld op prod-Webapps-pad. **TST-ODOO-02 verhuist** van `10.200.0.41` (VL-TST-SERVICE) naar `10.200.14.41` (VL-TST-WEBAPPS) bij Caddy-Quadlet-deploy. **TST-ODOO-01 blijft** in VL-TST-SERVICE tot prod-Odoo VLAN 33→36 shift — dan samen mee in dezelfde maintenance. Lopende Podman-experimenten op TST-ODOO-02 pragmatisch in 10.200.0.41 voortgezet tot verhuis-window.

**Historische context (verlaten):**
- 2026-05-18: eerste planning had haproxy/step-ca in 10.20.100.x (oude DMZ). Vervangen op 2026-05-21.
- 2026-05-21 ochtend: DMZ-tags waren 21/22 (in eerste DMZ-supernet-keuze). Bij volledige VLAN-review tags verhoogd naar 210/211 om collision met on-prem 1-99-range te voorkomen.
- 2026-05-31: DMZ-tags teruggebracht naar 21/22 (HAProxy → 21, step-ca → 22) na constatering dat 200-209 puur voor test-replicas bedoeld was en blok 21-29 ruim genoeg vrij is. Tag-conventie: VLAN-ID = `21 + 3e-octet van /24` (mnemonic: `10.21.0.x → VLAN 21`, `10.21.1.x → VLAN 22`). Operationele verhuis: UniFi-network-retag + Proxmox VM-NIC-tag flip — step-ca had ~5s NIC-bounce zonder consumer-impact (HAProxy nog niet gedeployed, geen ACME-clients). Connectiviteit post-shift bevestigd (ICMP+SSH OK, step-ca:8443 health-OK).
- 2026-05-21 ochtend: prod-Odoo zat in 10.33.0.0/24. Bij tier-review verhuisd naar 10.36.0.0/24 (WEBAPPS), VLAN 33 blijft enkel voor AD/print/file.

**Odoo-instances:** Rol = `app` / `identity` / `account-admin`; expose = `public` / `public-restricted` / `internal`. Zie [[project-identity-architecture]].

| Hostname | VMID | IP | Poort | FQDN | Env | Rol | Expose |
|---|---|---|---|---|---|---|---|
| SRVV-ODOO-01 | 220 | 10.33.0.40 (verhuist → 10.36.0.40) | 8069 | myschool.olvp.be | prod | app | public |
| SRVV-ODOO-01 | 220 | 10.33.0.40 (verhuist → 10.36.0.40) | 8070 | myschool-ict.olvp.be | prod | app (+MCP) | public |
| **SRVV-ID-01** (nieuw) | TBD | **10.36.0.41/24** | 8069 | id.olvp.be | prod | identity | public-restricted |
| **SRVV-ACC-01** (nieuw) | TBD | **10.36.0.42/24** | 8069 | myschool-acc.olvp.int | prod | account-admin | internal |
| **SRVV-ACC-01** (nieuw) | TBD | **10.36.0.42/24** | 8070 | myschool-acc-test.olvp.int | test | account-admin | internal |
| SRVV-TST-ODOO-01 | 10083 | 10.200.0.40/23 | 8069 | myschool-test.olvp.be | test | app | public |
| SRVV-TST-ODOO-01 | 10083 | 10.200.0.40/23 | 8070 | id-test.olvp.be | test | identity | public-restricted |
| SRVV-TST-ODOO-01 | 10083 | 10.200.0.40/23 | 8071 | myschool-dev.olvp.be | test | app | public |
| SRVV-TST-ODOO-02 | 10084 | 10.200.0.41/23 | 8069 | vrij | test | – | – |
| SRVV-TST-ODOO-02 | 10084 | 10.200.0.41/23 | 8070 | vrij | test | – | – |
| SRVV-TST-ODOO-02 | 10084 | 10.200.0.41/23 | 8071 | vrij | test | – | – |

**Architectuurpunten:**
- Meerdere Odoo-instances draaien als **aparte containers** (nu Docker, migreert naar Podman — zie [[project-containerization]] en ADR 0004) op verschillende poorten op dezelfde VM. HAProxy routeert naar Caddy:443 op de VM (één backend per VM), Caddy doet Host-header-routering naar de juiste Odoo-poort lokaal. Nieuwe Odoo op bestaande VM = alleen Caddy-edit + nieuwe container, geen HAProxy-wijziging.
- **`expose: internal` instances staan niet in HAProxy-config**: account-admin (`myschool-acc.olvp.int`:8069 prod én `myschool-acc-test.olvp.int`:8070 test, beide op SRVV-ACC-01) zijn enkel via interne DNS (`.olvp.int`) bereikbaar, met step-ca cert in Caddy. Geen publieke DNS-record, niet in fase 2 naar Cloudflare. De Caddy-automation (`caddy.yml`) rendert sinds 2026-06-02 ook `internal`-instances (var `caddy_instances` = alles behalve `direct_backend`); HAProxy-filtering op public/public-restricted blijft los.
- **`expose: public-restricted`** krijgt in HAProxy een path-allowlist (alleen `/web/login`, OAuth-paden, eventueel `/web/session/`); rest = 403.
- **Beheerders (VLAN 34)** komen direct bij VLAN 33/35/36 en gebruikers-VLANs. Naar DMZ (21/22) **uitsluitend via jump-01** (10.35.0.2).

**Naamconventie:**
- Prod-FQDN: `<naam>.olvp.be`
- Test-FQDN: `<naam>-test.olvp.be` of `<naam>-dev.olvp.be` (er bestaat al beide patronen)
- VM-hostnaam: `SRVV-<naam>` voor productie, `SRVV-TST-<naam>` voor test

**How to apply:** Verwijs naar deze tabel voor concrete waarden bij Ansible-vars, HAProxy backends, firewall-regels. Update bij wijzigingen. Bij toevoeging van nieuwe instance: zowel deze tabel als de Ansible `instances.yml` bijwerken. Voor de volledige VLAN-context (alle 30+ VLANs incl. on-prem tier-scheiding en BAWA): zie `vlans.md` in platform-handbook.
