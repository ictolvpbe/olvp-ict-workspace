---
name: project-mcp-and-itsm
description: "MCP-service en Odoo ITSM/PM-migratie als Projecten G en H. ⚠️ MOGELIJK ACHTERHAALD: programma bijgestuurd de voorbije weken (user, 2026-09-20), en een Frappe-platform met MCP voor projectopvolging overlapt met Project H. Verifieer vóór gebruik."
metadata:
  type: project
---

> ⚠️ **Mogelijk achterhaald — verifiëren vóór gebruik (gemeld door user, 2026-09-20).** Er is de
> voorbije weken "één en ander bijgestuurd" aan het programma; wat precies is niet vastgelegd.
> Bijkomend signaal: er komt een **Frappe-platform** (vier servers) waarvan één live door de
> IT-dienst gebruikt wordt voor projectopvolging, met MCP-toegang — functioneel hetzelfde doel als
> Project H hieronder. Open vraag: wordt Frappe de governance-SoT en wordt H omgelegd, of blijven
> Odoo en Frappe naast elkaar bestaan (en vervalt "single source of truth")?
>
> Vraag dit na bij de user en check `governance/programma-structuur.md` vóór je op onderstaande
> tekst voortbouwt. Zie [[feedback-verify-memory-against-repo]].

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


## Bijgesteld 2026-09-23 — het Frappe-spoor neemt het over

De kanttekening bovenaan ("mogelijk achterhaald, eerst verifiëren") is nu beslecht: **`melira_mcp`
vervangt de Odoo-MCP.** Eerst voor projectbeheer, later mogelijk om via **AppFoundry** rechtstreeks
met AI apps te bouwen. De ICT-toepassingen komen op **frame-acc** ([[project-frame-frappe-hosting]]).

Daarmee zijn MCP-1 t/m MCP-3 in de tracker, die vanuit het Odoo-spoor geschreven zijn, grotendeels
achterhaald. Nieuw item: **MCP-4**, met drie open punten.

**Bereikbaarheid — beslist 2026-09-23: intern, niet publiek.** De endpoint blijft op
`frame-acc.olvp.int`, zonder publicatie op de edge. Bereik loopt via een client die intern draait of
via de VPN binnenkomt; vandaag is dat Claude Code op het werkstation.

*Bewust niet gekozen*: een publieke FQDN die alleen `/mcp` doorlaat via een path-allowlist (patroon
`id.olvp.be`). Dat zou nodig zijn voor een cloud-client. Reden om het niet te doen: `melira_mcp`
groeit richting Studio-JSON genereren en app-provisioning — niet iets wat je van buitenaf bereikbaar
wil hebben zonder harde noodzaak. ⚠️ **Gevolg**: met deze MCP werken vanuit een cloud-dienst kan niet.

⚠️ **Scheid projectbeheer van AppFoundry.** Een MCP die records bijwerkt is een rechtenvraag; een MCP
waarmee een model apps aanmaakt en installeert is code-uitvoering in een draaiende omgeving. Acc is
de omgeving die de ICT-dienst zélf productief gebruikt. Die twee scopes horen niet achter dezelfde
toegang, en app-generatie hoort niet publiek bereikbaar te zijn.

**Uitfasering**: de huidige provider werkt op `myschool.project`/`myschool.task` in Odoo,
`melira_projects` heeft een eigen model. Migreren of opnieuw beginnen bepaalt of beide een tijd naast
elkaar draaien — maar twee MCP-servers met overlappende projectdata is verwarrend voor mens en model,
dus leg het moment van uitfaseren vast.

## Gemeten 2026-09-23 — wat `melira_mcp` werkelijk is

Zodra de VPN weer verkeer doorliet, gemeten op `frame-acc` (app-commit `e230a60`, geïmporteerd
2026-07-07). Het is **geen datamodel-in-wording maar een werkende server**, en dat verandert de aard
van de vraag: er valt op de edge niets fijner te knippen dan één pad, dus dit is een **rechten**vraag
geworden, geen configuratievraag.

- **Eén endpoint**: `POST /api/method/melira_mcp.api.mcp` — JSON-RPC 2.0 / Streamable HTTP, protocol
  `2025-03-26`, `serverInfo` = `melira-mcp 0.1.0`. Precies één `@frappe.whitelist(allow_guest=True)`,
  geen eigen route, geen tweede poort.
- **Auth-trap werkt, live geverifieerd**: `initialize` en `ping` mogen pre-auth (capability-discovery),
  daarna eist het `Authorization: token <key>:<secret>`. `tools/list` zonder token → **401** (`-32001`).
  `tools/call` eist bovendien de rol **`Melira MCP User`** → anders 403 (`-32003`).
- **24 tools, allemaal prefix `appfoundry_`**. Elf lezend (`list_projects`, `list_items`,
  `list_my_items`, `get_item`, `list_statuses`, `list_active_sprints`, `get_release_progress`,
  `list_build_queue`, `get_build`, `list_test_runs`, `generate_prompt`), dertien schrijvend
  (`set_status`, `assign`, `post_comment`, `update_item`, `create_item`, `create_project`,
  `create_release`, `create_sprint`, `link_blocked_by`, `link_process_node`, `report_test_run`,
  `claim_build`, `report_build`).
- **Audit bestaat al**: elke `tools/call` loopt via `melira_core.sys_event` naar Sys Event, ook bij
  fout. Rate-limit staat in `Melira MCP Settings` (`rate_limit_per_minute`, default 60).

⚠️ **De scheiding uit het punt hierboven bestaat vandaag niet.** `_provider_enabled()` kijkt naar de
prefix vóór de eerste underscore en kent één vlag: `provider_appfoundry`. Lezen, schrijven én de
build-queue hangen dus aan één vinkje en één rol — wie `Melira MCP User` krijgt om een projectstatus
bij te werken, mag meteen ook `create_project` en `claim_build`. Scheiden vraagt **werk in de app**
(tweede provider-prefix of per-tool-rollen), geen instelling. Wat de urgentie tempert: er is nog
**geen tool die code uitvoert of een app installeert**. `generate_prompt` levert tekst;
`claim_build`/`report_build` schrijven records in een wachtrij. De uitvoering zit aan de andere kant
van die wachtrij, buiten deze MCP.

## ⚠️ DNS volgt de VPN-tunnel niet (2026-09-23)

De endpoint antwoordt over de VPN op `https://frame-acc.olvp.int` met een geldig step-ca-cert — maar
alleen als je het IP meegeeft. `tun0` krijgt **geen resolver mee** (`resolvectl status tun0`:
`Current Scopes: none`), dus het werkstation vraagt `olvp.int` aan `192.168.2.1`/`8.8.8.8` en krijgt
NXDOMAIN. De interne DC `10.33.0.10` antwoordt wél correct over de tunnel — het is dus puur
resolver-configuratie, geen routering en geen zonekwestie.

Dat is meer dan een ongemak: de hele toegangsweg die voor MCP-4 gekozen is ("intern of via de VPN")
staat of valt hiermee. Oplossing in het VPN-profiel: push DNS `10.33.0.10` + search-domein
`olvp.int`. Tijdelijk per sessie kan het met
`sudo resolvectl dns tun0 10.33.0.10 && sudo resolvectl domain tun0 '~olvp.int'`. Zonder een van
beide werkt alleen een hosts-regel of een expliciet IP.

⚠️ Les die hier onder zit: een tunnel die *up* is en routes draagt, is nog geen bruikbare tunnel.
Test naambereikbaarheid, niet alleen `ping` op een IP — anders vind je dit gat pas als een client
faalt. Zelfde soort blinde vlek als bij [[project-netxms-monitoring]] NET-7, waar DNS over UDP 53
wél doorkwam en RPC niet, zodat het gat maandenlang onzichtbaar bleef.
