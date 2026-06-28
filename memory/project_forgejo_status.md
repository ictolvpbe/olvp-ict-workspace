---
name: project-forgejo-status
description: Forgejo provisioning werf #30 — SRVV-FORGEJO-01 op 10.35.0.11 (VLAN 35 mgmt). MISDIAGNOSE GECORRIGEERD 2026-06-28: het "postgres pq-driver"-issue was nooit postgres — Forgejo crash-loopt op SSH-bind poort 22 (UID 1000, geen CAP_NET_BIND_SERVICE). Postgres-in-podman werkt prima (121 tabellen). Fix: START_SSH_SERVER=false.
metadata:
  type: project
---

**Status 2026-06-28 — root cause gevonden, postgres-mysterie was MISDIAGNOSE**:

Vergelijking SRVV-FORGEJO-01 ↔ SRVV-ACC-01 (read-only, infra-ops) weerlegde de Podman-versie-hypothese: **beide draaien bit-identiek Podman 5.4.2 + netavark 1.14.0 + kernel 6.12.90, zelfde subnet 10.89.0.0/24**. NetworkAlias `db` registreert WEL correct (oude memory-notitie hierover was fout).

De echte oorzaak stond in de `forgejo.service`-journal (restart-counter **209.464**, continue crash-loop):
- DB-connectie LUKT bij poging #1: `PING DATABASE postgres` → `ORM engine initialization successful!`. De `forgejo`-DB heeft **121 tabellen** → migratie ooit volledig geslaagd, draait écht op postgres (geen sqlite-bestand).
- Crash: `[E] Unable to GetListener: listen tcp :22: bind: permission denied` → `[F] Failed to start SSH server` → `Received signal 15; terminating.`
- Forgejo's ingebouwde Go-SSH-server (`START_SSH_SERVER=true`, `SSH_LISTEN_PORT=22`) probeert `0.0.0.0:22` te binden als UID 1000 zonder `CAP_NET_BIND_SERVICE` → EACCES → FATAL.
- De `Connection reset by peer`-regels in de postgres-journal zijn **collateral**: gezonde DB-connecties die met RST sneuvelen als het proces ~2s later FATAL exit. Geen auth-/protocolfout. We hebben 2/3-juni het verkeerde spoor gevolgd.

**Conclusie**: postgres-in-podman is robuust op deze stack (zoals ACC-01) — Podman+Quadlet-conventie hoeft niet gebroken. Geschikt voor multi-team gebruik (reden om postgres i.p.v. sqlite te houden).

**Fix DOORGEVOERD 2026-06-28 (Optie A)**: `START_SSH_SERVER = false` in `app.ini [server]` (git-over-SSH via openssh in image, bindt 22 als root; bestaande `PublishPort=127.0.0.1:2222:22` blijft). Resultaat geverifieerd:
- `forgejo.service` = **active**, NRestarts=**0** (was failed/209k-loop); `curl :3000` = **200 OK**; Caddy publieke route **502→200** (HTTP/2, Host git.olvp.int).
- Backup live config: `/opt/forgejo/config/conf/app.ini.bak-20260628`.
- Template `templates/forgejo.app.ini.j2` ook gefixt — **uncommitted** in working tree, wacht op review/commit.
- Nevenfix: `caddy.service` stond failed (losse podman exit-125 van 13-jun) → hersteld, active.
- Cert-renewal-timer (`caddy-cert-renew.timer`) uitgerold met fail-isolatie. Host-pad-afwijking t.o.v. Odoo/Caddy-VMs: `step` op `/usr/bin/step`, root op `/root/.step/certs/root_ca.crt` (i.p.v. `/usr/local/...`). Nu OOK in playbook (commit `a0d3f1d`): `forgejo.yml` deployt script+service+timer; `caddy-cert-renew.sh.j2` geparameteriseerd met `step_bin`/`step_root_ca` (defaults = caddy.yml-gedrag ongewijzigd), forgejo.yml geeft de host-correcte overrides → handmatige uitrol nu idempotent reproduceerbaar.

**Admin-account 2026-06-28**: bootstrap via app.ini maakt GEEN admin (alleen install-wizard-skip); user-list was leeg → eerste admin handmatig aangemaakt via `podman exec -u git forgejo forgejo admin user create`. Account: **`dries.vanmoer`** (ID 1, IsAdmin=true, email dries.vanmoer@olvp.be). Wachtwoord in KeePassXC `olvp-forgejo-admin` (must-change-password=false; 2FA nog in te schakelen). Registratie staat uit (`DISABLE_REGISTRATION=true`) dus nieuwe users ook via CLI of door admin in UI. Login: `https://git.olvp.int/`. Alias `forgejo.olvp.int` losgelaten — we houden het bij `git.olvp.int` (alias werkte niet na reboot; niet verder onderzocht op user-verzoek).

**RESTERENDE BLOCKER**: `/etc/caddy/git.olvp.int.crt` is verlopen sinds 3-jun (24u-cert van 2-jun). `curl -k` verbergt dit; echte browser/HAProxy `verify required` weigert het. Renewal-timer kan verlopen cert NIET redden (verlopen=geen auth). Eerst vers cert uitgeven met provisioner-pw (KeePassXC `olvp-stepca-admin-provisioner-password`): `sudo step ca certificate git.olvp.int /etc/caddy/git.olvp.int.crt /etc/caddy/git.olvp.int.key --san git.olvp.int --san forgejo.olvp.int --not-after=24h --provisioner=admin` + `systemctl restart caddy`. Daarna neemt de timer het handenvrij over.

---

**Status 2026-06-03 (sessie-einde)**: Forgejo-werf gepauzeerd, niet operationeel. Reproduceer-poging met ACC-01-patroon (pg:15 + Podman-secret + NetworkAlias) faalde identiek aan 2026-06-02 setup. Forgejo binary's pq-driver geeft TCP-connection-reset zonder ooit auth te voltooien.

**Status 2026-06-02 (sessie-einde)**:

| Onderdeel | Status |
|---|---|
| VM SRVV-FORGEJO-01 op 10.35.0.11 (VLAN 35) | ✅ Live, ansible-user OK |
| AD-DNS records git.olvp.int + forgejo.olvp.int | ✅ |
| Tier 1 baseline (Podman + step + Cockpit) | ✅ Via SRVV-ODOO-TEMPLATE clone |
| step-ca bootstrap + cert voor beide FQDN's | ✅ |
| Vault `forgejo_db_password` | ✅ In group_vars/all/vault.yml |
| Caddy reverse-proxy met step-ca cert | ✅ Draait |
| Forgejo container | ⚠️ Draait maar web-server (poort 3000) antwoordt niet |
| **Eindstaat**: Forgejo geconfigureerd voor SQLite (na postgres-pivot), maar laatste curl http://127.0.0.1:3000/ gaf geen response | |

**Beslissingen genomen**:
- VLAN 35 mgmt (10.35.0.11), niet VLAN 36 webapps — Forgejo is mgmt-service
- Interne FQDN's `git.olvp.int` (primary) + `forgejo.olvp.int` (alias) via AD-DNS
- GitHub blijft authoritative; Forgejo wordt push-mirror (later op te zetten)
- **SQLite i.p.v. postgres** — pragmatisch besluit na ~2u debug-tijd op een hardnekkig connectie-issue tussen Forgejo 11 en postgres:16-alpine. SQLite is ruim voldoende voor single-admin mgmt-tool.

**Open issues**:
1. **Forgejo web-server start niet**: Na SQLite-pivot draait container, maar curl :3000 hangt zonder response. Niet geverifieerd of dit een herhalig is van postgres-symptoom OF nieuwe issue door SQLite-data-pad. Eerste check volgende sessie: `podman logs forgejo --tail 80`.
2. **Postgres-mysterie**: pq-driver in Forgejo binary kon niet verbinden met postgres:16-alpine ondanks dat psql-client wel werkte (zowel MD5 als SCRAM-SHA-256). Geparkeerd als [[task #35]] voor wanneer SQLite niet toereikend wordt.
3. **Browser cert-warning**: Step-ca-root cert niet in werkstation browser trust store. Toekomstig: deploy step-ca-root naar admin-werkstations.

**Architectuur vastgelegd in code**:
- `platform-ansible/inventory.yml` — group `mgmt_servers → forgejo_servers`
- `platform-ansible/tier1-baseline.yml` — algemene Tier 1 playbook (eerste keer geschreven deze werf)
- `platform-ansible/forgejo.yml` — Tier 2 overlay-playbook
- `platform-ansible/templates/forgejo.app.ini.j2` — Forgejo config (sqlite na pivot)
- `platform-ansible/templates/forgejo.Caddyfile.j2` — TLS-terminate met step-ca cert

**2026-06-03 update — postgres-reproduceer-poging gefaald**:

Geprobeerd met ACC-01's odoo-podman-role-patroon (vermoeden dat het zelfde issue zou zijn). Wijzigingen:
- `postgres:15` (i.p.v. :16-alpine) — werkende versie op ACC-01
- Podman-secret met `Secret=...,type=env,target=POSTGRES_PASSWORD` (i.p.v. plain `Environment=POSTGRES_PASSWORD`)
- `NetworkAlias=db` op postgres-container (faalde te registreren in podman-DNS — alleen short-ID kwam in Aliases-lijst)
- HOST=forgejo-db (ContainerName, wel resolveert) in app.ini
- Force-recreate beide containers (forgejo + forgejo-db)

Resultaat: **identiek hangend** als 2026-06-02. Forgejo logs: 10 retry-pogingen op "PING DATABASE postgres", elke 3s, allemaal "Connection reset by peer" volgens postgres-log. Daarna container exit (Restart=on-failure, geeft op).

**Wat is uitgesloten** (zekerheid > 90%):
- DNS-resolutie (`getent hosts forgejo-db` werkt vanuit Forgejo-container)
- TCP-connectiviteit (`nc -zv forgejo-db 5432` werkt)
- Postgres-versie (16-alpine + 15 identiek falen)
- Auth-encoding methodes (plain env-PASSWORD + Podman-secret-via-env identiek falen)
- Auth-algoritme (scram-sha-256 + md5 identiek falen — geverifieerd met `psql` van buitenaf)
- SSL_MODE (expliciet `disable` in app.ini)
- MTU (1500, standaard)
- Container env-caching (force-recreate van beide containers gedaan)

**Vermoeden bron**: Forgejo 11 binary's pq-driver heeft een specifiek issue met deze rootful-Podman + postgres-container setup. Mogelijk een Podman 4.x vs 5.x verschil (ACC-01 mogelijk op Podman 5.x). Niet eenvoudig binnen project-tijdsbestek te debuggen.

**Pragmatisch besluit 2026-06-03**: Forgejo-werf gepauzeerd voor onbepaalde tijd. Postgres-issue als open vraag in [[task #35]]. Alternatieven:
- Pivot naar SQLite (proven Forgejo-DB, eerder geprobeerd maar deploy onafgemaakt door tijdsdruk)
- Pivot naar Gitea i.p.v. Forgejo (Gitea heeft mogelijk andere pq-versie)
- Cloud-Forgejo (codeberg.org of self-hosted op andere VM met Docker i.p.v. Podman)
- GitHub blijven gebruiken (huidige werkende situatie, geen urgentie)

**VM-staat einde sessie**: SRVV-FORGEJO-01 draait, postgres-container draait, Forgejo-container in restart-loop (zal eventueel stoppen na 5x failures). Caddy reverse-proxy actief maar serveert 502. Step-ca cert geldig 24u, vernieuwt nu nog niet automatisch (renewal-timer niet uitgerold op deze VM).

Gerelateerd: [[project-vcs-strategy]] (GitHub blijft authoritative voor nu), [[project-template-strategy]], [[feedback-ansible-vault-loading]], [[feedback-test-from-user-vlan]].
