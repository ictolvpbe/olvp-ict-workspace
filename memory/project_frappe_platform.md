---
name: project-frappe-platform
description: "Nieuw spoor (user, 2026-09-20): vier Frappe-servers, waarvan één live door de IT-dienst voor projectopvolging met MCP-toegang voor Claude. Nog niets gebouwd, niets vastgelegd in de repo's. Overlapt met Project H (Odoo als governance-SoT)."
metadata:
  type: project
---

Aangekondigd door de user op 2026-09-20, te starten zodra de Tier-1-baseline geseald is
([[project-template-fleet-defects]]). **Vier servers**, met twee doelen: het platform verder
ontwikkelen, en één server die de IT-dienst live gebruikt voor onder meer projectopvolging, waar
Claude via MCP mee praat.

Frappe komt **nergens** voor in `platform-handbook` of `platform-ansible` — enkel een
commentaarregel in `inventory.yml` verwijst naar een groep `frappe_servers` die niet bestaat.

## Openstaande beslissingen (nog niets van beslist)

- **Verhouding tot Project H.** Functioneel hetzelfde doel als [[project-mcp-and-itsm]]: tracking
  met MCP-toegang. Wordt Frappe de governance-SoT en wordt H omgelegd, of bestaan Odoo en Frappe
  naast elkaar? In dat tweede geval vervalt "single source of truth" — mag, maar dan expliciet.
- **Welke vier.** Vermoedelijk dev/test/acc/prod, zoals de intranet-werf. Te bevestigen.
- **Scope.** Frappe Framework met Helpdesk/Projects, of het volle ERPNext.
- **⚠️ MariaDB.** Frappe draait officieel op MariaDB; Postgres-ondersteuning is gedeeltelijk en
  voor ERPNext afgeraden. Dat botst met de Postgres-standaard van ADR 0002 en wordt dus een
  bewuste, vast te leggen uitzondering.
- **Containerisatie.** `frappe_docker` is multi-container (backend, frontend, websocket, twee
  queue-workers, scheduler, MariaDB, twee Redis). Onder Podman + Quadlet (ADR 0004) is dat het
  grootste stuk werk van de werf.
- **Schijfprofiel** M per ADR 0008; de live server mogelijk ruimer.
- **Auth** via Keycloak-OIDC (Frappe Social Login), en een least-privilege `claude-mcp`-account
  zoals bij Odoo.

## Volgorde

1. Baseline sealen + bewijzen met een wegwerpkloon
2. ADR 0011 — rol van Frappe tegenover Project H, de MariaDB-uitzondering, de vier omgevingen
3. RB-2026-FRAPPE-DEPLOY — vier klonen, Quadlet, Caddy + step-ca, OIDC, MCP als laatste fase
4. Bouwen, dev eerst

Gerelateerd: [[project-mcp-and-itsm]], [[project-template-strategy]], [[project-containerization]],
[[project-intranet-hosting]] (zelfde vierfasen-patroon), [[reference-mcp-projects-provider]].
