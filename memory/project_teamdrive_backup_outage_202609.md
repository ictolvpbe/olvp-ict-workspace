---
name: project-teamdrive-backup-outage-202609
description: "Teamdrive-backup lag 87 dagen stil (22/06-17/09/2026): 151 Shared Drives zonder verse kopie, USB-schijven even oud. Oorzaak read-only CIFS-mount, onzichtbaar door 9 fouten. Ansible-rol cloud-backup klaar; migratie naar SRVV-P-BACKUP-01 loopt."
metadata:
  type: project
---

**Ontdekt 2026-09-17.** De Google-Teamdrive-backup draaide sinds **22 juni 16:31** niet meer. 151 van 154 Shared Drives zonder verse kopie — personeelsadministratie, boekhouding, leerlingenbegeleiding, schoolbestuur, secretariaten, alle vakwerkgroepen. Omdat de offline USB-keten `clbu_full` kopieert, droegen **ook de drie weekschijven** Teamdrive-data van 22 juni; de Proxmox-kant daarvan was wél actueel.

**Aanleiding (bijgesteld 2026-09-17 12:00 — zie hieronder, het is NIET opgelost)**: de CIFS-mount naar `//10.10.100.7/clbu_full` werd read-only (`Failed to copy: mkdir ...: read-only file system`). Waarschijnlijk het IP-conflict op `.7` met de oude gateway ([[project-gateway-cutover]]), maar niet te bewijzen — `dmesg` ging verloren bij de herstart van 17/09. **De NAS was op 17/09 gezond**: ARP gaf `1c:6a:1b:46:19:6f`, niet de oude gw-MAC; een verse mount was meteen weer `rw`.

**Negen fouten die het drie maanden onzichtbaar hielden** (alle opgelost in de rol, commit `f90f1cb`):
1. Health-check meldde dagelijks correct `UNHEALTHY` — **per e-mail**. Niemand keek. Dit is de kern.
2. `rotate_logs` draaide alleen ná een geslaagde run → ruimde juist niet op toen het moest.
3. Logrotate-regel matchte per-run-bestandsnamen → van elk bestand 26 weken bewaard.
4. 13.928 logbestanden, 1,3 GB → `/var` 100% vol.
5. Elke logregel door `echo | tee` met `pipefail` → volle schijf gaf SIGPIPE → exit 141. Het logmechanisme doodde de backup.
6. `Restart=on-failure` + `RestartSec=300` → crash-loop van weken.
7. `rclone version --check | head -1` → netwerkoproep + race die willekeurig 141 gaf.
8. PID-lock aan `trap cleanup EXIT` → een tweede instantie wiste de lock van de lopende instantie.
9. Lock-map in `/run` zonder `tmpfiles.d` → stuk na elke reboot.

**Correcties op wat gedocumenteerd stond**: er is **geen persoonlijk OAuth-token** maar een GCP-service-account met domain-wide delegation (`gdrive-backup-bot@olvp-cloud-backup.iam.gserviceaccount.com`, impersonating `ict@olvp.be`). De VM heet `SRVV-CLOUDBACKUP-01`. `SO-FOTO'S` kan **structureel** niet mee: `teamDriveDomainUsersOnlyRestriction` sluit een extern service-account per definitie uit.

**How to apply.** Bij elke bewaking die alleen mailt: dat is geen alarm. Zet er een exit-code-check op ([[project-netxms-monitoring]], BU-3). Bij "backup draait niet": kijk eerst of het doel **schrijfbaar** is, niet alleen bereikbaar — een read-only CIFS-mount blijft read-only tot je hem vervangt, een herstart van de dienst helpt niet. En bij een script met `set -euo pipefail`: elke pipe naar `head`/`grep -q` is een latente 141. Zie [[project-offline-backup-chain]], [[feedback-same-subnet-test-proves-nothing]].

## ⚠️ Bijstelling 2026-09-17 12:00 — de oorzaak is NAS-zijdig en nog niet weg

De verse mount van 10:50 was schrijfbaar (`touch` in de root slaagde). De achterstand-run startte om 10:57:00 en gaf om **10:57:07** — zeven seconden later — opnieuw `read-only file system`, eerst op `mkdir /mnt/CLBU-DIFF/BAPLE-AN`, daarna op submappen in CLBU-FULL. Run gestopt om 12:01 na 2 afgeronde drives van 151.

**Het beslissende spoor zit in de mount-opties.** `/proc/mounts` meldt nog `rw`, maar CLBU-FULL is onderweg **`serverino` kwijtgeraakt** terwijl CLBU-DIFF het behield. Dat wijst op een **heropgebouwde CIFS-sessie** waarbij de NAS andere capabilities teruggaf. De kernel houdt de superblock op `rw`, maar elke `mkdir` krijgt EROFS van de server. Dat verklaart ook waarom `touch` in de root eerder wél lukte: die test liep vóór de reconnect.

**Conclusie: dit is een probleem van de UNAS Pro, niet van het script.** De negen fixes in de rol `cloud-backup` blijven terecht — ze maken de storing zichtbaar in plaats van stil — maar lossen dit niet op. Zolang de NAS de SMB-sessie onder schrijfbelasting laat omvallen, faalt elke host.

**Te onderzoeken op de NAS**: share-rechten van het account op `clbu_full`/`clbu_diff` (read-only of read-write?), SMB-sessielogs, firmwareversie. **Zuiverste test**: het nieuwe account `svc-clbu-rw` op een apart mountpoint, en dan `mkdir` van een geneste map plus een aanhoudende schrijfreeks — niet alleen een `touch`, want juist `mkdir` faalde. Tijdens die test `grep CLBU /proc/mounts` herhalen: verdwijnt `serverino`, dan is de reconnect gereproduceerd.

**How to apply (aangescherpt)**: bij een CIFS-doel is "de mount bestaat en zegt rw" geen bewijs dat je kan schrijven. Test met **`mkdir`**, niet met `touch`, en test aanhoudend — niet één keer. Een mount die omslaat na een reconnect ziet er in `df` en `/proc/mounts` volstrekt gezond uit.
