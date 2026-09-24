---
name: project-glpi-srvv-glpi-01
description: "SRVV-GLPI-01 (10.10.100.9, VLAN 10): GLPI 11.0.4 ITSM, 33k tickets sinds 2012, draaide ongedocumenteerd buiten inventory. Mailgate-storing OPGELOST 2026-09-24 (PHP-CLI-tijdzone). Migratie naar VLAN 35 + Podman = werf L; GLPI blijft, Odoo-ITSM is langetermijn via FRAME."
metadata:
  type: project
---

**Werf L, geopend 2026-09-24.** GLPI stond tot die dag in **geen enkele repo**: niet in
`inventory.yml`, niet in het handbook, niet in memory — hetzelfde stille-dienst-patroon als de
cloud-pull-VM op `.8` en de Teamdrive-backup ([[project-teamdrive-backup-outage-202609]]).

## Gemeten stand 2026-09-24

| Veld | Waarde |
|---|---|
| Host | `srvv-glpi-01`, **10.10.100.9/16**, VLAN 10 (hoort in 35) |
| OS | Debian 13.3, kernel 6.12.63 |
| GLPI | **11.0.4**, in `/var/www/html/glpi`, var `/var/lib/glpi`, log `/var/log/glpi` |
| DB | MariaDB 11.8.3, schema `glpi`, **2082 MB / 671 tabellen** |
| Omvang | 33 476 tickets (sinds 2012-10-11), 1020 gebruikers, 1036 computers, 9238 documenten |
| Web | Apache 2.4.66 + **mod_php 8.2**, enkel HTTP :80, geen TLS |
| Cron | `/etc/cron.d/glpi-mailgate`, elke minuut, **PHP CLI 8.4** |
| Plugins | 21 stuks, waarvan 6 actief (state=4) en 3 in state=0 |
| Mailbox | POP3 `pop.gmail.com:995` voor `ict@olvp.be` (collector 9, actief) |
| Machine-id | `92a3746deda242d1953474106cca1275` (uniek, níét de vlootduplicaat) |
| Agents | **480** GLPI-agents (245 actief laatste week), 480 **unieke** deviceid's |
| DNS | `glpi.olvp.int` + `srvv-glpi-01.olvp.int` → 10.10.100.9; vhost heeft géén `ServerName` |

## De mailgate-storing — opgelost bewijs, oorzaak was de tijdzone

Symptoom: mailgate moest elke ochtend manueel gestart worden.

**Oorzaak**: `date.timezone` staat **niet** gezet in `/etc/php/8.4/cli/php.ini`, dus PHP-CLI valt
terug op **UTC**, terwijl het systeem op `Europe/Brussels` (+02:00) staat. De crontask `mailgate`
heeft `hourmin=7 / hourmax=19`. De cron draait op PHP-CLI en ziet dus een uur dat **twee uur
achterloopt** → effectief venster **09:00–21:00 wandklok** in plaats van 07:00–19:00.

Bewezen met `glpi_crontasklogs`: vier dagen op rij starten de runs in uur 09 en lopen ze door tot
en met uur 20. De losse 1–2 runs in uur 08 zijn de manuele starts.

**Waarom manueel starten wél werkt**: `/etc/php/8.2/apache2/php.ini` heeft
`date.timezone = Europe/Brussels` wél staan. De webinterface rekent dus correct, de cron niet.
Web en cron draaien bovendien op **verschillende PHP-versies** (8.2 vs 8.4).

**Fix uitgevoerd 2026-09-24**: `date.timezone = Europe/Brussels` in `/etc/php/8.4/cli/php.ini`
(origineel als `.bak-20260924`). Geverifieerd: PHP-CLI = systeemklok, cron draait met `errors=0`.
`core/timezone` in `glpi_configs` staat op `0`, dus GLPI forceert niets en volgt de php.ini.

**Het uurvenster blijft bewust 07:00–19:00** (afweging ICT): twee uur eerder 's ochtends dan
vóór de fix, maar ook twee uur vroeger 's avonds. Mail van na 19:00 wacht tot 07:00.

⚠️ **De fix is broos**: alleen de php.ini van 8.4 is ingevuld terwijl er twaalf PHP-versies op de
host staan. Wisselt de default-`php`, dan keert de storing stil terug. Hoort in de baseline-role.

**How to apply**: bij "taak draait wel maar op het verkeerde moment" — vergelijk `date('H')` van
**PHP-CLI** met `timedatectl`, niet met de DB-timestamps. MariaDB stond hier op `SYSTEM` en
schreef dus correcte lokale tijden weg; juist daardoor zag alles er in de logs gezond uit en
wees niets naar de tijdzone. Zie [[feedback-php-cli-timezone-cron]].

## Nevenbevindingen (niet de oorzaak, wel op te ruimen)

1. **Twaalf PHP-versies** geïnstalleerd: 5.6, 7.0–7.4, 8.0–8.5. Legacy-erfenis.
2. `wakeupAgents` (crontask 75) staat sinds **2025-08-23** in `state=2` (RUNNING) — 13 maanden
   vast. Verschijnt als `stuck` in `status.php`.
3. `Kan de DB niet ontgrendelen` in `cron.log`, **elke minuut**. Vermoedelijk de race tussen de
   systeem-cron en de web-trigger: de login-pagina bevat nog
   `<div style="background-image: url('/glpi/front/cron.php')">`, dus GLPI-modus staat óók aan.
4. Cron roept het verouderde `front/cron.php` aan i.p.v. `bin/console glpi:cron` (GLPI 11).
5. In de cron-regel staat `&>/dev/null`. Cron gebruikt `/bin/sh` (dash), waar `&>` **niet**
   bestaat: dash leest dat als "achtergrond + lege redirect". Alle fouten gaan verloren.
6. Collector gebruikt **POP3** met `collect_only_unread=1` — POP3 kent geen unread-flag.
7. `php-errors.log` is **17 MB**, vol met `glpi_plugin_accounts_accounttypes`-relatiewaarschuwingen.
8. Geen TLS (poort 443 dicht, aanmelding over kaal HTTP op VLAN 10), geen ansible-account tot
   2026-09-24, niet in Semaphore, niet in NetXMS.
9. Webmin op `0.0.0.0:10000` (beheert ook de Apache-config), `iperf3` permanent op
   `0.0.0.0:5201`, en **geen firewall** (nft leeg, ufw uit). Tracker SEC-6.
10. **Back-up bestaat wél** (bijgesteld door ICT 2026-09-24): Proxmox maakt 2×/week een
   VM-backup. Maar die is crash-consistent, niet applicatie-consistent, en is nooit
   teruggezet. Openstaand: restore-test + nachtelijke `mysqldump`.

## De agents zijn het kritieke pad (aangebracht door ICT, 2026-09-24)

Op de Windows-toestellen draait de GLPI-agent en die praat **op IP**. Bewezen uit het access-log:
`"GLPI-Agent_v1.14" host=10.10.100.9`. Verhuist de server zonder meer, dan stopt de inventaris
**stil** — een agent die zijn server niet vindt klaagt nergens, hij verdwijnt uit `last_contact`.

**Meevaller**: `glpi.olvp.int` bestaat al en wijst naar .9, en de vhost heeft geen `ServerName`,
dus hij aanvaardt nu al elke Host-header. Agents die op naam praten werken vandaag al.

**Strategie: DNS eerst, dual-IP enkel als vangnet.** De volgorde is bewust omgekeerd t.o.v. de
eerste ingeving (eerst twee IP's koppelen, clients later): zet de agents om via GPO **terwijl de
oude server draait** — mislukt dat, dan breekt er niets want het IP verandert niet. Pas als het
access-log **nul** agents op IP toont, migreren; de cutover is dan één DNS-wijziging.
Achterblijvers via **DNAT op de EFG**, níét via een tweede NIC — een been in VLAN 10 ondermijnt
precies de segmentatie die het doel is. Uitrol via GPP Immediate Task, zoals bij
[[project-netxms-windows-agents]].

**Access-logging aangezet 2026-09-24** (`/etc/apache2/conf-available/olvp-accesslog.conf`, met
`%{Host}i`): er werd sinds 2026-01-20 géén access-log meer geschreven. Bewust op server-niveau
en niet in de vhost, want die is door **Webmin** gegenereerd en wordt door Webmin herschreven.

**49 FusionInventory-agents** (v2.6/v2.3.17) posten naar `/glpi/plugins/fusioninventory/` en
krijgen **404** — die rapporteren al geruime tijd niet meer, zonder dat iets dat aangaf. ITSM-7.

## Order-plugin ODT's + database-omvang (2026-09-24)

**ODT-bestelbonnen** staan buiten de DB in `/var/lib/glpi/_plugins/order/templates/`
(`Bestelbon_BAPLE/BAWA/SO.odt`). ⚠️ Er staat een **byte-identieke duplicaatmap** `templates(1)/`
naast met **spaties** i.p.v. underscores, en `glpi_plugin_order_preferences.template` bevat één
voorkeur `[Bestelbon SO.odt]` — mét spatie. Alleen de nette map meenemen breekt die gebruiker,
pas op het moment dat er besteld moet worden. Corrigeer de voorkeur vóór de migratie. ITSM-9.

**Database 2082 MB — de tickets zijn de bulk niet.** `glpi_logs` was 1064 MB (51%),
`glpi_tickets` 353 MB. Van de 6,1 M logrijen was **50,9% itemtype `CronTask`**: de cron die elke
minuut `lastrun` bijwerkt, elke wijziging in de audit-historiek. ~540 MB zonder informatiewaarde,
**opgeruimd 2026-09-24**: 3 105 914 rijen in 63 batches, dump als vangnet
(`/var/backups/glpi/glpi_logs_pre-purge_20260924.sql.gz`, 74 MB). ITSM-8.

⚠️ **Meet zo'n opschoning niet in tabelgrootte.** InnoDB geeft na een DELETE niets terug aan het
bestandssysteem en het bestand groeit zelfs door de undo-administratie. Gemeten over beide
opschoningen samen: `glpi_logs` 1064 → **1179 MB**, `glpi_tickets` 353 → **429 MB**, database
2082 → **2272 MB**. De database werd op schijf dus *groter*.

De winst zit in de **gzip-dump**: ~165 MB vóór alles → **107 MB** erna, ongeveer een derde.
Dat is véél minder dan de 540 MB ruwe tabelruimte, want gzip comprimeert repetitieve logrijen
uitstekend — **reken ruwe tabelruimte nooit één-op-één door naar dumpgrootte** (fout die ik
tijdens deze werf maakte). Echte opbrengst: 3,1 M logrijen + 4911 tickets minder te doorzoeken
en te herstellen. `OPTIMIZE TABLE` bewust niet gedraaid; de restore op de nieuwe server bouwt
compact op.

**Prullenbak (5274 tickets)**: script klaar als `/usr/local/sbin/glpi-purge-trash.php`, kopie in
`platform-ansible/files/glpi/`. Dry-run standaard, `--older-than=N` verplicht in de praktijk (er
wordt dagelijks weggegooid; met 30 dagen 4911 van 5274). **Uitgevoerd 2026-09-24**: 4911 verwijderd, 0 mislukt, 363 recente bewaard door de 30-dagengrens.
**Wezencontrole: 0** in itilfollowups/tickets_users/documents_items/tickettasks — bewijs dat
GLPI's eigen delete de gekoppelde rijen meeneemt. Tickets 33 476 → 28 573.
Vier lessen uit het schrijven: GLPI 11 bootstrapt **niet** meer via `inc/includes.php` maar via
`vendor/autoload.php` + `new \Glpi\Kernel\Kernel()` + `boot()` (zoals `bin/console`);
**`GLPI_ROOT` niet zelf definiëren** want GLPI doet dat via `Safe\define()` en dat is een fatal
bij hergebruik; en gebruik GLPI's eigen `Ticket::delete($id, true)` i.p.v. SQL, anders blijven
followups/documents_items/tickets_users als wezen staan. En: GLPI's `DBmysqlIterator` levert de
**ticket-id als array-key**, niet een numerieke index — een voortgangsteller uit de foreach-key
toont dus ticket-id's ("94000/4911") in plaats van voortgang.

⚠️ **`purgeticket` NIET aanzetten zonder bewaartermijnbeleid.** Hij ruimt de prullenbak *niet* op
— hij verwijdert **gesloten tickets definitief** op basis van `autopurge_delay` per entiteit.
Alle 10 entiteiten staan op `-10` (CONFIG_NEVER), dus hij doet nu niets. **De valstrik is de
waarde `0`**: die betekent niet "uit" maar "alle gesloten tickets onmiddellijk", want het
`closedate`-filter wordt alleen toegevoegd als `delay > 0`. 27 997 gesloten tickets zouden in één
keer definitief verdwijnen. "Uit" is `-10`.

GLPI heeft **geen ingebouwde ticket-archivering op ouderdom**. 21 196 tickets >2 jaar (waarvan 3
nog niet afgesloten), 2792 documenten eraan gekoppeld, 5266 in de prullenbak. Maatwerk over ~15
tabellen voor ~210 MB winst — alleen zinvol als het doel een **bewaartermijn** (GDPR/DPIA) is,
niet als het doel "kleiner" is.

## Migratiekoers (beslist 2026-09-24 door ICT)

Herbouw op een **verse Tier-1-kloon** `SRVV-GLPI-01` in **VLAN 35** (mgmt) — de server wordt
enkel door ICT gebruikt en moet **niet extern bereikbaar** zijn. Podman + Quadlet (ADR 0004),
data via DB-dump + `/var/lib/glpi`-rsync. Oude host blijft staan als rollback.
**Project H uitgeklaard 2026-09-24**: de ITSM-migratie naar Odoo is een **langetermijnspoor** en
landt in het **FRAME-project**. Containeriseren van GLPI is dus geen weggegooid werk — GLPI
blijft voorlopig de ITSM-tool. Zie [[project-frame-frappe-hosting]], [[project-mcp-and-itsm]],
[[project-template-strategy]], [[feedback-tier1-kloon-procedure]].

Tracker: ITSM-1 (restore-test) · ITSM-2 (✅ mailgate) · ITSM-3 (migratie) · ITSM-4 (TLS) ·
ITSM-5 (hygiëne) · **ITSM-6 (agents naar DNS — kritiek pad)** · ITSM-7 (dode FI-agents) ·
SEC-6 (Webmin/iperf3/firewall) · ITSM-8 (DB-opschoning) · ITSM-9 (ODT-bestelbonnen) ·
ITSM-10 (3 oude 'nieuwe' tickets).
