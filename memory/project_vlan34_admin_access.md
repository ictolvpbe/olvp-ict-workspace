---
name: project-vlan34-admin-access
description: "VLAN 34 = admin-VLAN (4 toestellen: 2 Linux, 2 Win + wifi). Gemeten 24/09: staat BREED open (ook DMZ, strijdig met A-004) — niet te krap. Ontwerp AD-1..AD-8 in firewall-matrix; tracker NET-10 (regels), NET-11 (DHCP 8.8.8.8), NET-12 (VLAN 4, ná NET-10), NET-13 (802.1X EAP-TLS)."
metadata:
  type: project
---

Beslist door de user op **2026-09-24**: **VLAN 34 (`VL-SD-ADMIN`, 10.34.0.0/27) wordt het
admin-VLAN met toegang tot alle andere VLANs.** Op te pakken vóór verder werk dat vanaf de
beheerwerkplek servers moet bereiken. Nog niet uitgevoerd.

⚠️ Dit vervangt de aanname in oudere memories dat de beheerwerkplek in VLAN 10 zit: het
werkstation `/home/demm` hangt nu **bekabeld in VLAN 34** (`10.34.0.2/27`, gw `10.34.0.1`) en
daarnaast op wifi `WL-BSP001` (`10.4.2.138/20`). Zie de correctie in
[[project-firewall-strategy]], die nu zelf ook achterhaald is.

## Gemeten vanaf 10.34.0.2 (2026-09-24)

| doel | VLAN | resultaat |
|---|---|---|
| `10.33.0.10:53/445` | 33 | open |
| `10.10.0.10:53` | 10 | open |
| `10.35.0.2:22` (bastion), `10.35.0.10:3000` (Semaphore) | 35 | open |
| `10.35.0.11:443` (Forgejo), `10.35.0.20:443` | 35 | **dicht** |
| `10.36.0.42:443` (SRVV-ACC-01) | 36 | open — dat is regel **A-010**, bewust solo-host |
| `10.36.0.50/.51/.52` (FRAME acc/acc-test/kale frappe) | 36 | **dicht** |
| `10.200.14.51/.52` (FRAME dev/tst) | 200 | **dicht** |

Het patroon klopt precies met `firewall-rules-matrix.md`: er staan per-host-regels, geen
Admin→alles. De FRAME-VM's zijn er nooit bijgezet. Traceroute sterft na hop 1 en TCP krijgt een
actieve *no route to host* van `10.34.0.1` — de gateway zelf zegt geen pad te hebben.

## Tweede, losstaand euvel: DNS op VLAN 34

DHCP deelt op dit segment **`8.8.8.8` als eerste resolver** uit, met `10.34.0.1` als tweede en
**zonder zoekdomein**. systemd-resolved blijft bij de eerste, dus `olvp.int` geeft NXDOMAIN.
Bewezen: `dig @10.34.0.1 frame-acc.olvp.int` → `10.36.0.50` (de conditional forwarder op de
gateway werkt prima), `dig @8.8.8.8 ...` → niets. `dns.md` schrijft ook voor dat client-VLANs
de gateway als resolver horen te krijgen. **Fix: 8.8.8.8 uit de DHCP-DNS-optie van VLAN 34
halen** (of minstens achter de gateway zetten) en het zoekdomein `olvp.int` meegeven.

## Aandachtspunten bij uitvoering

* De vink **`Retourverkeer Automatisch Toestaan`** is verplicht op elke nieuwe inter-zone-allow —
  zie [[project-firewall-strategy]]. Zonder die vink krijg je timeouts die op routering lijken.
* Documenteer de regel in `platform-handbook/network-physical/network/firewall-rules-matrix.md`
  vóór activatie; dat is de afspraak in die werf.
* Admin→alles maakt het beheerwerkstation het enige overgebleven scharnier. Dat verhoogt het
  gewicht van [[project-it-workstation-hardening]], dat nog op nul staat.
* `10.35.0.11` en `.20` waren ook dicht — mogelijk hosts die niet draaien, niet per se een
  firewallgat. Apart nakijken.

**Waarom dit nu bovenkwam:** het blokkeerde de MCP-uitrol naar `frame-acc`/`frame-acc-test`
(zie de melira-frappe-werf). Zolang VLAN 34 VLAN 36 niet bereikt, werkt de beheerwerkplek daar
alleen via wifi.

## Nagemeten 2026-09-24, ná het uitschakelen van de kabel

De beheerwerkplek heeft drie mogelijke paden, en **maar één ervan bereikt de FRAME-VM's**:

| pad | adres | `10.36.0.42` (A-010) | `10.35.x` mgmt |
|---|---|---|---|
| kabel VLAN 34 | 10.34.0.2/27 | open | deels (bastion + Semaphore wel, `.11`/`.20` niet) |
| wifi `OLVP` (VLAN 10) | 10.10.150.27/16 | open | open |
| wifi `WL-BSP001` | 10.4.2.138/20 | open | — |

⚠️ **Correctie op een eerdere lezing in deze notitie.** De FRAME-VM's `10.36.0.50/.51/.52`
waren vanaf álle drie de paden onbereikbaar, en dat is **geen firewallkwestie**: traceroute
vanaf `10.4.0.1` geeft `!H` (host unreachable) en in het hele blok antwoordt alleen `.42`.
De gateway kan die hosts dus niet ARP'en — **ze stonden uit**. Zie de melira-frappe-werf.
Dat `.42` overal open staat terwijl `.50` nergens antwoordt, is het bewijs: zelfde VLAN,
zelfde regels, ander resultaat.

Wat blijft staan: VLAN 10 bereikt `10.35.0.10:3000` en `10.35.0.2:22` breed — de open
bevinding van 2026-08-28 in [[project-firewall-strategy]] — en VLAN 34 bereikt minder dan
VLAN 10. Dat is nog steeds omgekeerd aan de bedoelde trust-ladder.

## Tweede todo (user, 2026-09-24): VLAN 4 afschermen

**`10.4.0.0/16` (wifi `WL-BSP001`) moet afgeschermd worden.** Dat segment is vandaag het enige
pad naar de FRAME-VM's in VLAN 36 — ruimer dan zowel het admin-VLAN als het user-VLAN, en dus
precies omgekeerd aan de bedoelde trust-ladder.

⚠️ Let op de volgorde: **eerst** VLAN 34 → alle VLANs in orde brengen, **dan** pas VLAN 4
dichtzetten. Andersom snijdt de beheerwerkplek zichzelf af van `frame-acc` en `frame-acc-test`.

De DHCP-lease op dat net gaf een **/20** (`10.4.2.138/20`, gw `10.4.0.1`), terwijl de afspraak
`/16` noemt — uitzoeken wat de werkelijke scope is voor je regels schrijft.

## Vastgesteld 2026-09-24: `ADMIN-WORKSTATIONS` bestaat niet in UniFi

De host-list uit `firewall-rules-matrix.md` (bron van A-001/A-002/A-010/A-012/B-002) is nooit
aangemaakt — bevestigd door de user. Welke objecten de werkende paden (bastion, Semaphore, `.42:443`)
wél als bron gebruiken, is nog niet uitgelezen. Ontwerpvraag van de user: wie mag op VLAN 34
(toegangscontrole), en mag een toestel daar eens verbonden overal bij of beperken we? Voorstel in
gesprek: 802.1X met toestelcertificaat als toegangspoort, VLAN 34 zelf als bron-object (geen
host-list), en per bestemmingszone een poortgroep in plaats van Admin→any.

**Antwoorden user 2026-09-24:** VLAN 34 = **4 toestellen** (2 Linux, 2 Windows); er bestaat **wél een
wifi-netwerk op VLAN 34** (hoe beveiligd — PSK of 802.1X — nog na te kijken, dat is het grootste
mogelijke gat); remote support aan gebruikers loopt via een **cloudtool** → gebruikers-VLANs mogen
vanaf VLAN 34 volledig dicht.

## Gemeten 2026-09-24 namiddag vanaf 10.34.0.2 — de lezing hierboven klopt NIET

⚠️ "Lappendeken van solo-host-regels" was fout. VLAN 34 staat vandaag **breed open**: DC's
(`10.33.0.10`, `10.10.0.10`, `10.200.0.10`) op alle AD-poorten incl. 135/3389/5985, VLAN 35
(bastion, Semaphore, `.12`, UniFi `.15`), `10.36.0.42:22+443`, `10.200.14.41`, Proxmox
`10.10.100.80:8006` én de **DMZ** (`10.21.0.6:22`, `10.21.1.10:22/8443` — strijdig met A-004).
**Élke host die dicht leek, stond uit**: ARP `INCOMPLETE` vanaf de bastion in hetzelfde VLAN, en
`pvesh` toont ze `stopped` (Forgejo 209, MONITOR-01 223, ODOO-01 220, ACC-FRAME 224/225, …).
Er is dus geen enkel bewijs van een blokkade vanaf VLAN 34. Het probleem is eerder te ruim dan te
krap. Welke zone-policy dat doet, moet uit de controller komen. Ontwerp AD-1..AD-8 staat in
`firewall-rules-matrix.md` sectie "Admin-VLAN 34".

**Hermeten 2026-09-24 na herstart van de VM's:** vanaf 10.34.0.2 nu óók open: Forgejo `.11`,
MONITOR `.20`, ODOO-01 `10.36.0.40`, FRAME `10.36.0.50/.52`, `10.200.14.40/.51/.52` (22+443).
Enkel `10.36.0.51` dicht (geen ARP vanaf bastion; TST-ACC-FRAME-01 = VMID 225 zit op **.52**, niet
.51 — de toewijzing .50/.51/.52 in deze notitie klopt dus niet) en `10.19.0.20:22` (VLAN 19
geïsoleerd, verwacht). Conclusie blijft: VLAN 34 staat breed open, geen enkele blokkade.

**VLAN 36 volledig gescand 2026-09-24 (10.36.0.2-254, vanaf 10.34.0.2):** antwoorden op 22+443+ICMP:
`.40` ODOO-01, `.42` ACC-01, `.50` ACC-FRAME-01, `.52` TST-ACC-FRAME-01 — alle vier volledig open.
Niet antwoordend, maar niet door de firewall: `.44` SRVV-SSPR-01 (VMID 105, stopped, geen onboot)
en `.51` SRVV-FRAPPE-02 (VMID 401, stopped, **net0 op vmbr0 zonder VLAN-tag** — zou zelfs gestart
niet in VLAN 36 landen; inventory en vm-inventory.md zeggen 10.36.0.51).

## Stand einde sessie 2026-09-24 — tracker

Vastgelegd in `governance/tracker-additions.md`: **NET-10** (brede toegang vervangen door AD-1..AD-8;
eerst in de controller de regel vinden die de Admin-zone vandaag opent), **NET-11** (DHCP 8.8.8.8 eruit
+ zoekdomein), **NET-12** (VLAN 4 afschermen, pas ná NET-10), **NET-13** (802.1X + toestelcertificaat;
eerst nakijken of het VLAN-34-wifi een PSK heeft), **OPS-3** (startbeleid VM's na de reset).
Remote support aan gebruikers loopt via een cloudtool → gebruikers-VLANs dicht vanaf VLAN 34; op
de beheertoestellen enkel de technicus-client, geen onbeheerde agent.
