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

**OPEN BESLISSING — SoT voor groepslidmaatschap:**
- (a) MySchool → schrijft groepen naar **AD** → AD → Keycloak `groups`-claim → apps. Eén keten,
  AD blijft enige SoT. Sluit aan bij de acc-instance (kan naar AD schrijven) + de auth_oidc
  groups-claim-werf ([[project-identity-architecture]] OIDC-kant B). **Voorkeur** voor consistentie.
- (b) MySchool → pusht direct naar Moodle/Nextcloud-API's (login via Keycloak, structuur via push).
  Sneller maar tweede autorisatie-SoT naast AD.
- Te beslissen vóór de sync-laag gebouwd wordt. Hangt samen met de auth_oidc-werf (odoo-dev).

## HA + Backup + GDPR

- **HA** (Moodle nu mee-genoemd, Nextcloud expliciet later): app-nodes achter LB + gedeelde
  sessie-store (Redis) + gedeelde filestore (`moodledata` / Nextcloud-data op NFS/object) +
  DB-HA → koppelt aan **Project F** (Postgres HA/Patroni, Fase 3).
- **Backup**: Nextcloud = veel PII-bestandsopslag → strikt backup-regime + **DPIA-impact** hoog
  (Project C + `governance/dpia.md`). Moodle idem in mindere mate.
- **Security**: egress-allowlist + hardening (Project B); publiek = WAF-scope (SEC-2).

## Governance-vraag

Eigen **deelproject (Project I — aanvullende web-applicaties)** of uitbreiding onder **Project A**?
Nu als tracker-werven geparkeerd (APP-1/APP-2) met deze vraag **te ratificeren** door de user —
geen unilaterale herstructurering van `programma-structuur.md` (8 deelprojecten).

**How to apply:** bij Moodle/Nextcloud-werk: auth = Keycloak-OIDC (niet lokale of losse LDAP-bind),
provisioning = MySchool-gedreven, deploy = role-based Podman/Quadlet zoals odoo-podman. Eerst test,
dan HA + prod. Beslis de groeps-SoT-vraag (a vs b) vóór de sync-laag. Gerelateerd:
[[project-identity-architecture]], [[project-hosting-fase1-status]], [[project-template-strategy]],
[[project-myschool-repo]], [[reference-mcp-projects-provider]].
