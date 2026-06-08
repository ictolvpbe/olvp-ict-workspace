---
name: feedback-test-from-user-vlan
description: Bij OLVP zone-firewall (intra-zone-deny + per-VLAN-isolation) ALTIJD acceptance-testen vanaf een gebruikers-VLAN (10/etc), niet alleen vanaf Admin VLAN 34. Anders mis je dat de cross-zone-allow ontbreekt → users zien time-out terwijl admins denken dat alles werkt.
metadata:
  type: feedback
---

**Regel**: Acceptance-test voor publiek-bereikbare diensten (Odoo, HAProxy-backends, etc.) moet **minstens één keer vanaf VLAN 10 (USERS-zone)** of het meest-restrictieve user-VLAN gebeuren, niet enkel vanaf VLAN 34 (Admin). Admins hebben standaard breder firewall-toegang en zien niet wat gewone users zien.

**Why**: Geconstateerd 2026-06-02. Caddy no-cache mitigation voor stale-CSRF was groen vanaf VLAN 34, maar VLAN 10 (USERS) kreeg time-out: TCP-connect naar VIP `10.21.0.5:443` was unreachable. Diagnose-pad: hairpin-NAT uitsluiten (curl --resolve naar VIP faalde ook → niet hairpin) → zone-policy ontbrak. UniFi-rule toegevoegd: `USERS-zone (VLAN 10) → DMZ-RPROXY (VLAN 21)` voor TCP 80,443 dst `10.21.0.5/32`, met "Retourverkeer Automatisch Toestaan" ☑. Daarna werkt VLAN 10 ook.

**How to apply**:
- Na elke nieuwe publiek-bereikbare deploy: één manuele test vanaf VLAN 10 client (laptop/PC fysiek op userport, of via UniFi-test-client als die er is)
- Bij elk nieuw user-VLAN dat klanten moet zien (Wifi-personeel, Wifi-studenten, etc.): zelfde firewall-rule herhalen, of beter een **Network Group `INTERNAL-USERS`** maken met alle user-VLANs en daar één policy op zetten
- Bij intermitterende problemen: vraag óók waar de testers fysiek zitten (VLAN-context kan verklaring zijn)

**Open follow-ups na 2026-06-02 fix**:
- Wifi-VLANs (personeel, studenten) hebben mogelijk dezelfde gap → checken vóór gebruik live gaat
- Network Group `INTERNAL-USERS` overwegen voor onderhoudbaarheid
- Split-horizon DNS niet gedaan; hairpin via UniFi werkt voorlopig (geen performance-issue gerapporteerd)

Gerelateerd: [[project-firewall-strategy]] (zone-strategie), [[feedback-unifi-new-vlan-zone]] (zone-assignment + Retourverkeer-gotcha), [[project-hosting-fase1-status]] (firewall-rules-matrix TODO).
