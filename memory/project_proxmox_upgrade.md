---
name: project-proxmox-upgrade
description: "Werf Proxmox VE 8.4 -> 9 (EOL ~aug 2026). Test-cluster 'Datacenter2' eerst. Hyperconverged Ceph Reef 18.2.8 MOET eerst naar Squid 19.2, dan OS Bookworm->Trixie node-per-node. Runbook RB-2026-PVE89 + pve-config-backup role gepusht; volgende stap = LOKALE run (niet Semaphore)."
metadata:
  type: project
---

**Status 2026-06-17.** Proxmox VE upgraden van **8.4 -> 9** vóór EOL van PVE 8 (± augustus 2026). Eerst volledig op de **test-cluster**, daarna 1-op-1 op prod.

## Test-cluster (waar we testen)
- Naam **`Datacenter2`**, node **`srv-pmclust-t01`** = mgmt-IP **10.10.100.70** (corosync-ring `10.9.0.70/.71`).
- Bij de check 2026-06-16: **2 van 3 nodes** quorate (expected votes 3) → de 3de node bijzetten vóór de upgrade.
- **Hyperconverged Ceph Reef 18.2.8** (3 mon / 2 mgr / **12 OSD**), `HEALTH_OK`. Storages: `local`, `local-lvm`, `rdb` (RBD).
- Start-versie `pve-manager/8.4.19` (≥ 8.4.1 vereist ✅), 35 GB vrij op `/`.

## Upgrade-volgorde (bevestigd)
1. **Ceph eerst**: Reef 18.2.8 -> **Squid 19.2** (één sprong, géén Quincy-hop; Squid draait op PVE 8.2+ én 9). Mon -> mgr -> OSD, één node per keer, `noout` tijdens; afsluiten met `ceph osd require-osd-release squid`. Geen CephFS/MDS verwacht (pve8to9 gaf SKIP).
2. **Dan PVE 8 -> 9** node-per-node: repo's Bookworm->Trixie (deb822 `.sources`), `apt dist-upgrade`, reboot (kernel 6.14).

## pve8to9 --full bevindingen (2026-06-16, srv-pmclust-t01)
Schoon op één blokker na: **FAIL "Ceph 18 Reef too old"** (= verwacht, fase 1). WARN's: `noout` niet gezet; `systemd-boot` overbodig op legacy-boot (verwijderen); `intel-microcode` ontbreekt (non-free-firmware + installeren); locale `nl_BE.UTF-8` niet gegenereerd. **LVM-autoactivation al uitgeschakeld**, geen draaiende guests, certs/NTP ok.

## Geleverd + gepusht
- **Runbook** `platform-handbook/hosting/operations/upgrade-proxmox-8-to-9.md` (**RB-2026-PVE89**) — afvinkbaar, fase 0/1/2 + validatie/rollback/troubleshooting.
- **Ansible config-backup** `platform-ansible/`: role `roles/pve-config-backup/` + `pve-config-backup.yml` + inventory-groep **`pve_nodes`** (srv-pmclust-t01; t02/t03 als commentaar). On-node **systemd-timer** (dagelijks 03:30) snapshot van `/etc/pve`, netwerk, corosync, ceph, apt-sources + live-status; retentie 30d, `0600`/root, optionele NAS-sync via `-e pve_config_backup_remote=...`. Gevalideerd (YAML/ansible-syntax/jinja-render).

## Beslissing: LOKAAL draaien, NIET via Semaphore
`pve-config-backup` is **remote-first** -> hangt op de Semaphore-runner (ansible-core 2.20, zie [[feedback-semaphore-runner-remote-hang]]). De backup zelf is autonoom via de on-node timer; Semaphore is enkel deploy/refresh. Daarom lokaal uitrollen.

## VOLGENDE STAP (hier oppikken)
Lokale run-prereqs vanaf werkstation (VLAN 34):
1. **Netwerkpad** werkstation -> PVE-node `10.10.100.70` (apart mgmt-net `10.10.100.x` / `10.9.0.x` — routeerbaarheid te bevestigen).
2. **Root-SSH met sleutel** op de PVE-node (inventory zet `ansible_user: root` voor `pve_nodes`; PVE heeft geen `ansible`-user).
Dan: `ansible -i inventory.yml pve_nodes -m ping` -> `ansible-playbook -i inventory.yml pve-config-backup.yml --check --diff` -> echte run `-e pve_config_backup_run_now=true` (eerste snapshot vóór de upgrade). Pas daarna de Ceph- en OS-fasen volgens RB-2026-PVE89.

Gerelateerd: [[project-hosting-fase1-status]] (de VM's die op deze hypervisors draaien), [[project-gateway-cutover]] (apart lopend netwerk-werf).
</content>
