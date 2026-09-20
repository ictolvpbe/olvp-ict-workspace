---
name: project-mgmt-segmentatie-vlan30
description: "VLAN 30 VL-SD-MGMTOOB (10.30.0.0/24) voor beheerinterfaces, alleen bereikbaar vanaf VLAN 34. Geblokkeerd door Ceph: public_network staat op VLAN 10, dus het Proxmox-management verhuizen is een monitor-migratie. Ceph gaat naar VLAN 8 VL-CEPH-PUBNET. ADR 0010 + RB-2026-CEPH-PUBNET, beide 2026-09-20 bevestigd."
metadata:
  type: project
---

[ADR 0010](../platform-handbook/hosting/decisions/0010-segmentatie-managementadressen.md).
Aanleiding: waar horen de beheeradressen van Proxmox, NAS, alarmscherm en backup-PC?

## De beslissing in twee regels

**VLAN 30 `VL-SD-MGMTOOB` (`10.30.0.0/24`, gw `.1`)** voor interfaces die *uitsluitend* bestaan om
te beheren: Proxmox-web-UI, NAS-beheerpoort (de 1 Gbit-NIC), switch- en AP-management, IPMI.
Beleid: **binnenkomend alleen vanaf VLAN 34, uitgaand niets** behalve DNS, NTP, step-ca, NetXMS.

**Niet verhuizen wat ergens staat om een reden.** `PC-BACKUP-01` blijft in VLAN 19 (1,1 TB per run
naast de NAS, hoort op L2), het alarmscherm blijft in VLAN 73 (hangt onbeheerd in een lokaal en
hoort juist níét in een beheersegment). Hun beheerpad wordt met regels afgeschermd.

**Het principe:** plaatsing bepaalt de blast radius, beleid bepaalt de toegang. Een VLAN per
toestel lost niets op en maakt de regelverzameling onleesbaar — zie [[project-firewall-strategy]].

**Waarom niet VLAN 35:** dat huisvest mgmt-*diensten* (Semaphore, Forgejo, Keycloak, NetXMS) die
naar de hele vloot moeten praten. Zet je beheerinterfaces daarbij, dan erven precies de interfaces
die niets mogen initiëren het ruimste uitgaande beleid.

## Wat de verhuizing blokkeert: Ceph

Gemeten op het productiecluster (2026-09-19, root-SSH op 10.10.100.80-82, alle drie identiek):

- **corosync staat al netjes apart** — `ring0_addr` op `10.9.0.8x`, eigen NIC, VLAN 9. Het
  managementadres verhuizen raakt het cluster dus niet.
- **Ceph is maar half gescheiden.** `cluster_network` = 10.7.0.0/24 (VLAN 7), maar
  `public_network` = **10.10.100.0/16** en `mon_host` staat op de VLAN-10-adressen. Elke
  VM-schijf-I/O kruist dus het gebruikers-VLAN, en het managementadres verhuizen is een
  **Ceph-monitor-migratie**, geen VLAN-wissel.

Het VLAN-register noemde VLAN 7 "Proxmox Ceph-netwerk" en suggereerde een isolatie die er maar
voor de helft is; dat is rechtgezet.

**Bestemming Ceph-clientverkeer: VLAN 8 `VL-CEPH-PUBNET` (`10.8.0.0/24`)**, bevestigd 2026-09-20.
Symmetrisch met 7 (replicatie) en 9 (corosync), op de 10 Gbit-NIC. VLAN 19 viel af wegens de
NAS-backupstroom; VLAN 30 omdat management geen hot path hoort te dragen.

## Fasering (ADR 0010)

1. VLAN 30 aanmaken + beleid → 2. NAS-beheerpoort → 3. switch-/AP-management inventariseren →
4. beheerpaden backup-PC en alarmscherm dichtzetten → 5. **Ceph public_network
([RB-2026-CEPH-PUBNET](../platform-handbook/hosting/operations/migrate-ceph-public-network.md))** →
6. Proxmox-management naar VLAN 30.

Fase 5 en 6 **eerst op het testcluster**: `srv-pmclust-t01..t03` (10.10.100.70-72) staat standaard
uit en los van het stroomnet, en is tegelijk het DR-doel voor productie. Mislukte oefening kost
daar niets, en het toetst meteen of de DR-rol werkt. Productie = `p01..p03` op 10.10.100.80-82.

## Feiten die de uitvoering sturen

- Ceph **19.2.6 squid**, 3 mons, 12 OSD's (4/node, SSD, ~21 TiB raw), pool `rdb` size 3/min_size 2.
- Ceph 19 aanvaardt **meerdere** `public_network`-subnetten → gefaseerd migreren, niet springen.
- `HEALTH_OK` is niet haalbaar: het cluster meldt `HEALTH_ERR`, volledig uit `AUTH_INSECURE_*` over
  het verouderde `aes`-cephx-sleuteltype. Geen PG-, OSD- of quorumprobleem. **Niet in dezelfde
  beweging oplossen** — twee ingrepen tegelijk maakt een storing onherleidbaar.
- Nooit meer dan één monitor tegelijk weg (3 → 2 = quorum, 3 → 1 = alles stil).
- Proxmox-**backups** lopen wél correct over VLAN 19 (`PROXMO_BU` → 10.19.0.7). Wat nog over
  VLAN 10 loopt is de ISO-share op 10.10.100.7.

## Open

- `PRODSTOR` wijst naar Ceph's interne pool `.mgr` (`pg_num 1`) met 23 GiB weesdata van twee
  half-verwijderde images; niets gebruikt het. Opruimen = fase 8 van RB-2026-CEPH-PUBNET, ná de
  netwerkaanpassingen.
- `AUTH_INSECURE_*` op de cephx-sleutels.
- Switch-, AP- en IPMI-beheeradressen nog te inventariseren.

Gerelateerd: [[project-firewall-strategy]], [[project-infrastructure-params]],
[[project-offline-backup-chain]], [[project-template-fleet-defects]].
