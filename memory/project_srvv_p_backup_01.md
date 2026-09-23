---
name: project-srvv-p-backup-01
description: "SRVV-P-BACKUP-01 (10.10.100.6, Debian 13): Bacula 15 + Bacularis 6 + PG17, draait sinds 2026-09 ook de Teamdrive-backup. Openstaand: machine-id dupliceert die van .8, Webmin op 0.0.0.0, Bacula luistert alleen op localhost, ufw i.p.v. nftables, VLAN-verhuizing naar 10.35.0.60."
metadata:
  type: project
---

Doelhost voor de geconsolideerde backup-rollen. Vastgesteld 2026-09-17 via read-only inventarisatie.

| | |
|---|---|
| OS | Debian 13 trixie — jullie standaard, geen migratieschuld |
| Bacula | 15.0.3 + Bacularis 6.0.0, director/sd/fd draaien, PostgreSQL 17 |
| NAS | share `bacula_bu` gemount op `/mnt/bacula_storage` |
| Resources | 2 vCPU, 7,7 GB RAM; `/` 28%, `/var` 77% |
| Tweede NIC | `ens19` aanwezig maar DOWN |

**Waarom hier en niet op een eigen VM.** De cloud-pull stond op `SRVV-CLOUDBACKUP-01` (Debian 11, EOL, onbeheerd). Tegen consolidatie pleit dat de Bacula-Director de backup-control-plane is en straks FD-credentials naar élke client draagt — daar uitgaand internet bijzetten verbreedt de host die je het minst wil verliezen. Vóór pleit dat de oude VM die Google-sleutel tóch al draagt, zonder beheer of monitoring, en dat bus-factor 1 "minder VMs" echt laat tellen. **Besluit: doen, mits afgeschermde rol** (eigen service-account zonder shell, systemd-hardening, vault-credentials, egress alleen naar de Google-API). Methode is **herbouwen uit code**, niet de VM verhuizen. Zie [[project-teamdrive-backup-outage-202609]].

**Openstaande punten (stand 2026-09-17):**
1. **Machine-id identiek aan `SRVV-CLOUDBACKUP-01`** (`24a516d46f714444a68e97843c763d1a`) — beide van hetzelfde template gekloond zonder reset. Botst op DHCP-DUID, journal-identificatie en monitoring-identiteit.
2. **Webmin luistert op `0.0.0.0:10000`** — dezelfde bevinding als SEC-5 op `PC-MONITORING-01`.
3. **Bacula luistert alleen op localhost** (9101/9102/9103 op 127.0.0.1) — kan dus geen enkele remote client backuppen. Dat is wat "in de steigers" concreet betekent.
4. **ufw actief, nftables inactief** — tegen ADR 0005 in; verklaart waarom 9097 (Bacularis) van buiten dicht is.
5. **Docker aanwezig, Podman niet** — tegen ADR 0004 in.
6. **NetXMS-agent wijst naar `10.10.100.2`**, de oude server, niet `10.35.0.20` ([[project-netxms-monitoring]]).
7. **NAS-credentials in klare tekst in `/etc/fstab`**, mode 644, op naam van `bob.benny` — tracker BU-6, zie [[project-offline-backup-chain]].
8. **VLAN-verhuizing naar `10.35.0.60`** (VLAN 35) staat al in `vm-inventory.md` gepland als `SRVV-BACULA-01`. Bewust **niet** samen met de backup-migratie uitgevoerd: twee wijzigingen tegelijk maken een storing onherleidbaar — de les van 22/06 ([[feedback-risk-aware-changes]]).

**Stand 2026-09-17 einde dag**: rol `cloud-backup` uitgerold en werkend. Beide NAS-shares gemount via fstab met het nieuwe service-account `svc-clbu-rw`, timers ingeschakeld (backup 01:17, health 06:00/18:00), `--list-only` detecteert alle Shared Drives, en de eerste volledige inhaalrun draait sinds 15:17. Machine-id vernieuwd (`775701c0…`).

**Opgelost sinds de eerste inventarisatie**: punt 1 (machine-id) en punt 7 (NAS-credentials — de cloud-backup-keten gebruikt nu `svc-clbu-rw` uit de vault; `bob.benny` staat nog wél in klare tekst in `/etc/fstab` voor de Bacula-mount, tracker BU-6).

**Nieuw gevonden**: `/var` is maar 2,9 GB en stond op 79% — zie [[project-template-fleet-defects]], TPL-1. Online te groeien met `lvextend -r`, er staat 9,91 GB vrij in de VG.

Ansible: groep `cloud_backup` in `inventory.yml`, playbook `cloud-backup.yml`, rol `roles/cloud-backup`.

## Bacula backupt niets — vastgesteld 2026-09-22, gepland na FRAME (BU-8)

Punt 3 van de inventarisatie heeft een concreet gevolg dat pas nu gemeten is. In de catalogus staan **266 jobs**, maar het is uitsluitend de standaardtaak `BackupCatalog`, en die eindigt sinds minstens 07/09 **elke nacht** op *Backup Error*: 0 bestanden, 0 bytes, 30 minuten retries. Het joblog zegt het letterlijk: `[DE0039] Unable to connect to Storage Daemon "bacula_storage" on SRVV-P-BACKUP-01:9103 — Verbinding is geweigerd`. De director belt de hostnaam, de SD luistert alleen op `127.0.0.1`. Op `/mnt/bacula_storage` staat één leeg bestand `test` uit november 2025. Geen alarm, dus het viel niemand op.

**Bewust geparkeerd tot na de FRAME-servers** ([[project-frame-frappe-hosting]]), op vraag van de user: eerst die werf af en de projectplanning overzichtelijk, dan Bacula. Tracker **BU-8**; de studie naar databasebeschikbaarheid (PITR / warme standby BAWA / HA-cluster) staat als **BU-9**.

**How to apply:** noem deze host niet "Bacula in de steigers" — er is een director, een catalogus en een planning, maar er is nooit één byte geschreven. Bij het oppakken: eerst de SD-bind (of `localhost` in de director-config), en voor Odoo hoort de filestore bij de database — een DB-only backup geeft een restore waarin elke bijlage stuk is.
