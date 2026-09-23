---
name: feedback-ontwerp-runbook-niet-geverifieerd
description: "Een runbook met status 'ontwerp' bevat concrete artefacten — image-tags, IP's, paden — die nog nooit zijn aangeraakt. De Euro-Office-tag 9.3.1 uit RB-2026-NC-EO-DEPLOY bestond niet. Trek bij de uitvoering élk genoemd artefact één keer echt aan vóór je erop bouwt."
metadata:
  type: feedback
---

**Vastgesteld 2026-09-23** bij het uitvoeren van RB-2026-NC-EO-DEPLOY ([[project-eurooffice-nextcloud]]).

Het runbook stond op status *ontwerp* en noemde
`ghcr.io/euro-office/documentserver:9.3.1` — met versienummer, poort, geheugeneis en healthcheck
erbij, dus met alle uiterlijke tekenen van een geverifieerd feit. **Die tag bestaat niet.**
`podman pull` gaf `manifest unknown`. Het ghcr-repo draagt uitsluitend CI-build-artefacten
(`build-document-server-custom-<sha>`) plus `latest`, en géén enkele semver-tag. Het versienummer
9.3.1 kwam uit een blogpost, niet uit het register.

Hetzelfde patroon in hetzelfde document, drie keer:

- **IP's** `10.200.14.45/.46` stonden er met "⚠️ te bevestigen" — dat was eerlijk, en ze klopten.
- **Het WAN-IP** stond als open vraag ("welk IP draagt de DNAT vandaag?") — ook eerlijk, en de
  meting gaf een antwoord dat het werk kleiner maakte: de DNAT staat per **poort**, niet per FQDN,
  dus er was helemaal geen gateway-wijziging nodig. Ook de firewall bleek al open.
- **De architectuur van spoor B** was op één punt gewoon mis: de `collaboration`-dienst van
  OpenCloud draait *in* het hoofdproces, niet als losse container ernaast.

**Why:** bij het schrijven van een ontwerp komen feiten uit documentatie, blogposts en redenering.
Dat is legitiem — je kan niet alles bouwen om het te kunnen plannen. Maar het verschil tussen
"gelezen" en "aangeraakt" verdwijnt in de opmaak: een tabelrij met een versienummer ziet er precies
zo uit of hij nu uit een `podman pull` komt of uit een persbericht.

**How to apply:**

- Bij de **eerste uitvoering** van een ontwerp-runbook: trek elk genoemd artefact één keer echt aan.
  `podman pull` de image, `dig` de naam, `ping` het adres, open de poort. Dat is een half uur en het
  verplaatst de ontdekking van "midden in fase 5" naar "voor fase 1".
- **Pin op digest wanneer er geen semver-tags zijn.** `latest` volgen op een server die documenten
  bewerkt betekent dat je editor onder je voeten verandert. Noteer er de pakketversie bij
  (`dpkg -l` in het image), want een digest zegt een mens niets.
- Schrijf een bevinding op als **"niet vast te stellen"** wanneer dat het eerlijke antwoord is. De
  sessielimiet-vraag hier is niet statisch te beantwoorden: geen sleutel in de config, niets in de
  licentiedocumentatie, telmachinerie wél in de binary maar die zit in de versie zónder limiet ook.
  Dat in de wijzigingslog zetten als "gecontroleerd ✅" zou een verkeerd gerust gevoel geven op
  precies het punt waar de productiebeslissing aan hangt.
- Werk het runbook bij **tijdens** de uitvoering, niet erna. De correctie is het waardevolste wat
  een eerste uitvoering oplevert.

Gerelateerd: [[feedback-verify-memory-against-repo]], [[feedback-ansible-check-mode-verify]],
[[feedback-same-subnet-test-proves-nothing]] — alle drie varianten van dezelfde reflex: meet het,
leid het niet af.
