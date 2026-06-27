---
name: project-intranet-hosting
description: "Nieuwe Odoo-app 'intranet' (repo ictolvpbe/intranet, branches dev/test/prod LOWERCASE) hosten op de bestaande Odoo-stack, deels bestaande MySchool-instances vervangend. 4 fasen: DEV → ACC → TEST → PROD(juli). Start 2026-06-27."
metadata:
  type: project
---

**Start 2026-06-27.** Tweede Odoo-app naast MySchool: **intranet** uit `github.com/ictolvpbe/intranet` (privaat, lokaal op `/home/demm/SchoolProjects/intranet`, 17 modules). Vervangt deels bestaande MySchool-instances, deels nieuw. Zelfde role-based `odoo-podman`-patroon ([[project-odoo-role-migration]], [[project-hosting-fase1-status]]).

## Doel-config (per VM/slot)
| Instance | VM (IP) | slot/poort | FQDN | expose | channel→branch | vervangt |
|---|---|---|---|---|---|---|
| intranet-dev | srvv-dev-odoo-01 (10.200.14.41) | odoo1/8069 | intranet-dev.olvp.be | public | dev→`dev` | myschool-dev2 |
| intranet-acc | srvv-acc-01 (10.36.0.42) | odoo1/8069 | intranet-acc.olvp.int | internal | prod→`prod` | myschool-acc |
| intranet-acc-test | srvv-acc-01 (10.36.0.42) | odoo2/8070 | intranet-acc-test.olvp.int | internal | test→`test` | myschool-acc-test |
| intranet-test | srvv-tst-odoo-01 (10.200.14.40) | odoo2/8070 | intranet-test.olvp.be | public | test→`test` | *nieuw (8070 vrij)* |
| intranet (prod) | srvv-odoo-01 (10.36.0.40) | odoo1/8069 | intranet.olvp.be | public | prod→`prod` | **myschool.olvp.be** |

myschool-dev (8070 dev-VM), myschool-test (8069 tst-VM), myschool-ict (8070 srvv-odoo-01) blijven.

## Beslissingen (user, 2026-06-27)
- **Branches LOWERCASE**: `dev`/`test`/`prod` (LET OP: anders dan MySchool `Dev`/`master`). acc-test → `test`, acc → `prod`.
- **ACC**: account-admin (myschool-acc + -acc-test) wordt **clean geretired**; SRVV-ACC-01 volledig herbestemd naar intranet. vzdump als vangnet. auth_ldap/AD-delegaties ([[project-ad-member-delegation]]) blijven staan maar ongebruikt. NB intranet brengt z'n eigen account-sync mee (`intranet_core_ldap` + `intranet_admin`).
- **PROD (juli)**: intranet **vervangt myschool.olvp.be** op srvv-odoo-01:8069. Server mag vooraf klaargezet; **DNS-record `intranet.olvp.be` blijft ±1 week naar de OUDE intranet-toepassing wijzen → DNS-cutover NIET aanraken tot die oude app weg is.** Publieke cutover (DNS+HAProxy+LE) = laatste stap.
- **Volgorde**: ① DEV → ② ACC → ③ TEST → ④ PROD.

## Infra-feiten (geverifieerd 2026-06-27)
- **Image: GEEN rebuild nodig.** Intranet python-deps (`google-api-python-client, google-auth, ldap3, requests, zeep`) zijn subset van bestaande superset `localhost/odoo-olvp:19.0` (requests=base Odoo).
- **Per-instance `addons_repo`**: nieuw veld in `vars/instances.yml` (default afwezig = MySchool_addons). `addons-update.yml` aangepast → `item.addons_repo | default(addons_repo)` zodat een VM met gemengde apps (MySchool+intranet) in één run klopt.
- **PAT OK (2026-06-27)**: classic-PAT `github-pat-semaphore` leest intranet + MySchool_addons (geverifieerd via `git ls-remote`). Geen access-wijziging nodig. Test-gotcha: `read -rs PAT` → `PAT` is de variabele-NAAM, token plak je ná de prompt (lege token gaf valse "geen toegang"). Test exact = `git ls-remote --heads https://oauth2:${PAT}@github.com/ictolvpbe/intranet.git`.
- **HAProxy = instances.yml-driven** (`haproxy.cfg.j2` loopt over `odoo_vms`): host-ACL `is_<vm>` = alle publieke FQDN's op die VM, health-check Host = eerste publieke instance. Na SoT-edit → `site.yml`-re-render (vault), geen handmatige ACL's. LE-cert apart uitgeven (certbot HTTP-01 via be_acme op haproxy-1, NA DNS-live; deploy-hook synct naar haproxy-2).
- **Caddy = instances.yml-driven** (`caddy.yml`): re-render geeft step-ca-cert + Caddyfile-blok (default_sni + no-cache login-paths).
- **Poort-afleiding**: container/dirs uit positie N (odoo_instance_N, db/addons/config/odoo-data N); host-poort uit `odoo_port` (`PublishPort 127.0.0.1:<odoo_port>:8069`). Positie 1↔8069, 2↔8070.
- Top-level intranet-modules: `intranet_admin, intranet_dashboard, intranet_directie_dashboard, intranet_kosten_dashboard, intranet_web` + thema **`intranet_theme`** (trekken samen 16 intranet-modules mee).
- **GOTCHA addons-layout**: intranet houdt modules in een **`extra-addons/`-subdir**, MySchool in de repo-root. → nieuw instance-veld **`addons_subdir: extra-addons`** + `odoo.conf.j2` schuift `addons_path` een niveau dieper. Zonder dit: `invalid module names, ignored: ...`. (template+SoT gepusht commit aeb7e80)
- **GOTCHA thema**: `intranet_theme_olvp` staat op **`installable: False`** (dev-branch, OLVP-styling verhuisd naar basis `intranet_theme`) → installeer **`intranet_theme`**, NIET `_olvp` (die wordt `uninstallable`).
- **GOTCHA auto-bootstrap**: een verse role-deploy + draaiende worker met `db_name`+healthcheck → Odoo **auto-init't de DB met `base`** (14 core-modules) vóór je `-i` draait. Daarna enkel de app-modules `-i`-en (base/deps staan er al). DB-init geen drama → werkte met live worker (geen SerializationFailure, want `-i` van nieuwe modules, geen `-u all`).
- **Semaphore lege target_limit**: `-e target_limit=` is een extra-var (hoogste precedence) → set_fact op target_limit werkt NIET; opgelost met aparte `target_limit_eff` (commit dee8afe).

## Standaard post-deploy (ELKE nieuwe instance)
- **Taal `nl_BE`** (Belgisch Nederlands) verplicht als default: `base.language.install` wizard (`lang_ids`) → `ir.default.set('res.partner','lang','nl_BE')` (default nieuwe users) → bestaande `res.users` op nl_BE → commit. In runbook Fase 7. Toegepast op intranet-dev + intranet_acc + intranet_acc_test (2026-06-27); doen bij TEST + PROD.
- admin in intranet-app-groepen + sterk admin-USER-pw (zie Fase 7 / [[task #39]]).

## Voortgang
**Samenvatting:** DEV ✅ + ACC ✅ functioneel klaar (2026-06-27). TEST + PROD nog te doen. Per fase rest enkel: user zet admin-USER-pw + cleanup `.bak` na acceptatie.

**Fase DEV — ✅ FUNCTIONEEL KLAAR (2026-06-27):**
- ✅ SoT: `instances.yml` slot-1 srvv-dev-odoo-01 → intranet-dev (channel dev, branch dev, db intranet_dev, addons_repo ictolvpbe/intranet). + header gedocumenteerd.
- ✅ Playbook: `addons-update.yml` per-instance addons_repo (3 plekken + header). YAML gevalideerd.
- ✅ **Decommission myschool-dev2** via SSH (ansible@10.200.14.41): odoo_instance_1+postgres_odoo_1 gestopt/verwijderd; `db1-data/odoo1-data/addons1/config1` → `.myschool-dev2-bak` (vangnet, db-drift-gotcha). myschool-dev (8070) bleef 200.
- ✅ Stap 1 deploy (`odoo-podman.yml --limit dev-odoo-01`, user-vault) — intranet-dev slot 1 vers gebouwd.
- ✅ Stap 2 addons-update Semaphore (target_fqdn=intranet-dev.olvp.be) → intranet@dev in addons1.
- ✅ Stap 3 DB-init (door Claude via SSH): addons_path-fix + 16 intranet-modules + intranet_theme geïnstalleerd; intranet-dev lokaal `/web/health` 200, `/` 303. myschool-dev (8070) bleef 200.
- ✅ Stap 4 Caddy (user-vault): cert + Caddyfile-blok intranet-dev, lokale TLS 200.
- ✅ Stap 5 DNS: gecorrigeerd `.84`→`.82`. ⚠️ **GOTCHA `.82` vs `.84`**: stond eerst per ongeluk op `.84` (nieuwe gw-blok, geen DNAT) → publiek onbereikbaar + LE HTTP-01 faalt. DNAT 80/443→VIP zit op OUDE gw `.82` ([[project-gateway-cutover]]). Alle publieke FQDN's (ook intranet-test + prod) MOETEN `.82` tot de gateway-cutover. Verifieer `dig +short <fqdn> @1.1.1.1`.
- ✅ Stap 6 LE-cert + HAProxy: `site.yml --limit haproxy` (user-vault) rendert ACL/backend (UP) maar geeft cert NIET uit. LE-cert door Claude via SSH op haproxy-1: `certbot certonly --standalone --http-01-port 8080 --key-type ecdsa -d intranet-dev.olvp.be`. ⚠️ **GOTCHA**: directory-deploy-hook draait NIET bij eerste `certonly` (enkel bij `renew`) → eenmalig handmatig `sudo env RENEWED_LINEAGE=/etc/letsencrypt/live/<fqdn> /etc/letsencrypt/renewal-hooks/deploy/haproxy-deploy.sh` (concat→/etc/haproxy/certs/<fqdn>.pem + reload + sync naar haproxy-2). Renewals daarna automatisch.
- ✅ **Publieke E2E bewezen**: `https://intranet-dev.olvp.be` → /web/health 200, / 303→/odoo→/web/login, cert CN=intranet-dev.olvp.be (Let's Encrypt). Keten DNAT(.82)→HAProxy(LE)→Caddy(step-ca)→Odoo werkt.
- **RESTEERT:**
  7. Post-deploy admin-pw (sterk, KeePassXC) + admin-in-intranet-groepen ([[task #39]]) — **geblokkeerd op user-go** (step-by-step gating; classifier weigerde DB-write na stap 6).
  8. Verify vanaf VLAN 10 ([[feedback-test-from-user-vlan]]).
  9. Cleanup na acceptatie: `*.myschool-dev2-bak`-dirs op dev-odoo-01.

**Fase ACC — ✅ FUNCTIONEEL KLAAR (2026-06-27):** acc-01 herbestemd, account-admin geretired.
- ✅ SoT: `instances.yml` srvv-acc-01 → intranet-acc (8069, prod, db intranet_acc, internal) + intranet-acc-test (8070, test, db intranet_acc_test, internal), beide repo ictolvpbe/intranet + addons_subdir. Gepusht `5c4ab2f`.
- ✅ vzdump door user. ✅ **Decommission** (Claude/SSH): myschool-acc (slot1, was enkel base+1user) + myschool-acc-test (slot2, 47 mod) gestopt/verwijderd; db/odoo/addons/config 1+2 → `.myschool-acc[-test]-bak`. caddy bleef up.
- ✅ (1) deploy `odoo-podman.yml --limit srvv-acc-01` (user, lokaal).
- ✅ (2) addons-update **LOKAAL** (niet Semaphore — runner→prod-webapps VLAN36 timeout, [[feedback-semaphore-proxyjump-prod-webapps]]): `ansible-playbook addons-update.yml -e target_fqdn=intranet-acc.olvp.int,intranet-acc-test.olvp.int --ask-vault-pass` (+ `export github_pat`, GEEN `--limit` want PLAY1=localhost). ⚠️ **prod-webapps (VLAN36) addons-update moet lokaal** tot de runner→VLAN36-firewall gefixt is (geldt ook voor PROD srvv-odoo-01).
- ✅ **DB-pw-drift gefixt** (zal op PROD ook gebeuren bij slot-hergebruik!): fresh postgres-init las stale secret `odoo-db-passwd` (oude pw) ≠ odoo.conf (nieuwe vault-pw) → `password authentication failed` → odoo crash-loop. **Fix zonder vault**: pw uit `config1/odoo.conf` lezen (sudo) → `printf 'ALTER USER odoo PASSWORD $pwq$<pw>$pwq$;' | podman exec -i postgres_odoo_N psql -U odoo -d postgres` (lokale trust-socket, dollar-quote tegen specialtekens; psql `-v :'x'` werkt NIET door de exec-laag) → `podman secret rm/create odoo-db-passwd` resync → reset-failed + restart odoo.
- ✅ (3) DB-init beide (intranet_acc + intranet_acc_test): 44 modules / 16 intranet-modules elk, lokaal /web/health 200.
- ✅ **(4-7) DONE 2026-06-27**: step-ca-certs beide .olvp.int (user-provisioner-pw) + `caddy.yml --limit srvv-acc-01 --ask-vault-pass`; (5) AD-DNS `intranet-acc[-test].olvp.int` → 10.36.0.42 (user); GEEN HAProxy/LE (intern); (6) post-deploy groepen (Claude) + admin-pw (user); (7) verify vanaf VLAN 34 (FQDN, geen bare-IP). Oude `myschool-acc[-test].olvp.int` AD-records later weg.
- Cruft op acc-01: `_legacy-addons2-20260611-*` + slot3-wezen (db3/odoo3/addons3/config3) — opruimen na acceptatie.

**Fase ACC = functioneel klaar 2026-06-27** (groepen ✅, intern E2E 200/303 ✅). Rest user: admin-USER-pw op beide DBs + KeePassXC; cleanup `.bak`/cruft + oude AD-records na acceptatie.

**Cleanup (2026-06-27, na admin-pw's gezet):**
- ✅ Sessie-vangnet weg: `*.myschool-dev2-bak` (dev-odoo-01, ~203MB) + `*.myschool-acc[-test]-bak` (acc-01, ~285MB) + oude FQDN Caddy-certs (myschool-dev2 / myschool-acc[-test]).
- ✅ HAProxy wees-certs `myschool-dev2.olvp.be` + `id-test.olvp.be` weg op haproxy-1 + -2 (certbot delete + pem + reload).
- ⏳ User: acc-01 pre-existing cruft (`_legacy-addons2-*` + slot3-wezen db3/odoo3/addons3/config3) — classifier blokkeerde Claude (pre-existing data), user ruimt op. DNS-records (C): oude `myschool-acc[-test].olvp.int` (AD) + `myschool-dev2.olvp.be` (one.com).
- ⚠️ **Gotcha**: bij FQDN-teardown ook de Caddy-cert-files verwijderen, niet enkel het blok → anders wees-certs.

**🔴 INCIDENT tijdens cleanup ontdekt + opgelost — myschool-test ~16d publiek 503:** wees-certs braken de step-ca renew-loop (set -e) → cascade-expiry → HAProxy verify required → backend DOWN. Volledig in [[feedback-caddy-cert-renew-fail-isolation]]. Fix: template `caddy-cert-renew.sh.j2` per-cert fail-isolatie (commit d18ec3d) + wees-certs weg + myschool-test-cert vers uitgegeven (user). **Status 2026-06-27: myschool-test publiek 200, backend UP.** RESTEERT: (a) template uitrollen `caddy.yml --limit webapps`; (b) dev-odoo-01 + acc-01 checken op wees-certs/idem renewal-breuk.

**Fase TEST — ✅ FUNCTIONEEL KLAAR (2026-06-27):** intranet-test.olvp.be op srvv-tst-odoo-01 slot 2 / 8070, NAAST myschool-test (8069, ongemoeid behalve korte restart). Geen decommission, geen db-drift (secret actueel). 16 intranet-modules + intranet_theme + nl_BE + admin in 11 groepen. Publiek E2E: DNS .82, /web/health 200, / 303, cert CN=intranet-test.olvp.be (LE), HAProxy backend UP. `caddy.yml --limit tst-odoo-01` rolde meteen de gefixte renew-script uit op tst-odoo-01. RESTEERT: admin-pw (user) + VLAN10-verify. **Cert-check 2026-06-27: dev-odoo-01 + acc-01 schoon** (certs geldig tot 28/6, renew-timer OK, geen wees-certs) → incident was geïsoleerd tot tst-odoo-01.

**PROD**: nog niet gestart (juli). ⚠️ **PROD-les uit TEST**: een whole-VM `odoo-podman.yml`-deploy HERSTART de co-tenant-instance kort (myschool-test herstartte tijdens de TEST-deploy) → op srvv-odoo-01 betekent dit een korte myschool-ict-restart; inplannen. TEST = intranet-test (8070) NAAST myschool-test (8069) op srvv-tst-odoo-01, branch test, public — geen decommission. PROD = srvv-odoo-01 (VLAN36, addons-update lokaal!), vervangt myschool.olvp.be, DNS-cutover pas ná oude intranet-app (juli).

## Te schrijven
Runbook `platform-handbook/hosting/operations/deploy-intranet-instances.md` (na DEV-bewijs, analoog RB-2026-TST-ROLE-MIGRATE).

Gerelateerd: [[project-odoo-addons-repo]] (branch/repo-conventie), [[project-odoo-role-migration]], [[project-hosting-fase1-status]], [[feedback-semaphore-runner-remote-hang]] (deploys lokaal), [[project-ad-member-delegation]] (acc account-admin dat geretired wordt).
