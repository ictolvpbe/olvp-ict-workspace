---
name: project-unifi-os-server-migration
description: "Geplande werf: huidige UniFi-controller migreren naar SRVV-UNIFI-01 (Debian 13 + UniFi OS Server), VLAN 35 / 10.35.0.15, met firewall inform-rules NM-001/NM-002."
metadata:
  type: project
---

Migratie-werf (gepland, geen deadline — tracker-rij **NET-1**): huidige UniFi Network-controller verhuist naar een eigen VM **`SRVV-UNIFI-01`** met **UniFi OS Server** op **Debian 13 minimal**.

**Plaatsing (vastgelegd 2026-06-07):**
- VLAN **35** (`VL-SD-MGMTSVC`), IP **`10.35.0.15`** (gewijzigd van `.3` → `.15` op 2026-06-09, user-keuze; appliance-blok bij Semaphore `.10`/Forgejo `.11`/Keycloak `.12` i.p.v. core-blok `.2-.9`), DNS `SRVV-UNIFI-01.olvp.int` + alias `unifi.olvp.int`. Zie [[project-infrastructure-params]].

**OS/runtime-let-op:**
- UniFi OS Server draait op Linux **bovenop Podman ≥ 4.3 + slirp4netns ≥ 1.2** → sluit aan op Podman-default ([[project-containerization]], ADR 0004) en golden-image baseline ([[project-template-strategy]]).
- Ubiquiti noemt officieel **Debian 12+**; Debian 13 werkt in praktijk maar staat (nog) niet expliciet in de supportmatrix — geen blocker, wél melden bij support-case.

**Firewall (toegevoegd aan firewall-rules-matrix, status "Te activeren bij controller-migratie"):**
- Host-list **`UNIFI-DEVICES`** (switch/AP mgmt-IPs, TBD). Gateway praat lokaal met controller-host → geen inter-zone rule.
- **NM-001**: `UNIFI-DEVICES` → `10.35.0.15` TCP **8080** (device-inform).
- **NM-002**: `UNIFI-DEVICES` → `10.35.0.15` UDP **3478** (STUN).
- Beide met verplichte "Retourverkeer Automatisch Toestaan"-vink ([[feedback-unifi-new-vlan-zone]]).
- `SRVV-UNIFI-01` toegevoegd aan `MGMT-APPLIANCES` → admin-SSH (A-001) + web-UI 443 (A-002) dekken de controller al.
- L2-discovery (UDP 1900/10001 broadcast) routeert niet cross-zone; cross-VLAN-adoptie via `set-inform`/DHCP-optie-43/DNS naar `10.35.0.15`.

**Migratie-aanpak:** nieuwe VM → UniFi OS Server installeren → controller-backup (.unf) restoren → devices re-inform naar `10.35.0.15` → oude controller pas afbreken na verificatie. Restore-point (Proxmox-vzdump + .unf) vóór cut-over; window tijdens schoolvakantie (controller-down = geen adoptie/visibility, géén dataplane-impact).

**How to apply:** Ansible-inventory (`mgmt_servers`-groep) bewust NOG NIET ingevuld — pas bij effectieve provisioning. Doc-uitwerking: `network-physical/physical/network-equipment.md` ("Open werf"-sectie) + `reference/ip-plan.md`.
