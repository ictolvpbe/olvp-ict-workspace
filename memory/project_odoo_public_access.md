---
name: project-odoo-public-access
description: "Project om Odoo MySchool-instances publiek toegankelijk te maken via HAProxy HA-stack in DMZ, met dual-ISP redundantie. Gestart mei 2026, gefaseerd over schooljaar + zomervakantie 2026."
metadata: 
  node_type: memory
  type: project
  originSessionId: 59a419f3-43f1-477b-bb2b-196d0c16926a
---

Multi-fase project gestart 2026-05-18: publieke toegankelijkheid opzetten voor Odoo MySchool-instances die nu enkel intern bereikbaar zijn.

**Why:** Odoo-instances draaien intern op test- en productie-VLAN's; gebruikers (leerkrachten, leerlingen, ouders, administratie) moeten ze ook van buiten het schoolnetwerk kunnen bereiken. UniFi Gateway Pro heeft 2 ISP's beschikbaar — beide moeten benut worden voor redundantie. Toekomstgericht: Enterprise Fortress upgrade gepland, mogelijk meer applicaties dan enkel Odoo achter dezelfde stack.

**How to apply:** Bij vragen over dit project altijd uitgaan van de gefaseerde aanpak (zie [[project-phasing]]) en de gekozen architectuur (zie [[project-architecture-haproxy]]). Tot juli 2026: single-ISP publicatie via one.com DNS + HTTP-01. Daarna: Cloudflare-migratie en dual-ISP failover.

Resterende openstaande info-vragen (per 2026-05-18):
- 3 nog-te-verifiëren netwerkdetails (zie [[project-infrastructure-params]]): test-VLAN range, mask SRVV-ODOO-01, DMZ-VLAN structuur
- Volledige inventaris van one.com DNS-records voor Fase 2 (mail-records, baple.olvp.be, andere subdomeinen) — pas relevant in juli 2026
- Wat draait er op baple.olvp.be (relevant voor grijs/oranje-wolk-keuze in Fase 2)
- Bevestiging dat myschool-ict.olvp.be eveneens publiek bereikbaar moet zijn (zit op prod-VM)

Reeds bevestigd (zie [[project-infrastructure-params]]): hypervisor (Proxmox, user provisiont zelf), VM-inventaris (3 hosts, 4 actieve FQDNs + 4 vrije slots), DMZ IP-toekenning, management-subnet 10.34.0.0/27 met jump-host op .2.
