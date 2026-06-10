---
name: project-hosting-fase1-status
description: "Live status van Fase 1 deploy ([[project-phasing]]) — wat in productie staat. Alle 3 Odoo-VMs (TST-ODOO-02, TST-ODOO-01, SRVV-ODOO-01) volledig live via HAProxy → Caddy → Odoo TLS-re-encrypt chain. 6 publieke FQDN's e2e bewezen sinds 2026-06-01."
metadata:
  type: project
---

**Status na 2026-06-01:**

## Live in productie

- **HAProxy HA-paar** draait op `haproxy-1` (10.21.0.6 MASTER, prio 110) en `haproxy-2` (10.21.0.7 BACKUP, prio 100), beide in VLAN 21 (`VL-SD-DMZ-RPROXY`). Services enabled + active, configs gedeployed via [`platform-ansible/site.yml`](platform-ansible/site.yml).
- **Keepalived VRRP** actief, VIP `10.21.0.5` zit op haproxy-1, failover bewezen (BACKUP heeft IP niet).
- **step-ca** draait al sinds 2026-05-28 op `10.21.1.10` (verhuisd naar VLAN 22 op 2026-05-31).
- **UniFi DNAT** actief: `WAN2 84.199.147.82:80+443 → VIP 10.21.0.5:80+443`. UniFi 9.x pad: Settings → Policy table → Create policy → type Forward. **WAN2 nu, dual-WAN volgt in Fase 2.**
- **Placeholder TLS-cert** op beide haproxy-hosts in `/etc/haproxy/certs/placeholder.pem` (30 dagen self-signed, CN=placeholder.olvp.be). Vervangen door ACME volgt in [[task #16]].
- **DNS A-records** bij one.com: `myschool-dev2.olvp.be → 84.199.147.82` actief (vermoedelijk ook andere FQDN's al — te verifiëren).
- **TST-ODOO-02** verhuisd naar `10.200.14.41` (VLAN 207 `VL-TST-WEBAPPS`). Odoo-containers draaien: `odoo_instance_1` (8069), `odoo_instance_2` (8070), `odoo_instance_3` (8071) met postgres-backends. `/web/health` geeft 200.

## Blockers van gisteren — beide opgelost (2026-06-01)

1. **VLAN 207 zone-mismatch**: VLAN was automatisch in zone DMZ gezet i.p.v. Services → verplaatst naar Services → jump-01 → TST-WEBAPPS:22 + HAProxy → TST-WEBAPPS:8069 ontgrendeld. Zie [[feedback-unifi-new-vlan-zone]].
2. **HAProxy health-check 500**: Odoo's multi-database setup vereist Host-header op `/web/health` — zonder Host = 500 → backend DOWN. Template aangepast met `http-check send meth GET uri /web/health hdr Host {{ inst.fqdn }}`. Zie [[feedback-haproxy-odoo-healthcheck-host]].

End-to-end bewezen:
```
curl -sI https://myschool-dev2.olvp.be/
→ HTTP/2 303, location: /odoo, server: Werkzeug/3.0.1 Python/3.12.3
```

Op 2026-06-01 ook **Caddy-Quadlet-pad** opgezet → `direct_backend` tijdelijke route is uit instances.yml weggehaald. Architectuur nu zoals in [[project-architecture-haproxy]] bedoeld: HAProxy (LE-cert, TLS-terminate) → Caddy:443 (step-ca-cert, TLS-re-encrypt) → Odoo:8069 (HTTP). Backend `be_srvv_tst_odoo_02` = **UP, L7OK 200** (Caddy-pad).

**Op-host config op TST-ODOO-02** (niet onder git, kandidaat voor [[task #21]] Ansible-role):
- `/etc/containers/systemd/caddy.container` — Quadlet, host-network, image `docker.io/library/caddy:2-alpine`
- `/etc/caddy/Caddyfile` — `admin off`, `auto_https off`, **`default_sni myschool-dev2.olvp.be`** (zie [[feedback-caddy-default-sni-for-haproxy-check]]), site-block met `tls cert key` + `reverse_proxy 127.0.0.1:8069 { header_up Host {host} }`
- `/etc/caddy/myschool-dev2.{crt,key}` — step-ca cert, 24h validity
- `/usr/local/bin/step` — Smallstep CLI 0.30.2 (gekopieerd van stepca-host)
- `/usr/local/bin/renew-caddy-cert.sh` + `caddy-cert-renew.{service,timer}` — 3×/dag renewal, `--exec` restart Caddy alleen bij daadwerkelijke renewal

**LE-cert auto-sync HA-paar (haproxy-1 ↔ haproxy-2, sinds 2026-06-01):**
- Root SSH-key `/root/.ssh/id_ed25519` op haproxy-1 → public key in `/home/ansible/.ssh/authorized_keys` op haproxy-2 (ansible-user heeft NOPASSWD sudo)
- Renewal-hook `/etc/letsencrypt/renewal-hooks/deploy/haproxy-deploy.sh` op haproxy-1: lokaal concat + reload + ssh-pipe naar peer + remote reload, fail-safe met `logger -t certbot-haproxy-deploy`
- Force-renewal test 2026-06-01: lokaal + remote mtime binnen 1s, cert identiek op beide hosts
- 60-daagse renewal-cyclus nu volledig handenvrij

**TST-ODOO-01 volledig live (2026-06-01) — Docker→Podman migratie geslaagd:**
- Verhuisd 10.200.0.40 → 10.200.14.40 (VL-TST-SERVICE → VL-TST-WEBAPPS)
- Containers gemigreerd Docker → Podman+Quadlets met behoud van databases (`odoo-test`, `odoo-dev`)
- Caddy via `caddy.yml` Ansible-playbook (zelfde pattern als TST-ODOO-02)
- 3 LE-certs op HAProxy (myschool-test, id-test, myschool-dev) + 3 step-ca certs op Caddy
- id-test.olvp.be DNS-A-record gefixt bij one.com (was legacy 46.30.213.100/IPv6 → nu 84.199.147.82)
- E2E bewezen: 4 FQDN's HTTP/2 303 → /odoo via volledige TLS-re-encrypt chain
- Open: Docker package nog geinstalleerd (service disabled), reboot voor iptables-reset bij gelegenheid

**SRVV-ODOO-01 volledig live (2026-06-01, vroeger dan gepland) — Docker→Podman migratie geslaagd:**
- Verhuisd 10.33.0.40 → 10.36.0.40 (VL-SD-SERVICE → VL-SD-WEBAPPS)
- Server niet in gebruik → schoolvakantie-window niet nodig, migratie direct uitgevoerd
- Containers gemigreerd Docker → Podman+Quadlets met behoud van `ICT-PROD` database
- Caddy via `caddy.yml --limit srvv-odoo-01` (inventory uitgebreid met `prod_webapps` group als kind van `webapps`)
- 2 prod LE-certs (myschool, myschool-ict) op HAProxy + 2 step-ca certs op Caddy
- E2E bewezen: `https://myschool.olvp.be/` + `https://myschool-ict.olvp.be/` → HTTP/2 303 → /odoo

**Inventory-gotcha**: ansible-controllen vanuit werkstation (VLAN 34 ADMIN) heeft directe toegang naar Webapps (VLAN 36), maar group_vars/all.yml zet ProxyJump via jump-01 (VLAN 35). Voor srvv-odoo-01 is op host-level `ansible_ssh_common_args: ""` gezet om de ProxyJump te overrulen. Voor Semaphore-runner (later, in mgmt-VLAN 35) is firewall-rule `MGMT-BASTIONS → prod_webapps:22` nodig — TODO firewall-rules-matrix.

**id.olvp.be e2e live (2026-06-01)**: identity-FQDN was eerder placeholder voor SRVV-ID-01 (nooit geprovisioneerd). Pragmatic herzien: identity-instance **co-located** op SRVV-ODOO-01:8071 (instance 3). vars/instances.yml: `id.olvp.be → port 8071, role: identity, expose: public-restricted`. HAProxy identity-allowlist actief (alleen `/web/login` + `/auth_oauth/` + `/web/session/` publiek; volle paths intern). Aparte VM blijft lange-termijn-ambitie per [[project-identity-architecture]].

**SRVV-ODOO-TEMPLATE Tier 1 baseline live (2026-06-01)** op 10.200.0.40 (was vroeger TST-ODOO-DEV, hostname hernoemd). Pure Tier 1 + Odoo-overlay-skeleton (Dockerfile + certs/ + addons-skeleton). Cockpit + cockpit-podman op port 9090 als standaard web-UI per host. Klaar voor Proxmox-snapshot. Toekomstige clones uit deze template:
- **SRVV-ACC-01** (account-admin, 10.36.0.42 VLAN 36) — **netwerk-/edge-laag LIVE 2026-06-02**: clone op 10.36.0.42, interne DNS (`.olvp.int`) gezet, 2 step-ca certs uitgegeven (`myschool-acc.olvp.int`:8069 prod + `myschool-acc-test.olvp.int`:8070 test, beide `expose: internal`), `caddy.yml` deployed (Caddy + renewal-timer up). **ACC-01 container-laag LIVE 2026-06-03** via nieuwe reproduceerbare role `roles/odoo-podman/` (playbook `odoo-podman.yml`, gedreven door instances.yml): 5 containers Up = caddy + postgres_odoo_1/2 + odoo_instance_1/2 (8069 prod `myschool_acc`/master, 8070 test `myschool_acc_test`/Dev). Beide DB's geïnitialiseerd (`odoo -d <db> -i base --stop-after-init --no-http`). Kernkeuzes role: gedeelde image `localhost/odoo-olvp:19.0` (odoo:19.0 + ldap3 + interne CA-certs), DB-pw via Podman-secret (postgres) + db-conn in odoo.conf (odoo, zodat directe CLI werkt), Odoo op 127.0.0.1 (Caddy = enige edge), journald-logging (geen log-bind-mounts), `db_name`+`dbfilter` per instance (deterministisch). **Hobbels onderweg (alle opgelost)**: caddy.yml internal-support + check_mode-safe; log-dir perms → journald; group_vars dir-migratie brak vault-refs ([[project-ansible-groupvars-layout]]); odoo:19.0 uid=100 niet 101 ([[feedback-odoo19-podman-uid]]); DB-init poortconflict → `--no-http`. **Resteert** (status 2026-06-03, oppakken volgende sessie):
  1. **account-admin app-config** — runbook **geschreven**: `hosting/operations/config-account-admin.md` (RB-2026-ACC-CONFIG): LDAP-bind naar AD (`auth_ldap`, LDAPS:636, filter op ICT-groep, JIT-users), TOTP-2FA verplicht voor alle users, audit (login-trail + OCA `auditlog`). **Klaar om uit te voeren**, geblokkeerd op: (a) AD-parameters invullen (DC-IP, base DN, bind-account-DN uit KeePassXC, ICT-groep-DN); (b) verifiëren dat `python-ldap` in de image zit (auth_ldap-dep, normaal core). **`ldap3` NIET aanraken** — bewust aanwezig voor myschool-accountbeheer, image/pip-deps geparkeerd, zie [[project-odoo-image-ldap3]]. Test-instance (`myschool_acc_test`) eerst, dan prod.
  2. **host-firewall lockdown + egress-allowlist** (Fase 1b prio-1): inbound enkel VLAN 34 → :443 + step-ca-pad; egress alleen AD-LDAP(S), NTP, APT-mirror, backup-target. Nog geen runbook.
  3. **strikt backup-regime** voor de acc-DB's (audit-data) inplannen.
  - **Diagnose 2026-06-09 (acc "login niet bereikbaar"-melding)**: server-side is alles **gezond** — vanaf VLAN 34 (controller) geven beide acc-FQDN's `/`→303 en `/web/login`→200, step-ca-cert vers + 24h-auto-renew werkt, TCP 443 open, DNS `*.olvp.int`→10.36.0.42 ✅. Dus géén outage. De melding kwam van **VPN**: het netwerk-/TLS-pad naar 10.36.0.42:443 werkt vanaf VPN (IP-test gaf TLS+HTTP 200), dus de oorzaak is **DNS-over-VPN** (VPN pusht de AD-resolver/`olvp.int`-split-DNS niet → hostnaam resolt niet op het toestel) en/of **step-ca root niet vertrouwd** (cert-warning). **GOTCHA — nooit via IP testen**: bare-IP → geen SNI → valt op `default_sni`=`myschool-acc.olvp.int` (=PROD, 8069, db `myschool_acc`), maar Host=IP matcht `dbfilter ^myschool_acc$` niet → Odoo selecteert geen DB → **blanco scherm** (200, 0 bytes). Altijd de FQDN gebruiken. NB: acc-DB's hebben nog enkel `base` (login enkel lokale admin tot RB-2026-ACC-CONFIG). **Morgen oppakken**: VPN-DNS goed zetten (of hosts-entry + step-ca-root-import als workaround), daarna RB-2026-ACC-CONFIG (auth_ldap + 2FA + audit).
  - **Uitgewerkt 2026-06-10 (acc-bereikbaarheid netwerk + VPN)**: **netwerk-kant (VLAN 34) werkt volledig** (DNS `*.olvp.int` via AD-DC **10.10.0.10** search `olvp.int`; 443 open; HTTP 200). **VPN-kant: enige ontbrekende stuk = interne DNS-push** (netwerk-/TLS-pad werkt al). Beslissingen vastgelegd in `network-physical/network/vpn.md` (§Split-DNS, 2026-06-10) + firewall-matrix rules **A-010** (`ADMIN-WORKSTATIONS`→acc:443, werkt empirisch) + **A-011** (`VPN-ADMIN-CLIENTS` `192.168.3.0/24`→`10.36.0.42`:443, te activeren). **Jouw UniFi/GPO-acties**: (1) OpenVPN-server pusht DNS **`10.10.0.10` + `10.33.0.10`** (push-both) + domain `olvp.int`; (2) firewall-rule A-011 (VPN→acc:443) + **D-001** (VPN→`10.33.0.10:53`, voorwaarde voor de `.33`-helft) + retourverkeer; (3) step-ca root (`platform-ansible/files/step-ca-root.crt`) via AD-GPO in machine-trust-store (warning-free). **Push-both besloten 2026-06-10** (i.p.v. enkel `.10`): `.33`=SRVV-SERV-01 is al gepromoveerde DC/NS maar `:53` nog firewall-dicht vanaf clients → koppelt aan de gefaseerde DNS-client-cutover [[project-ad-serv01-migration]] tracker **NET-4** (firewall→verify→secundair→primair, vóór NET-2). Tot D-001 open is, valt VPN terug op `.10`. **Gotcha herhaald**: nooit via IP testen (default_sni=prod + host-gebonden dbfilter → blanco scherm); altijd de FQDN.
  - **RB-2026-ACC-CONFIG voortgang 2026-06-10**: ✅ python-ldap 3.4.4 in image, ✅ LDAPS 636 OK naar **10.33.0.10** (= bind-DC, SERV-01), ✅ admin/admin op beide consoles, ✅ console cert-trust (Chrome-Linux NSS-import van step-ca-root; bredere uitrol = SEC-5). **AD-params**: base `DC=olvp,DC=int`; **binddn `CN=svc-odoo-ldap,CN=Managed Service Accounts,DC=olvp,DC=int`** (⚠️ container is gMSA-default → **verifiëren dat het een gewone user is, geen gMSA** — gMSA kan geen simple-bind met pw); **login-filter-groep `CN=grp-ict,OU=groepen,OU=olvp,DC=olvp,DC=int`**. **⚠️ ONTHOUDEN: de groepsnaam `grp-ict` kan tijdens de AD-migratie nog wijzigen → login-filter dan bijwerken.** **TLS-gotcha bevestigd (2026-06-10)**: Odoo 19 `auth_ldap` (`res_company_ldap.py:106-113`) doet `ldap://` + `start_tls_s()` = **StartTLS op 389, GEEN LDAPS:636**. 636 is wél open vanaf VLAN 36 maar Odoo kan het niet gebruiken → **389 nodig**. 389 vanaf VLAN 36 → `10.33.0.10` stond **dicht** (timeout) → firewallregel toe te voegen: **intra-zone `Services`** (VLAN 36 én VLAN 33 zitten in dezelfde UniFi-zone `Services`!) `10.36.0.42 → 10.33.0.10 TCP 389` + retourverkeer (spiegel van de werkende 636-regel). **Per-instance AD-domein (bevestigd 2026-06-10): acc-test → olvp.test, acc-prod → olvp.int** (test-instance tegen test-AD = prod-AD niet raken). svc-account = gewone user (PS-script, geen gMSA ✅); grp-ict = **directe** membership (geen chain-match nodig). Params:
  - **acc-test** (`myschool_acc_test`, 8070): DC **`10.200.0.10`** (SRVV-TST-WIN-01, olvp.test, VLAN 200); base `DC=olvp,DC=test`; binddn `CN=svc-odoo-ldap,OU=Managed Service Accounts,DC=olvp,DC=test`; filter-groep `CN=grp-ict-od,OU=grp,OU=od,DC=olvp,DC=test`.
  - **acc-prod** (`myschool_acc`, 8069): DC `10.33.0.10` (olvp.int); base `DC=olvp,DC=int`; binddn `CN=svc-odoo-ldap,CN=Managed Service Accounts,DC=olvp,DC=int`; filter-groep `CN=grp-ict,OU=groepen,OU=olvp,DC=olvp,DC=int`.
  - Firewall: prod-pad VLAN36→`10.33.0.10:389` ✅ open. **Test-pad VLAN36→`10.200.0.10:389` nog te openen** (intra-zone Services, maar **kruist prod→test-grens** → bewuste uitzondering). **Cert-trust-risico**: StartTLS naar olvp.test valideert cert van issuer `olvp-SRVV-TEST-OLVP-CA` → die CA zit vermoedelijk NIET in de odoo-image-truststore (wel step-ca + olvp.int-CA) → StartTLS kan falen → mogelijk olvp.test-CA aan image toevoegen. Te bevestigen via bind-test.
  Volgende: test-firewall open → StartTLS bind-test (cert+binddn+pw+filter) → `auth_ldap` installeren + LDAP-config acc-test.
  - **Voortgang 2026-06-10 (LDAPS-trust opgelost, 2 gotcha's):** (1) **CA-trust**: olvp.test-root-CA (`olvp-SRVV-TEST-OLVP-CA`, self-signed tot 2071) ontbrak in de odoo-image → toegevoegd aan `platform-ansible/files/olvp-test-root-ca.crt` + copy-task in `roles/odoo-podman`; **role-fix**: image-rebuild notifyt nu Restart (anders bleef container op oude image — Quadlet `state:started` no-op). (2) **Hostname-verificatie**: python-ldap StartTLS checkt cert-SAN; DC-cert-SAN = `DNS:SRVV-TST-WIN-01.olvp.test` (geen IP) → verbinden op `10.200.0.10` gaf `CONNECT_ERROR`. Fix: `odoo_extra_hosts` `{ srvv-tst-win-01.olvp.test → 10.200.0.10 }` op srvv-acc-01 (inventory) + Odoo LDAP-server = **FQDN** (niet IP). `openssl s_client -starttls ldap -CAfile … → Verify return code 0` bevestigde dat de chain klopt. **Geldt ook voor prod**: gebruik daar de olvp.int-DC-FQDN, niet `10.33.0.10`. (3) **libldap mist default-CA**: zelfs met cert in de image-bundle + FQDN bleef `start_tls_s()` falen — `/etc/ldap/ldap.conf` heeft geen `TLS_CACERT`, dus libldap (auth_ldap's kale start_tls_s) leest de system-bundle niet. **Fix**: `Environment=LDAPTLS_CACERT=/etc/ssl/certs/ca-certificates.crt` op de odoo-container (Quadlet-template, naast REQUESTS_CA_BUNDLE). Bewezen met `podman exec -e LDAPTLS_CACERT=… → OK-env-CA`. Drie gotcha's samen (CA-in-image + extra_hosts/FQDN + LDAPTLS_CACERT) = volledige LDAPS-trust. StartTLS naar FQDN = `STARTTLS OK` ✅.
  - **INCIDENT 2026-06-10 — 502 na de role-apply (DB-pw-drift, opgelost):** beide acc-instances crashten (`password authentication failed for user "odoo"`). Oorzaak = latente drift: `odoo.conf` rendert `db_password` altijd uit `vault_odoo_db_password`, maar `postgres_odoo_N` zet z'n pw enkel bij init (06-03, oude pw in data-dir) en de Podman-secret `odoo-db-passwd` wordt door `skip_existing: true` nooit bijgewerkt. De db-pw-rotatie van 06-05 wijzigde de vault → de apply renderde odoo.conf met de nieuwe pw → mismatch → crash-loop. **Fix:** `\password odoo` (nieuwe vault-pw) op `postgres_odoo_2` én `_1` (lokale trust-socket in de container) + `systemctl restart odoo_instance_*` → beide `Up (healthy)`. **Geldt voor ÁLLE Odoo-VMs**: elke apply ná een vault-db-pw-wijziging breekt draaiende instances tot postgres mee-geroteerd wordt. **TODO (volgende werf, test op dev eerst):** (a) idempotente pw-sync-task in de role (ALTER USER = vault) zodat dit niet meer kan; (b) de stale `odoo-db-passwd`-secret op acc bijwerken naar de vault-pw. Drift-gotcha + rotatie-procedure staan nu als comment in `roles/odoo-podman/tasks/main.yml` (secret-task). Koppelt aan [[task #40]].
- **SRVV-FORGEJO-01** (10.35.0.x in VLAN 35 gepland) — clone + Forgejo-overlay-role (cleanup `/opt/odoo-multi-instance/`-residu, deploy Forgejo Quadlet)

Architectuur-design vastgelegd: één multi-doel template (Tier 1) i.p.v. één per applicatie. Zie [[project-template-strategy]].

## Volgende werven

1. **[[task #16]] ACME-cert** voor `myschool-dev2.olvp.be` (of paralel meerdere FQDN's) — placeholder vervangen door Let's Encrypt via HTTP-01 challenge (be_acme backend staat klaar)
2. **[[task #17]] Caddy-Quadlet** op TST-ODOO-02 — vervangt het tijdelijke `direct_backend`-pad door reguliere TLS-re-encrypt architectuur (haproxy → Caddy:443 → Odoo). Dan kan `direct_backend: true` uit instances.yml + template-loop kan eventueel terug naar één-pad-only
3. Andere FQDN's via dezelfde flow: `myschool-test.olvp.be`, `id-test.olvp.be`, `myschool.olvp.be` (na prod-Odoo VLAN-shift)

## Updates 2026-06-02

**SRVV-FORGEJO-01 in opbouw**: VM gekloond van SRVV-ODOO-TEMPLATE → 10.35.0.11 (VLAN 35 mgmt). AD-DNS A-records `git.olvp.int` + `forgejo.olvp.int` → 10.35.0.11. Tier 1 baseline aanwezig. step-ca cert uitgegeven. Forgejo geconfigureerd voor SQLite (na postgres-pivot — zie [[project-forgejo-status]] + [[task #35]]). Web-server start onstabiel — startpunt volgende sessie. Niet productief in gebruik.

**Nieuwe Ansible-werven**:
- `platform-ansible/tier1-baseline.yml` — generieke Tier 1 baseline-playbook
- `platform-ansible/forgejo.yml` — Tier 2 overlay voor Forgejo
- `platform-ansible/group_vars/` herstructureerd naar `all/{vars,vault}.yml` (was `all.yml` + `all_vault.yml` — vault werd niet auto-geladen, zie [[feedback-ansible-vault-loading]])

**Stale-CSRF mitigation uitgerold op ALLE Odoo-VMs (2026-06-03)**: Caddy no-cache headers op `/web/login*`, `/web/database/*`, `/web/session/*`, `/auth_oauth/*` in Caddyfile-template. Voorkomt 'F5 na lange wait → CSRF-error'. Login UI ook merkbaar sneller. Bevestigd op alle 6 publieke FQDN's via `curl -kI ... | grep cache` — `cache-control: no-store, no-cache, must-revalidate, max-age=0` aanwezig op `myschool-test`, `id-test`, `myschool-dev`, `myschool-dev2`, `myschool`, `myschool-ict`, `id`.

**Firewall VLAN 10 → DMZ-VIP geopend**: User-VLAN had geen toegang tot HAProxy VIP 10.21.0.5. Policy USERS → DMZ:80,443 toegevoegd. Wifi-VLANs check als follow-up [[task #34]].

**Identity-werf design-shift 2026-06-04**: dedicated IdP (Keycloak/Authentik) op SRVV-ID-01 vervangt Odoo-als-id-instance plan. Huidige id.olvp.be (Odoo:8071 op SRVV-ODOO-01) is **tijdelijke placeholder** tot cutover naar dedicated IdP-VM. Werf [[task #41]] epic met 5 sub-tasks. Bestaande [[project-identity-architecture]] memory bijgewerkt met design-update; 3-rollen (id/acc/app) BLIJVEN, alleen id-implementatie wijzigt. ADR 0006 (Keycloak vs Authentik) te schrijven. Eerste aanbeveling: Keycloak.

**PRIO security-hygiëne open 2026-06-03**: Wachtwoord-rotaties op alle Odoo-VMs:
- [[task #39]] — `admin_passwd` (Odoo DB-manager master-pw) per instance in odoo.conf
- [[task #40]] — DB-pw (Odoo↔Postgres) per instance: ALTER USER + odoo.conf db_password + Podman-secret (ACC-01)
Beide vereisen: nieuwe waarden in KeePassXC + ansible-vault, render via Ansible waar mogelijk. ACC-01 al klaar voor secret-rotation via [[project-ansible-groupvars-layout]] vault. Oude VMs (SRVV-ODOO-01, SRVV-TST-ODOO-01/02) hebben odoo.conf-edit + restart nodig per instance.

**Semaphore Addons Update operationeel**: Werf #29 voltooid. Per-env updateerbaar (prod/test/dev gescheiden), repo `ictolvpbe/MySchool_addons` met case-sensitive branches `master`/`Dev` — zie [[project-odoo-addons-repo]].

## Updates 2026-06-05

**addons-update channel-refactor + flexibelere selectors** (zie [[project-odoo-addons-repo]]): per-instance `env` → `channel` (prod/test/dev), losgekoppeld van VM-tier `env`. Nieuwe survey-var `target_fqdn` (komma-lijst exacte FQDN's, wint van `target_channel`); `target_env` → hernoemd `target_channel`. Drie selectors: target_limit (VMs) → target_channel (kanaal) → target_fqdn (exact). Plus: restart_after/upgrade_modules/upgrade_db (activeren na pull), boolean-guards, env-pattern-skip bij upgrade, fail-fast bij niet-matchende selector. **myschool-ict her-gechanneld prod→dev** (Dev-branch op prod-hardware SRVV-ODOO-01). Semaphore-template TODO: survey `target_env`→`target_channel` hernoemen + `target_fqdn` toevoegen + vault-key `olvp-ansible-vault` koppelen + Repository=platform-ansible (zie [[feedback-semaphore-template-repo-vault]]).

**TODO volgende sessie — id-instances teardown op de VM's**: `id.olvp.be` + `id-test.olvp.be` zijn uit `instances.yml` verwijderd (worden Keycloak op SRVV-ID-01). SoT-change staat in `main`; routing valt weg bij volgende caddy/haproxy-render. Resterende teardown:
- **id-test.olvp.be** (TST-ODOO-01, test, laag risico): haproxy+caddy re-render → `odoo_instance_2` (port 8070) stoppen+verwijderen + addons2/cert opruimen. Eerst doen.
- **id.olvp.be** (SRVV-ODOO-01, **prod-VM** — expliciete go per stap): `odoo_instance_3` (port 8071). Staged tot Keycloak-cutover om identity-downtime te vermijden; DNS-record blijft (re-point naar Keycloak:8443 = [[task #41]]).
- Destructieve commando's pas op uitvoeringsmoment genereren ([[feedback-no-preview-destructive]]).

## Updates 2026-06-06

**VM-rename SRVV-TST-ODOO-02 → SRVV-DEV-ODOO-01** (Fase 1-2 van [[project-odoo-role-migration]] item 41a): schone test/dev-split. Hostname/Proxmox/Cockpit-cert + SoT (`instances.yml` name, `inventory.yml` nieuwe `dev_webapps`-groep) hernoemd, IP/VMID ongewijzigd (`10.200.14.41`/`10084`). Role matcht op IP → geen container-impact, e2e `myschool-dev2.olvp.be` = HTTP/2 303. Commit platform-ansible `fe87928`. Resteert: HAProxy-redeploy (backendnaam-reconcile, Fase 3) + myschool-dev-relocatie (item 41b). DEV-VM mapt op Keycloak-realm `olvp-dev`/AD `olvp.test` ([[project-identity-architecture]]).

## Vault-pw delivery (technische schuld)

KeePassXC-wrapper-script `~/.local/bin/ansible-vault-pw-olvp` bestaat maar werkt niet met `ansible-vault` (stdin-sluit-issue, ook met `</dev/tty`-redirect). Workaround: `--ask-vault-pass` met copy-paste uit KeePassXC entry `olvp-ansible-vault-master`. **Eerste pw-poging gisteren werd geweigerd door typo** — altijd copy-paste, niet typen. Voor non-interactief gebruik (Semaphore, cron): later secret-tool/libsecret-cache opzetten.
