---
name: project-identity-architecture
description: "Identity-architectuur voor [[project-odoo-public-access]]: drie gescheiden rollen (id/acc/app), AD als single source of truth, acc volledig intern. Beslist 2026-05-18."
metadata: 
  node_type: memory
  type: project
  originSessionId: 59a419f3-43f1-477b-bb2b-196d0c16926a
---

Aparte identity-laag voor [[project-odoo-public-access]] met drie rollen, beslist 2026-05-18 vanuit GDPR-dataminimalisatie. Detailplan: `a-id-test-ook-publiek-rosy-stroustrup.md`.

**Why:** De oorspronkelijke opzet zou alle PII (max ~2500 leerlingen, ouders, staff) op dezelfde publieke applicaties laten staan als de Odoo-apps zelf. Extra zorg: `myschool-ict.olvp.be` host een MCP-server voor Claude-app-dev — die extra programmatische attack-surface maakt fysieke scheiding van accountbeheer-data des te belangrijker.

**Drie rollen:**
1. **identity (`id`)** — pure auth-broker: AD-bind via LDAP + 2FA + OAuth2-tokens uitgeven. Geen rijke PII, geen accountbeheer-UI. **Publiek bereikbaar met path-allowlist** (alleen auth-paden).
2. **account-admin (`acc`)** — ICT-werkplek voor accountbeheer + audit. Rijke workflow- en audit-data bovenop AD. **Volledig intern**: niet in HAProxy/DMZ, alleen interne DNS.
3. **app** — myschool, myschool-ict (incl. MCP), myschool-test, myschool-dev. **Publiek**. JIT-provisioning van `res.users` bij eerste OAuth-login; alleen minimale token-claims (`sub`, `email`, `name`, `groups`).

**AD = single source of truth voor credentials:**
- id én acc binden **onafhankelijk** naar AD via LDAP. Geen onderlinge afhankelijkheid.
- acc gebruikt **direct LDAP voor zijn eigen login** (niet via id) → ICT kan accountbeheer blijven doen als id down is.
- acc kan optioneel naar AD **schrijven** (account create/modify) voor ICT-workflows.

**Plaatsing:**
| Rol | Productie | Test |
|---|---|---|
| identity | dedicated VM **SRVV-ID-01** (10.33.0.41) | co-located op SRVV-TST-ODOO-01:8070 |
| account-admin | dedicated VM **SRVV-ACC-01** (10.36.0.42, VLAN 36) — `myschool-acc.olvp.int`:8069 | **co-located op SRVV-ACC-01:8070** — `myschool-acc-test.olvp.int` |
| app | bestaande SRVV-ODOO-01 | bestaande SRVV-TST-ODOO-01 + 02 |

Productie krijgt 2 nieuwe VMs. Test consolideert op bestaande VMs (test-data minder sensitief). **Uitzondering account-admin (beslist 2026-06-02):** acc-test draait NIET op de test-VMs maar co-located op de prod-VM SRVV-ACC-01:8070, omdat zowel prod- als test-account-admin `internal` (nooit vanaf internet bereikbaar) moeten blijven — de test-VMs zijn public/public-restricted en dus ongeschikt. Het eerder geplande SRVV-TST-ODOO-01:8072 is nooit gebouwd en vervalt. Beide acc-instances in `.olvp.int` (niet `.olvp.be`).

**Publieke routering:**
| Rol | Publiek? | Pad |
|---|---|---|
| app | ja | HAProxy DMZ → Caddy op VM, alle app-paden |
| identity | beperkt | HAProxy DMZ → Caddy op VM, path-allowlist (alleen auth) |
| account-admin | nee | Geen HAProxy. Interne DNS → Caddy direct op VM (alleen step-ca cert, geen LE) |

**2FA:**
- id-instance: TOTP-2FA verplicht voor admin + employees vanaf go-live.
- acc-instance: TOTP-2FA verplicht voor **alle gebruikers** (= ICT-staff alleen).

**Risico's:**
- **SPOF id-instance**: bij uitval geen nieuwe app-logins; bestaande sessies werken tot token-expiry. Acc blijft werken (onafhankelijke LDAP-bind). HA voor id-instance overwegen in Fase 3.
- **OCA `auth_oauth_provider` Odoo 19**: compatibiliteit nog te verifiëren (taak F1-X). Bij blockers: zelf porten (~1 dag) of Authentik-fallback.
- **MCP-server auth op myschool-ict**: aparte auth-strategie nodig (service-token of OAuth client-credentials grant); ontwerp in sub-task F1-AA-MCP, geen blocker voor overige identity-stack.
- **AD-uitval**: alle logins falen. Geaccepteerd risico — AD is al kritieke infra vandaag.
- **Step-ca root cert distributie naar ICT-laptops**: nodig om acc-instance zonder browser-warning te bereiken. Voorkeur: group policy via AD.

**Nieuwe Fase 1-taken (bovenop [[project-phasing]]):**
- F1-X — OCA `auth_oauth_provider` Odoo 19 verifiëren (blocker voor F1-Z)
- F1-Y — SRVV-ID-01 provisionen (4 vCPU / 4 GB / 40 GB, 10.33.0.41/24)
- F1-Y2 — SRVV-ACC-01 provisionen (2 vCPU / 4 GB / 40 GB, 10.33.0.42/24)
- F1-Z — id-instance config (LDAP, OAuth2-provider, 2FA, Caddy)
- F1-Z2 — acc-instance config (LDAP onafhankelijk, audit-logging, 2FA, Caddy)
- F1-AA — App-instances OAuth-client config (4 instances + MCP sub-task)
- F1-BB — Caddy-config update op alle VMs (incl. acc met enkel step-ca)
- F1-CC — Interne DNS-resolutie voor `myschool-acc[-test].olvp.be` (split-DNS, niet in publieke DNS)

**Implicaties Fase 1b (security prio 1):**
- Egress-allowlist nieuwe VMs: alleen AD-LDAP, NTP, APT-mirror.
- HAProxy admin-deny gedifferentieerd per rol (identity = strict allowlist; app = bredere deny; acc = niet in HAProxy).
- Backups: acc strikt regime (audit-data kritisch voor incidentonderzoek).

**Implicaties Fase 2 (DNS):** `myschool-acc[-test].olvp.be` **niet** publiceren in Cloudflare; blijven in interne resolver (split-DNS).

**How to apply:** Bij elk voorstel rond authenticatie, accountbeheer of OAuth: deze 3-rollen-scheiding respecteren. acc-instance hoort nooit in publieke DNS, nooit in HAProxy-config, nooit op DMZ-pad. id-instance enkel via path-allowlist publiek. Apps doen JIT-provisioning, geen lokale wachtwoorden meer.

---

## Update 2026-06-04 — dedicated IdP i.p.v. Odoo-als-id-instance

**Status van bovenstaand plan**: 3-rollen-architectuur (id/acc/app) BLIJFT. Wat WIJZIGT is wat er op de **id-rol** draait.

**Was**: Odoo-instance met OCA `auth_oauth_provider`-module (huidig live op SRVV-ODOO-01:8071 als placeholder; SRVV-ID-01 dedicated VM was gereserveerd maar nooit geprovisioneerd).

**Wordt**: Dedicated IdP (**Keycloak** of **Authentik** — keuze in [[task #42]] / ADR 0006) op SRVV-ID-01 (10.33.0.41), in container via Quadlet. Odoo wordt enkel OIDC-client, geen OIDC-provider meer.

**Waarom de wijziging**:
- Odoo is geen volwassen identity-provider; auth_oauth_provider is OCA-add-on met beperkte security-track-record voor internet-facing role
- Wachtwoord-handling in Odoo-codebase houden vergroot aanvalsoppervlak
- Dedicated IdP (Keycloak/Authentik) is purpose-built: hardening, rate-limit, OIDC+SAML, LDAP-federatie first-class
- De eerder gedocumenteerde "Authentik-fallback" (Risico's-sectie hierboven) wordt hoofdpad

**Implicaties**:
- F1-X taak (OCA auth_oauth_provider verifiëren) **vervalt**
- F1-Y (SRVV-ID-01 provisionen) blijft, MAAR overlay verandert: container-IdP i.p.v. Odoo-instance
- F1-Z (id-instance config) wordt nu Keycloak/Authentik realm + LDAP-federatie + 2FA
- Huidige id.olvp.be → Odoo:8071-route moet uit `platform-ansible/vars/instances.yml` na cutover
- Apps blijven OIDC-clients met `auth_oidc`-module (zelfde patroon, ander OP-endpoint)

**Uitrol in 4 fasen via [[task #41]] (epic) + #42-#46**:
1. ADR 0006 + Keycloak-vs-Authentik PoC
2. SRVV-ID-01 + IdP-deploy + HAProxy/Caddy + firewall
3. First OIDC-client (myschool-test)
4. Full cutover + disable local password

**Beslissing 2026-06-04**: **Keycloak** (ADR 0006, status accepted onder voorbehoud van PoC LDAP-federatie). Argumenten: best-class LDAP/AD-federatie, volwassen hardening, bus-factor (docs/community), simpelere container-stack (2 ipv 4 containers).

**Cutover-vereenvoudiging 2026-06-04**: Geen actieve OIDC-clients op huidige id.olvp.be Odoo:8071-placeholder → directe cutover op zelfde FQDN mogelijk zonder parallel-route. HAProxy-backend voor id.olvp.be wijzigen van Odoo:8071 → SRVV-ID-01:443 in één move, daarna Odoo:8071-instance uit instances.yml weghalen.

**VLAN/IP-keuze definitief 2026-06-04 einde dag**: SRVV-ID-01 blijft op **10.35.0.12 in VLAN 35 (mgmt)** — pragmatische keuze. Oorspronkelijke plan 10.33.0.41/VLAN 33 (VL-SD-SERVICE) uit ADR 0001 vervalt. Reden: Keycloak in mgmt-VLAN past bij Semaphore (.10) en Forgejo-aanloop (.11) — automation/auth-tier samen. Firewall: VLAN 35 → SRVV-INFRA002:636 al actief; HAProxy DMZ → 10.35.0.12:443 nieuwe regel nodig bij cutover #44.

**Cutover-plan 2026-06-05** (morgen, direct-move zonder maintenance-window mogelijk omdat Odoo:8071-placeholder geen actieve clients heeft):
1. Caddy + step-ca cert op SRVV-ID-01
2. Keycloak config switch: start-dev → start --optimized, proxy=edge, hostname-strict=true, bind 127.0.0.1
3. HAProxy backend voor id.olvp.be wijzigen Odoo:8071 → 10.35.0.12:443
4. Verwijder Odoo:8071 als instance uit platform-ansible/vars/instances.yml
5. End-to-end test https://id.olvp.be → Keycloak admin/realm-pages

**SPOF-mitigation (HA) — [[task #47]] open**: Single-point-of-failure blijft staan na #44 — bij IdP-uitval geen nieuwe app-logins (bestaande sessies werken tot JWT-TTL; acc blijft werken via onafhankelijke LDAP-bind). Voor Fase 3 (najaar 2026): onderzoek warm-standby (passieve 2e VM met realm-export-restore cron + DNS-failover) vs active-active cluster (Infinispan + shared postgres). Voorlopige aanbeveling: warm-standby vanwege OLVP-schaal en bus-factor (1-2 admins). Niet blocking voor cutover morgen.

**Gerelateerd**: ADR 0001 (three-role-identity) blijft van kracht voor de architectuur-shape; ADR 0006 supersedes alleen het Odoo-als-id-instance-aspect.

---

## Update 2026-06-06 — realm-strategie test/prod (Keycloak)

**Beslissing (brainstorm)**: test- en prod-identity scheiden via **realms binnen één Keycloak**, NIET via een aparte id-test-VM. Gekozen **optie B** uit 3:
- A. 1 Keycloak, 1 realm (test+prod clients samen) — ❌ verworpen: test-client deelt signing-keys/sessies/vertrouwensgrens met prod.
- **B. 1 Keycloak op SRVV-ID-01, 2 realms die elk een ander AD-domein federeren** — ✅ gekozen. Realm `olvp` (prod + test apps: myschool, myschool-ict, **myschool-test**) → AD `olvp.int`; realm `olvp-dev` (dev apps: myschool-dev, myschool-dev2) → AD `olvp.test`. acc staat los (directe LDAP-bind).
- C. 2 Keycloak-VM's (id + id-test) — enkel zinvol om een Keycloak-**versie-upgrade/breaking config** te testen; los je dan op met een wegwerp-container of Proxmox-snapshot van SRVV-ID-01, niet met een permanente 2e VM.

**Waarom B**: realm = volledig geïsoleerd security-domein (eigen users, clients, **signing-keys**, sessies, token-issuer `iss`). Een `olvp-test`-token (`iss=.../realms/olvp-test`) is per definitie ongeldig bij een prod-client. Past bij bus-factor (1-2 admins): 1 VM, 1 cert, 1 backup.

**AD-federatie (twee domeinen — gecorrigeerd 2026-06-06, eerdere "één AD"-aanname was fout)**: er zijn **twee AD-domeinen** — productie `olvp.int` (SRVV-INFRA002) en test `olvp.test`. Realm `olvp` bindt naar `olvp.int` (prod + test apps → testgroep = echte olvp.int-users, dus zelfde account/pw/SSO als prod). Realm `olvp-dev` bindt naar `olvp.test` (dev apps). Keycloak federeert elk realm onafhankelijk op LDAP-niveau → **géén AD-trust nodig**. Firewall VLAN 35 → `olvp.int`:636 bestaat al. **`olvp.test` bestaat (bevestigd 2026-06-06)**: test-DC **`SRVV-TST-WIN-01` @ `10.200.0.10`** (VLAN 200 / VL-TST-SERVICE). Nog te regelen voor `olvp-dev`: firewallregel VLAN 35 (Keycloak SRVV-ID-01 10.35.0.12) → `10.200.0.10`:636 (LDAPS) + olvp.test-AD-cert in de Keycloak-truststore ([[feedback-keycloak-ldaps-truststore]]). Geen existentie-blocker meer, wel concrete config.

**VM-split (besloten 2026-06-06)**: `SRVV-TST-ODOO-02` → hernoemen naar **`SRVV-DEV-ODOO-01`** met de dev-instances erop (channel `dev`); `SRVV-TST-ODOO-01` blijft pure test (channel `test`, myschool-test). FQDN's wijzigen niet — enkel VM/hostname + inventory + instances.yml `name:` + Proxmox-naam + Caddy-host. Meeliften op de myschool-dev-verhuizing ([[project-odoo-role-migration]] item 41). Lijnt 1-op-1 met de realm-split: TST+prod → `olvp`/olvp.int, DEV → `olvp-dev`/olvp.test.

**Hostname**: realm zit in de URL (`https://id.olvp.be/realms/{olvp|olvp-dev}/...`) → technisch is een aparte hostname NIET nodig. **Optioneel**: `id-test.olvp.be` als **2e HAProxy-frontend naar dezelfde Keycloak-backend** (10.35.0.12:443) — puur routing, geen extra server — voor schone test-client-URL's + een **eigen rate-limit/path-allowlist** (test soepeler, prod strenger). Vereist dan hostname-v2-config die beide hostnames toelaat.

**Aandachtspunten**: 2× LDAP-sync naar dezelfde DC (verwaarloosbaar); realm-export-backup voor beide realms (koppelt aan [[task #47]] warm-standby + Project C-BACKUP); realm-separation test client-integraties wél, Keycloak-upgrade zelf niet (= optie C's niche).

**Vastgelegd 2026-06-06**: addendum-sectie *"Realm-strategie test/prod"* toegevoegd aan `platform-handbook/hosting/decisions/0006-dedicated-idp-keycloak.md` (status accepted). Bevat opties A/B/C, LDAP-aanpak, hostname/routing + WBS-mapping. Project-item GOV/132 op done. Naam `id-test.olvp.be` is vrij sinds de Odoo-id-test-placeholder uit instances.yml verwijderd is. ⚠️ ADR-body heeft nog oude IP `10.33.0.41`; correcte waarde `10.35.0.12` (VLAN 35) staat in het addendum + deze memory.
