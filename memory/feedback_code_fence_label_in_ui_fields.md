---
name: feedback-code-fence-label-in-ui-fields
description: Zet geen taallabel op code-blokken waarvan de inhoud in een UI-veld geplakt moet worden — het label wordt mee gekopieerd en breekt het script stil.
metadata:
  type: feedback
---

Code-blokken waarvan de inhoud **rechtstreeks in een invoerveld** geplakt wordt (NXSL-filters in NetXMS, Survey-vars in Semaphore, transform-expressies in Grafana, server-side scripts in Odoo) krijgen **geen taallabel** achter de openingsfence. In een terminal is het taallabel gewoon zichtbare tekst en wordt het mee geselecteerd.

Concreet voorval 2026-08-26: een autobind-filter in NetXMS aangeleverd als een blok met `nxsl` als taallabel. Dat woord belandde als eerste regel in het filterveld van twee containers. Resultaat: `Error in line 2: syntax error` bij elke autobind-poll, containers bleven leeg, geen enkele melding in de UI. Diagnose kostte een sessie; de fix was één regel wissen.

**Why:**
- Scriptvelden in beheer-UI's geven zelden zichtbare compileerfouten — het falen is stil en ziet eruit als "de regel klopt niet" i.p.v. "het script is stuk".
- De gebruiker kopieert wat hij ziet. Alles in het blok is dus een instructie, ook het label.
- Zelfde familie als [[feedback-terminal-paste-long-lines]]: wat in een terminal gerenderd wordt, is niet wat in een browser gerenderd wordt.

**How to apply:**
- Blok bedoeld om te *plakken in een veld* → kale fence, geen taal.
- Blok bedoeld om te *lezen of in een bestand te zetten* → taallabel mag.
- Zeg er expliciet bij waar de eerste regel begint wanneer het veld al inhoud heeft ("volledig vervangen", niet "toevoegen").
- Bij "het script doet niets": eerst het *opgeslagen* script uit de database of het log terughalen, niet het script dat je dénkt te hebben aangeleverd.

Gerelateerd: [[project-netxms-monitoring]] (waar dit gebeurde), [[feedback-yaml-colon-space]] (zelfde reflex: toon fout-vs-correct expliciet).
