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

Ansible: groep `cloud_backup` in `inventory.yml`, playbook `cloud-backup.yml`, rol `roles/cloud-backup`.
