---
name: feedback-caddy-default-sni-for-haproxy-check
description: "Caddyfile achter HAProxy moet `default_sni <fqdn>` in global block hebben — HAProxy's `sni req.hdr(host)` levert lege SNI tijdens health-checks, Caddy weigert SNI-loze handshakes met TLS alert 80 → backend DOWN."
metadata:
  type: feedback
---

**Regel:** elke Caddyfile die als backend van HAProxy fungeert (met HAProxy's standaard `sni req.hdr(host)` op de server-line) krijgt een `default_sni <fqdn>` in het global block. Anders weigert Caddy de HAProxy health-checks met TLS alert 80.

```caddyfile
{
    admin off
    auto_https off
    default_sni myschool-dev2.olvp.be   # ← essentieel voor HAProxy health-checks
}
```

**Why:** 2026-06-01 — na een Caddy-restart (cert-renewal cron) ging `be_srvv_tst_odoo_02` plots DOWN met `L6RSP "SSL handshake failure"` in 2ms (te snel voor een netwerk-call). Diagnose:

1. HAProxy's `sni req.hdr(host)` op de server-line is een sample-expression. Voor proxied requests = de Host-header. Voor health-checks = **geen request-context** → expression returnt null → HAProxy stuurt **lege SNI** in de TLS ClientHello.
2. Caddy met static-cert sites (`<fqdn>:443 { tls cert key }`) en `auto_https off` heeft **geen default cert** voor onbekende SNI → weigert handshake met TLS alert 80 (internal_error).
3. Vóór de Caddy-restart accepteerde Caddy de handshake mogelijk doordat een eerdere config-state een default cert had; na restart strikt.

Geprobeerd-en-verworpen alternatieven:
- `check-sni str(<fqdn>)` op HAProxy server-line: in onze HAProxy 3.0.11 had dit geen effect. Mogelijk syntax-issue of 3.0-specifiek.
- `sni str(<fqdn>)` static: werkte ook niet (geen verschil), bovendien verliezen we dynamic-routing-voordeel voor multi-FQDN Caddy.
- Catch-all `:443 { }` site in Caddyfile: werkt maar minder elegant dan `default_sni`.

**Caddy `default_sni` is de canonieke fix**: 1 regel global, Caddy selecteert het genoemde cert/site bij elke handshake zonder of met onbekende SNI. Voor multi-FQDN Caddy: pak een willekeurige eigen FQDN (of de "primary" site).

**How to apply:**
- Bij elke nieuwe Caddyfile achter HAProxy (handmatig of via Ansible-role [[task #21]]): include `default_sni <fqdn>` in global.
- Bij debug van plotse `L6RSP SSL handshake failure` in HAProxy stats: eerst Caddyfile-default_sni checken vóór dieper TLS-debugging.
- HAProxy template-comment houden bij de server-line zodat de coupling tussen `sni req.hdr(host)` ↔ Caddy `default_sni` zichtbaar blijft.

Gerelateerd: [[project-architecture-haproxy]] (HAProxy + Caddy TLS-re-encrypt pad), [[feedback-haproxy-odoo-healthcheck-host]] (Host-header in HAProxy health-check, complementair issue).
