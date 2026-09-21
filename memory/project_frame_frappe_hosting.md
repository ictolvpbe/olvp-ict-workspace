---
name: project-frame-frappe-hosting
description: "FRAME = de werfnaam voor het Frappe-spoor bij OLVP (8 eigen Frappe v16-apps van de leverancier) op de OLVP-stack, model B bench-per-env/site-per-tenant met Podman-Quadlet. Vier omgevingen dev/test/acc/acc-test (beslist 2026-09-21) plus de kále Frappe SRVV-FRAPPE-02 als naam-uitzondering. Fase 0 en Fase 1 e2e bewezen (dev-bench, 2026-09-21); code was teruggehaald uit gesloten PR #1."
metadata:
  type: project
---

**Kennisbank staat buiten deze repo:** `/home/demm/OwnProjects/melira-project/hosting/`
(acht genummerde documenten: context, architectuur, image-build, deployment, roadmap, gotcha's,
caddy, app-updates). Die map is het **ontwerpregister**; de uitvoerende code hoort in
`platform-ansible`. Lees bij twijfel eerst die map — ze is uitgebreider dan wat hier past.

## Wat het is

FRAME is de OLVP-werfnaam voor dit spoor. De leverancier levert **8 eigen Frappe v16-apps** op
GitLab (`git@gitlab.com:melira/core/*`, privé, SSH-only), met `melira_core` als verplichte basis
voor alle andere — die repo- en modulenamen zijn feiten over andermans systeem en blijven dus
staan. Bestuursbeslissing B3 (2026-07-04): **MySchool blijft Odoo**, dit wordt greenfield Frappe. Gehost achter dezelfde
HAProxy→Caddy→step-ca-keten als Odoo.

**Model B** (beslist 2026-07-08): één **bench** per omgeving, meerdere **sites** per bench.
Herevaluatie naar model A (volle stack per instance) voorbehouden als security om harde
inter-tenant-isolatie vraagt.

## ⚠️ De code stond niet in de repo

PR #1 (`melira-frappe-fase0`) is op **2026-08-25 gesloten zonder merge** en de branch is
verwijderd, terwijl de kennisbank `platform-ansible` als canonical bron aanwijst. Niemand wist
meer waarom; het lijkt niet opzettelijk. De commits waren nog ophaalbaar via
`git fetch origin refs/pull/1/head` — 28 bestanden, 1796 regels — en staan sinds 2026-09-21 op
branch `melira-frappe-fase1`. **Les:** een gesloten PR met verwijderde branch is niet weg, maar
hij is ook niet vindbaar voor wie er niet naar zoekt.

## Topologie (beslist 2026-09-21)

Vier omgevingen, gespiegeld op [[project-intranet-hosting]]. Blootstelling volgt de suffix:
`.olvp.be` publiek, `.olvp.int` intern.

| VM | bench | kanaal/branch | FQDN | expose |
|---|---|---|---|---|
| `srvv-dev-frame-01` | dev | `dev` | `frame-dev.olvp.be` | publiek |
| `srvv-tst-frame-01` | test | `tst` | `frame-test.olvp.be` | publiek |
| `srvv-acc-frame-01` | acc | **`prod`** | `frame-acc.olvp.int` | intern |
| `srvv-tst-acc-frame-01` | acc-test | `tst` | `frame-acc-test.olvp.int` | intern |
| `srvv-frappe-02` | prod | `prod` | `frappe.olvp.be` | publiek |

**Acc draait het prod-kanaal** omdat de ICT-dienst die omgeving zélf productief gebruikt, onder
meer voor projectbeheer met `melira_projects`. Acc-test is het voorportaal op het test-kanaal.
Exact het intranet-patroon. **Accountbeheer blijft tot nader order in intranet.**

**Naamregel:** noch de productnaam noch `frappe` in server-namen of FQDN's — beide worden `frame`.
Bevestigd 2026-09-21: in code, Podman-namen en mapnamen mág de productnaam blijven staan; FQDN's
niet.
`SRVV-FRAPPE-02` (kále Frappe met ERPNext-stack, eigen image, bestaat al als VMID 401) is een
bewuste uitzondering en behoudt haar naam. Daardoor blijft `frame.olvp.be` vrij voor de latere
publieke productieserver.

## Meerdere instances per server — twee niveaus

| Niveau | Isolatie | Wanneer |
|---|---|---|
| **Sites in één bench** | zwak (gedeelde runtime) | edities of klanten op dezelfde app-versie |
| **Benches op één VM** | sterk (eigen containerset, netwerk, `http_port`) | verschillende branches — het equivalent van Odoo's 8069/8070 |

De role kan beide: `tasks/main.yml` loopt over `benches`, `bench.yml` over `sites` (geverifieerd).

**Waarom acc en acc-test tóch aparte VM's:** een bench draagt **één versie van de app-code** (apps
worden op bench-niveau uit git gehaald op de branch van het kanaal), dus twee omgevingen kunnen
geen twee sites in dezelfde bench zijn. Als twee benches op één host zou het 18 containers en twee
MariaDB's worden — te krap op 8 GB.

## Oude VM's opgeruimd (2026-09-21)

`SRVV-FRAPPE-01` (400), `SRVV-DEV-FRAPPE-01` (10050) en `SRVV-TST-FRAPPE-01` (10051) zijn
**verwijderd** — opnieuw klonen uit de geseald baseline is sneller dan ze bijwerken.
`SRVV-FRAPPE-02` (401, kále Frappe) **blijft bestaan**.

10050 en 10051 hadden elk vier backups in `PROXMO_BU`; **400 had er geen enkele**, dus daar is
vooraf een `vzdump` van gemaakt (3,75 GB) zodat het besluit omkeerbaar blijft.

⚠️ 401 is nu de enige VM in dit spoor die **niet** uit de nieuwe baseline komt en dus nog de oude
template-fouten draagt: gedeelde machine-id, scheve indeling, geen versiemarkering. Neem hem mee
in fase 9 van [[project-template-fleet-defects]] of kloon hem alsnog opnieuw.

## Stand

| Fase | Status |
|---|---|
| **0** — image-build (Frappe 16 + 8 apps, ~2,16 GB) + dev-site | ✅ e2e bewezen 2026-07-08 |
| **1** — `roles/frappe-podman`, SoT, `caddy-frappe.yml`, topologie | ✅ **e2e bewezen 2026-09-21** op de dev-bench |
| **2** — Keycloak-OIDC, MariaDB-backup, branch-sync-wiring | ⬜ open |

**De blocker is weg:** Fase 1 kon niet geverifieerd worden omdat er geen VM's waren en de
Tier-1-master nog in aanbouw was. Die master is geseald op 2026-09-21
([[project-template-fleet-defects]]) — klonen kan.

## ▶ Eerste bench draait (2026-09-21)

`SRVV-DEV-FRAME-01` (VMID 10050, 10.200.14.51): negen containers gezond, site
`frame-dev.olvp.be` met database en db-gebruiker, `/api/method/ping` → `pong`,
`/login` → 200, `developer_mode: 1`.

Bewust met een **runtime-only image** (`melira-frappe:16-runtime`, gebouwd met
`MELIRA_APPS_JSON=/dev/null`) en een **lege app-lijst**: zo testte de run alleen
de role, niet de apps-laag. Geen GitLab-sleutel nodig. Overgezet met
`docker save | ssh 'sudo podman load'` — **let op die `sudo`**: Quadlet-units zijn
systeem-units en gebruiken de rootful store, niet die van de ansible-gebruiker.

Volgende stap: de apps-volume + deploy-key uit `frappe/decisions/0001`, daarna
test, acc en acc-test.

## Vijf vertaalfouten compose → Quadlet (alle gedicht)

De Fase 0-compose was bewezen, de vertaling ernaar niet. Alle vijf kwamen pas bij
de eerste echte run boven, en ze zijn generiek genoeg om bij elke
compose→Quadlet-migratie op te letten:

1. **YAML folded scalar houdt nieuwe regels** zodra een regel dieper inspringt.
   `bench new-site` werd daardoor zónder opties aangeroepen, vroeg interactief om
   het root-wachtwoord en stierf op `EOFError`. **Dit was de echte oorzaak van
   vier mislukte runs.**
2. **Bind-mount ≠ named volume.** Podman vult een named volume bij de eerste
   mount vanuit het image; een bind-mount maskeert de image-inhoud. Zonder
   `sites/apps.txt` weigert élk bench-commando.
3. **Datadirs van sidecars niet chownen.** mariadb en redis chownen hun datadir
   naar uid 999; zet Ansible dat terug op root, dan faalt de database-aanmaak met
   `errno 13` — een bestandssysteemfout die als SQL-rechtenprobleem leest.
4. **Wachtwoorden via Podman-secrets**, niet via de commandoregel en niet via
   `environment:` + `podman run -e VAR` (die bereiken het podman-proces niet
   onder `become`).
5. **`bench new-site` schrijft `site_config.json` vóór de database.** Een half
   mislukte run laat dus een site achter die compleet lijkt. ⚠️ De
   idempotentie-check kijkt nu naar dat bestand en is daarmee nog niet
   waterdicht — open punt.

Zie ook [[feedback-toon-het-echte-commando]] over hoe die vier ronden te
vermijden waren.

Gerelateerd: [[project-frappe-platform]] (de oorspronkelijke aankondiging),
[[project-intranet-hosting]] (het gespiegelde patroon), [[project-containerization]],
[[project-mcp-and-itsm]] (`melira_mcp` raakt aan het MCP-spoor).
