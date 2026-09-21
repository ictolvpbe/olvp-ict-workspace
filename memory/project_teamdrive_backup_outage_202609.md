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

## Eindstand 2026-09-17 15:10 — draait op SRVV-P-BACKUP-01

De rol `cloud-backup` is uitgerold op [[project-srvv-p-backup-01]] en `--list-only` detecteert alle Shared Drives. De eerste volledige run is gestart.

**Correctie op mijn conclusie van 12:00.** Ik schreef toen stellig dat het read-only worden een **NAS-probleem** was, op grond van het verdwijnen van `serverino` bij een reconnect. Dat is niet bewezen. Op de nieuwe host — **kernel 6.12 tegenover 5.10**, cifs-utils 7.4 tegenover 6.11 — houden beide mounts `serverino` en blijven ze schrijfbaar, met een vers service-account. Twee dingen veranderden tegelijk (kernel én account), dus welke van beide de doorslag gaf is niet vastgesteld. De oude kernel is de waarschijnlijkste verdachte; de NAS zelf bleef de hele dag gezond en bediende ondertussen een andere share zonder klacht.

**Nog vier bugs gevonden tijdens de migratie**, bovenop de negen van vanochtend — allemaal van dezelfde soort: code die werkte omdat de omgeving er toevallig naar gevormd was.

| # | Fout | Gevolg |
|---|---|---|
| 10 | `main()` deed `mkdir "${LOG_DIR}"` vóór `load_config` | logmap werd met de ingebakken standaard aangemaakt; op een andere host meteen "Toegang geweigerd" |
| 11 | rclone werd aangeroepen zonder `--config` | werkte onder systemd (`Environment=`) maar niet handmatig → "remote 'gdrive' not found" |
| 12 | `cloud-backup.service` had `[Install] WantedBy=multi-user.target` | een `systemctl enable` zou bij élke boot een volledige backup starten |
| 13 | `flock` als apt-pakket opgevoerd | bestaat niet in Debian; zit in `util-linux` |

**How to apply (aangevuld)**: een script dat "al jaren werkt" op één host bevat vrijwel zeker aannames over die host. Verhuizen naar een rol legt ze één voor één bloot — reken daar tijd voor in, en leg elke aanname vast in plaats van hem alleen te repareren.

## Waar het morgen verder gaat (stand 2026-09-17 einde dag)

De inhaalrun draait sinds 15:17 op `SRVV-P-BACKUP-01`. Controleren met drie getallen: afgeronde drives (doel 151), `read-only file system`-fouten (moet **0** blijven) en `df -h /var`.

**Waarneming die de diagnose opnieuw bijstelt**: ook op kernel 6.12 raakte `CLBU-FULL` onderweg `serverino` kwijt — er is dus óók daar een reconnect geweest — maar de mount bleef schrijfbaar, nul fouten. Het verlies van `serverino` zegt dus alleen dát er een sessie is heropgebouwd, niet dat die degradeert. Mijn eerdere koppeling tussen die twee was te snel. Dat schuift de verdenking naar het oude account `bob.benny`, dat op de NAS in een rare staat bleek (het moest opnieuw aangemaakt worden, en een eerste poging strandde op [[feedback-unas-local-account-smb]]).

**Open, in volgorde:**
1. Run afwachten; pas na een **tweede** geslaagde run mag `SRVV-CLOUDBACKUP-01` weg.
2. `netxms-agent.yml -e target_limit=srvv-p-backup-01` — zonder dat is er nog steeds geen alarm, alleen mail. Dat was de kern van deze storing.
3. `/var` groeien (TPL-1), `apt clean`, en de 950 MB dode containerd-data.
4. `svc-clbu-rw` terug van owner naar editor (least privilege).
5. BU-6: `bob.benny` roteren, uit `/etc/fstab` van `.6`, en `svc-bacula-rw` + de USB-keten op `PC-MONITORING-01` meenemen.
6. Vóór het opruimen van `.8`: de test `svc-clbu-rw` op kernel 5.10, en het oude service-account-token uit 2023 (`/opt/olvp-rclone-387510-*.json`) intrekken in de Google-admin.
7. rclone is op de nieuwe host 1.60 (Debian) tegenover 1.73 op de oude — overwegen uit de officiële bron te halen en in de rol vast te leggen.

## Nacontrole 2026-09-21 — de keten draaide, maar meldde niets

Vier dagen na de migratie nagekeken. De backup liep elke nacht, maar er was **geen enkele mail** aangekomen en er stond nog steeds geen alarm. Zes fouten erbij, bovenop de dertien van 17/09 — allemaal van dezelfde soort: code die werkte omdat de omgeving er toevallig naar gevormd was, plus twee die ik zelf introduceerde.

| # | Fout | Gevolg |
|---|---|---|
| 14 | Rapportmail ging via `mpack`, dat in geen enkele pakketlijst stond | script logde "mpack not found" en ging door; sinds de uitrol nooit gemaild |
| 15 | Exim4 stond als `dc_eximconfig_configtype='local'`, loopback-only, geen smarthost | post kon de host niet verlaten; `mainlog` had geen enkele afleveringspoging |
| 16 | `health-check.sh` keek naar `teamdrive-backup.timer` en `/var/run/teamdrive-backup.lock` — de namen uit de handmatige opzet | twee keer per dag onterecht UNHEALTHY + een gefaalde unit |
| 17 | 64 van de 153 drives elke nacht op FAIL door 2.613 `dangling shortcut`-fouten en macOS-rommel | `summary.failed` betekende niets meer, precies het mechanisme van juni |
| 18 | `discover.conf` werd niet uitgerold → `discover-drives.py` stierf bij élke run op `PermissionError` | discovery draaide nooit; en omdat rclone **niet** impersoneert (`subject =` leeg) betekent dat: een nieuwe Shared Drive komt stil nooit in de backup |
| 19 | `discover-drives.py --config` las altijd de standaardconfiguratie | een vlag die stil iets anders doet dan ze belooft |

**Twee fouten van mezelf, gevonden doordat ik het getest heb.** De outbox-unit is `Type=oneshot`; systemd ruimde de cgroup op en legde daarmee het aflever-proces om dat `sendmail` had afgesplitst — exim logde de ontvangst en verder niets. En de nieuwe lock-controle riep na élke geslaagde run "stale lock", terwijl een flock-bestand nu juist hoort te blijven liggen. Beide opgelost (`sendmail -odf`, en flock zelf laten antwoorden).

**Bewezen op 2026-09-21:** testrun `--drive ICT-INTRANET` = 251 MB in 15 s, 0 mislukt; mail afgeleverd bij `smtp-relay.gmail.com` met `250 2.0.0 OK` over TLS 1.3 (`CV=yes`); health-check exit 0; agent levert `hours/failed/drives/warned`.

**Discovery-dry-run relativeert de zorg**: 154 drives via impersonation, 153 toegankelijk, alleen `SO-FOTO'S` buiten — en dat is een grens, geen storing. Er was dus geen stille achterstand; het risico gold de toekomst.

**Mailketen**: Google Workspace SMTP-relay `smtp-relay.gmail.com::587`, **geen login — herkenning op het publieke IP** `91.183.159.250`. Afzender moet `@olvp.be` zijn (`dc_readhost` + `dc_hide_mailname`), anders weigert de relay. Let op IPv6: de allowlist is IPv4.

Werk staat op branch `backup/cloud-backup-rapportage` in `platform-ansible` (8 commits), tracker **BU-7**. Twee commits wachten nog op een uitrol: de discovery-exclusie en de flock-fix.

**How to apply.** Een keten die "draait" is niet hetzelfde als een keten die meldt. Controleer na elke uitrol drie dingen apart: draait de taak, klopt wat ze meet, en komt het signaal ergens aan. Alle zes fouten hierboven zaten in dat tweede en derde stuk. En bij een systemd-unit van het type oneshot: alles wat het script afsplitst sterft mee — post moet dus synchroon verstuurd worden, of door een aparte unit.

