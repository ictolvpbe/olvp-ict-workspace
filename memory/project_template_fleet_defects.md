---
name: project-template-fleet-defects
description: "TPL-1/TPL-2 in het golden image: gedeelde machine-id en een scheve partitie-indeling. Vloot gemeten 2026-09-20 — TWEE template-generaties (5 legacy-home / 7 legacy-opt), niemand op baseline. ADR 0008 (dun image + profiel), ADR 0009 (versiemarkering, geïmplementeerd). Runbook RB-2026-TPL-BASELINE klaar, nog niet uitgevoerd."
metadata:
  type: project
---

Vastgesteld 2026-09-17 tijdens de Teamdrive-backup-migratie ([[project-teamdrive-backup-outage-202609]]). Beide fouten komen uit het golden image van [[project-template-strategy]] en zijn dus twaalf keer uitgerold.

## De vloot heeft twee generaties (gemeten 2026-09-20)

Niet één scheve indeling maar **twee**, en elke generatie maakt dezelfde fout op een andere map:

| Generatie | Kenmerk | Hosts |
|---|---|---|
| `legacy-home` | `/home` 27,5 G (leeg), `/var` 2,9 G | haproxy-1, haproxy-2, sspr-01, unifi-01, p-backup-01 |
| `legacy-opt` | `/opt` 63 G (leeg), `/var` 12 G | forgejo-01, id-01, monitoring-01, tst-odoo-01, dev-odoo-01, srvv-odoo-01, srvv-acc-01 |

**Geen enkele host staat op de baseline.** Meet dus altijd eerst welke generatie je voor je hebt —
bij de ene komt de ruimte uit `/home`, bij de andere uit `/opt`. Zie
[[feedback-verify-memory-against-repo]].

## De baseline-VM: VMID 516, géén verse installatie

**In Proxmox:** VMID **516**, naam `DEB13-PODMAN-TEMPLATE`, node `srv-pmclust-p01`, 100 G op `rdb`.
Snapshot `pre-diskfix-2026-09-20` gemaakt op 2026-09-20 — daarvóór was er géén enkel rollback-punt.
Staat in `inventory.yml` als groep `baseline_vm`. Voorganger `WEG-SRVV-ODOO-TEMPLATE` (VMID 10082,
p02) is gestopt en teruggetrokken.

**Vorm:** bewust een gewone VM met snapshots, géén Proxmox-template-object — een template kun je
niet starten, en dan is elke bijwerkronde klonen → aanpassen → opnieuw converteren.

⚠️ **Afsluiten met `/usr/local/sbin/olvp-seal-template.sh --seal`**, nooit met een kale `shutdown`.
Het script ruimt op, wist de host-sleutels, maakt de machine-id leeg en sluit in dezelfde handeling
af; het weigert op elke andere host via een SMBIOS-UUID-grendel. Bron:
`platform-ansible/files/olvp-seal-template.sh`. Zie [[feedback-proxmox-clone-identiteit]].

## ✅ Baseline-VM klaar (2026-09-21): fase 1–8 afgerond

Uitgevoerd op VMID 516. `/opt` 63,25 → **2 G**, `/` → 15 G, `/var` → 20 G, `/tmp` → 2 G, `/srv`
nieuw op 5 G (fstab op UUID), **46,57 G vrij** in de VG. Krimp over twee PV's zonder `pvmove`;
herstart bracht alle zeven volumes correct op. `tier1-baseline.yml --limit baseline_vm` zette
log-hygiene, logrotate-cap en de step-ca-root.

Markering: `baseline_version: "0"`, generatie `baseline`, **zes van de zeven** sleutels.

**De baseline bereikt nooit `13.1`, en dat is correct**: op een geseald template is de machine-id
juist leeg, dus `machine-id-unique` kan daar niet waar zijn. De promotie gebeurt op de kloon, bij
diens eerste `tier1-baseline`-run.

**Geseald en bewezen op 2026-09-21.** Snapshot `Tier1-baseline-2026-09-21` staat op de
uitgeschakelde VM. Wegwerpkloon (VMID 9516, intussen vernietigd) toonde: uniek machine-id, opnieuw
aangemaakte SSH-host-sleutels, werkende aanmelding met de bestaande sleutel, correcte indeling,
markering mee gekloond, en **alle zeven sleutels vervuld** — de promotie naar `13.1` is daarmee
aantoonbaar (bewust niet gedraaid).

**De baseline is bewezen in de praktijk (2026-09-21).** Vier FRAME-VM's uit deze baseline gekloond
(10050, 10051, 224, 225): elk een eigen machine-id, `/srv` gemount, geen gefaalde units, en alle
vier naar `13.1` gepromoveerd door `tier1-baseline.yml`. TPL-2 is daarmee structureel opgelost voor
nieuwe VM's — de elf bestaande hosts blijven open (fase 9).

⚠️ **Klonen kost per stuk een volledige kopie van 100 GiB** op Ceph, ook al is er maar ~10 GiB in
gebruik: `qm clone --full` kopieert de gealloceerde grootte. Reken op enkele minuten per kloon.

Eerste afnemer was het Frappe-platform
([[project-frappe-platform]], vier VM's).

**Twee dingen om te onthouden bij het klonen:**
1. De nieuwe machine-id is exact de **SMBIOS-UUID zonder streepjes** — systemd leidt hem in een VM
   af uit de DMI-UUID. Uniciteit komt dus van de hypervisor, niet van toeval.
2. Een kloon erft de **statische IP-configuratie** van het template en komt dus op `10.10.200.1`
   te staan, waar de inventorygroep `baseline_vm` naar wijst. Zet hostname en adres meteen na de
   eerste boot, of laat geen kloon rondslingeren. Argument voor de cloud-init-werf.

**Fase 9 — de elf bestaande hosts — is nog volledig open.**

## Gemeten indeling

`SRVV-DEBIAN-TEMPL` op **10.10.200.1** vervangt het teruggetrokken `SRVV-ODOO-TEMPLATE`
([[project-template-strategy]]). Gemeten: schijf 100 GB, `/` 7,72 · `/opt` **63,25** (268 K in
gebruik) · `/var` 12 · `/home` 8 · `/tmp` 0,58 · VG vrij 7,03. Zelfde `machine-id` als de hele
vloot en `/lost+found` uit 2023 — het is dus **dezelfde afstamming**, geen schone start. Wel al
rolneutraal: het Odoo-skelet is eruit.

Gevolg voor het runbook: de ruimte komt van **`/opt`**, niet van `/home`. En de VG-naam blijft
`SRVV-DEBIAN-TEMPL-vg` — hernoemen naar `vg0` is ingetrokken omdat `/etc/fstab` op device-pad
mount (alleen `/boot` op UUID), dus `vgrename` zou de boot breken.

Het image mist ook het **step-ca root-cert** en het log-hygiene-filter, terwijl de
template-strategie beide als aanwezig beschreef. `tier1-baseline.yml` zet ze wel.

## TPL-1 — `/var` te klein, `/home` te groot

Het template geeft `/home` **27,5 GB** (werkelijk in gebruik: 128 KB) en `/var` **2,9 GB**, terwijl `/var` de logs, container-images en databases draagt. Op een server hoort die verhouding omgekeerd.

Concreet gemeten op `SRVV-P-BACKUP-01`: VG 49,52 GB met **9,91 GB ongebruikt**, `/var` op 79% met nog een volledige backup-run te gaan. Op `SRVV-CLOUDBACKUP-01` liep `/var` in juni-september volledig vol, wat de backup mee om zeep hielp.

**Per host online op te lossen** — `/var` is ext4 en er staat ruimte vrij in de VG, dus geen herstart nodig:

    sudo lvextend -r -L +8G /dev/SRVV-DEBIAN-TEMPL-vg/var

`-r` groeit meteen het bestandssysteem mee. Krimpen van `/home` is niet nodig zolang de VG vrije ruimte heeft.

## TPL-2 — dezelfde machine-id op 12 VM's

`/etc/machine-id` is niet leeggemaakt vóór het wegschrijven van het template, dus elke kloon erft `24a516d46f714444a68e97843c763d1a`. Gemeten op haproxy-1/2, tst-odoo-01, dev-odoo-01, srvv-odoo-01, srvv-acc-01, forgejo-01, id-01, monitoring-01, unifi-01, sspr-01 en p-backup-01.

Impact nu beperkt — de VM's hebben statische adressen (geen DHCP-DUID-botsing) en NetXMS gebruikt een eigen agent-id — maar het raakt journal- en loglabeling en software die er installatie-id's uit afleidt. **Eerst nakijken of Alloy/Loki op machine-id labelt** ([[project-netxms-monitoring]]); dat bepaalt de urgentie.

Saneren vraagt per VM een herstart. `SRVV-ODOO-01` en `SRVV-ACC-01` zijn productie en krijgen een eigen venster. **Let op**: `/var/lib/dbus/machine-id` is op deze VM's een echt bestand, geen symlink, en moet mee.

Gedaan op `SRVV-P-BACKUP-01` en `SRVV-CLOUDBACKUP-01`.

## Stand 2026-09-19

**Beslist (user):** `/home` krimpen naar **2 GB**, alle vrijgekomen ruimte naar `/var`. Niet `/home`
volledig verwijderen (isolatie blijft), niet enkel de vrije VG-ruimte bijzetten.

**Geschreven, nog niet uitgevoerd:** runbook **RB-2026-TPL-BASELINE** →
`platform-handbook/hosting/operations/repair-tier1-template-disk.md`. Negen fases: snapshot → meten
→ `/home` veiligstellen → krimpen → `/var` groeien → Tier-1-stack controleren → machine-id →
nieuwe snapshot + wegwerpkloon als bewijs → sanering van de elf bestaande hosts. Met rollback,
troubleshooting-tabel en 8 gotcha's.

**Waarom dit vóór Euro-Office komt:** Fase 0 van RB-2026-NC-EO-DEPLOY kloont uit dit image. Een
document-server met container-images in een `/var` van 2,9 GB is een storing die je zelf plant.

**Drie gotcha's die de uitvoering bepalen:**
1. Via de **Proxmox-console**, niet SSH — `/home/ansible/.ssh/authorized_keys` staat op de partitie
   die je afbreekt. Root-wachtwoord bij de hand.
2. Volgorde ligt vast: eerst `/home` krimpen (ext4 kan niet online krimpen → unmount), dán `/var`
   groeien (dat kan wél online, daarom is de vloot-sanering van TPL-1 veilig tijdens de werkuren).
3. **machine-id als állerlaatste stap vóór het afsluiten.** Eén extra boot "om iets te checken" en
   systemd vult hem opnieuw — dat is vrijwel zeker hoe de oorspronkelijke fout ontstond.

**Zijspoor genoteerd, niet in scope:** staan `/etc/machine-id` én de **SSH-host-sleutels** gelijk op
twee klonen, dan is dat dezelfde oorzaak. Kandidaat TPL-3; saneren vraagt een eigen venster want
alle `known_hosts` moeten mee.

## Open ontwerpvraag — init-procedure bij eerste boot (user, 2026-09-19)

User stelde voor: een init-procedure die na een kloon bij de eerste opstart machine-id reset,
IP en hostname zet, het ansible-account klaarzet, enz. **Antwoord: ja, cloud-init is het juiste
gereedschap — maar het vervangt de image-fix niet, het is er van afhankelijk.**

- machine-id: systemd genereert alleen een nieuwe id als het bestand **leeg** is. Staat er een
  waarde in het image, dan is er niets te genereren. Geen init-script repareert dat achteraf.
- Schijfindeling: `growpart`/`resize_rootfs` groeien enkel de laatste partitie — bij LVM met aparte
  `/var` en `/home` helpt dat niets.

Wat cloud-init wél overneemt (vandaag handwerk per kloon, en handwerk is waar TPL-2 vandaan komt):
hostname, `/etc/hosts`, IP/gateway/DNS, `ansible`-user + sleutel, en het **opnieuw genereren van de
SSH-host-sleutels**. Het breekt ook het kip-en-ei dat `provision-ansible-account.md` nu met de hand
oplost: Ansible kan geen host bereiken zonder IP en sleutel.

Voorgestelde verdeling in vier lagen: **image** (lege machine-id, géén host-sleutels, juiste
LV-indeling, cloud-init geïnstalleerd) → **cloud-init** (identity + netwerk) → **tier1-baseline.yml**
(bestaat al, idempotent) → **Tier-2-playbook** (bestaat al).

Drie punten om vooraf te wegen: (a) Proxmox vraagt een echt **template-object met CloudInit-drive**
(`qm set --ipconfig0 --ciuser --sshkeys`) — het huidige image is een gewone VM met een snapshot;
(b) cloud-init wil de **netwerkconfig bezitten** → kiezen tussen laten renderen of
`network: {config: disabled}` met enkel identity — dit is waar zulke trajecten meestal op stranden;
(c) het is een extra bewegend deel met een eigen faalmodus (VM boot zonder netwerk omdat de
datasource niet gelezen werd). Bij ~12 VM's per jaar wint het weinig tijd, maar wél uniformiteit —
en dát is hier het probleem.

**Advies: eerst het image repareren (blokkeert Euro-Office), cloud-init daarna als aparte werf.**
Nog te beslissen: ADR schrijven, het runbook uitbreiden met een fase "image cloud-init-klaar", of
voorlopig enkel als werf registreren.

## ▶ Waar het verder gaat

1. **De VM controleren die nu als master/template dienst doet** — bestaat hij nog, staat hij uit,
   is het een gewone VM met snapshot of een Proxmox-template-object, en welke indeling heeft hij
   werkelijk? Dat beantwoordt meteen de niet-uniform-vraag hierboven. Laatst bekend als
   `SRVV-ODOO-TEMPLATE` op **10.200.0.40**, snapshot `Tier1-baseline-2026-06-01`.
2. Daarna RB-2026-TPL-BASELINE uitvoeren.
3. Beslissen over de cloud-init-werf.

⚠️ **Werkstation stond 2026-09-19 buiten het schoolnetwerk** (192.168.2.107, geen VPN) — 10.200.0.40,
de Proxmox-node 10.10.100.70 en drie andere hosts gaven allemaal timeout op poort 22. Niets kon
nagemeten worden. Test dat eerst opnieuw bij hervatting.

**Nog niet gedaan** (bewust buiten scope gehouden): TPL-1/TPL-2 staan nog steeds nergens in
`governance/tracker-additions.md`, en `project_template_strategy.md` zegt nog altijd niets over de
partitie-indeling of machine-id — een volgende template-build herhaalt de fout dus. Handbook-wijziging
nog niet gecommit.

## How to apply

De sanering per host is dweilen: zolang het template ongewijzigd blijft, erft elke nieuwe VM beide fouten. Repareer eerst het template — `/etc/machine-id` leegmaken (niet verwijderen; systemd vult hem bij de eerste boot) en de partitie-indeling omkeren — en saneer daarna de bestaande hosts. Zie [[project-srvv-p-backup-01]].
