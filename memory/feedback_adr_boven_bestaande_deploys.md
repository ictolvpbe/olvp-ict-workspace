---
name: feedback-adr-boven-bestaande-deploys
description: "Een conventie lees je in de ADR, niet af van bestaande deployments — die zijn vaak precies wat de ADR corrigeert. Controleer ook of een Accepted ADR daadwerkelijk geïmplementeerd is."
metadata:
  type: feedback
---

Bij het opleveren van `SRVV-GLPI-01` (2026-09-24) zocht ik de conventie voor dataplaatsing op door
te kijken waar bestaande rollen hun data zetten. `group_vars` gaf `/opt/keycloak`,
`/opt/melira`, `/opt/odoo-multi-instance` — dus `/opt`, dacht ik, en ik blies `/opt` op tot 20 G.

**Fout.** [[ADR 0008]] (`0008-partitie-baseline-tier1.md`) zegt letterlijk: *"Géén applicatiedata
in `/opt`. Dat is hoe de 63 GB `/opt` op de baseline-VM ontstaan is."* Data hoort in
`/srv/<dienst>`, en de maat komt uit het **schijfprofiel** (`disk_profile: S/M/L` als host-var),
niet uit een handmatige `lvextend`.

Die drie `/opt`-paden zijn **precies de deployments die de ADR corrigeert**. Ik had de patiënt
als voorbeeld genomen.

**Why:** in een repo die groeit is de oudste code het talrijkst, dus "wat doet de rest" wijst
statistisch naar de situatie vóór de laatste beslissing. Een ADR is een breuk met wat ervoor
stond; hem afleiden uit de omgeving werkt dus per definitie niet.

De schade was hier niet cosmetisch: die 20 G liet te weinig vrije ruimte in de VG over om
profiel M (`/var` 40 G, `/srv` 20 G) nog te kunnen toepassen. Mijn "vooruitziende" ingreep
blokkeerde de standaard die hij had moeten volgen.

**How to apply:**
- Raak je maatvoering, paden, netwerkindeling of naamgeving aan, **lees dan eerst
  `hosting/decisions/`**. Die map is klein; hem doorlopen kost minder dan het terugdraaien.
- Wijkt een bestaande deployment af van de ADR, dan is dat een **bevinding**, geen voorbeeld.
- Zet zulke waarden waar het ontwerp ze wil hebben (hier: een variabele in de inventory die een
  role uitvoert), niet met de hand op de host. Handwerk is onzichtbaar voor de volgende run.

**En omgekeerd: een ADR is geen bewijs dat iets bestaat.** Diezelfde dag noteerde ik dat de
herbouw het ontbreken van een firewall zou oplossen "omdat de baseline nftables meebrengt" — op
gezag van [[ADR 0005]]. Gemeten op de verse kloon: `nftables` disabled, ruleset leeg, en de role
`baseline-host-firewall` **bestaat niet** in `platform-ansible`. Die ADR staat sinds 2026-05-31
op *Proposed*. → SEC-8.

Dus: **lees de ADR voor de conventie, en controleer in de code of ze ook gedraaid is.** Let op
de status — *Proposed* betekent besloten noch gebouwd. Zie [[feedback-verify-memory-against-repo]]
voor dezelfde rangorde: de draaiende machine wint van elk document.
