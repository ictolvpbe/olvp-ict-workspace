---
name: project-phasing
description: "Vier-fasen aanpak voor [[project-odoo-public-access]]. Fase 1 (mei-juni 2026): build + 1b security prio1. Fase 2 (zomer 2026): DNS-migratie + failover. Fase 3 (najaar 2026): security prio2. Fase 4 (2027+): security prio3."
metadata: 
  node_type: memory
  type: project
  originSessionId: 59a419f3-43f1-477b-bb2b-196d0c16926a
---

Project [[project-odoo-public-access]] is bewust gesplitst in vier fasen met verschillende risicoprofielen en investeringsniveaus (beslist 2026-05-18, zie ook [[feedback-risk-aware-changes]] en [[project-security-layers]]).

**Why:** User wil mail/DNS-migratie van olvp.be niet tijdens schooljaar uitvoeren wegens onaanvaardbare mail-uitval-impact. Splitsen ontkoppelt risico's. Security-hardening is opgedeeld in een prio1-onderdeel dat **mee in eerste go-live** moet, en latere lagen die na stabilisatie aangepakt worden.

**Fase 1 — start mei 2026 (Build):**
- DMZ-netwerk (supernet 10.21.0.0/16, VLAN 21 = 10.21.0.0/24 voor HAProxy, VLAN 22 = 10.21.1.0/24 voor step-ca), HAProxy HA-paar, step-ca, Caddy-reconfig op Odoo-servers, UniFi DNAT op WAN1, monitoring, instances achter HAProxy.
- **Aparte werf in Fase 1**: prod-Odoo SRVV-ODOO-01 verhuizen van 10.33.0.40 (VLAN 33 SD-SERVICE) naar 10.36.0.40 (VLAN 36 SD-WEBAPPS). Tier-scheiding tussen schoolservices (AD/print/file) en publieke web-apps. Te plannen tijdens schoolvakantie (downtime + Caddy + interne DNS).
- DNS blijft bij one.com — per Odoo-instance één A-record handmatig, wijzend naar WAN1-IP.
- ACME via HTTP-01 (geen API nodig bij one.com); per-host certs.
- Single-ISP inbound, dual-ISP outbound.
- Geaccepteerd risico: bij ISP1-uitval onbereikbaar tot ISP1 herstelt.

**Fase 1b — geïntegreerd in fase 1, vóór eerste publieke gebruiker (Security hardening prio 1):**
- Egress-filtering op prod- en test-VLAN (UniFi zone-based firewall, default-deny outbound + allowlist legitieme destinations).
- 2FA verplicht op Odoo staff-accounts; admin-accounts gescheiden van dagelijks gebruik.
- Admin-endpoints (`/web/database/...`) afgeschermd op HAProxy + `list_db=False` en `dbfilter` in odoo.conf.
- Automatische OS-security-updates (`unattended-upgrades`) op alle stack-VM's.
- Backups encrypted, off-site, **ransomware-resistent** (pull-based of immutable), maandelijkse restore-test.

**Fase 2 — zomervakantie 2026 (DNS + failover):**
- DNS-zone olvp.be migreren one.com → Cloudflare (mail-records, DKIM, DMARC, eventueel DNSSEC, alle subdomeinen mee — niet enkel Odoo).
- Volledige inventaris one.com-records vooraf; kritieke records (MX/SPF/DKIM/DMARC/CAA) manueel verifiëren.
- TTL's 24u vooraf naar 300s.
- ACME omschakelen van HTTP-01 naar DNS-01 met Cloudflare API (wildcard mogelijk).
- DNAT-rules op WAN2 activeren in UniFi.
- Cloudflare Free + Load Balancer add-on ($5/maand): beide WAN-IPs als origins, health checks.
- Per Odoo-record DNS overzetten naar Cloudflare LB-hostname (oranje wolk); mail/baple/andere blijven grijs.

**Fase 3 — najaar 2026 (Security hardening prio 2):**
- Inter-instance isolatie binnen prod-VLAN (client isolation of micro-segmentatie).
- Management-plane scheiden + jump-host met 2FA.
- PostgreSQL-architectuur evalueren (aparte DB-VM's).
- Centrale logging (Loki/Grafana/Promtail) + alerting-regels (failed logins, 5xx-spikes, cert-expiry, egress-events).
- HAProxy rate-limiting (stick tables) op login-paths.
- GeoIP-blokkering — afhankelijk van gebruikersnoden.

**Fase 4 — 2027+ (Security hardening prio 3):**
- WAF activeren: Coraza op HAProxy met OWASP Core Rule Set (voorkeur), of Cloudflare Pro Managed Rules.
- UniFi IDS/IPS in detection → selectief prevention.
- Disk-encryption (LUKS) op data-VM's evalueren.
- Externe pen-test / vulnerability scan.
- Security Incident Response Plan + jaarlijkse tabletop oefening.

**How to apply:** Bij ieder voorstel checken in welke fase het hoort en niet vooruitlopen. Fase 1b is **niet optioneel**: geen publieke gebruiker mag op het systeem voor 1b is afgevinkt. Fase 2 mag pas starten als Fase 1 + 1b operationeel en stabiel zijn. Fase 3 en 4 zijn opvolgcommitments — bij elke nieuwe sessie checken of er aan iets uit deze fasen gewerkt moet worden.
