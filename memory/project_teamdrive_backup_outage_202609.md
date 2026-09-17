---
name: project-teamdrive-backup-outage-202609
description: "Teamdrive-backup lag 87 dagen stil (22/06-17/09/2026): 151 Shared Drives zonder verse kopie, USB-schijven even oud. Oorzaak read-only CIFS-mount, onzichtbaar door 9 fouten. Ansible-rol cloud-backup klaar; migratie naar SRVV-P-BACKUP-01 loopt."
metadata:
  type: project
---

**Ontdekt 2026-09-17.** De Google-Teamdrive-backup draaide sinds **22 juni 16:31** niet meer. 151 van 154 Shared Drives zonder verse kopie — personeelsadministratie, boekhouding, leerlingenbegeleiding, schoolbestuur, secretariaten, alle vakwerkgroepen. Omdat de offline USB-keten `clbu_full` kopieert, droegen **ook de drie weekschijven** Teamdrive-data van 22 juni; de Proxmox-kant daarvan was wél actueel.

**Aanleiding**: de CIFS-mount naar `//10.10.100.7/clbu_full` werd read-only (`Failed to copy: mkdir ...: read-only file system`). Waarschijnlijk het IP-conflict op `.7` met de oude gateway ([[project-gateway-cutover]]), maar niet te bewijzen — `dmesg` ging verloren bij de herstart van 17/09. **De NAS was op 17/09 gezond**: ARP gaf `1c:6a:1b:46:19:6f`, niet de oude gw-MAC; een verse mount was meteen weer `rw`.

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
