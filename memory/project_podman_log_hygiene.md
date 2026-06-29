---
name: project-podman-log-hygiene
description: Werf — Podman health_status-events vullen /var/log via rsyslog op álle Podman-hosts met healthchecks. Ontdekt via FORGEJO-01-incident (/var 100%). Fix = rsyslog drop-filter + logrotate maxsize, te codificeren in tier1-baseline + retrofit bestaande hosts. Status: OPEN (discovery), 2026-06-28.
metadata:
  type: project
---

**Werf aangemaakt 2026-06-28.** Tracker: provisioneel onder **D-MON** (monitoring/observability) in de WBS — registreren in `myschool.project` zodra de myschool-MCP-key hersteld is (zie [[reference-mcp-projects-provider]], key dood sinds 2026-06-07).

## Probleem
Podman logt elk health-check-event (`container health_status ... health_status=healthy`) naar journald; rsyslog forwardt dat naar `/var/log/syslog` ÉN `/var/log/user.log` (dubbel). Bij `HealthInterval=30s` per container is dat ~2880 events/dag/container. Default-logrotate roteert **dagelijks, niet op grootte** → de logbestanden groeien ongebreideld tot GB's. Sluipend: geen alarm tot `/var` 100% raakt en container-builds/writes falen.

Ontdekt toen `/var` op **SRVV-FORGEJO-01** 100% vol liep (12G, ~9,6G alleen syslog+user.log) en een podman-build met `no space left on device` faalde. Zie incident-detail in [[project-forgejo-status]].

## Fix (bewezen op FORGEJO-01)
1. **rsyslog drop-filter** `/etc/rsyslog.d/49-drop-podman-health.conf`:
   `if ($programname == "podman") and ($msg contains "health_status") then stop`
   (health blijft queryable via `podman inspect` / journald; alleen de tekstlog-spam verdwijnt). `systemctl restart rsyslog`.
2. **logrotate size-cap backstop** in `/etc/logrotate.d/rsyslog`: `maxsize 200M` + `compress` op de syslog/user.log-stanza.
3. **Cleanup** indien al volgelopen: `truncate -s 0 /var/log/{syslog,user.log}` + `rm` van de `.1`-kopieën. (truncate, GEEN rm op de actieve inode.)
4. **HealthInterval** bewust NIET aangepast — de rsyslog-filter lost de bron al op zonder container-restart.

## Scope — Podman-hosts met healthchecks (uit inventory.yml)
| Host | IP | Rol | Survey/fix |
|---|---|---|---|
| srvv-forgejo-01 | 10.35.0.11 | Forgejo (mgmt) | ✅ GEFIXT 2026-06-28 |
| tst-odoo-01 | 10.200.14.40 | TEST Odoo | ✅ GEFIXT 2026-06-29 (was 51% /var, ~437M logs) |
| dev-odoo-01 | 10.200.14.41 | DEV Odoo | ✅ GEFIXT 2026-06-29 (was 55% /var, ~509M logs) |
| srvv-id-01 | 10.35.0.12 | Keycloak | ✅ GEFIXT 2026-06-29 (was 21% /var, mild) |
| srvv-unifi-01 | 10.35.0.15 | UniFi OS Server | ✅ GEFIXT 2026-06-29 (was 6% /var, mild) |
| srvv-odoo-01 | 10.36.0.40 | PROD Odoo | ✅ GEFIXT 2026-06-29 (was 52% /var, 4 healthy, user-OK "niet in gebruik") |
| srvv-acc-01 | 10.36.0.42 | PROD account-admin | ✅ GEFIXT 2026-06-29 (was 51% /var, 4 healthy) |

**ALLE Podman-healthcheck-hosts gedekt 2026-06-29.** Werf D-MON fix-deel afgerond; rest = WBS-registratie + optionele playbook-validatie.

HAProxy-1/2 + pve-nodes: vermoedelijk geen Podman-healthchecks (HAProxy native, PVE = hypervisor) → waarschijnlijk N.V.T., bevestigen.

## Plan (fasering)
1. **Strategisch (root-cause-codificatie)**: rsyslog drop-filter + logrotate `maxsize` toevoegen aan **`tier1-baseline.yml`** (golden-image baseline) → élke nieuwe Tier-1-VM krijgt het automatisch. Dit is de echte fix; retrofit is de inhaalslag. Zie [[project-template-strategy]].
2. **Retrofit non-prod** (tst/dev-odoo, id, unifi): survey → fix uitrollen (kan via de baseline-playbook of gericht). Laag risico.
3. **Retrofit prod** (srvv-odoo-01, srvv-acc-01): in onderhoudsvenster, met OK; read-only survey eerst om severity te kennen (kan al kritiek zijn).
4. **Verificatie**: per host `/var`-% gedaald/stabiel + 0 nieuwe health_status-regels na rsyslog-restart.

## Status
- 2026-06-28: werf aangemaakt. FORGEJO-01 gefixt (referentie-implementatie).
- 2026-06-28: **Fase 1 (baseline-codificatie) KLAAR** — commit `10a986e` op platform-ansible: `tier1-baseline.yml` heeft nu rsyslog drop-filter (`/etc/rsyslog.d/49-drop-podman-health.conf`, gated op rsyslog-aanwezigheid) + globale logrotate `maxsize 200M` + `Restart rsyslog` handler. Elke NIEUWE Tier-1-VM krijgt het automatisch. Nog niet via playbook-run getest (vault-pw vereist; tier1-baseline raakt group_vars/all/vault auto-load).
- **GEPLAND voor binnenkort (richt: begin juli 2026)** — retrofit bestaande hosts via `ansible-playbook tier1-baseline.yml --tags log-hygiene --limit <host>` (idempotent, raakt enkel rsyslog/logrotate, geen stack-restart). Volgorde:
  1. **Non-prod eerst** (tst-odoo-01, dev-odoo-01, srvv-id-01, srvv-unifi-01) — laag risico, normale uren. Survey loopt mee (severity zichtbaar bij de run).
  2. **Prod** (srvv-odoo-01, srvv-acc-01) — in een onderhoudsvenster, met expliciete OK van de user. Read-only severity-check eerst (kan al kritiek zijn → dan eerder inplannen).
- **REMINDER gezet 2026-06-28**: cloud-routine `trig_01MFXyFp64NXLV74WQx7S6Po` vuurde 2026-06-29 09:00 CEST als checklist-reminder.
- **2026-06-29: Fase 2 (non-prod retrofit) ✅ KLAAR.** Uitgerold op tst-odoo-01, dev-odoo-01, srvv-id-01, srvv-unifi-01. **Methode: DIRECT via SSH** (user-keuze), NIET via playbook — dus de baseline-codificatie (`tier1-baseline.yml`, commit 10a986e) is nog steeds **niet via een echte playbook-run getest** (alleen handmatig gerepliceerd). Script = exact identiek aan de tag: drop-filter `/etc/rsyslog.d/49-drop-podman-health.conf` + logrotate `maxsize 200M` in `/etc/logrotate.conf` (idempotent, insertbefore include-line) + `systemctl restart rsyslog` + truncate syslog/user.log + rm geroteerde kopieën. Vanaf werkstation directe SSH (`~/.ssh/ansible_olvp`, user `ansible`, NOPASSWD sudo) — geen vault nodig.
  - **Verificatie OK alle 4**: journald krijgt nog health-events (8/8/2/1 in 60s → health queryable blijft), `/var/log/syslog` = 0 health_status-regels (filter werkt), containers healthy (tst 4, dev 4, id 1).
  - **UniFi-gotcha**: `sudo podman ps` (root-store) toont 0 containers — UOS beheert zijn containers in een eigen store; health-events komen van transient `podman[...]`-invocaties (programname `podman`) → filter `$programname == "podman"` vangt ze tóch. UniFi gedekt.
  - Directe SSH werkt vanaf werkstation (VLAN 34) ondanks dat inventory-default ProxyJump is; tst/dev (VLAN 200/207) + id/unifi (VLAN 35) allemaal direct bereikbaar met de canonical key.

- **2026-06-29: Fase 3 (PROD) ✅ KLAAR.** User gaf expliciete OK ("werk prod ook maar af, is niet in gebruik momenteel") → geen apart window nodig (fix is HTTP-neutraal, geen container-restart). srvv-odoo-01 + srvv-acc-01 direct via SSH (VLAN36, werkstation bereikt beide direct). Survey: beide ~52%/51% /var, ~15k health-regels, 4 healthy containers elk. Na fix: journald 8 events/60s (queryable), syslog 0 health_status, 4 healthy. **Observatie**: `https://myschool.olvp.be/web/health` → HTTP 404 (snel, 0,1s) — NIET door deze wijziging (alleen rsyslog/logrotate geraakt); past bij "myschool.olvp.be niet in gebruik, wordt juli door intranet vervangen" ([[project-intranet-hosting]]). Keten antwoordt, alleen die route 404.

## TODO (resterend)
- [x] **Playbook-validatie ✅ KLAAR 2026-06-29** — user draaide `ansible-playbook tier1-baseline.yml -e target_limit=srvv-id-01 --tags log-hygiene --diff --ask-vault-pass` (eigen terminal; `!`-shell kan vault-prompt niet voeden → EOFError, draai in echte terminal). Idempotentie bevestigd via SSH: filter-`mtime` bleef `08:31:30` (= SSH-apply-tijd, niet de playbook-tijd) → `copy` herschreef niet → taak `ok` niet `changed`. `maxsize` niet gedupliceerd. Codificatie (commit 10a986e) bewezen idempotent; elke nieuwe Tier-1-VM krijgt het automatisch.
- [ ] **WBS-registratie** onder D-MON in `myschool.project` zodra myschool-MCP-key hersteld is (dood sinds 2026-06-07, [[reference-mcp-projects-provider]]).
- [x] **HAProxy-1/2 + pve → N.V.T. bevestigd 2026-06-29.** HAProxy-1 (10.21.0.6) + -2 (10.21.0.7): **Podman niet geïnstalleerd**, 0 health-spam, /var 28%/24%, haproxy native active → geen log-hygiene nodig. Bereik via bastion ProxyJump (`-o ProxyJump=ansible@10.35.0.2`); direct vanaf werkstation timeout (DMZ VLAN21). pve-node srv-pmclust-t01 (10.10.100.70): **niet bereikbaar** vanaf werkstation én bastion (PVE-mgmt-net 10.10.100.x geïsoleerd); functioneel N.V.T. want Proxmox = hypervisor (LXC/KVM, geen Podman). Definitieve check enkel in Proxmox-context-sessie ([[project-proxmox-upgrade]]).

## Gerelateerd
- [[project-forgejo-status]] — incident-oorsprong + bewezen fix-details
- [[project-template-strategy]] — tier1-baseline (doel voor codificatie)
- [[feedback-ansible-check-mode-verify]] — verifieer changed echt op de targets
- [[project-infrastructure-params]] — VM-inventaris
