---
name: reference-mcp-projects-provider
description: MCP-spec voor `myschool_mcp.provider_projects` — interageert met `myschool_projects`-app (project/task-management). Tools, datamodel, workflow voor push van project-data vanaf Claude/clients. Verschilt van de oudere `appfoundry`-provider.
metadata:
  type: reference
---

Spec aangeleverd 2026-06-04. **`provider_projects` is een andere provider** dan `appfoundry` in myschool_mcp — werkt tegen `myschool.project` + `myschool.project.task` modellen (niet `appfoundry.item`/sprint/release). Beide providers kunnen parallel bestaan in dezelfde myschool_mcp-module via de `provider_projects = True` toggle.

## Verbinding

| Wat | Waarde |
|---|---|
| Endpoint | `POST http://<host>/mcp` (lokaal: `http://localhost:8069/mcp`) |
| Auth-header | `X-API-Key: <odoo-apikey>` |
| Key-source | Odoo → Preferences → Account Security → New API Key (scope `rpc` of leeg) |
| Rechten | `group_mcp_user` + `group_projects_manager`. `group_myschool_core_admin` impliceert beide. |
| Provider-toggle | `myschool_mcp.provider_projects = True` (default ON) |

Claude Code registratie:
```bash
claude mcp add --transport http myschool http://<host>/mcp --header "X-API-Key: ..."
```

## Actieve registratie (sinds 2026-06-06)

De `myschool`-MCP-server (Claude Code, **scope = extra-addons werf**
`/home/demm/PyCharm/odoo-myschool/extra-addons` in `~/.claude.json`) wijst nu naar
de **remote instance** i.p.v. localhost:

- **Endpoint**: `https://myschool-ict.olvp.be/mcp` (publiek via HAProxy→Caddy→Odoo,
  master-branch, role-based conf-pattern op SRVV-ODOO-01). Was `http://localhost:8069/mcp`.
- **Auth**: API-key van een **dedicated service-user `claude-mcp`** (login `claude-mcp`,
  interne gebruiker) — NIET de admin-key. Bewuste least-privilege-keuze: een Odoo
  API-key heeft géén fijnmazige scope (lege/`rpc`-scope = volledige RPC *als die user*),
  dus de begrenzing komt volledig uit de **groepen**: enkel `group_mcp_user` +
  `group_projects_manager`, géén Administration/Settings. Zo blijft blast-radius bij
  key-lek beperkt tot projects-data + is MCP-actie auditbaar als `claude-mcp`.
- **Key-opslag**: enkel in `~/.claude.json` (header X-API-Key). NOOIT in memory/git.
  Roteren = nieuwe key in de UI + `claude mcp remove/add` opnieuw.
- **Geverifieerd**: `tools/list` geeft 25 tools (appfoundry 14 + projects 11). `myschool_mcp`
  is mee geïnstalleerd op `myschool_ict` (module zit nu óók op master, niet enkel Dev-MCP).
- **Gotcha**: `~/.claude.json`-wijziging wordt pas geladen bij een **nieuwe** sessie in
  de extra-addons-map; lopende sessie daar herstarten.
- **Post-deploy-TODO**: dit `claude-mcp`-service-account + key handmatig aangemaakt;
  hoort op termijn in de reproduceerbare post-deploy-stap ([[project-odoo-role-migration]],
  samen met admin-password + app-groepen). Patroon sluit aan bij [[feedback-service-accounts]].
- ⚠️ **Key dood sinds 2026-06-07**: de key `e217552f…` geeft nu **HTTP 401** op `/mcp`
  (myschool-ict zelf gezond, login 200) — vermoedelijk vervaldatum of per ongeluk verwijderd.
  **Om WBS-updates te hervatten**: nieuwe API-sleutel aanmaken (admin/claude-mcp → Voorkeuren
  → Accountbeveiliging) en in `~/.claude.json` (extra-addons scope) zetten via `claude mcp`.
- **OPENSTAANDE WBS-edits (geblokkeerd op nieuwe key)**: item **136** → `done` (myschool-dev
  relocatie + teardown oude instance, KLAAR 2026-06-07); item **137** → herkadreren naar enkel
  "id-test-container (instance_2) teardown op TST-ODOO-01 — routing al weg, container deferred
  tot Keycloak-cutover", priority 0.

## Datamodel

**Project** (`myschool.project`):
- `name`, `code` (uniek, voor lookups), `parent_id` (sub-project / WBS-tak)
- `responsible`, `members`
- `date_start`, `date_end`, `description`
- `state` ∈ {`new`, `active`, `on_hold`, `done`, `cancelled`}
- `progress` (berekend, niet schrijfbaar)

**Work item** (`myschool.project.task`):
- `name`, `item_type` ∈ {`task`, `milestone`, `phase`, `epic`, `bug`}
- `project_id`, `parent_id` (self-hiërarchie)
- `assigned_id`, `state` ∈ {`todo`, `in_progress`, `blocked`, `done`, `cancelled`}
- `priority` ∈ {`'0'`=Low, `'1'`=Normal, `'2'`=High, `'3'`=Critical}
- `date_start`, `date_deadline` (YYYY-MM-DD)
- `planned_hours`, `milestone_id`, `depends_on_ids`

**Conventies**:
- Milestone = work item met `item_type=milestone`
- Een item met kinderen = "summary" (voortgang rolt op)
- Resolutie: projecten op `id` of `code` (case-insensitief); work items op `id`; milestones op `id` of `naam` binnen project; gebruikers op `login`/`id`/`"me"`

## Tools

### Read
| Tool | Parameters |
|---|---|
| `projects_list` | `parent?`, `top_level_only=true`, `my_only=false`, `limit=100` |
| `projects_get` | `project` → project + children + milestones + dependencies |
| `projects_tree` | `project?` → geneste WBS-boom van projecten |
| `projects_list_items` | `project`, `item_type?`, `state?`, `assignee?`, `open_only=false`, `limit=200` |

### Write
| Tool | Parameters |
|---|---|
| `projects_create_subproject` | `name*`, `parent?`, `code?`, `responsible?`, `state=new`, `date_start?`, `date_end?` |
| `projects_create_item` | `project`, `name`, `item_type=task`, `parent?`, `assignee?`, `state=todo`, `priority='1'`, `date_start?`, `date_deadline?`, `planned_hours?`, `milestone?`, `description?` |
| `projects_update_item` | `item*` (id) + elk veld om te wijzigen. `parent`/`milestone`/`assignee` = `id` (of `naam`/`login`); `false`/`0` wist |
| `projects_add_milestone` | `project`, `name`, `date?`, `state=todo`, `description?` |
| `projects_set_dependency` | `project`, `depends_on`, `remove=false` (project-niveau) |

`*` = verplicht.

## Workflow — volledig project pushen

```
1. Project        projects_create_subproject {name:"OLVP-ICT", code:"OLVP-ICT", state:"active"}
2. Milestone      projects_add_milestone {project:"OLVP-ICT", name:"Netwerk live", date:"2026-09-01"}
3. Fase           projects_create_item {project:"OLVP-ICT", name:"Fase 1 — Netwerk", item_type:"phase"}
4. Taak           projects_create_item {project:"OLVP-ICT", name:"Switch config",
                    parent:<phase_id>, assignee:"jan.peeters",
                    state:"todo", priority:"2",
                    date_deadline:"2026-08-20", milestone:"Netwerk live"}
5. Sub-project    projects_create_subproject {name:"Servers", parent:"OLVP-ICT"}
6. Controleren    projects_get {project:"OLVP-ICT"} · projects_tree · projects_list_items
```

## Aandachtspunten bij push

- **Geen upsert** — `create_*` maakt altijd nieuw record. Voor idempotentie: gebruik unieke `code` per project + check eerst via `projects_list` / `projects_list_items`
- **Volgorde**: fases (parents) en milestones aanmaken vóór taken die ernaar verwijzen — `parent`/`milestone`-params verwachten bestaande id (milestone op naam binnen zelfde project mag)
- **Hiërarchie op id**: `projects_list_items` geeft platte lijst met `parent_id`; bouw boom zelf, of gebruik `projects_tree` voor projectniveau
- **Cycli + cross-project deps** worden server-side geweigerd → MCP-fout
- **Format**: datums `YYYY-MM-DD`, priority is `string '0'..'3'`

## Gotchas geconstateerd in PoC 2026-06-04

1. **`description` veld**: NIET in `projects_create_subproject`-args. Alleen in `projects_create_item`. Project-description kan eventueel achteraf via update.
2. **State-vocabulary verschilt** tussen project en task:
   - Project: `new`, `active`, `on_hold`, `done`, `cancelled`
   - Task: `todo`, `in_progress`, `blocked`, `done`, `cancelled`
   - `on_hold` als task-state = `Internal error: Wrong value for myschool.project.task.state`. Voor pause/wachtend-op-iets task → gebruik `blocked`.
3. **Milestones zijn project-scoped**: sub-project-task kan NIET linken naar parent-project-milestone. Error: "Doel-milestone moet in hetzelfde project zitten." Workaround: maak milestone op het sub-project zelf, OF plaats task direct op hoofdproject onder een phase.
4. **`depends_on` / `depends_on_ids` niet via `projects_update_item`**: error "No updatable fields provided". Spec zegt `projects_set_dependency` is "project-niveau". Voor item-niveau-deps: mechanisme onduidelijk, te onderzoeken — mogelijk dit is een gap in v1 van de provider.
5. **Geen delete-tool**: alleen `state="cancelled"` (+ eventueel naam-prefix `(DUPLICATE)` voor zichtbaarheid). Volledige verwijdering vereist Odoo-UI.
6. **Unicode in JSON-args via shell**: werkt server-side, maar response-parsing kan struikelen. Voor robuuste curl-bridge: ASCII-alternatief (`->` i.p.v. `→`) of `json.dumps` voor escape via Python in plaats van bash-string-interpolatie. Failed-parse betekende NIET dat server het record niet aanmaakte (PoC: dubbele creation door retry op fout-vermoeden).
7. **`projects_tree` toont enkel project-hiërarchie**: items binnen project zichtbaar via `projects_get` (één laag) of `projects_list_items` (vlakke lijst met parent_id).
8. **`projects_update_item` heeft beperkte whitelist** van updateable velden. Bevestigd werkend: `name`, `state` (+ vermoedelijk `parent`, `milestone`, `assignee`, `priority`, `date_start`, `date_deadline`, `planned_hours`). Bevestigd NIET werkend: `description`, `depends_on_ids`. Fout: `"No updatable fields provided"`. Workaround voor description: gebruik bij `projects_create_item` (waar wél geaccepteerd), of prefix in `name` (bv. `[#41] EPIC: ...` voor Claude-task-tracker-mapping zoals gedaan in PoC).
9. **External-ID-mapping**: spec heeft geen apart veld voor externe identifiers. Voor mapping naar externe trackers (Claude-tasks, Jira, etc.): gebruik `[#NN]`-prefix in name of een dedicated `code`-pattern. Voor PoC-OLVP-ICT-push: `[#NN]` in name werkt als visual-mapping in lijsten.

## Gerelateerd

- `myschool_mcp` module + `appfoundry`-provider (oudere v1) in `MySchool_addons` repo, branch `Dev-MCP` — zie [[project-myschool-repo]] + module-doc in repo
- [[project-mcp-and-itsm]] — strategische context (Project G + H)
- ~~Task #48: OLVP-ICT project-data pushen via MCP~~ **DONE 2026-06-06**: volledige WBS
  gepusht naar `myschool.project` op myschool-ict — root **OLVP-ICT** (id 2) → 9 sub-projecten
  (A-HOST/B-SEC/C-BACKUP/D-MON/E-NET/F-PGHA/G-MCP/H-ITSM/GOV) → **102 work-items** (phases/
  epics/tasks) + 4 program-milestones. States gezet (done voor afgewerkt werk, blocked voor
  myschool-test/Forgejo, in_progress voor lopend). Push via throttled script (rate-limit 60/60s
  → 1 call/sec). Bug-les: `projects_create_item` met deadline maar zónder parent → datum NIET
  in parent-positie meegeven (anders "Invalid work item reference"). Phase-items kregen daardoor
  eerst geen children; fix = parent apart aanmaken + `projects_update_item parent=<id>` reparent.
