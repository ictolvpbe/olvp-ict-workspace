---
name: project-offline-backup-chain
description: "Offline backup-keten NAS→USB: ontdekt 2026-09-01 bij storing, draaide ongedocumenteerd op het kiosk-toestel. Besluit: eigen toestel PC-BACKUP-01 in VLAN 19. Cloud-pull blijft aparte VM."
metadata:
  node_type: memory
  type: project
---

Naast de Proxmox-VM-backups op de NAS draait een **offline keten**: 2× per week (ma + wo 16:00) wordt **1,115 TB** van de NAS naar een externe USB-schijf gesynct met `rclone` over CIFS. **3 weekschijven roteren**, bewaard op ± 100 m van het serverlokaal; een tweede reeks van **3 maandschijven** (1e zaterdag) is nooit ingepland en werkt dus niet. Volledig gedocumenteerd in `platform-handbook/network-physical/physical/storage.md` (2026-09-01 herschreven).

**Keten**: Google Teamdrives → cloud-VM `10.10.100.8` → NAS-share `clbu_full` → USB. Daarnaast Proxmox → NAS `proxmo_bu/dump` → USB. NAS = **UniFi UNAS Pro**, `10.19.0.7` (storage-NIC, VLAN 19) + `10.10.100.7` (routeerbare NIC).

**Scripts** in `/opt/backup/` op het uitvoerende toestel, met de hand onderhouden:
- `BuNasToExtHDD.sh` — in cron, ma + wo 16:00
- `BuTeamDrivesToExtHDD.sh` — **niet** in cron, wordt *geketend* aan het einde van het vorige script. Praat ondanks de naam níét met Google (dat zit op de VM); kopieert alleen NAS `clbu_full` → USB
- `BuNasToExtHDDMonthly.sh` — staat in **geen enkele crontab**, vandaar dat de maandbackup niet werkt

**⚠️ Beide reeksen zijn voor de scripts niet te onderscheiden**: ze pakken het eerste `/dev/sd[b-z]` met een (leeg) `hdd.txt` en schrijven naar dezelfde map. Hangt er op een maandag een maandschijf aan, dan wist `rclone sync` de maandhistoriek. Markerbestand per reeks fixen vóór de maandtaak wordt ingepland.

**Besluit 2026-09-01: backup-rol verhuist naar eigen toestel `PC-BACKUP-01`** (VLAN 19, naast de NAS, ≥2,5 GbE, Debian 13 + Tier 1, geen scherm). Reden: de keten draaide op `PC-MONITORING-01`, hetzelfde toestel als het alarmscherm ([[project-netxms-monitoring]]) — historisch gegroeid, geen ontwerp. 1,1 TB per run hoort niet over de gateway te lopen, zeker niet met UniFi IDS in detection-modus op komst ([[project-network-detection]]). Bij de bouw: Ansible-rol `backup-offline`, **read-only** service-account i.p.v. persoonsgebonden account in klare tekst ([[feedback-service-accounts]]), geen uitgaand internet behalve statusmail via interne smarthost, en run-bewaking in NetXMS.

**Code klaar 2026-09-01** (commits `421f406` ansible / `0179100` handbook): rol `roles/backup-offline` + `backup.yml`, inventory-groep `backup_devices`, runbook **RB-2026-BACKUP-DEPLOY**, firewall-regels **B-001/B-002/B-003**, tracker **BU-1 t/m BU-5**. De drie oude scripts zijn één script met taakdefinities in `/etc/backup-offline/jobs/`; markerbestand met `set=` per schijfreeks dwingt af dat een taak niet op de verkeerde reeks draait. Geblokkeerd op: hardware, adres in VLAN 19 (10.19.0.20 voorgesteld, te bevestigen), read-only NAS-account `svc-backup-ro` + `vault_backup_nas_password`, en regel B-001 (per-VLAN-isolation binnen VLAN 19).

**Cloud-pull-VM `10.10.100.8` niet samenvoegen met PC-BACKUP-01** — tegengestelde blootstelling: die heeft internet + OAuth-token op de Google-tenant nodig en **schrijft** naar de NAS, terwijl de offline keten juist géén internet nodig heeft en **read-only** kan. Wel herbouwen als beheerde VM `SRVV-CLOUDBU-01` (staat nu niet in `inventory.yml`, geen ansible-account, nergens gedocumenteerd), in de serverband, niet in VLAN 19.

**Bacula + Bacularis** in de steigers op `10.10.100.6` (poort 9097), VLAN aan te passen; doel = centrale DB-backup met restore-pad. Zie [[project-hosting-fase1-status]] en `management-tools/bacula.md`.

**Onderliggende oorzaak gevonden 2026-09-15: IP-conflict met de oude gateway.** De oude UniFi-gateway had na de EFG-cutover nog interfaces op `.7` in meerdere VLANs (MAC `0c:ea:14:19:e4:11`) — ook op het NAS-adres. Bewezen via haproxy-2 (`10.21.0.7`, zelfde MAC in ARP); user haalde de `.7`-interfaces weg en bevestigde dat het NAS-conflict **de backup-problemen veroorzaakte**. De eerdere diagnose "NAS heeft geen werkende default gateway" was dus het symptoom (antwoorden gingen via ARP naar het verkeerde toestel), niet de oorzaak. Zie [[project-gateway-cutover]] en [[project-stepca-cert-incident-202609]].

**How to apply:** Bij netwerkwijzigingen die `10.10.100.7`, VLAN 19 of het backup-toestel raken: deze keten meewegen. Bij "host antwoordt alleen in eigen subnet": vergelijk de MAC in `ip neigh` met de echte MAC van de host — een vreemde MAC = IP-conflict. Ze is 5 dagen stil uitgevallen na de VLAN-verhuizing van 27/08 + EFG-cutover van 28/08 omdat ze nergens gedocumenteerd stond. Zie [[feedback-same-subnet-test-proves-nothing]] voor de diagnose-valkuil.

**Update 2026-09-17 — de Teamdrive-tak van deze keten lag 87 dagen stil.** De cloud-pull naar `clbu_full` draaide sinds 22 juni niet meer, dus de USB-schijven droegen in die periode Teamdrive-data van 22 juni; de Proxmox-tak was wél actueel. Zie [[project-teamdrive-backup-outage-202609]]. Twee gevolgen voor deze memory: er is **geen persoonlijk OAuth-token** maar een GCP-service-account met domain-wide delegation, en de cloud-pull verhuist niet naar een nieuwe `SRVV-CLOUDBU-01` maar wordt herbouwd op [[project-srvv-p-backup-01]], samen met Bacula. De scheiding met `PC-BACKUP-01` blijft wél overeind — die afweging verandert niet.

Ook bevestigd: het NAS-account `bob.benny` is persoonsgebonden en staat op `SRVV-P-BACKUP-01` **in klare tekst in `/etc/fstab` (mode 644)**. Tracker BU-6.
