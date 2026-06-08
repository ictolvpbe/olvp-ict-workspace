---
name: project-odoo-image-ldap3
description: "ldap3 in de odoo-image zit er bewust voor het myschool-accountbeheer-project. Image/pip-deps niet unilateraal wijzigen (parked). auth_ldap heeft python-ldap nodig — coördinatiepunt, geen solo image-edit."
metadata:
  type: feedback
---

`ldap3` is een **bewuste** dependency in de gedeelde odoo-image (`localhost/odoo-olvp:19.0`, Containerfile in [[project-template-strategy]] / `roles/odoo-podman/templates/Containerfile.j2`) — het wordt gebruikt door het **myschool-accountbeheer-project**. Beslist door user 2026-06-03.

**Why:** De image is gedeeld over alle Odoo-instances (app + account-admin). Een wijziging aan de pip-deps raakt meerdere tracks. De image en z'n deps zijn daarom **geparkeerd** — niet op eigen houtje aanpassen.

**How to apply:**
- `ldap3` NOOIT verwijderen of vervangen omdat een runbook iets anders "nodig" lijkt te hebben.
- Odoo's `auth_ldap` (account-admin LDAP-login) importeert `ldap` = **python-ldap**, een ánder pakket. Dat zit normaal al in de odoo:19.0 image (core-dependency). Eerst verifiëren (`podman exec ... python3 -c "import ldap"`), niet aannemen.
- Ontbreekt python-ldap toch → **coördinatiepunt** met de myschool-track, geen solo image-edit/force-rebuild. Zie runbook `hosting/operations/config-account-admin.md` stap 1.2.

Gerelateerd: [[project-containerization]], [[project-template-strategy]], [[feedback-odoo19-podman-uid]].
