---
name: project-sspr-password-selfservice
description: Nieuwe werf — vervang verouderde self-service wachtwoordtool (pw.olvp.be) door gehardende LTB Self Service Password op dedicated VM SRVV-SSPR-01. NIST-beleid + runbook geschreven 2026-06-29; bouw nog te starten (user cloont VM).
metadata:
  type: project
---

**Start 2026-06-29.** De oude SSPR-tool op `pw.olvp.be` (verouderde LTB-versie, publiek exposed) wordt vervangen.

## Beslissingen (user, 2026-06-29)
- **Beleid eerst, dan tool.** NIST 800-63B als basis: lengte > complexiteit, breach-screening (HIBP), géén rotatie-als-security maar wél een **educatieve** jaarlijkse wijzig-campagne, geen security-vragen.
- **Tool = LTB Self Service Password** (`ltbproject/self-service-password` v1.7.x). Afgewogen tegen PWM; PWM's enige pluspunt (helpdesk-module) verviel toen **LS-leerling-self-service geschrapt** werd (reset zeldzaam → ad-hoc AD-side). Entra bewust afgewezen (open source / technologische zelfstandigheid).
- **Doelgroepen**: personeel (≥14) + SO-leerlingen (≥12) self-service; LS-leerlingen NIET in de tool.
- **Dedicated VM** `SRVV-SSPR-01`, VLAN 36, **10.36.0.44** (niet co-located op Odoo-prod — AD-reset-creds + publieke app isoleren). User cloont de Tier-1 golden-image zelf.
- **Self-hosted, publiek via `https://pw.olvp.be`**.

## Geleverd 2026-06-29
- `platform-handbook/governance/password-policy.md` — NIST-beleid + selectiecriteria (commit handbook).
- `platform-handbook/hosting/operations/deploy-sspr.md` — **RB-2026-SSPR-DEPLOY**, volledige uitrol-runbook (8 fasen). Commit `86432c6`.
- Governance-index bijgewerkt.

## Architectuur (uit runbook)
`DNS pw.olvp.be → .82 (oude gw DNAT) → HAProxy (LE) → Caddy (step-ca) op SRVV-SSPR-01 → LTB-container (127.0.0.1:8084) → AD LDAPS srvv-serv-01.olvp.int:636 (10.33.0.10)`.
- **AD-bind = direct op SERV-01 (migratie-DC), niet infra002** (beslist 2026-06-30): binden meteen op `srvv-serv-01.olvp.int`/10.33.0.10 zodat de geplande AD-verhuizing géén herbind vereist. Account+dsacls zijn domein-breed → script op SERV-01 draaien. CA key-behoudend (`olvp-SRVV-INFRA002-CA`) → truststore-cert verandert niet. Zie [[project-ad-serv01-migration]].
- **Service-patroon = Forgejo** (los Quadlet-service + eigen Caddy + overlay-playbook `sspr.yml` nog te schrijven), NIET het Odoo-instances-patroon.
- **HAProxy = `be_keycloak`-patroon**: handmatig toegevoegde niet-Odoo publieke backend (ACL `is_sspr` + `be_sspr` server :443 ssl verify required sni req.hdr(host)). Niet instances.yml-driven.
- AD: dedicated least-privilege bind-account `svc-sspr` + `dsacls` reset-delegatie (`CA;Reset Password;user` + `WP;pwdLastSet;user`) op personeel+SO-OU's (analoog [[project-ad-member-delegation]]).

## Belangrijkste gotchas (in runbook)
- LDAPS op **FQDN** (cert-SAN-match, [[feedback-keycloak-ldaps-truststore]]); AD-CA-cert in container-truststore.
- AD eist LDAPS voor pw-set; **firewall VLAN36→10.33.0.10:636 (VLAN33/SERV-01) nieuw nodig — bevestigd DICHT 2026-06-30**, openen vóór Fase 3. (VLAN36→VLAN10:636 stond al open maar gebruiken we niet.)
- DNS `.82` niet `.84` ([[project-gateway-cutover]]); `pw.olvp.be` bestaat al → cutover = laatste stap, oude tool tot dan laten draaien.
- `default_sni pw.olvp.be` in Caddy ([[feedback-caddy-default-sni-for-haproxy-check]]); LE eerste `certonly` → deploy-hook handmatig.
- AD verhuist later naar SRVV-SERV-01 ([[project-ad-serv01-migration]]) → daarom op FQDN binden.
- Eén `$pwd_min_length` per instance → vloer 12, personeel via policy-tekst naar ≥14.
- Geen reCAPTCHA (Google-dep) → rate-limiting via HAProxy/fail2ban.

## Oude tool — config geharvest (2026-06-30)
Oude pw-server = **`srvv-pw001`, 10.20.100.1** (VLAN20 legacy-DMZ, Ubuntu 20.04, **bare-metal** LTB-package op `/usr/share/self-service-password/conf/config.inc.php`, geen container). Tijdelijk `ansible`-account aangemaakt (NOPASSWD) voor read-only harvest → **op decommissie-opruimlijst zetten**. Login werkt via `~/.ssh/ansible_olvp`.
- **Bevestigd**: `ldaps://srvv-infra002.olvp.int` (LDAPS+FQDN), base `dc=olvp,dc=int`, login-attr `sAMAccountName`, `$ad_mode=true`+`change_expired_password`, `$hash=clear`.
- **2 verbeteringen overgenomen in runbook**: (1) betere filter `(&(objectClass=user)(sAMAccountName={login})(!(userAccountControl:1.2.840.113556.1.4.803:=2)))` — sluit disabled accounts uit; (2) **UPN-bind** `svc-sspr@olvp.int` i.p.v. volledige DN (overleeft OU-verhuizing). Oude bind = `PWM-proxyuser@olvp.int`.
- **NIET overgenomen** (juist wat we vervangen): oud `$pwd_min_length=10`+complexity-regels+`use_pwnedpasswords=false`; `$debug=true`; `$use_recaptcha=true` met hardcoded Google-keys cleartext (droppen — gotcha 9).
- **SMTP-beslissing (user 2026-06-30)**: **Gmail-relay NU** (`smtp.gmail.com:587` TLS, `ict@olvp.be` — hergebruikt uit oude tool, `sspr_smtp_pass`/Podman-secret), **mét geplande toekomstige migratie naar niet-Google/EU-SMTP** (tech-debt). Oude box had `$use_tokens=false` (mailflow nooit af) → nieuwe flow wél `use_tokens=true` + echte `mail_from`. Runbook Fase 6 bijgewerkt.

## Status / volgende stap
- **▶ BEZIG (2026-06-30): bouw SRVV-SSPR-01.** Fase 0 (clone) klaar, oude config geharvest, runbook bijgewerkt, **inventory-groep `sspr_servers` toegevoegd** (top-level, eigen groep, ssh_common_args="").
- **Fase 0-bevinding**: de clone is een **kále Debian 13-base** — géén Podman/step/`/etc/containers/`/log-hygiene (dus NIET van een volledig golden-image). Geen probleem: **Fase 1 `tier1-baseline.yml` installeert dat allemaal** (standalone playbook, idempotent). Runbook Fase 0/1 aangepast aan deze realiteit + step-ca bootstrap-noot.
- **✅ Fase 1 GROEN (2026-06-30)**: baseline gedraaid — Podman 5.4.2, step CLI 0.30.6, step-ca root bootstrapped, log-hygiene (rsyslog-drop + logrotate), Quadlet-dir, Cockpit active.
- **✅ LDAPS empirisch bewezen vanaf de VM (2026-06-30)**: DNS resolvet `srvv-infra002.olvp.int` → **10.10.0.10 + 10.10.0.11** (round-robin, beide :636 open, zelfde FQDN-cert → DC-failover); firewall VLAN36→636 **stond al open**; cert-SAN = enkel `DNS:SRVV-INFRA002.olvp.int` (FQDN-bind verplicht); issuer = `olvp-SRVV-INFRA002-CA`.
- **⚠️ Truststore-correctie (runbook bijgewerkt)**: DC stuurt enkel de **leaf** (geen keten) → `openssl x509` pakt niet de CA. Voor `TLS_CACERT` de **CA-cert** apart halen: (a) `certutil -ca.cert` op DC, of (b) self-bootstrap via `ldapsearch` op `cACertificate` met `LDAPTLS_REQCERT=never` (svc-sspr-creds).
- **▶ BEZIG = Fase 2 (AD-beheer / user, op SERV-01)**: PowerShell-script geleverd (2026-06-30) → `New-ADUser svc-sspr` (UPN, PasswordNeverExpires + CannotChangePassword + AccountNotDelegated) + `dsacls` reset-delegatie per OU (`/I:S … ;user`). User vult de OU-DN's in (Personeel/SO/ServiceAccounts) via `Get-ADOrganizationalUnit`. Daarna: pw → KeePassXC (later vault `sspr_ldap_bindpw`) + `certutil -ca.cert` voor truststore.
- **⚠️ BLOKKEERT Fase 3**: firewall VLAN36→10.33.0.10:636 openen (nu dicht). Dan Fase 3 (container/config intern testen).
- Te schrijven tijdens bouw: overlay-playbook `sspr.yml` + templates (`sspr.Caddyfile.j2`, config-template) — codificatie analoog forgejo.yml (eerst manueel/SSH bewijzen, dan codificeren).
- Open beleidsvragen (schoolleiding): min-lengtes bevestigen, recovery-kanaal, MFA-timing.

## Gerelateerd
- [[project-identity-architecture]] (AD SoT + LDAPS), [[project-forgejo-status]] (service-deploy-patroon), [[project-firewall-strategy]] (VLAN36→10:636-regel), [[feedback-test-from-user-vlan]] (VLAN10-verify).
