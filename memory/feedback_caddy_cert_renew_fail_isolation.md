---
name: feedback-caddy-cert-renew-fail-isolation
description: "step-ca cert-renewal op de Caddy-VMs brak volledig zodra ÉÉN cert verlopen was: de renew-loop had set -euo pipefail en 'step ca renew' faalt op een verlopen cert (kan zichzelf niet meer autoriseren) → loop abort → álle certs stoppen met renewen. Gevolg: stille cascade-expiry → HAProxy backend DOWN → publiek 503. Wees-certs van geteardownde FQDN's zijn de trigger."
metadata:
  type: feedback
---

**Incident 2026-06-27.** `myschool-test.olvp.be` bleek **~16 dagen publiek 503** (Odoo zelf gezond, lokaal 200). 

**Keten:**
1. Na teardown/verhuizing van FQDN's (id-test, myschool-dev) bleven hun **24u step-ca-certs achter** in `/etc/caddy/*.crt` op tst-odoo-01 → verliepen 07-06.
2. `renew-caddy-cert.sh` loopt over álle `/etc/caddy/*.crt` met **`set -euo pipefail`**. `step ca renew` op een **verlopen** cert faalt ("certificate expired ... lacked necessary authorization") want [[feedback-step-ca-renew-no-password]]: het bestaande cert IS de auth — verlopen = geen auth meer.
3. `set -e` → de eerste fail **aborteert de hele loop** → de nog-geldige certs (myschool-test) worden nooit bereikt → verliepen op hun beurt (11-06).
4. Caddy serveert het verlopen cert nog (curl -k = 200), maar **HAProxy `verify required` weigert het** → backend DOWN → **503**. `step ca renew` (timer) kan een verlopen cert niet redden → vaststaande outage.

**Fix (structureel, commit `d18ec3d` in platform-ansible):** `templates/caddy-cert-renew.sh.j2` → per-cert **fail-isolatie**: `if ! step ca renew ...; then logger WARN; fi` (set -e wordt in een if-conditie onderdrukt) → één slecht cert blokkeert de rest niet meer. Uitrollen via `caddy.yml --limit webapps`.

**Herstel verlopen cert (renew kan NIET, want verlopen = geen auth):** wees-certs verwijderen + **vers** uitgeven met provisioner-pw: `step ca certificate <fqdn> ... --provisioner admin --not-after 24h --force` → naar `/etc/caddy/` → `systemctl restart caddy`. Daarna HAProxy backend weer UP.

**How to apply / preventie:**
- **Bij élke FQDN-teardown/verhuizing: ook de Caddy-cert-files (`<fqdn>.crt/.key`) verwijderen** (niet enkel het Caddyfile-blok + caddy.container Volume). De role-Caddy (`caddy.yml`) re-rendert het blok + Volume uit instances.yml, maar laat de losse cert-files staan → wezen.
- **Monitoring-gat**: lokale Odoo-health (200) verbergt dit; enkel publieke E2E + HAProxy-backendstatus toont het. Test publiek, niet enkel lokaal ([[feedback-test-from-user-vlan]]).
- Verwant: [[feedback-step-ca-renew-no-password]] (renewal handenvrij ZOLANG geldig), [[feedback-caddy-default-sni-for-haproxy-check]], [[feedback-haproxy-odoo-healthcheck-host]], [[project-intranet-hosting]] (incident dook op tijdens de cleanup).
