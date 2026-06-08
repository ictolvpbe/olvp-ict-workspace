---
name: feedback-odoo19-podman-uid
description: "odoo:19.0 container-user = uid 100 / gid 101 (niet 101:101 zoals oudere images). Bind-mount odooN-data + odoo.conf moeten naar uid 100 gechowned worden, anders filestore/config PermissionError."
metadata:
  type: feedback
---

De officiële **`odoo:19.0`** image draait zijn proces als **uid 100, gid 101** (`podman exec <c> id` → `uid=100(odoo) gid=101(odoo)`). Oudere odoo-images gebruikten 101:101 — die aanname is fout voor 19.0.

**Why:** Onder rootful Podman (= directe uid-mapping host↔container) moeten de bind-mounts die Odoo *beschrijft/leest* eigendom zijn van uid 100. Twee concrete symptomen toen `odooN-data`/`odoo.conf` op 101:101 stonden (role `odoo-podman`, ACC-01, 2026-06-03):
- `odoo.conf` als `101:101 0640` → odoo (uid 100, wél in gid 101) kon 'm via group-read net lezen → leek te werken.
- `odooN-data` als `101:101 0755` → uid 100 valt onder *group* (r-x) → **geen write** → `PermissionError: '/var/lib/odoo/filestore'` bij DB-init / runtime.
- Verkeerde/lege config gaf eerder `configparser.NoSectionError: No section: 'options'`.

**How to apply:**
- Chown bind-mounts voor de container-user naar het **echte** uid: `podman exec <c> id` om te verifiëren, niet aannemen.
- In role `odoo-podman` vastgelegd als vars `odoo_uid: 100` / `odoo_gid: 101` (defaults/main.yml); `odooN-data` + `config/odoo.conf` chownen daarheen.
- postgres-data niet zelf chownen — de postgres-entrypoint doet dat (uid 999).

Gerelateerd: [[project-containerization]], [[project-template-strategy]], [[project-hosting-fase1-status]].
