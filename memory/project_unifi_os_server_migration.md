---
name: project-unifi-os-server-migration
description: "Geplande werf: huidige UniFi-controller migreren naar SRVV-UNIFI-01 (Debian 13 + UniFi OS Server), VLAN 35 / 10.35.0.15, met firewall inform-rules NM-001/NM-002."
metadata:
  type: project
---

Migratie-werf (gepland, geen deadline — tracker-rij **NET-1**): huidige UniFi Network-controller verhuist naar een eigen VM **`SRVV-UNIFI-01`** met **UniFi OS Server** op **Debian 13 minimal**.

**Volledig draaiboek geschreven 2026-06-11**: `hosting/operations/migrate-unifi-controller.md` (**RB-2026-UNIFI-MIGRATE**), 6 fasen + rollback. Werf-taken opgezet (TaskList).

**Bron-controller (bevestigd 2026-06-11)**: GEEN CloudKey/UDM — klassieke self-hosted **`unifi`-package (Java+MongoDB) op een Debian 12-host**, Network-versie **10.0.162**, enkel de Network-applicatie (géén Protect/Access/Talk), beheert **switches + WAPs** (gateway = UniFi Gateway Pro, praat lokaal met de controller-host).

**Feasibility-verdict ✅ GO (2026-06-11)**:
- `.unf` Network-backup = exact het migratie-backup-type; met enkel Network is 100% van de config porteerbaar.
- Gateway Pro = extern-beheerde UXG (geen embedded controller zoals UDM) → volledig adopteerbaar door self-hosted controller; toekomstige Enterprise Fortress Gateway idem.
- **Versie-eis**: nieuwe controller moet Network **≥ 10.0.162** draaien vóór restore (anders "backup is newer than installed") → eerst updaten, dán restoren.
- UniFi OS Server-reqs (Podman ≥ 4.3.1, slirp4netns ≥ 1.2, systemd, libc ≥ 2.31) zitten in de Tier-1 baseline.
- **UniFi OS Server gekozen** (niet de klassieke package opnieuw) — Ubiquiti's aanbevolen next-gen self-hosting, containerized op Podman, lijnt met ADR 0004 + golden-image.
- **Huidige controller-IP = `10.10.100.15`** (VLAN 10-range). **`unifi.olvp.int` bestaat al** en wijst naar die controller → wordt de **cut-over-hefboom**: bij build alleen `SRVV-UNIFI-01.olvp.int`→.15 aanmaken, en pas op cut-over-moment `unifi.olvp.int` flippen `10.10.100.15`→`10.35.0.15`.
- **Inform-host bevestigd (2026-06-11)**: devices informeren via **hardcoded IP** `http://10.10.100.15:8080/inform` (gezien op een WAP), NIET via de hostname → kale DNS-flip werkt pas na conversie. **Strategie (Fase 0b in runbook)**: op de OUDE controller eerst `Override Inform Host = unifi.olvp.int` zetten + provisionen (geen downtime — hostname resolvt nu al naar .15-oud = 10.10.100.15) → daarna is de cut-over louter het DNS-record flippen `10.10.100.15`→`10.35.0.15` (Pad 1). Voorwaarde: devices kunnen `unifi.olvp.int` resolven via hun DHCP-DNS. Anders fallback Pad 2 = `set-inform http://10.35.0.15:8080/inform` per device. **Firewall NM-001/NM-002 moet actief zijn vóór de flip** (devices op hun VLAN → controller VLAN 35).
- **Fase 0b-test geslaagd (2026-06-11)**: `unifi.olvp.int` resolvt naar `10.10.100.15` op devices in 3 ranges (WAP 10.70.x, switch 10.10.110.x, WAP 10.1.111.x) → DNS-flip-cut-over bevestigd haalbaar voor de **10.x-estate**.
- **SCOPING-BESLISSING 192.168-devices (2026-06-11)**: aparte fysieke locatie **mét** routed/VPN-pad naar centraal → kandidaat voor een aparte UniFi *Site* op SRVV-UNIFI-01 (single-pane) óf een dedicated on-site controller. **Besloten: 10.x-estate eerst migreren (DNS-flip), 192.168 apart beslissen** ([[feedback-risk-aware-changes]], gefaseerd). ⚠️ **Maar "10.x eerst" isoleert 192.168 niet automatisch** als die op dezelfde huidige controller (10.10.100.15) staan: globale Override Inform Host (Fase 0b) pusht naar állen, `.unf`-restore neemt alle sites mee, DNS-flip verlegt iedereen die de hostname gebruikt. **Actie vóór globale Fase 0b**: ping-resolutietest óók op een 192.168-device. Resolvt+bereikbaar → laten meeliften naar centraal (splitsen kan later); resolvt niet/link onzeker → 192.168 niet via globale override, per-device set-inform op enkel 10.x + 192.168 apart regelen vóór decommission oude controller.

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
