---
name: feedback-lege-output-is-geen-meting
description: "Een geneste SSH-loop met meerlaagse quoting faalt stil en levert lege output. Lees 'geen treffers' nooit als 'niets gevonden' zonder een positieve controle die bewijst dat de meting liep."
metadata:
  type: feedback
---

Na de watchdog-reset van 2026-09-24 ([[project-pve-watchdog-incident-202609]]) scande ik alle
drie de Proxmox-nodes op VM's die `onboot: 1` hebben maar niet draaien. De uitkomst was één
treffer, en ik concludeerde: alles is teruggekomen op één VM na.

**Fout.** Er stonden er op dat moment nog negentien stil, waaronder productie
(`SRVV-ODOO-01`) en `SRVV-NETXMS-01` — die laatste heeft wél `onboot: 1` en had dus in mijn
lijst moeten staan.

De scan was een `for`-loop over de nodes met daarin een geneste `ssh` en daarin weer een
`for`-loop met `awk` en `grep`. Drie lagen quoting (lokale shell → ssh → remote bash → awk),
elk met eigen escaping. In diezelfde beurt gaf `uptime` voor `p01` óók niets terug, en dát had
het signaal moeten zijn: een node die antwoordt op `ssh` maar geen uptime teruggeeft, meet niet.

**Why:** een mislukte meting en een geslaagde meting zonder treffers zien er identiek uit — beide
geven lege output. Bij een zoekopdracht naar problemen is leeg juist het antwoord dat je *wíl*
zien, dus de bevestigingsdrang doet de rest. Dit is dezelfde klasse als de 49 agents die stil een
404 kregen en de mailgate die stil niet liep: **de afwezigheid van een signaal is zelf een
signaal**, ook als je het zelf veroorzaakt hebt.

**How to apply:**
- Bouw in elke scan een **positieve controle**: laat hem ook iets rapporteren wat er zeker ís.
  Tel de onderzochte objecten en toon dat aantal. `0 VM's gecontroleerd op p01` valt op,
  `geen treffers` niet.
- Draai een meting over meerdere hosts liefst **per host apart** in plaats van in één geneste
  loop. Trager, maar je ziet per host of hij antwoordde.
- Vermijd drie lagen quoting. Zet het script in een bestand en pipe het naar `ssh 'bash -s'`,
  of gebruik base64 zoals bij [[feedback-tier1-kloon-procedure]].
- Bij Proxmox specifiek: `pvesh get /cluster/resources --type vm` geeft de **hele cluster** in
  één aanroep, zonder per-node-ssh. Dat was hier het juiste gereedschap en het scheelt de hele
  foutbron. Let op de derde categorie die geen enkele status-scan vindt: een VM **zonder
  `onboot` én zonder HA** komt na een reset nooit terug. Daarvoor heb je een lijst nodig van wat
  er hóórt te draaien, niet een scan van wat er staat.

Verwant: [[feedback-ansible-check-mode-verify]] (vertrouw de PLAY RECAP niet),
[[feedback-toon-het-echte-commando]] (inspecteer de echte argv vóór je hypothesen bedenkt).
