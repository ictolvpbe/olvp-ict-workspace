---
name: project-naming-convention
description: Hostname-naming-conventie OLVP. PREFIX-ROL-NN-formaat (SRV/SRVV/SW/WAP/CAM/PR/TEL). DNS heeft canonical A-record + optionele korte CNAME-alias. Mgmt-tools geconsolideerd één-VM-per-logische-stack i.p.v. één-IP-per-container.
metadata:
  type: project
---

OLVP hostname- en DNS-conventie (vastgelegd 2026-05-26):

**Formaat**: `<TYPE-PREFIX>-<ROL>-<NN>`, hoofdletters + hyphens. Voorbeelden: `SRVV-HAPROXY-01`, `SRVV-ODOO-01`, `SW-CORE-02`, `WAP-LOKAAL-A1-03`.

**Type-prefixen**:
- `SRV` = fysieke server (bare-metal)
- `SRVV` = Virtuele server (VM op Proxmox)
- `SW` = switch
- `WAP` = Wireless Access Point
- `CAM` = IP-camera
- `PR` = printer
- `TEL` = IP-telefoon
- `NAS` = Netwerk-storage
- `FW` = firewall/gateway
- `LPT` = laptop (user-device, sinds 2026-05-26)
- `PC` = desktop (user-device, sinds 2026-05-26)

User-devices (LPT/PC) gebruiken `<ROL>` op drie manieren: user-naam (`LPT-DRIES-01`), functie (`PC-ADMIN-01`), of locatie (`PC-LOK101-05`). Per device-type kiezen.

Volledige conventie + DNS-strategie: `platform-handbook/shared/naming.md`.

**DNS-strategie — dubbele A-records (canonical + alias):**
- Per host **twee A-records** met hetzelfde IP:
  - Canonical: `SRVV-HAPROXY-01.olvp.int → IP`
  - Alias: `haproxy-1.olvp.int → IP` (zelfde IP, geen CNAME)
- Bewuste keuze 2026-05-26: CNAMEs werken niet betrouwbaar in UniFi-DNS-resolver (NXDOMAIN bij lookup van CNAME binnen eigen zone). Dubbele A-records overal hetzelfde, kleine trade-off bij IP-wijziging (twee records updaten).
- Canonical naam dominant in logs/audit-trail (uit `hostnamectl`-output)
- Korte alias = handig voor dagelijks beheer (SSH, browser-bookmarks)
- In Ansible-inventory: canonical naam als `inventory_hostname`, `ansible_host=<IP>` of `<canonical-fqdn>`
- Bij IP-wijziging: **beide A-records updaten** + eventuele extra aliassen (container-FQDNs op zelfde host)

**Mgmt-tier consolidatie (zelfde datum):**
Eerder voorgesteld één-IP-per-container voor monitoring (Grafana 10.35.0.21, Prometheus 10.35.0.22, Loki 10.35.0.23) — over-engineered voor onze schaal. Nu één-VM-per-logische-stack:
- `SRVV-MONITORING-01` (10.35.0.20): Grafana + Prometheus + Loki + Promtail + NetXMS in containers
- `SRVV-WAZUH-01` (10.35.0.50): manager + indexer + dashboard samen
- `SRVV-BACULA-01` (10.35.0.60): Director + SD + Bacularis samen

Container-specifieke DNS (`grafana.olvp.int`, `prometheus.olvp.int`, `loki.olvp.int`) wijst naar dezelfde host-IP; reverse-proxy (Caddy) op de VM routeert per host-header.

**Why:**
- Bus-factor: nieuwe collega ziet meteen type uit hostname
- Logging-leesbaarheid: canonical naam in audit-trail
- Schaalbaarheid: consolidatie verlaagt VM-management-overhead voor schoolschaal
- Lower latency: Grafana/Prometheus/Loki op zelfde host = container-network i.p.v. WAN

**How to apply:**
- Nieuwe VM: hostname volgt `SRVV-<rol>-<nn>` direct bij provisioning
- Bestaande VMs zonder conventie (alleen `jump-01` nog): `hostnamectl set-hostname SRVV-JUMP-01` + `/etc/hosts` update
- Nieuwe DNS-record: canonical A-record verplicht, korte CNAME-alias optioneel (alleen toevoegen als waarde toevoegt)
- In Ansible-inventory en docs: canonical als primary, korte alias secundair
- Bij stack-consolidatie (Grafana+Prometheus+Loki samen): containers met Caddy host-routing op één VM

**Status bestaande VMs (2026-05-26):**
- `SRVV-ODOO-01`, `SRVV-TST-ODOO-01/02`, `SRVV-ANSIBLE-01`, `SRVV-HAPROXY-01/02`, `SRVV-JUMP-01` ✓ Conform
- DNS-records voor canonical + aliases aangemaakt in olvp.int
- Toekomstige: `SRVV-STEPCA-01`, `SRVV-MONITORING-01`, `SRVV-WAZUH-01`, `SRVV-BACULA-01`, `SRVV-FORGEJO-01/02`, `SRVV-MCP-01`, `SRVV-ID-01`, `SRVV-ACC-01`

Gerelateerd: [[project-infrastructure-params]] (VM-IPs), [[reference-dns-olvpbe]] (olvp.int + olvp.be + olvp.test), [[project-bus-factor]] (waarom canonical naming).
