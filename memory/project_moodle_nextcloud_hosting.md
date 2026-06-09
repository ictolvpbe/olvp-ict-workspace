---
name: project-moodle-nextcloud-hosting
description: "Twee nieuwe publieke app-hosting-werven op planning gezet 2026-06-09: testserver Moodle (+ HA, accounts gesynct vanuit MySchool) en testserver Nextcloud (accounts/groepen/mappen vanuit MySchool, HA later). Beide internet-bereikbaar via het bestaande HAProxy/Caddy-platform. Tracker APP-1/APP-2."
metadata:
  type: project
---

**Op planning gezet 2026-06-09 (user-verzoek, "tussendoor")** — twee nieuwe self-hosted
web-apps die OLVP publiek wil aanbieden, bovenop de Odoo-stack:

1. **Testserver Moodle** — HA-setup; accounts beheerd/gesynct vanuit MySchool. Publiek bereikbaar.
2. **Testserver Nextcloud** — accounts/groepen/mappen beheerd vanuit MySchool; HA later. Publiek bereikbaar.

Tracker: **APP-1** (Moodle) + **APP-2** (Nextcloud) in `governance/tracker-additions.md`.

## Past op het bestaande platform (Project A)

Beide hergebruiken de hosting-keten van [[project-hosting-fase1-status]]: HAProxy HA in DMZ
(LE-cert) → Caddy op de VM (step-ca, TLS re-encrypt) → app op 127.0.0.1. Deploy bij voorkeur
role-based + Podman/Quadlet, analoog aan `roles/odoo-podman/` ([[project-template-strategy]],
Tier 1 baseline + app-overlay). Nieuwe FQDN's (kandidaat `moodle.olvp.be` + `cloud.olvp.be`/
`nextcloud.olvp.be`), DNS one.com → Cloudflare in Fase 2 ([[reference-dns-olvpbe]]).

## Identity-spanning (kernpunt — NIET in tegenspraak met de architectuur)

"Accounts beheerd vanuit MySchool" ligt **bovenop** [[project-identity-architecture]], niet ertegenin:
- **Authenticatie/SSO** hoort via **Keycloak-OIDC** (realm `olvp`, AD-backed) — net als de Odoo-apps.
  Moodle (OIDC/OAuth2-auth-plugin) en Nextcloud (`user_oidc`-app) worden OIDC-clients. Login = AD-
  credentials via Keycloak, centraal 2FA. Respecteert "AD = SoT voor credentials".
- **Provisioning/autorisatie** komt uit **MySchool** (Odoo kent klassen/vakgroepen/leerling↔leerkracht):
  - Moodle: users + cohorts/course-enrolments via Moodle **Web Services API** (token).
  - Nextcloud: users + groepen + **group-folders** via de **Provisioning/OCS API**.
  - Mechanisme sluit aan bij de myschool-addons + MCP-provider ([[reference-mcp-projects-provider]]).

**BESLIST 2026-06-09 — sync-aanpak (groepslidmaatschap + provisioning):**
- **PRIMAIR = directe backend-task sync (optie b):** MySchool (Odoo) pusht via backend-tasks
  rechtstreeks naar de app-API's — Moodle **Web Services API**, Nextcloud **Provisioning/OCS-API**
  (users + groepen + group-folders). Snelste pad; bouwen in de myschool-addons / MCP-laag.
- **FALLBACK ("scenario 1") als dit niet vlot gaat = de AD-keten (optie a):** MySchool → schrijft
  groepen naar **AD** → AD → Keycloak `groups`-claim → apps. Eén SoT (AD), sluit aan bij de
  acc-instance (kan naar AD schrijven) + de auth_oidc groups-claim-werf ([[project-identity-architecture]]).
- _(Auth/SSO blijft hoe dan ook via Keycloak-OIDC; dit gaat enkel over de provisioning/autorisatie-laag.
  "scenario 1" = mijn lezing van de user-instructie = de AD-keten; corrigeer indien anders bedoeld.)_

## HA + Backup + GDPR

- **HA** (Moodle nu mee-genoemd, Nextcloud expliciet later): app-nodes achter LB + gedeelde
  sessie-store (Redis) + gedeelde filestore (`moodledata` / Nextcloud-data op NFS/object) +
  DB-HA → koppelt aan **Project F** (Postgres HA/Patroni, Fase 3).
- **Backup**: Nextcloud = veel PII-bestandsopslag → strikt backup-regime + **DPIA-impact** hoog
  (Project C + `governance/dpia.md`). Moodle idem in mindere mate.
- **Security**: egress-allowlist + hardening (Project B); publiek = WAF-scope (SEC-2).

## Governance — Project I (bevestigd 2026-06-09)

User koos **eigen deelproject**: **Project I — Aanvullende web-applicaties (Moodle & Nextcloud)**.
Toegevoegd aan `governance/programma-structuur.md` (nu 9 deelprojecten + governance-laag) + mermaid.
Tracker-werven APP-1 (Moodle) / APP-2 (Nextcloud).

**How to apply:** bij Moodle/Nextcloud-werk: auth = Keycloak-OIDC (niet lokale of losse LDAP-bind),
provisioning = MySchool-gedreven, deploy = role-based Podman/Quadlet zoals odoo-podman. Eerst test,
dan HA + prod. Beslis de groeps-SoT-vraag (a vs b) vóór de sync-laag. Gerelateerd:
[[project-identity-architecture]], [[project-hosting-fase1-status]], [[project-template-strategy]],
[[project-myschool-repo]], [[reference-mcp-projects-provider]].
