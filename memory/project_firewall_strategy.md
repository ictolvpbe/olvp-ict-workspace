---
name: project-firewall-strategy
description: UniFi-firewall zone-strategie OLVP — 14 zones bewust geconsolideerd voor UI-beheersbaarheid. Intra-zone-deny + per-VLAN-isolation als echte enforcement, niet de zone-grouping zelf. Test in prod-zones bewust voor firewall-rule-validatie.
metadata:
  type: project
---

OLVP UniFi-firewall-strategie (bevestigd 2026-05-24 op basis van werkelijke UniFi-config):

**Zone-aanpak**: 14 zones, bewust **niet** verder uitgesplitst voor UI-beheersbaarheid. Bundeling = puur UI-grouping, **niet** security-grens.

**Echte security-enforcement op twee niveaus:**
1. **Intra-zone-traffic staat bijna overal op deny** (UniFi-instelling per zone). VLANs in zelfde zone praten niet automatisch met elkaar.
2. **Per-VLAN "isolated"-vlag** in UniFi network-definitie → niet-overrulbare firewall-rules. Sterker dan zone-rules. Gebruikt voor pure-quarantine + cluster-only-VLANs (VL-ISOLATION, VL-PROXM-CEPH, VL-PFSYNC).

**Why deze aanpak:**
- 17+ zones (zoals onze eerdere strategie-doc suggereerde) maakt UniFi-UI onhanteerbaar voor 1-2 admins.
- Intra-zone-deny + VLAN-isolation samen geven dezelfde scheiding zonder UI-explosie.
- Aparte zones voor Webapps/Mgmt-appliances/Quarantine zou alleen waarde toevoegen indien intra-zone-policy ooit op allow gezet wordt.

**Hoe te toepassen:**
- Bij nieuwe VLAN: bepaal of het in bestaande zone past op basis van **trust-tier**, niet functie. Allemaal zelfde tier = zelfde zone OK.
- Bij twijfel of intra-zone hierop deny staat: checken in UniFi (Settings → Firewall → Zone-config).
- Voor VLANs die echt geïsoleerd moeten zijn (quarantine, cluster-only): gebruik **per-VLAN-isolation-vlag**, niet aparte zone.

**Test in prod-zones is bewust:**
- VL-TST-SERVICE staat in `Services`-zone naast VL-SD-SERVICE; VL-TST-SD-TEACHER in `Personeel`; etc.
- Reden: test fungeert als firewall-rule-validator — dezelfde regels moeten werken voor test én prod.
- **Voorwaarde**: nooit prod-PII in test-DBs (geen prod-leerlinggegevens in test-snapshots). DPIA-implicatie indien dat zou veranderen.

**Legacy / opruimwerk lopend:**
- **VL-SD-INFRA-ORG** is een mix van admin/infra/service-VLAN-functies. Stap-per-stap opgesplitst naar VL-ADMIN (Admin-zone), VL-INF-* (Infra-zone), VL-SD-SERVICE (Services-zone). Blijft daarna leeg.
- **Default (VLAN 1) + VL-BSP-WEG + VL-SD-VRIJ** zijn opruim-targets — devices te verhuizen, daarna VLANs deactiveren.

**Host-lists in plaats van losse IPs (sinds 2026-05-26):**
Voor rule-management gebruiken we **UniFi object-groups (host-lists)** waar mogelijk. Reden: schaalbaar bij groei (nieuwe host = lijst-edit, niet rule-edit), audit-vriendelijk, bus-factor. Naam-conventie = **rol** (niet VLAN): `MGMT-BASTIONS`, `DMZ-HAPROXY`, `ODOO-VMS`, etc. Lijst pas zinvol bij ≥2 hosts of verwachte groei — solo-hosts direct in rules. Bij host-uitfasering: verwijder uit lijst, **niet** de lijst zelf weggooien (rules verwijzen ernaar). Volledige lijst-inventaris: `firewall-rules-matrix.md` sectie "Host-lists". Concrete lijst-content wordt ingevuld na UniFi-configuratie door gebruiker.

**Gotcha — `Retourverkeer Automatisch Toestaan` (2026-05-28):**
Elke inter-zone ACCEPT-regel in UniFi heeft een checkbox **`Retourverkeer Automatisch Toestaan`** (auto-allow return / conntrack-vink). Staat die uit, dan matcht het inkomende SYN-pakket wél de regel, maar het SYN-ACK terug naar de source matcht niets en wordt door default-deny gedropt. Resultaat: client-side `Connection timed out`; op de destination `tcpdump` ziet wel SYN binnenkomen en `ss -tn` toont kortstondig `SYN-RECV`-state. Symptoom dat hier specifiek naar wijst: ping vanaf destination naar source faalt terwijl source-naar-destination wel een pad heeft. **Bij elke nieuwe inter-zone-regel deze vink expliciet checken.** Vastgelegd na de step-ca-bringup waar regel A-005 (`jump-01 → SRVV-STEPCA-01:22`) zonder vink in productie stond.

**Te volgen items:**
- Intra-zone-policy *bijna overal* deny — exacte status per zone te documenteren. Audit-script later om periodiek te valideren.
- VPN-zone heeft `SRV-WUDM-01` twee keer — UniFi-config-curiositeit, bij gelegenheid opschonen.
- **Firewall-rules-matrix op te vullen**: `platform-handbook/network-physical/network/firewall-rules-matrix.md` heeft de structuur (per categorie: WAN-inbound, Admin-flow, Hosting, Webapp↔Services, Test, Monitoring, Pen-test, BAWA). Elke nieuwe rule documenteren vóór UniFi-activatie + status-veld bijhouden.
- **Review-cyclus rules vóór prod**: elke nieuwe/gewijzigde rule eerst testen tegen test-VLAN-equivalent, pas daarna toepassen op prod. Inplannen als checklist in [[project-security-testing]] Audit-cyclus en als hard-gate Fase 1b.

**Wanneer her-evalueren:**
- Indien intra-zone ooit op allow gezet wordt voor een zone met meerdere trust-tiers (vooral `Services` met AD + Mgmt + Webapps) → dan wel splitsen.
- Bij significante scaling (meer apps, meer admins) — vermoedelijk pas Fase 4+.

Gerelateerd: [[project-security-layers]] (defense-in-depth), [[project-infrastructure-params]] (VLAN-lijst), [[feedback-risk-aware-changes]] (waarom we voorzichtig zijn met grote firewall-revisies). Doc: `platform-handbook/network-physical/network/firewall-zones.md`.
