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
`DNS pw.olvp.be → .82 (oude gw DNAT) → HAProxy (LE) → Caddy (step-ca) op SRVV-SSPR-01 → LTB-container (127.0.0.1:8084) → AD LDAPS srvv-infra002.olvp.int:636`.
- **Service-patroon = Forgejo** (los Quadlet-service + eigen Caddy + overlay-playbook `sspr.yml` nog te schrijven), NIET het Odoo-instances-patroon.
- **HAProxy = `be_keycloak`-patroon**: handmatig toegevoegde niet-Odoo publieke backend (ACL `is_sspr` + `be_sspr` server :443 ssl verify required sni req.hdr(host)). Niet instances.yml-driven.
- AD: dedicated least-privilege bind-account `svc-sspr` + `dsacls` reset-delegatie (`CA;Reset Password;user` + `WP;pwdLastSet;user`) op personeel+SO-OU's (analoog [[project-ad-member-delegation]]).

## Belangrijkste gotchas (in runbook)
- LDAPS op **FQDN** (cert-SAN-match, [[feedback-keycloak-ldaps-truststore]]); AD-CA-cert in container-truststore.
- AD eist LDAPS voor pw-set; firewall VLAN36→VLAN10:636 nieuw nodig.
- DNS `.82` niet `.84` ([[project-gateway-cutover]]); `pw.olvp.be` bestaat al → cutover = laatste stap, oude tool tot dan laten draaien.
- `default_sni pw.olvp.be` in Caddy ([[feedback-caddy-default-sni-for-haproxy-check]]); LE eerste `certonly` → deploy-hook handmatig.
- AD verhuist later naar SRVV-SERV-01 ([[project-ad-serv01-migration]]) → daarom op FQDN binden.
- Eén `$pwd_min_length` per instance → vloer 12, personeel via policy-tekst naar ≥14.
- Geen reCAPTCHA (Google-dep) → rate-limiting via HAProxy/fail2ban.

## Status / volgende stap
- **WACHT OP**: user cloont SRVV-SSPR-01 (Fase 0). Daarna Fase 1→8 met Claude-begeleiding.
- Te schrijven tijdens bouw: overlay-playbook `sspr.yml` + templates (`sspr.Caddyfile.j2`, config-template) — codificatie analoog forgejo.yml (eerst manueel/SSH bewijzen, dan codificeren).
- Open beleidsvragen (schoolleiding): min-lengtes bevestigen, recovery-kanaal, MFA-timing.

## Gerelateerd
- [[project-identity-architecture]] (AD SoT + LDAPS), [[project-forgejo-status]] (service-deploy-patroon), [[project-firewall-strategy]] (VLAN36→10:636-regel), [[feedback-test-from-user-vlan]] (VLAN10-verify).
