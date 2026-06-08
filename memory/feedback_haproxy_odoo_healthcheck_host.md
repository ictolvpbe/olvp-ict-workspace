---
name: feedback-haproxy-odoo-healthcheck-host
description: HAProxy health-check naar Odoo MOET een Host-header sturen — anders geeft Odoo (multi-database setup) HTTP 500 en blijft de backend DOWN. Trigger: nieuwe HAProxy-backend voor Odoo definiëren.
metadata:
  type: feedback
---

**Regel:** elke HAProxy-backend die naar Odoo wijst, krijgt een health-check met **expliciete Host-header**:
```haproxy
option httpchk
http-check send meth GET uri /web/health hdr Host <fqdn>
http-check expect status 200,301,302,303
```
**Niet doen** (oude `option httpchk GET /web/health` zonder send/hdr) = HAProxy stuurt request zonder Host → Odoo's multi-database router weet niet welke DB → HTTP 500 → backend DOWN → 503 naar client.

**Why:** 2026-06-01 — `be_myschool_dev2_olvp_be` bleef DOWN met `L7STS 500`. Verifieerde direct op TST-ODOO-02:
- `GET /web/health` ZONDER Host → `500 INTERNAL SERVER ERROR`
- `GET /web/health` MET `Host: myschool-dev2.olvp.be` → `200 {"status": "pass"}`

Odoo's WSGI-router heeft de Host nodig om de juiste database (`myschool_dev2`) te selecteren; zonder Host crasht het in een DB-lookup-fout. `/web/health` is geen "echte" health-endpoint maar gewoon een Odoo-controller die mee in de multi-DB-flow valt.

**How to apply:**
- In `platform-ansible/templates/haproxy.cfg.j2` heeft de `direct_backend`-pad deze syntax al; **niet** terugdraaien naar legacy `option httpchk GET ...`.
- Voor Caddy-backends (TLS re-encrypt) is dit minder kritiek omdat Caddy zelf SNI/Host uit het TLS-pad afleidt, maar verifieer met `curl --resolve` of de check 200 geeft.
- Alternatief health-endpoint indien `/web/health` problemen geeft: `/web/database/manager` of `/` (verwacht status 303). Pas dan ook `http-check expect status` aan.
- Bij toekomstige Odoo-instances: gebruik dezelfde template-loop in `haproxy.cfg.j2`. Het `direct_backend`-pad genereert automatisch `hdr Host {{ inst.fqdn }}` per instance.

Gerelateerd: [[project-architecture-haproxy]] (HAProxy-stack), [[project-containerization]] (Odoo containers per VM).
