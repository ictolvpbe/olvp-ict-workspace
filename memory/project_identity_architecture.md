---
name: project-identity-architecture
description: "Identity-architectuur voor [[project-odoo-public-access]]: drie gescheiden rollen (id/acc/app), AD als single source of truth, acc volledig intern. Beslist 2026-05-18."
metadata: 
  node_type: memory
  type: project
  originSessionId: 59a419f3-43f1-477b-bb2b-196d0c16926a
---

Aparte identity-laag voor [[project-odoo-public-access]] met drie rollen, beslist 2026-05-18 vanuit GDPR-dataminimalisatie. Canonieke detailbeslissingen: ADR `0001-three-role-identity.md` (architectuur-shape) + ADR `0006-dedicated-idp-keycloak.md` (Keycloak i.p.v. Odoo-als-id) in `platform-handbook/hosting/decisions/`, plus deze memory. _(De oorspronkelijke plan-export `a-id-test-ook-publiek-rosy-stroustrup.md` van 2026-05-21 is superseded — beschreef nog de Odoo-als-id-opzet + acc op :8072.)_

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

---

## Update 2026-06-08 — geverifieerde stand SRVV-ID-01 + blocker slice C (olvp-dev realm)

**Live stand Keycloak (PoC #43 draait sinds 2026-06-04)**: SRVV-ID-01 (10.35.0.12) up; Keycloak 26 + Postgres 15 (Quadlet) healthy. Realm `olvp` + LDAP-federatie `srvv-infra002-ad` (olvp.int) met **3075 users** gesynct (pagination werkt). Realm `olvp-dev` bestaat nog **niet**. Run-modus nog PoC: `start-dev`, `KC_PROXY=none`, `KC_HOSTNAME_STRICT=false`, `KC_HOSTNAME=id.olvp.int`. **Geen Caddy/TLS** op de VM (443 dicht). Truststore: enkel `srvv-infra002.crt` (olvp.int). DNS `id.olvp.be` + `id-test.olvp.be` → 84.199.147.82 (VIP, restant placeholder). HAProxy heeft **geen id-backend** (haproxy.cfg.j2 is 100% `odoo_vms`-gedreven → Keycloak vereist een **special-case** frontend/backend + eigen Keycloak-path-allowlist, niet de Odoo `/web/login`-allowlist).

**✅ SLICE C FUNCTIONEEL BEWEZEN (2026-06-08, vroeg-validatie tegen oude DC).** Realm `olvp-dev` + LDAP-provider `olvp-test-ad` aangemaakt; **688 users gefedereerd uit olvp.test via LDAPS** ("Sync all users finished: 688 imported"). Bewijst de hele keten: truststore-fetch, AddHost (FQDN-resolutie + cert-SAN-match), bind-account, LDAPS/636, pagination. Prod-realm `olvp` (3075) ongemoeid. **Maar: gebonden aan de OUDE DC `srvv-test-olvp.olvp.test` (10.200.0.9)**, want `.10` had nog geen cert. Methode: role-flow met `.9`-override extra-vars (truststore + AddHost) + realm-script (commit `a75c97a`); bind-DN `CN=keycloak,CN=Users,DC=olvp,DC=test`, pw in vault `vault_keycloak_olvp_dev_bind_credential`. Provider-naam DC-neutraal (`olvp-test-ad`) → **re-point naar `.10` = enkel connectionUrl-wissel** na de CA-migratie (RB-2026-ADCS-TEST → `.10` enrollt cert; role-defaults targetten al `.10`). Hiccup onderweg: admin copy/paste-typo (kcadmin bootstrap-pw staat vast sinds first-boot 4 juni, latere vault-wijziging verandert kcadmin niet). **KC-log waarschuwt "Hostname v1 options [proxy, hostname-strict-https] still in use"** → adresseren in slice A (prod-harden = hostname-v2).

**✅ SLICE A+B AF EN GEVERIFIEERD (2026-06-08).** Keycloak draait in productie-modus (`Cmd=[start]`, "Profile prod activated", v1-warning weg) HTTP-only op `127.0.0.1:8080`, met hostname-v2 (`KC_PROXY_HEADERS=xforwarded` + `KC_HOSTNAME_STRICT=false` → dynamisch dual-host). Caddy (commit `02a930c`, in de keycloak-role) TLS-terminate op `:443` met step-ca certs per FQDN (`id.olvp.be` + `id-test.olvp.be`, issuer OLVP Internal CA) → `reverse_proxy 127.0.0.1:8080`. **Dual-host bewezen**: `id.olvp.be/realms/olvp` → issuer `https://id.olvp.be/realms/olvp`; `id-test.olvp.be/realms/olvp-dev` → issuer `https://id-test.olvp.be/realms/olvp-dev`. Run via `-e @vars/keycloak-slice-ab.yml` (draagt de `.9`-binding mee). Hobbels onderweg (alle in role gefixt): step CLI ontbrak op de clone (handmatig gekopieerd); Quadlet-wijziging herstartte de container niet (`state: started` no-op → nu `notify: Restart`, commit `2c56fed`); lege `id.olvp.be.crt` liet Caddy crash-loopen (cert-gate valideert nu PEM + reset start-limit, commit `4d23322`).

**✅ ITEM 1 COMPLEET (2026-06-08)**: CA-migratie `.9→.10` geslaagd ([[project-ad-serv01-migration]]) + olvp-dev re-pointed naar `.10` (connectionUrl `ldaps://srvv-tst-win-01.olvp.test:636`, sync "688 updated", geen PKIX-fout). `keycloak_production`+`keycloak_caddy_enabled` staan nu in inventory-hostvars (srvv-id-01); `vars/keycloak-slice-ab.yml` verwijderd (commit `08e2472`). Een plain `keycloak.yml`-run wijst nu naar `.10`.

**✅ SLICE D AF EN GEVERIFIEERD (2026-06-08)**: HAProxy special-case voor Keycloak live (commit `f8f2f5f` haproxy.cfg.j2 + group_vars `keycloak_backend_ip`/`keycloak_public_fqdns`; `site.yml` gehard naar `hosts: haproxy` commit `e8ab8b9`). `be_keycloak` UP/L7OK200 (firewall DMZ→10.35.0.12:443 staat). Publiek e2e bewezen: `https://id.olvp.be/realms/olvp/...` + `https://id-test.olvp.be/realms/olvp-dev/...` → correcte per-host issuers, **Let's Encrypt-cert op HAProxy** (geen -k nodig), TLS re-encrypt naar Caddy→KC. public-restricted allowlist (`/realms/ /resources/ /js/ /robots.txt`; `/admin`+`/metrics`+`/health` intern) — **extern bevestigd 2026-06-08**: `/admin` geeft 403 vanaf een echte internet-bron (gsm/mobiele data); intern = from_internal-exempt. DNS id.olvp.be + id-test.olvp.be → VIP (al sinds placeholder-tijdperk). Deploy lokaal met `site.yml --tags haproxy --ask-vault-pass` (Semaphore hangt op remote-first).

**RESTEERT voor [2]**: (1) ~~CA-migratie + re-point~~ ✅; (2) ~~slice D~~ ✅. **Volgende: Odoo-instance als OIDC-client testen** (auth_oidc → Keycloak realm). Plus losse cleanups: externe /admin-deny bevestigen (gsm/mobiele data → 403), stale `olvp-SRVV-TST-WIN-01-CA` AD-reg + .9-retire + truststore-cruft ([[project-ad-serv01-migration]]).

**App→IdP back-channel pad (ontwerp-leemte, 2026-06-08 — gevonden bij OIDC-test)**: OIDC-clients (Odoo) doen server-side token/userinfo-calls naar Keycloak. Geverifieerd dat een app-VM (dev-odoo-01, VLAN 207) Keycloak NERGENS kon bereiken: `10.35.0.12:443` dicht, DMZ-VIP `10.21.0.5:443` dicht, publieke VIP-hairpin timeout. **Gekozen patroon = split-horizon per-app**: browsers gebruiken publieke DNS → VIP → HAProxy(LE) → Caddy → KC (onveranderd); **app-servers krijgen een per-instance `AddHost id-test.olvp.be/id.olvp.be → 10.35.0.12`** (interne Caddy) + firewall app-VLAN → `10.35.0.12:443`. Odoo-container vertrouwt al de step-ca root (`olvp-ca.pem` in image) → TLS valideert op de step-ca-cert (CN matcht), issuer blijft de publieke FQDN (Caddy zet X-Forwarded-Host). **Split-DNS verworpen**: id.olvp.be intern naar .12 voor iedereen zou interne browsers de step-ca-cert geven → cert-warning. Geldt 1-op-1 voor prod (VLAN 36 → .12:443). Te realiseren in de odoo-podman-role (AddHost-var) + firewallregel.

**Centraal-beheer-eisen voor app-OIDC (besproken 2026-06-08)**: user-eis = groepslidmaatschap + identiteit centraal (AD/Keycloak), niet per-app lokaal beheren. Verduidelijkt: (a) de lokale `res.users.id` (integer-PK) verschilt inherent per Odoo-DB — **geen probleem**, correlatie gebeurt op de stabiele externe sleutel (OIDC `sub`=`oauth_uid`, en/of login=email/UPN); (b) de Odoo-login is stuurbaar via claim-mapping → zet op **email** voor een consistente, schone login in elke instance (geen `...@realm@...`-vorm); (c) **centraal groepsbeheer doet het built-in `auth_oauth` NIET** — vereist een `groups`/`roles`-claim in het Keycloak-token (client-scope/mapper) + een Odoo-module die die claim bij elke login → Odoo-groepen mapt (`auth_oidc` OCA basis, volledige rol-mapping vaak custom). **Dit is dé reden dat de architectuur `auth_oidc` voorschrijft i.p.v. auth_oauth.** Test met auth_oauth bewijst enkel de plumbing (connectie/redirect/JIT); central-management = follow-up met auth_oidc + groups-claim-mapping (odoo-dev-werf). Eventueel lokaal groeps-/user-edit afschermen tegen divergentie.

**✅ OIDC-PLUMBING-TEST GESLAAGD (2026-06-08)** — myschool-dev2 (dev-odoo-01) ↔ realm olvp-dev. Volledige keten bewezen: browser → HAProxy(LE) → Caddy(step-ca) → Keycloak → olvp.test AD-auth (user `mark.demeyer`/mark.demeyer@olvpedu.be) → token → Odoo back-channel (intern via AddHost→.12, step-ca-trust via REQUESTS_CA_BUNDLE) → userinfo → JIT-user → ingelogd. **auth_oauth-gotchas (Odoo 19 ↔ Keycloak)**: (1) system-param `auth_oauth.authorization_header=True` nodig — anders stuurt Odoo het token als query-param i.p.v. Bearer → KC userinfo "Missing token"; (2) **Data URL (`data_endpoint`) leeg laten** — userinfo (validation_endpoint) levert alle claims; een waarde als "Keycloak" → `MissingSchema`; (3) Keycloak-client `odoo-myschool-dev2` (implicit+standard flow) via `roles/keycloak/files/create-oidc-client.sh`. **JIT-user kreeg PORTAL-toegang** (niet internal) → bevestigt dat auth_oauth onvoldoende is voor central-management. **Follow-up = auth_oidc**: internal-access + groups-claim→Odoo-group-mapping + login=email-mapping (odoo-dev-werf). Infra (back-channel/CA-trust/AddHost/firewall VLAN207→.12:443) wordt hergebruikt door auth_oidc.

**✅ AUTH_OIDC KEYCLOAK-KANT (A) AF (2026-06-08)**: centraal groepsbeheer voorbereid. `roles/keycloak/files/add-groups-claim.sh` (commit `5f97250`/`fed34a3`): (1) group-ldap-mapper op olvp-test-ad → **235 AD-groepen** uit olvp.test gesynct (groups.dn=DC=olvp,DC=test, member/DN-strategie); (2) oidc-group-membership-mapper op client odoo-myschool-dev2 → `groups`-claim in id/access/**userinfo**-token. **Geverifieerd**: token van mark.demeyer bevat `['bgrp-ict-od','bgrp-pers-bawa','grp-ict-od','grp-pers-bawa']`. Gotcha: protocol-mapper-config = platte strings (component-config = arrays) → arrays gaven "Cannot parse the JSON". LDAP-federated memberships staan NIET in `user_group_membership` (LOAD_GROUPS_BY_MEMBER_ATTRIBUTE = live bij login) — verifieer via het token, niet de DB-tabel.

**RESTEERT = auth_oidc Odoo-kant (B), odoo-dev-werf**: OCA `auth_oidc` vendoren in MySchool_addons + installeren; OIDC-provider (code-flow, discovery, client_id `odoo-myschool-dev2` + secret); `groups`-claim → Odoo-groep-mapping (auth_oidc-basis of custom); **internal user** i.p.v. portal; **login=email + geen JIT/link-bestaande + sAMAccountName/employeeID-fallback** (beslist 2026-06-09, zie update onderaan). Infra (back-channel/AddHost/REQUESTS_CA_BUNDLE/firewall) staat klaar.

**(historisch) slice D was**: publieke cutover (HAProxy special-case Keycloak-frontend/backend + path-allowlist + firewall DMZ→`10.35.0.12:443`, beide FQDN's; raakt DMZ → expliciete go). Step-ca certs zijn 24h-validity met auto-renew-timer (zoals Odoo-pattern).

**Prereq-1-resolutie (besloten 2026-06-08): "optie C" = key-behoudende CA-migratie.** De olvp.test-CA staat op de af te bouwen 2016-DC; i.p.v. enrollen van een stervende CA migreren we de CA naar SRVV-TST-WIN-01 — meteen dry-run voor de prod-AD-verhuizing. Zie [[project-ad-serv01-migration]] + runbook RB-2026-ADCS-TEST. (Diagnoses op de oude CA waren groen → enkel-enrollen had ook gekund, maar C gekozen voor de dry-run-waarde.)

**BLOCKER slice C (realm olvp-dev → olvp.test) — twee externe prereqs (gebruiker-actie):**
1. **Test-DC `SRVV-TST-WIN-01` (10.200.0.10, olvp.test) heeft GEEN LDAPS-cert** (geverifieerd 2026-06-08): TCP 636 open, maar TLS-handshake `read 0 bytes` = geen servercert. Controle: prod-DC olvp.int 10.33.0.10:636 `read 2071 bytes` = cert OK. → test-DC moet eerst een Server-Auth-cert in de machine-store krijgen (AD CS olvp.test / step-ca-cert / self-signed) vóór Keycloak kan federeren.
2. **Firewall VLAN35 (10.35.0.12) → 10.200.0.10:636 nog DICHT** — UniFi-regel + "Retourverkeer Automatisch Toestaan" nodig.

**olvp.test PKI-topologie (geverifieerd 2026-06-08)**: olvp.test heeft AL een werkende Enterprise CA `olvp-SRVV-TEST-OLVP-CA` + werkende LDAPS — maar op de **oude DC `SRVV-TEST-OLVP` (10.200.0.9, Win2016, wordt uitgefaseerd)**: 10.200.0.9:636 levert cert subject=`CN=SRVV-TEST-OLVP.olvp.test`, issuer=`CN=olvp-SRVV-TEST-OLVP-CA`, geldig tot mei 2027. De nieuwere DC `SRVV-TST-WIN-01` (10.200.0.10) heeft enkel nog **geen DC-cert geënrolld** → daarom 0-bytes-handshake. Prereq-1-fix is dus enrollment, GEEN CA-opzet: op .10 `gpupdate /force` + `certutil -pulse` (of handmatig template *Domain Controller Authentication* via certlm.msc) + reboot. **CA-migratie 10.200.0.9 → SRVV-TST-WIN-01** is een aparte werf bij de 2016-decommission: doe key-behoudend (`Backup-CARoleService` + "use existing private key" + zelfde CA-naam + CRL/AIA/CDP herzien) zodat de root identiek blijft en niets her-uitgegeven/herverdeeld hoeft. **Test-Odoo-instances staan hier los van**: die gebruiken step-ca (Caddy) + LE (HAProxy), niet de olvp.test-CA; key-behoudende migratie raakt ze niet.

Na beide prereqs: olvp.test toevoegen aan `keycloak_truststore_hosts` (role auto-fetch) + realm `olvp-dev` + LDAP-provider (vendor=AD, pagination ON) config. Roadmap-slices voor [2]: A=role prod-klaar (optimized/edge/hostname-strict/localhost-bind), B=Caddy op VM, C=realm olvp-dev (geblokkeerd), D=publieke cutover (HAProxy special-case + firewall DMZ→.12:443, raakt DMZ). Gebruiker wil **beide FQDN's** (id.olvp.be + id-test.olvp.be) als 2 HAProxy-frontends naar dezelfde Keycloak. Zie [[feedback-keycloak-ldaps-truststore]].

---

## Update 2026-06-09 — auth_oidc login/matching BESLIST (na collega-overleg)

Drie punten, te implementeren in de **auth_oidc Odoo-kant (#45, odoo-dev werf)**. Vervangt de
eerdere losse aanbeveling "login=email".

1. **Login = email.** Gebruikers zijn dit gewoon (= hun M365-login). → Keycloak: username-mapping
   van de LDAP-federatie naar `mail`/`userPrincipalName` (i.p.v. `sAMAccountName`); `res.users.login`
   = email. **`sAMAccountName` + `employeeID` worden als extra token-claims meegegeven** (KC
   protocol-mappers) en dienen als **fallback-matchsleutel** (eerder besproken stabiele sleutel).
   Primaire IdP-correlatie blijft `sub` → `oauth_uid`.

2. **Geen JIT — link met bestaande lokale Odoo-users.** Signup/auto-create **uit**. Eerste
   OIDC-login moet de **bestaande** `res.users` vinden (match op email, fallback
   `sAMAccountName`/`employeeID`) en daaraan `oauth_provider` + `oauth_uid` (sub) koppelen; daarna
   matcht het op sub. Vereist **custom matching-logica bovenop OCA `auth_oidc`** (default matcht
   enkel op oauth_uid en creëert bij signup). Users zijn al lokaal aanwezig (bestaande data /
   MySchool-provisioning, vgl. [[project-moodle-nextcloud-hosting]]).

3. **Google-SSO op termijn — haalbaar via Keycloak Identity Brokering.** Google als externe
   OIDC-IdP in Keycloak ("Sign in with Google"); apps blijven ongewijzigd (Keycloak = broker).
   Randvoorwaarden: account-**linking op email** (sluit aan op 1+2 — Google-email moet matchen met
   AD-email), welke Google-tenant (Workspace for Education?), Google = **extra** auth-pad naast
   AD-als-credential-SoT (vervangt AD niet), en 2FA-interactie (Google-MFA vs Keycloak-TOTP). Geen
   beslissing nu; **mini-ADR wanneer concreet**.

**How to apply (auth_oidc-werf):** `res.users.login`=email + signup uit + match-bestaande met
fallbacks (sAMAccountName/employeeID-claims) + `sub`→oauth_uid. KC: login-met-email aan,
extra protocol-mappers voor sAMAccountName + employeeID. Google-brokering pas later, mits
email-linking rond is.

**Punt 1 (KC-kant) — gedraaid op olvp-dev (2026-06-09):** `roles/keycloak/files/add-login-claims.sh`
(platform-ansible commit `6564e2c`) — idempotent: realm `loginWithEmailAllowed=true` + LDAP-attr-mappers
`sAMAccountName`+`employeeID` + client protocol-mappers → claims `email`/`samaccountname`/`employee_id`.
Gedraaid op realm `olvp-dev` (client `odoo-myschool-dev2`, provider `olvp-test-ad`): de 3 protocol-mappers
staan bevestigd op de client (kcadm-check). **Nog te doen**: claim-verificatie via voorbeeld-token
(`generate-example-access-token`) of echte login; daarna **prod `olvp`** (provider `srvv-infra002-ad`,
per app-client). Caveat: `employeeID` moet in olvp.test-AD gevuld zijn (anders claim leeg = ok, fallback);
`mail` gevuld + uniek (duplicateEmailsAllowed=false).

**kcadmin-pw GEROTEERD (2026-06-09) — first-boot-gotcha opgelost.** De live `kcadmin`-pw stond vast op
de first-boot-waarde (4 juni) terwijl vault/KeePassXC sinds de rekey (5 juni) een andere waarde hielden →
auth faalde. Opgelost via **bootstrap-admin recovery** (KC 26): `systemctl stop keycloak` → one-off
`podman run … quay.io/keycloak/keycloak:26.0 bootstrap-admin user --username tmpadm --password:env …`
(tegen dezelfde DB via `--network keycloak` + secret `keycloak-db-passwd`) → start → als `tmpadm`
inloggen → `kcadm set-password` op `kcadmin` → `tmpadm` verwijderd. Nieuwe pw nu in KeePassXC-entry
`keycloak-admin-srvv-id-01` + vault `vault_keycloak_admin_password` (gelijk). Podman-secret
`keycloak-admin-passwd` opnieuw te renderen (role-task heeft `skip_existing: true` → eerst `podman secret rm`,
dán `keycloak.yml --tags secret`). NB: secret-re-render is enkel hygiëne voor een toekomstige verse-DB-bootstrap;
de live `kcadmin` is al via DB geroteerd. Bron-doc: keycloak.org/server/bootstrap-admin-recovery.
