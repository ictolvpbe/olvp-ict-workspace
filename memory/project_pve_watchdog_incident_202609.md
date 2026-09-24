---
name: project-pve-watchdog-incident-202609
description: "2026-09-24 12:00: switchstoring brak het Proxmox-quorum, HA-watchdog resette srv-pmclust-p02 en p03 hard. Alles kwam terug behalve SRVV-NXFILTER-V11. Switch opgelost door ICT."
metadata:
  type: project
---

**2026-09-24, omstreeks 12:00.** Twee van de drie Proxmox-nodes zijn **hard gereset** door de
HA-watchdog. Ontdekt doordat `10.10.100.9` (GLPI) midden in een sessie onbereikbaar werd —
ping gaf `no route to host`, óók vanaf een andere host in hetzelfde `/16`.

```
12:00:03  watchdog-mux: client (PID 38188) watchdog is about to expire
12:00:13  watchdog expired - disable watchdog updates
12:00:14  exit watchdog-mux with active connections
-- Boot 2dce69bfa14342f88fa007919bf60e4e --
12:02:16  Starting VM 207
```

**Oorzaak (ICT, 2026-09-24)**: een **probleem met een switch**. Daardoor verloren de nodes
elkaar, viel het quorum weg en deed de HA-watchdog waarvoor hij bedoeld is — de nodes fencen.
De switch is opgelost.

## Wat geraakt werd

- `srv-pmclust-p02` en `p03` herstart; **p01 niet**. Quorum daarna weer 3/3.
- Alles kwam vanzelf terug **op één na**: **`SRVV-NXFILTER-V11` (VMID 241, p03, VLAN 11)**
  bleef `stopped` terwijl HA hem op `state started` had staan.
  **Handmatig gestart 2026-09-24 13:15** (`qm start 241` → "Requesting HA start"); draait op
  `10.11.0.9` en beantwoordt DNS. Er bestaat ook een `SRVV-NXFILTER-V14` (VMID 240, p02) en een
  tweede DNS-antwoordend adres `10.11.0.10`.
- **Meet niet te vroeg.** `SRVV-BSP001` en `SRVV-EDU001` stonden twee minuten na de reset nog op
  `stopped` en leken uitgevallen; ze waren gewoon aan het opstarten en draaiden even later.
- GLPI (VMID 207, `SRV-GLPI-01`) kwam binnen ~2 minuten terug; Apache en MariaDB actief,
  `status.php` weer HTTP 200. Geen schade aan de opschoning die die ochtend gedraaid was.

## Les

**Een VM die na een node-reset niet terugkomt, meldt dat nergens.** De node is gezond, het
cluster is quorate — en toch draait hij niet. Maar er zijn **twee** startmechanismen, en één
controle dekt ze niet allebei:

| VM-soort | Wie start hem | Controle |
|---|---|---|
| Gewone VM | `onboot: 1` | `qm config <id>` vs `qm status <id>`, per node |
| **HA-beheerde VM** | de HA-manager, **`onboot` telt niet** | `ha-manager status` — gewenste state vs `qm status` |

⚠️ Een scan op `onboot: 1` **mist precies de HA-VM's**. Dat was hier bijna misgegaan: NXFILTER-V11
kwam wél in die scan naar voren, maar de andere HA-resources (102, 103, 201, 203, 242, 513) niet —
die hadden net zo goed stil kunnen blijven staan zonder dat de controle iets liet zien.

Draai na élke ongeplande node-herstart dus **beide**, vóór je naar de diensten zelf kijkt. En
verifieer daarna de **dienst**, niet alleen de VM-status: voor NXFilter is dat een `dig` tegen
`10.11.0.9`, niet `qm status`.

Diezelfde ochtend was er nóg een geval van hetzelfde patroon: 49 FusionInventory-agents die al
maanden een 404 kregen zonder dat iets alarmeerde ([[project-glpi-srvv-glpi-01]]). En het is
dezelfde les als MON-2 en de backup-keten: **de afwezigheid van een signaal is zelf een
signaal**.

## Corosync heeft één ring — opgenomen als NET-9

**Bevestigd 2026-09-24**: corosync draait op **één link**. `ring0_addr` = `10.9.0.80/81/82`
(VLAN 9) over NIC **`eno6`**, `linknumber: 0`, transport **knet** (kan 8 links),
`link_mode: passive`. Eén netwerkpad kon dus twee van de drie nodes meenemen.

Alle drie de nodes zijn identiek: `eno5`→vmbr0 (10.10.100.x), **`eno6` = corosync**,
**`eno7`+`eno8` ongebruikt**, `ens1f0`→vmbr1 (10.1.0.x + VLAN 19), `ens1f1` = Ceph (10.7.0.x).

Twee routes voor een tweede ring: over **`ens1f1`** (andere fysieke kaart `38:ea:a7` tegenover
onboard `d4:f5:ef`, geen bekabeling nodig — en met `passive` draagt ring1 in normale werking
geen verkeer, dus Ceph stoort niet), of over **`eno7`/`eno8`** (vrij op alle drie, zuiverder,
maar bekabelen).

⚠️ **Het draait om de switch, niet om de NIC.** Zit `ens1f1` op dezelfde switch als `eno6`, dan
is een tweede ring schijnveiligheid en lost ze dit incident niet op. Stel de bekabeling eerst
vast. En: een corosync-wijziging raakt het quorum zelf — `config_version` omhoog, via pmxcfs
naar alle nodes; fout uitgevoerd splijt het cluster.

Zie [[project-mgmt-segmentatie-vlan30]].
