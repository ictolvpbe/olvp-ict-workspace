---
name: project-mcp-and-itsm
description: MCP-service en Odoo ITSM/PM-migratie zijn strategische Projecten G en H — doel is Claude-gestuurde updates van Odoo + governance-tracking in Odoo i.p.v. git/Sheets.
metadata:
  type: project
---

Sinds 2026-05-21 zijn Project G (MCP) en Project H (Odoo PM/ITSM-migratie) toegevoegd aan de programma-structuur — strategisch doel: Claude moet rechtstreeks via MCP Odoo kunnen updaten en governance-tracking gebeurt op termijn in Odoo i.p.v. git/Sheets.

**Status MCP (Project G):**
- Bestaande proof-of-concept lokaal draaiend
- Productie-hardening + deploy nog te doen: auth via id-instance, scope-beperkte permissions, rate-limit, audit-logging
- Parallel design mogelijk tijdens Fase 1 uitvoering; deploy pas na A's hard gates

**Status ITSM-app (Project H pre-requisite):**
- Bestaande basis aanwezig met: Tickets/Incidents + Knowledge-base, Change-management workflow, Asset-management/CMDB + Risk-register
- Nog niet aanwezig: SLA-tracking + reporting (mogelijk apart te bouwen)
- Odoo Project module wordt eerst geëvalueerd voor portfolio-management vóór beslissing om eigen `myschool_projectmanager`-module te bouwen

**Migratie-strategie:**
- Architectuur-docs, ADRs, runbook-tekst → **git blijft canonical** (docs-as-code voor versioning + PR-review)
- Risk-register, incident-log, change-log, tasks/projecten, SLA-tracking → **migreren naar Odoo** (workflow, audit, multi-user, MCP-toegankelijk)
- DPIA → canonical in git + audit-trail in Odoo
- Tracker-Sheet vervalt na migratie (gearchiveerd, niet verwijderd)

**Why:**
- Doel "Claude updatet via MCP" vereist MCP-service operationeel
- Doel "Odoo wordt single-source-of-truth voor governance" vereist ITSM-uitbreiding
- Volgorde: hosting eerst, dan MCP, dan PM-evaluatie + migratie — anders bootstrap-paradox

**How to apply:**
- Bij ITSM-uitbreiding of MCP-werk: nooit zonder check of A's hard gates klaar zijn
- MCP wordt SPOF voor Claude-productiviteit en moet in monitoring + risk + SLA opgenomen
- Odoo wordt na H-migratie bedrijfskritisch — **versterkt prioriteit van Project F (Postgres HA)**, ook al was die initieel Fase 3
- Tijdens H-migratie: bewaar git/Sheets-fallback tot Odoo-tracking bewezen stabiel is
- Collega in opleiding kan ITSM-uitbreiding mee oppakken (myschool-dev-track) — past in workforce-management

Gerelateerd: [[project-programma-structuur]], [[project-bus-factor]], [[project-identity-architecture]] (id-instance is auth-broker voor MCP), [[project-phasing]].
