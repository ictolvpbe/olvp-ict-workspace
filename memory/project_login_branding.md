---
name: project-login-branding
description: "OLVP login-branding (schoollogo + petrol #007E8C loginscherm) als standaard provisioning op ELKE Odoo-instance, via de role-meegeleverde module olvp_login_branding (repo-agnostisch, /mnt/branding-addons). Module+role klaar 2026-06-27; uitrol DEV/ACC/TEST te doen, PROD juli."
metadata:
  type: project
---

**Vraag (2026-06-27):** op elke instance het **schoollogo** + **loginscherm in de logo-kleur** — toegevoegd aan de **standaard provisioning** (niet per app-repo).

**Oplossing — module `olvp_login_branding`, meegeleverd door de odoo-podman role** (`roles/odoo-podman/files/olvp_login_branding/`), dus repo-agnostisch (intranet én myschool):
- **Logo:** `post_init_hook` zet `res.company.logo` (alle bedrijven) uit `static/src/img/olvp_logo.png` (= `/home/demm/Downloads/OLVP-logo_Transparant.png`, 274×240, transparant). Voedt login (`/web/binary/company_logo`) + navbar + rapporten. Draait **enkel bij install**, niet bij `-u` → handmatige wijziging blijft.
- **Kleur:** `static/src/scss/login.scss` in `web.assets_frontend` → `body.bg-100` petrol **#007E8C**; login-kaart terug wit. Backend-body = `o_web_client` (níét bg-100) → backend ongemoeid. Merkkleur #007E8C = OLVP-petrol (logo-bubble; gedocumenteerd in `myschool_theme_olvp`).
- `depends:['web']`, `auto_install:True` → nieuwe DB's automatisch; bestaande via `-i olvp_login_branding`.

**Role-changes (platform-ansible):** `tasks/main.yml` sync-task → `/opt/odoo-multi-instance/branding-addons/`; `odoo_instance.container.j2` `Volume=…/branding-addons:/mnt/branding-addons:ro`; `odoo.conf.j2` `addons_path` += `/mnt/branding-addons`. **Geen image-rebuild** (gemount, niet gebakken).

**Uitrol (runbook RB-2026-LOGIN-BRANDING, `olvp-login-branding.md`):** per VM `ansible-playbook odoo-podman.yml --limit <host> --ask-vault-pass` (re-rendert conf/quadlet + restart), dan per DB `podman exec odoo_instance_N odoo -d <db> -i olvp_login_branding --stop-after-init --no-http` + restart. Verifiëren via `https://<fqdn>/web/login` (níét bare-IP). Volgorde DEV→ACC→TEST; **PROD srvv-odoo-01 juli** (VLAN36 → lokaal; herstart myschool+myschool-ict kort).

**Status 2026-06-27:** code/runbook klaar + gevalideerd (py-compile, manifest, jinja, yaml, syntax-check). **Uitrol nog niet gedraaid** (user draait ansible — vault). 8 DB's totaal: dev(2)/acc(2)/test(2) nu, prod(2) juli.

**Gotcha's:** DB-pw-drift bij role-apply (odoo.conf rendert vault-pw; moet matchen met lopend postgres-pw — zie [[project-intranet-hosting]]). SVG ongeschikt als `res.company.logo` → PNG gebruikt. Petrol-logo is knockout → daarom logo op witte kaart, niet op petrol vlak.

Verwant: [[project-intranet-hosting]], [[feedback-test-from-user-vlan]] (verifieer publiek, niet bare-IP).
