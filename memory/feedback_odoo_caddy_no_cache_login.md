---
name: feedback-odoo-caddy-no-cache-login
description: Caddy moet no-cache headers sturen op /web/login* en auth-paths van Odoo. Anders krijgen users intermittent CSRF-fouten + trage UI bij F5 op /web/login na lange wait. Geconstateerd 2026-06-02 op myschool-test.
metadata:
  type: feedback
---

**Regel**: In de Caddyfile (template `platform-ansible/templates/caddy.Caddyfile.j2`) elke Odoo site-block voorzien van:

```caddy
@auth path /web/login /web/login/* /web/database/* /web/session/* /auth_oauth/*
header @auth {
    Cache-Control "no-store, no-cache, must-revalidate, max-age=0"
    Pragma        "no-cache"
    Expires       "0"
}
```

**Why**: User-klacht 2026-06-02 — myschool-test login soms traag + soms CSRF-error. Diagnose-pad (klok-sync OK, proxy_mode OK, geen workers, geen LDAP-call) liet stale-cache + stale-session over. Repro-pattern: open /web/login, wacht ~15min (Odoo session-GC ruimt sessie), F5 → browser POST oude CSRF-token tegen nieuwe session → 403/CSRF. Mitigation gedeployd via commit 49ca110 — direct effect: snellere login-UI + geen CSRF meer.

**How to apply**:
- Bij elke nieuwe Odoo-instance via `caddy.yml`-playbook → al automatisch correct via template
- Bij Caddy-config buiten Ansible-pad (manueel ooit?) → vergeet deze block niet
- Bij andere reverse-proxy keuze in toekomst (Traefik, Nginx) → equivalent maken
- Test na deploy: `curl -sI https://<fqdn>/web/login | grep -i cache-control` → moet `no-store...` tonen

**Niet de oorzaak gebleken** (uitsluiten in toekomstige debug-sessies):
- Klok-drift (alle 4 VMs binnen 600ms NTP-synced)
- `proxy_mode` False (was True)
- Multi-worker session-split (workers default = threaded, single proc)
- HAProxy VRRP-flap (geen events in journalctl)
- Caddy reload tijdens dag (renewal-check 3×/dag maar restart enkel bij echte renew = 1×/nacht ~02:07)
- LDAP-traagheid (interne users only voor test; LDAPS-port 636 wel `connection refused` maar parked als aparte issue)

Gerelateerd: [[project-hosting-fase1-status]] (Caddy-Quadlet pad), [[feedback-caddy-default-sni-for-haproxy-check]] (andere Caddyfile-gotcha).

**Side-observation parked**: `srvv-infra002.olvp.int:636` (LDAPS) geeft `Connection refused` vanaf Odoo-container in VL-TST-WEBAPPS. Niet relevant zolang test/dev geen AD-login gebruikt. Wel relevant bij `id-test.olvp.be` (identity-rol) en later prod-AD-integratie — apart aan te pakken werf wanneer LDAP-binding nodig wordt.
