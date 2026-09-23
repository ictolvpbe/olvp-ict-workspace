---
name: project-ad-replication-bsw001
description: "NET-7 — SRVV-BSW001 was niet over RPC te bevragen (repadmin-fout 58) door een ontbrekende EFG-regel Services→VPN; opgelost en geverifieerd 2026-09-23. Les: een DC die je niet kan uitlezen is een blinde vlek, niet een gezonde DC."
metadata:
  type: project
---

**Opgelost en geverifieerd 2026-09-23** (tracker NET-7). Bewaard om de blinde vlek, niet de fix.

## Wat er misliep

Gevonden op 2026-09-23 terwijl een DNS-zone werd opgeruimd die telkens terugkwam.
`repadmin /replsummary` gaf voor alle DC-paren **0 mislukkingen** met 8–9 minuten vertraging — dus
op het eerste zicht een gezonde replicatie — maar sloot af met:

```
Experienced the following operational errors trying to retrieve replication information:
   58 - SRVV-BSW001.olvp.int
```

Fout 58 = de DC was **niet over RPC te bevragen**. Gevolg: `SRVV-BSW001` (remote campus,
`192.168.1.10`) stond wél in de **bron**-tabel — anderen halen van hem op — maar ontbrak in de
**bestemmings**-tabel. Over zijn eigen *inkomende* replicatie was dus níets bekend, terwijl hij
clients op die campus bedient.

**Oorzaak**: op de EFG ontbrak een regel **Services (`10.33.0.10`) → VPN (`192.168.1.10`)** met
retourverkeer toegestaan. Toegevoegd 2026-09-23; daarna is fout 58 weg en verschijnt BSW001 ook als
bestemming. Het dynamische RPC-bereik 49152–65535 hoefde niet apart opengezet te worden.

## ⚠️ De les — waarom dit maandenlang onzichtbaar bleef

**DNS over UDP 53 kwam er wél door, RPC niet.** Elke `dig` tegen die DC werkte dus gewoon, en elke
controle die op naamresolutie steunde zei "in orde". Alleen het protocol dat je nodig hebt om de
*status* op te vragen, was geblokkeerd. Een halfopen pad ziet er van buitenaf uit als een open pad.

Twee gevolgen die blijven gelden:

1. **Een DC waarvan je de status niet kan uitlezen, is niet "geen nieuws is goed nieuws".** De
   samenvattingsregel "0 mislukkingen" sloeg alleen op de DC's die wél antwoordden. Lees altijd het
   staartje van `repadmin /replsummary`, niet enkel de tabel.
2. **Test het protocol dat je werkelijk gebruikt, niet een makkelijker protocol op dezelfde host.**
   Zelfde patroon dook dezelfde dag op bij de VPN: de tunnel was *up* en droeg routes, maar `tun0`
   kreeg geen resolver mee, dus namen in `olvp.int` faalden terwijl `ping` op een IP werkte — zie
   [[project-mcp-and-itsm]].

Hieruit volgt **MON-2**: een NetXMS-alarm als de AD-replicatie meer dan 30 minuten achterloopt **of
helemaal niet uitleesbaar is**. Die tweede helft is de eigenlijke vangst — precies dit geval zou
anders opnieuw onzichtbaar zijn. Vereist de NetXMS-agent op de DC's
([[project-netxms-windows-agents]]).

Het hoort ook in het rijtje gaten dat de gateway-cutover van augustus achterliet, naast de VLAN's
zonder gateway-IP en de DHCP op de oude gateway ([[project-gateway-cutover]]). Te kennen vóór de
AD-migratie naar `SRVV-SERV-01` ([[project-ad-serv01-migration]]): migreren met een DC waarvan je de
stand niet kent, is vragen om problemen.
