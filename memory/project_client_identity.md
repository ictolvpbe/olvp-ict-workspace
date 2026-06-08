---
name: project-client-identity
description: Forensische traceerbaarheid leerling-internet-activiteit + 802.1X/RADIUS-architectuur. Huidige status (NPS+dynamic-VLAN werkt al) en tier-ladder voor verbetering. FreeRADIUS als doelarchitectuur.
metadata:
  type: project
---

OLVP-netwerk-identity-context (vastgelegd 2026-05-21):

**Huidige situatie (werkt):**
- **802.1X + dynamic VLAN-assignment** is al actief over alle leerling/personeel-VLANs via een Windows NPS-server (LDAP-bind naar AD).
- Leerlingen: managed **Chromebooks** (Google Admin Console). Geen betekenisvolle hostname, wél MAC ↔ device ↔ assignedUser via GAC Directory API.
- Computerklassen + administratie: **Windows AD-joined**.
- Leerkrachten: **Windows Intune-joined**.
- ICT plant migratie personeel-laptops naar **Linux** op middellange termijn (MDM-keuze nog open).
- **NXfilter** doet DNS-filtering op alle leerling-VLANs. DNS-query-logs bewaard 30 dagen.
- DHCP-server is gemixt: UniFi-gateway voor sommige VLANs, AD-DC voor andere.

**Het gap dat we vastleggen:**
NXfilter-logs (IP+MAC+query+timestamp) en NPS RADIUS-accounting-logs (user+IP+MAC+session) zijn er beide, maar **niet automatisch gecorreleerd**. Bij incidenten = handmatig twee logs kruisen op timestamp+IP. Officieel beleidsdocument over internet-activiteit-logging (doel/retentie/toegang/transparantie) ontbreekt nog.

**Doelarchitectuur (vastgelegd, niet kritisch pad):**
- **FreeRADIUS** in VLAN 35 als vervanger voor Windows NPS. Reden: config-in-Ansible, open-source, betere Loki-integratie, past in Linux-migratie. Geen deadline.
- **Management approach voor FreeRADIUS**: in eerste instantie **Ansible + Grafana** (config-in-Git + Loki/Grafana voor accounting + dashboards). Geen aparte UI; AD blijft SoT dus geen parallelle user-store nodig.
- **Tier-ladder voor mitigatie van het correlatie-gap**:
  - Tier 1: DHCP-leases (UniFi + AD-DC) naar Loki
  - Tier 1.5: NPS RADIUS-accounting naar Loki — **primair forensisch instrument**
  - Tier 2: GAC API → asset-register (Project H, ITSM in Odoo)
  - Tier 3: FreeRADIUS-migratie
  - Tier 4: forensisch Grafana-dashboard "IP+tijd → user"
- Tier 1.5 + 4 hangen af van Loki-deploy (Fase 3 monitoring).

**WPA3-werf (latere fase, geen deadline):**
- WPA3-Enterprise duwt EAP-TLS naar voren (in 192-bit Suite B mode verplicht). Migratie van huidige PEAP-MSCHAPv2 naar EAP-TLS = grote operationele werf: client-cert per device.
- Cert-provisioning is per OS verschillend en NIET PacketFence's domein:
  - Windows AD: AD-CS + GPO auto-enrollment
  - Windows Intune: Intune SCEP/NDES
  - Chromebooks: Google Admin Console
  - Linux: step-ca + Ansible / MDM-SCEP (MDM-keuze nog open)
- **PacketFence wordt op dat moment toegevoegd** als NAC-orchestrator (niet als cert-uitgever): centrale cert-revocation, cross-MDM device-inventory, posture-checks, optionele BYOD-onboarding-flow als beleid ooit verandert.
- Te plannen na: FreeRADIUS-migratie (Tier 3) + step-ca operationeel.

**Why:** AD is al SoT ([[project-identity-architecture]]). RADIUS-accounting bestaat al — we benutten enkel de data nog niet. Volgorde van tiers volgt cost/benefit, niet technische blokkade.

**How to apply:** Bij elke vraag rond "wie heeft wat bezocht / waar zit user X" verwijs naar `platform-handbook/network-physical/network/client-identity.md` voor de tier-ladder. Bij FreeRADIUS-implementatie zie `management-tools/freeradius.md`. Bij DPIA/legal-vragen rond leerling-logging zie `governance/dpia.md` + risk R-26.

**BYOD voor leerlingen blijft geweerd** uit student-VLANs (huidig beleid bevestigd) — MAC-randomization-issues + identity-attestation maken het complex. Gast-WiFi (`PD-GUEST` VLAN 40) is voor BYOD het juiste pad.

Gerelateerd: [[project-identity-architecture]] (AD-SoT), [[project-phasing]] (Tier 1.5 + 4 in Fase 3), [[project-mcp-and-itsm]] (Project H = ITSM/asset-register), [[project-security-layers]] (NXfilter is bestaand maar valt onder security-laag).
