---
name: project-myschool-repo
description: MySchool_addons Odoo-modules-repo bevat ~18 addons + sinds 2026-06-03 een centrale docs/-structuur (analoog aan platform-handbook). Branch-conventie master/Dev/Dev-* case-sensitive. Lokaal pad /home/demm/PyCharm/odoo-myschool/extra-addons/, remote github.com/ictolvpbe/MySchool_addons.
metadata:
  type: project
---

## Repo-locatie

- **Lokaal werkstation**: `/home/demm/PyCharm/odoo-myschool/extra-addons/`
- **GitHub**: `git@github.com:ictolvpbe/MySchool_addons.git`
- **Parent project**: `/home/demm/PyCharm/odoo-myschool/` (full PyCharm-project met Odoo-source, venv, config, data-dir)

## Lokale dev-setup (PyCharm)

- Python-interpreter: "Python 3.13 (PyCharm)" — PyCharm-managed venv
- `config/odoo.conf`: db_host=localhost, db_user=myschool, addons_path=/home/demm/PyCharm/odoo-myschool/extra-addons
- Beschikbare dev-DBs op localhost:5432: `odoo`, `odoo-dev` (heeft myschool_core + myschool_appfoundry installed), `odoo-dev-001`, `odoo-dev-002`
- Run-config in PyCharm — user start Odoo zelf via IDE, niet via bash-pad

## Branch-strategie (case-sensitive)

- **`master`** → productie (myschool, myschool-ict, id, myschool-acc). Wordt gepulled door Semaphore voor `target_env=prod`
- **`Dev`** → test + dev environments (myschool-test, myschool-dev, ...). `target_env=test`/`target_env=dev`
- **`Dev-<Topic>`** → feature-branches. Niet auto-pulled; test via Semaphore Survey-override `addons_branch_override=Dev-<Topic>`

Volledige conventie: in repo zelf via [docs/decisions/0002-branch-strategy.md](https://github.com/ictolvpbe/MySchool_addons/blob/Dev-Docs-structure/docs/decisions/0002-branch-strategy.md).

## Actieve feature-branches (stand 2026-06-03)

| Branch | Doel | Status |
|---|---|---|
| `Dev-MCP` | myschool_mcp deploy-spoor (cherry-pick uit Dev-Admin-and-Core) | live op GitHub, lokale test in progress |
| `Dev-Docs-structure` | docs/-submap opzetten | **gepushed 2026-06-03, klaar voor review/merge** |
| `Dev-Admin-and-Core` | grote refactor, 195 commits ahead van master | actief, lang openstaand |
| 12+ andere `Dev-*` | diverse | review pending, mogelijk stale |

## Docs-structuur (sinds 2026-06-03)

Branche `Dev-Docs-structure` bevat compleet centraal docs-systeem analoog aan `platform-handbook`:

```
docs/
├── architecture/  — overview, module-graph, mcp-architecture, data-model
├── modules/       — per-module overzicht (18 modules)
├── guides/        — gebruikers-handleidingen (legacy top-level files verplaatst)
├── operations/    — deploy-via-semaphore, module-install, branch-promote, backup-restore
├── reference/     — branches, environments, git-workflow
├── prompts/       — AI prompt-templates (legacy verplaatst)
└── decisions/     — ADRs: 0001 mcp-as-module, 0002 branch-strategy
```

Plus repo-root: `README.md`, `CONTRIBUTING.md`, `CLAUDE.md` (agent-context). 32 docs-files totaal, ~110 KB content.

**Status**: branch `Dev-Docs-structure` gepushed, niet gemerged. Toekomstige merge: ofwel direct naar `master` (puur additive, geen code-impact) ofwel via `Dev` voor consistentie.

## Belangrijke modules (categorisering)

- **Fundament**: `myschool_core` (orgs, persons, roles, periods, LDAP, BeTask, Informat), `myschool_admin`, `myschool_theme`, `myschool_sync`
- **Werknemers/activiteiten**: `activiteiten`, `afwezigen`, `planner`, `professionalisering`
- **Project + service**: `myschool_appfoundry` (dev-pm), `myschool_devhub` (alt-pm), `myschool_itsm` (ITIL), `myschool_asset` (CMDB), `process_mapper`, `knowledge_builder`, `myschool_tasks` (WIP), `myschool_processcomposer` (WIP)
- **AI + integratie**: `myschool_mcp` (in `Dev-MCP`, niet op master)
- **Security**: `security_phishing`
- **Dashboard**: `myschool_dashboard`

## Deploy-pad

Semaphore template "Odoo extra-addons Update" pullt `MySchool_addons` naar `/opt/odoo-multi-instance/addons<N>/` op de Odoo-VMs. Survey-vars: `target_env`, `addons_branch_override`, `target_limit`. Module-install/upgrade is **manueel** (Odoo UI of CLI `-i <module>`/`-u <module>`) — Semaphore doet alleen git-pull.

Volledige details: [docs/operations/deploy-via-semaphore.md](https://github.com/ictolvpbe/MySchool_addons/blob/Dev-Docs-structure/docs/operations/deploy-via-semaphore.md).

## How to apply (voor agent-sessies)

- **Werken op MCP**: branch `Dev-MCP`, focus op `myschool_mcp/` modulair, tests via `odoo -i myschool_mcp --test-tags myschool_mcp --stop-after-init`
- **Docs aanpassen**: branch `Dev-Docs-structure`, edit `docs/<subdir>/<file>.md`. ADR's: kopieer `docs/decisions/0000-template.md`, increment nummer
- **Nieuwe module**: skeleton kopiëren van `myschool_core/` of `myschool_mcp/`, manifest aanpassen, registreer in `docs/modules/README.md` index
- **Deploy testen**: NIET direct naar master mergen. Eerst feature-branch via Semaphore Survey-override op test-instance, dan Dev, dan master
- **Lokaal Odoo**: PyCharm Run-config (Python 3.13 PyCharm-venv), DB `odoo-dev`
- **DB-naam vragen**: niet relevant — Odoo addon installeert tabellen in dezelfde DB als Odoo-instance

Gerelateerd: [[project-odoo-addons-repo]] (branch-conventie samenvatting), [[project-mcp-and-itsm]] (Project G strategische context), [[project-vcs-strategy]] (GitHub vs Forgejo lange-termijn).
