---
name: feedback-unifi-new-vlan-zone
description: Bij elke nieuwe VLAN in UniFi expliciet de Network Zone toewijzen — anders valt het VLAN in een default-zone, en zone-paar-policies matchen het verkeerde paar. Trigger: nieuw VLAN aanmaken in UniFi.
metadata:
  type: feedback
---

**Regel:** na elke nieuwe VLAN-aanmaak in UniFi (Settings → Networks): controleer het veld **Network Zone** / **Zone Assignment** en wijs expliciet de juiste zone toe. Niet doen = VLAN valt in een default-bucket en geen enkele zone-paar-policy matched.

**Why:** 2026-05-31 — VLAN 207 (`VL-TST-WEBAPPS`) werd aangemaakt en TST-ODOO-02 daarheen verhuisd. Volgens `firewall-zones.md` hoort dit in zone **Services**. UniFi had het VLAN echter automatisch in zone **DMZ** geplaatst. Gevolg: HAProxy (DMZ) → TST-ODOO-02 (DMZ) werd geblokkeerd door intra-DMZ default-deny, ook al hadden we expliciet een Accept-rule aangemaakt. Symptoom: alleen VPN-pool kon erbij (ruime VPN→DMZ-rule), interne mgmt-flows (jump-01 → SSH) en DMZ→backend (HAProxy → Odoo:8069) waren dicht.

**How to apply:**
- Bij elke `Create new Network/VLAN` in UniFi: open de advanced settings en kies de Network Zone die overeenkomt met onze [firewall-zones.md](platform-handbook/network-physical/network/firewall-zones.md).
- Achteraf gemiste zone-toewijzing herkennen aan deze pattern: rule aangemaakt + Apply gedaan + nog steeds default-deny ergens, OF "alleen VPN werkt, niets anders". Dan eerst zone checken vóór rule-details debuggen.
- Documenteer in `[[project-firewall-strategy]]` als pre-checklist bij VLAN-creatie.

Gerelateerd: [[project-firewall-strategy]] (zone-bundeling rationale), [[project-infrastructure-params]] (VLAN-toekenning).
