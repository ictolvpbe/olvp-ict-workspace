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
- **SCOPING-BESLISSING 192.168-devices (2026-06-11)**: aparte fysieke locatie **mét** routed/VPN-pad naar centraal → kandidaat voor een aparte UniFi *Site* op SRVV-UNIFI-01 (single-pane) óf een dedicated on-site controller. **Besloten: 10.x-estate eerst migreren (DNS-flip), 192.168 apart beslissen** ([[feedback-risk-aware-changes]], gefaseerd). ⚠️ **Maar "10.x eerst" isoleert 192.168 niet automatisch** als die op dezelfde huidige controller (10.10.100.15) staan: globale Override Inform Host (Fase 0b) pusht naar állen, `.unf`-restore neemt alle sites mee, DNS-flip verlegt iedereen die de hostname gebruikt. **Actie vóór globale Fase 0b**: ping-resolutietest óók op een 192.168-device.
- **192.168-RESOLUTIETEST GEDAAN (2026-06-12)** op `WAP-WD0B1-01` (UAP-AC-LR, 192.168.0.210, site **BZ**): `unifi.olvp.int` **resolvt** ✅ (→10.10.100.15), maar **géén pad naar VLAN 35** ❌ (`ping 10.35.0.10` = 100% loss; link reikt tot VLAN 10 niet 35). Device informeert via **korte naam** `http://unifi:8080/inform` (≠ hardcoded IP van de 10.x-WAP → heterogene inform-host-estate). **Besluit: BZ/192.168 kan NIET meeliften** — na DNS-flip zou de naam resolven maar de controller (10.35.0.15) onbereikbaar zijn → controller-loos. **BZ vóór globale Fase 0b + DNS-flip uit het centrale pad halen.** Sporen: (1) routing+firewall naar VLAN 35 uitbreiden (open diagnose: firewall-deny vs ontbrekende VPN-route — check op gateway of VLAN 35 gerouteerd is over de BZ-link); (2) **dedicated on-site controller voor BZ — aanbevolen**, minst risicovol, ontkoppelt de werven. 10.x-DNS-flip-migratie blijft ongewijzigd doorgaan.

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

**FASE 1 AFGEROND (2026-06-15) ✅:**
- ✅ Proxmox-clone: `srvv-unifi-01` op **`10.35.0.15`**, ping + SSH-22 open vanaf werkstation.
- ✅ Inventory-entry `unifi_servers > srvv-unifi-01` in `platform-ansible/inventory.yml`, mét `ansible_ssh_common_args: ""` (zie ProxyJump-gotcha hieronder) — **NOG NIET gecommit** (bewust, klaar om te committen nu baseline slaagde).
- ✅ **`ansible`-account bootstrap (RB provision-ansible-account)**: account bestond al (golden-image, uid 1001, sudo-groep) maar miste `~/.ssh/authorized_keys` → daarom faalde SSH, NIET de key. Canonical pubkey geplaatst (let op console-paste: 1e `s` viel weg → `sh-ed25519`, gefixt), pw locked, NOPASSWD-sudoers, `/etc/hosts`-regel `127.0.1.1 SRVV-UNIFI-01` (hostname-resolutie). De "verkeerde key"-diagnose van 2026-06-12 was VALS — `~/.ssh/ansible_olvp` IS de canonical key, zie gecorrigeerde [[project-workstation-ansible-key]] + [[feedback-semaphore-key-canonical]].
- ✅ **Baseline geslaagd**: `ok=17 changed=7 failed=0`. Prereqs ruim voldaan: **Podman 5.4.2, slirp4netns 1.2.1, glibc 2.41, systemd 257, Debian 13.5**. step-ca nog te bootstrappen (handmatige eenmalige actie — debug-reminder, geen fout).
- ⚠️ **ProxyJump-gotcha**: `group_vars/all/vars.yml` zet default `ansible_ssh_common_args: -o ProxyJump=ansible@10.35.0.2` voor álle hosts. Voor `.15` faalde dat met `UNREACHABLE port 65535 timed out` want jump-01→.15 = intra-VLAN-35 verkeer, geblokkeerd door per-VLAN-isolatie ([[project-firewall-strategy]]). Werkstation bereikt `.15` wél direct (admin-rule) → override `ansible_ssh_common_args: ""`. (Idem aandachtspunt voor latere Semaphore-deploy: runner .10 → .15 is óók intra-VLAN-35.)
- ⚠️ **AD-DNS**: `SRVV-UNIFI-01.olvp.int` resolvt nog **NXDOMAIN** bij AD-DC (10.10.0.10) → record nog aanmaken. `unifi.olvp.int` (cut-over-hefboom) bevestigd intact → 10.10.100.15.
- ⚠️ **Gateway-DNS-gotcha (2026-06-15)**: `unifi.olvp.int` resolvt verschillend per resolver — bij **AD-DC** correct → 10.10.100.15; bij de **UniFi-gateway (10.10.0.1)** als DNS-server → al z'n interface-IP's (.1/.5 op elk VLAN), want de gateway kaapt de naam `unifi`. Devices die de gateway als DHCP-DNS gebruiken zouden de DNS-flip-cut-over dus NIET volgen. Fase 0b-geteste devices gebruiken AD-DC (oké), maar af te dekken vóór de globale flip.

**FASE 2 AFGEROND (2026-06-15) ✅ — UniFi OS Server geïnstalleerd:**
- **Disk eerst uitgebreid**: clone had 50 GB met krappe verdeling (/ 7,7G, /var 2,9G, /home 27,5G leeg). UOS-eis = **min. 20 GB vrij, aanbevolen 40-50 GB**. User vergrootte vda in Proxmox naar **100 GB** (msdos: extended vda2 + logical vda5 met LVM). Online opgelost zonder reboot/shrink: `growpart` (cloud-guest-utils) op vda2+vda5 → `pvresize /dev/vda5` → `lvextend -r` **`/`→40G, `/var`→15G**. `/home` met rust gelaten (ruimte genoeg). VG-buffer ~15 GB. (growpart zat niet in baseline; /home-shrink bewust NIET gedaan want overbodig + risicovol offline.)
- **Installer**: UOS Server **5.1.15** linux-x64 van `fw-download.ubnt.com` (volledig = **874 MB**). ⚠️ Gotcha: eerste download naar `/tmp` (eigen LV, maar maar **551 MB**) gaf **truncated 496 MB**-bestand (curl ENOSPC, gemaskeerd door `2>/dev/null`); herdownload naar `/home/ansible` (26 GB) = volledig. Les: download grote bestanden NIET naar `/tmp` op deze golden-image (551 MB-LV) + check curl-rc.
- **Install**: `uosserver-installer --help` toont `--non-interactive`. Gedraaid als transient `systemd-run --unit=uos-install` (overleeft SSH-drop). `INSTALLATION COMPLETE`, exit 0, ~1,5 min. Data/binaries onder **`/var/lib/uosserver/`** (daarom was /var-groei juist), user `uosserver` (uid 1002), 2 systemd-services (`uosserver.service` + `-updater`), container **healthy**.
- **Netwerk-mode = `pasta`** (UOS 5.1.15 bundelt eigen pasta-binary en koos die, NIET slirp4netns) → de slirp4netns≥1.2-prereq uit het runbook is voor deze versie achterhaald.
- **Web-UI op poort `11443`** (NIET 443!) → `https://10.35.0.15:11443/`, geeft HTTP 200, bereikbaar vanaf werkstation. Inform-poort **8080** luistert ook. ⚠️ **Firewall-matrix A-002 gaat uit van 443 → moet 11443 worden** voor admin-toegang (af te dekken in Fase 5).
- Installer-bestand opgeruimd na install.

**Fase 2 volledig afgerond (2026-06-15)**: lokale UniFi OS-admin aangemaakt (browser, https://10.35.0.15:11443), en **Network-applicatie 10.4.57 stond al op de OS Server** (≥ 10.0.162 → restore-eis ruim voldaan, geen update nodig). Firewall-matrix A-002/A-003 al gecorrigeerd naar 11443 (handbook, doc-edit — commit-status: zie sessie).

**FASE 0b-PREP — estate volledig in kaart + flip-script gebouwd (2026-06-18):**
- **API-read op de OUDE controller** (`https://10.10.100.15:8443`, klassieke Network-API, login `zeus`): 1 site `default` (OLVP-Sint-Niklaas), **236 devices** = 174 AP (`uap`) + 62 switch (`usw`). 229 connected, 7 disconnected.
- **Inform-host-verdeling (vóór flip)**: 114 op hardcoded `10.10.100.15`, 37 op korte naam `unifi`, 85 op FQDN `unifi.olvp.int`. De 122 die via de naam informeren+connected zijn, bewijzen al resolutie+bereik.
- **Ping-test (alle 236, SSH `mca-cli-op` per device vanaf werkstation)**: 226 PING_OK, 2 partial (BZ-link), 8 SSH_FAIL. **227/228 resolven `unifi.olvp.int → 10.10.100.15` correct.**
- ⚠️ **Gateway-DNS-kaping LIVE bevestigd op `SW-PPARKING-02` (10.10.110.39)**: resolvt `unifi.olvp.int → 10.1.0.1` (gateway-interface, niet de controller). Nu veilig want informeert via hard IP; mág NIET mee-geflipt worden tot zijn DHCP-scope de AD-DC als DNS krijgt. = de gotcha uit Fase-1-status, nu echt waargenomen.
- **8 SSH_FAIL**: 7 stale/disconnected (5 letterlijk `-OUD`, state=0 → controller-opruimkandidaten) + 1 LIVE switch `SW-PP202-01` (10.10.110.226) die SSH weigert (verder gezond volgens controller).
- **BZ/192.168-poot volledig geënumereerd: 30 devices (8 sw + 22 ap, 192.168.0.200-235)**, allen resolven `unifi.olvp.int→10.10.100.15` en bereiken de OUDE controller (VLAN 10). Bevestigt: **split-off blijft verplicht** — na DNS-flip wijst de naam naar 10.35.0.15 (VLAN 35), voor hen onbereikbaar. Lijst beschikbaar voor dedicated on-site controller / routing-fix.
- **Doelgroep Fase 0b-flip = 125 10.x-devices** (107 op IP + 18 op korte naam), uitgesloten: 78 al-FQDN + 30 BZ + 1 gateway-kaping + 8 SSH/stale.
- **Script gebouwd**: `hosting/operations/unifi-fase0b-set-inform.sh` — flipt één-voor-één naar `unifi.olvp.int` met LIVE pre-checks (her-resolve+ping vlak vóór flip, skip bij resolve≠controller) + device-side post-check. `MODE=dry` default, `MODE=apply [LIMIT=N]` voor gefaseerde rollout.
- **APPLY GEDRAAID + GROTENDEELS AFGEROND (2026-06-18)**: 3 runs (1 + 10 + rest). **112 devices geflipt naar FQDN, 0 warnings, allen Connected gebleven.** Estate-stand inform-host: **85→197 op `unifi.olvp.int`**, 37→20 korte naam (≈BZ), 114→19 hard IP. Resterend bewust níét geflipt:
  - **~13 SSH-weigerende switches** (alle `usw`, op 10.10.110.x + SW-PC007-01 op 10.10.100.51): SW-PP101-01, PC112-01, PL208-C1-BACKUP, PK2T1-01, PG1B1-01, PP103-02, PN101-01, PP202-01, PA3B1-01, PC007-01, PC102-01, PM008-01, PP106-01. Per-device lukt niet → optie (a) SSH aanzetten via controller + herdraaien, of (b) controller-side globale Override Inform Host als sluitstuk (⚠️ raakt ook BZ → pas ná BZ-split-off).
  - **2e gateway-DNS-kaping ONTDEKT (live-check ving het)**: `WAP-PA104-01` (10.1.111.12) resolvt `unifi.olvp.int → 10.1.0.1`, idem `SW-PPARKING-02` (10.10.110.39). ⇒ de kaping is **niet scope-beperkt**; vóór de globale DNS-flip moeten **alle DHCP-scopes de AD-DC als DNS** krijgen i.p.v. de gateway. Deze 2: eerst DNS fixen, dan flippen.
  - **BZ/192.168-poot** (≈20 op korte naam) bewust onaangeroerd → split-off-beslissing.
- **Device-CLI-gotcha** (zie [[feedback-unifi-device-noninteractive-cli]]): `info`/`set-inform` zijn interactieve shell-aliassen → non-interactief `mca-cli-op info` / `mca-cli-op set-inform <url>`; káál `mca-cli-op` loopt oneindig (genereerde 154MB). SSH-automation: device-user `zeus`, `ssh -n` verplicht in while-read-loop, `SSH_ASKPASS`-truc voor pw.

**VOLGENDE — de eigenlijke cut-over (runbook RB-2026-UNIFI-MIGRATE, vakantie-window aanbevolen, géén dataplane-impact maar controller-down):**
- **Fase 0** (nu veilig, niet-destructief): `.unf`-backup downloaden van de OUDE controller (10.10.100.15) → Settings → Control Plane → Backups → Download. Bewaren buiten de oude host.
- **Fase 3**: vzdump nieuwe VM → oude `systemctl stop unifi` → `.unf` restoren op SRVV-UNIFI-01.
- **Fase 5 vóór Fase 4**: firewall NM-001 (8080) + NM-002 (3478) activeren zodat devices .15 bereiken.
- **Fase 4**: DNS-flip `unifi.olvp.int` 10.10.100.15→10.35.0.15 (na Fase 0b: devices informeren via hostname).
- ⚠️ **PREREQ cut-over**: BZ/192.168-devices vóór de globale Fase 0b/DNS-flip uit het centrale pad halen (dedicated on-site controller aanbevolen) — anders controller-loos. Zie scoping-besluit hierboven.
- **AD-DNS** `SRVV-UNIFI-01.olvp.int → 10.35.0.15` nog aanmaken (NXDOMAIN; user doet dit op de Windows-DC). `unifi.olvp.int` met rust laten.
- Werkstation-hygiëne: `~/Downloads/ansible_olvp(.pub)` (private key) verwijderen; `~/.ssh/known_hosts` chown naar kristof.
