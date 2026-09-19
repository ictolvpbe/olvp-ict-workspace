---
name: project-template-fleet-defects
description: "Twee fouten in het golden image op alle 12 klonen: /etc/machine-id niet leeggemaakt en een scheve partitie-indeling. Runbook RB-2026-TPL-BASELINE klaar 2026-09-19 (nog niet uitgevoerd). LET OP: de vloot blijkt NIET uniform — twee verschillende indelingen gedocumenteerd. Tracker TPL-1/TPL-2."
metadata:
  type: project
---

Vastgesteld 2026-09-17 tijdens de Teamdrive-backup-migratie ([[project-teamdrive-backup-outage-202609]]). Beide fouten komen uit het golden image van [[project-template-strategy]] en zijn dus twaalf keer uitgerold.

## ⚠️ De vloot is niet uniform (vastgesteld 2026-09-19)

De cijfers hieronder komen van **één meting op `SRVV-P-BACKUP-01`**. Het handbook spreekt ze deels
tegen: `hosting/operations/deploy-netxms.md:121` documenteert voor `SRVV-MONITORING-01` een schijf
van **200 GB** met `/` 7,6 G, `/var` **12 G**, `/opt` **62 G** en ongeveer de helft ongepartitioneerd
— tegenover 49,5 GB / `/var` 2,9 G / `/home` 27,5 G op P-BACKUP-01. Beide heten Tier-1-klonen.

Óf het image is onderweg gewijzigd, óf niet elke VM komt eruit voort. **Niet uitgezocht.** Dus:
behandel elk getal in deze memory als voorbeeld, niet als vaststelling, en meet per host vóór je
iets wijzigt. Zie [[feedback-verify-memory-against-repo]] — dit is precies dezelfde soort fout,
nu in mijn eigen memory.

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
