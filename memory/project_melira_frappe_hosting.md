---
name: project-melira-frappe-hosting
description: "FRAME = Melira (8 eigen Frappe v16-apps) op de OLVP-stack, model B bench-per-env/site-per-tenant met Podman-Quadlet. Vier omgevingen dev/test/acc/acc-test (beslist 2026-09-21) plus de kále Frappe SRVV-FRAPPE-02 als naam-uitzondering. Fase 0 bewezen, Fase 1 geschreven maar nooit e2e; code teruggehaald uit gesloten PR #1."
metadata:
  type: project
---

**Kennisbank staat buiten deze repo:** `/home/demm/OwnProjects/melira-project/hosting/`
(acht genummerde documenten: context, architectuur, image-build, deployment, roadmap, gotcha's,
caddy, app-updates). Die map is het **ontwerpregister**; de uitvoerende code hoort in
`platform-ansible`. Lees bij twijfel eerst die map — ze is uitgebreider dan wat hier past.

## Wat het is

Melira = **8 eigen Frappe v16-apps** op GitLab (`git@gitlab.com:melira/core/*`, privé, SSH-only),
met `melira_core` als verplichte basis voor alle andere. Bestuursbeslissing B3 (2026-07-04):
**MySchool blijft Odoo**, Melira wordt greenfield Frappe. Gehost achter dezelfde
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

**Naamregel:** noch `melira` noch `frappe` in server-namen of FQDN's — beide worden `frame`.
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

## Stand

| Fase | Status |
|---|---|
| **0** — image-build (Frappe 16 + 8 apps, ~2,16 GB) + dev-site | ✅ e2e bewezen 2026-07-08 |
| **1** — `roles/frappe-podman`, SoT, `caddy-frappe.yml`, topologie | 🟡 geschreven, **nooit e2e** |
| **2** — Keycloak-OIDC, MariaDB-backup, branch-sync-wiring | ⬜ open |

**De blocker is weg:** Fase 1 kon niet geverifieerd worden omdat er geen VM's waren en de
Tier-1-master nog in aanbouw was. Die master is geseald op 2026-09-21
([[project-template-fleet-defects]]) — klonen kan.

Gerelateerd: [[project-frappe-platform]] (de oorspronkelijke aankondiging),
[[project-intranet-hosting]] (het gespiegelde patroon), [[project-containerization]],
[[project-mcp-and-itsm]] (`melira_mcp` raakt aan het MCP-spoor).
