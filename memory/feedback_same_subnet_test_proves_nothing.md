---
name: feedback-same-subnet-test-proves-nothing
description: "Bij 'host onbereikbaar': een test vanaf hetzelfde subnet raakt de routering niet. Test vanuit een andere zone én tegen een tweede host in het doelsubnet om firewall van eindtoestel te scheiden."
metadata:
  node_type: memory
  type: feedback
---

Bij een onbereikbaarheidsklacht: **een test vanaf een host in hetzelfde subnet als het doel bewijst niets over routering of firewall.** Dat verkeer gaat rechtstreeks over de interface zonder gateway-hop. Controleer met `ip route get <doel>` of er werkelijk een gateway in het pad zit vóór je uit zo'n test een conclusie trekt.

**De test die wél scheidt**: ping vanuit de andere zone naar een **tweede host in hetzelfde doelsubnet**.
- Antwoordt die tweede host wél → het inter-VLAN-pad en de firewall zijn in orde, het probleem zit in het **eindtoestel** (meestal een ontbrekende of dode default gateway: het kan alleen zijn eigen subnet beantwoorden).
- Antwoordt die ook niet → dán pas naar zone-policy, regel-volgorde en "Retourverkeer Automatisch Toestaan" kijken ([[project-firewall-strategy]]).

**Why:** Bij de mislukte NAS-backup van 2026-08-31 leek de NAS gezond omdat ze antwoordde vanaf een werkplek in `10.10.0.0/16` — maar dat was L2-verkeer. Diagnose ging daardoor eerst richting een ontbrekende firewall-regel; die werd gemaakt en hielp niet. Pas een ping vanuit VLAN 35 naar de AD-DC in datzelfde `10.10.x`-subnet (die wél antwoordde) wees de NAS zelf aan. Zie [[project-offline-backup-chain]].

**Ook**: een open TCP-poort is nog geen werkende dienst. Verifieer bij CIFS/SMB de echte mount met credentials — dat dekt protocol-onderhandeling en authenticatie, wat een poortcheck niet doet. Analoog aan [[feedback-haproxy-odoo-healthcheck-host]].

**How to apply:** Vraag bij elke "X is onbereikbaar" eerst: vanaf welk subnet is er getest, en zat er een gateway in dat pad? Combineer met [[feedback-test-from-user-vlan]] (test vanaf de plek waar de dienst echt benaderd wordt) en met het scheiden van naam en transport (`curl --resolve`, zie de DNS-les in [[project-netxms-monitoring]]).
