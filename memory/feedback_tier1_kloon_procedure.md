---
name: feedback-tier1-kloon-procedure
description: "Een VM uit de Tier-1-baseline klonen vraagt drie dingen die niemand onthoudt: de baseline hangt op vmbr0 zonder tag terwijl de doel-VLAN's op vmbr1 met tag zitten, er is geen cloud-init dus hostname/IP gaan via de QEMU-guest-agent, en qm resize geeft blokken maar geen bestandssysteem."
metadata:
  type: feedback
---

**Vastgesteld 2026-09-23** bij het klonen van `SRVV-TST-CLOUD-01` en `SRVV-TST-OCLOUD-01`
([[project-eurooffice-nextcloud]]). De kloon zelf lukt in één commando; de drie stappen erna kosten
de tijd, en ze zijn alle drie vergeetbaar op een manier die pas later pijn doet.

**Why:** de baseline is bewust een kále Debian zonder cloud-init ([[project-template-strategy]]).
Dat is prima voor de reproduceerbaarheid, maar het betekent dat een verse kloon opstart met het
IP en de hostname van de template — in een VLAN waar dat adres niet thuishoort.

## How to apply

**1. Het bridge/tag-verschil.** De baseline (VMID 516) hangt op `bridge=vmbr0` **zonder** VLAN-tag
(dat is VLAN 10, `10.10.200.1`). De doel-VLAN's zitten op **`vmbr1` mét tag**, bijvoorbeeld
`tag=207` voor `VL-TST-WEBAPPS`. Vergeet je dat bij `qm set`, dan boot de kloon in het verkeerde
VLAN met het adres van de template — en als de template ooit tegelijk aanstaat, heb je een
adresconflict met je eigen golden image.

```
qm set <id> --net0 virtio,bridge=vmbr1,tag=207 --cores 2 --sockets 2 --memory 8192 --onboot 1
```

`cores 2 × sockets 2` = 4 vCPU; Proxmox rekent dat zo, niet met `cores 4`.

**2. Hostname en IP via de guest-agent, niet via de console.** `agent: 1` staat aan op de baseline,
en de agent loopt over **virtio-serial** — dus hij werkt óók wanneer het netwerk nog helemaal
verkeerd staat. Dat maakt `qm guest exec` het aangewezen pad en bespaart het geklik in noVNC:

```
qm guest exec <id> --timeout 60 -- /bin/bash -c 'echo <base64> | base64 -d > /root/init.sh; bash /root/init.sh <HOSTNAME> <IP>'
```

Base64 eromheen omdat de quoting anders door drie lagen shell moet (lokale shell → ssh → qm → bash)
en je daar niets aan hebt. Het script zet `hostnamectl set-hostname`, vervangt de oude naam in
`/etc/hosts` en schrijft `/etc/network/interfaces`. Let op het **netmask `255.255.254.0`** (`/23`)
in de test-webapps-band, niet `/24`.

**3. `qm resize` geeft blokken, geen bestandssysteem.** Na het vergroten van de virtuele schijf is
er alleen ongebruikte ruimte áchter de laatste partitie. De hele keten is nodig:

```
parted mkpart primary <start> 100%  →  pvcreate  →  vgextend  →  lvextend  →  resize2fs
```

De VG van een kloon heet nog naar de **template** (`SRVV-DEBIAN-TEMPL-vg`); dat is cosmetisch,
hernoemen raakt de bootconfiguratie en is het risico niet waard. Vraag de naam op met `vgs` in
plaats van hem hard te coderen.

**En dan de maatvoering zelf.** De baseline geeft `/srv` **5 G**. Voor een bestandsplatform, een
Bacula-doel of wat dan ook dat data vasthoudt is dat geen "kleine start" maar een vergissing die
staat te wachten. Zet `/srv` en `/var` (container-images!) meteen goed — achteraf groeien kan wel,
maar dan doe je het onder tijdsdruk.

Gerelateerd: [[feedback-proxmox-clone-identiteit]] (dekt de identiteitskant: machine-id,
host-sleutels, sealen), [[project-template-fleet-defects]], [[project-baseline-versiemarkering]].
