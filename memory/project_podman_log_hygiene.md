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
| srvv-forgejo-01 | 10.35.0.11 | Forgejo (mgmt) | ✅ AL GEFIXT 2026-06-28 |
| tst-odoo-01 | 10.200.14.40 | TEST Odoo | te surveyen (non-prod) |
| dev-odoo-01 | 10.200.14.41 | DEV Odoo | te surveyen (non-prod) |
| srvv-id-01 | 10.35.0.12 | Keycloak | te surveyen (non-prod) |
| srvv-unifi-01 | 10.35.0.15 | UniFi OS Server | te surveyen (non-prod) |
| **srvv-odoo-01** | 10.36.0.40 | **PROD Odoo** | survey+fix vereist expliciete OK + window |
| **srvv-acc-01** | 10.36.0.42 | **PROD account-admin** | survey+fix vereist expliciete OK + window |

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
- **Open vraag voor user**: prod-onderhoudsvenster vastleggen.
- **WBS-registratie**: nog in `myschool.project` onder D-MON zetten (geblokkeerd op dode myschool-MCP-key sinds 2026-06-07, [[reference-mcp-projects-provider]]).

## Gerelateerd
- [[project-forgejo-status]] — incident-oorsprong + bewezen fix-details
- [[project-template-strategy]] — tier1-baseline (doel voor codificatie)
- [[feedback-ansible-check-mode-verify]] — verifieer changed echt op de targets
- [[project-infrastructure-params]] — VM-inventaris
