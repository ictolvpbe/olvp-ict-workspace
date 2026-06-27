---
name: project-login-branding
description: "OLVP-branding (instelbare thema-kleuren + logo + favicon + petrol loginscherm) zit in de intranet_theme Odoo-module (intranet-repo, branch dev), NIET in een Ansible-role. De eerder gebouwde role-module olvp_login_branding is teruggedraaid op verzoek van de user (minder provisioning-werk)."
metadata:
  type: project
---

**Vraag (2026-06-27):** schoollogo + loginscherm in merkkleur op de intranet-instances, plus instelbare thema-kleuren (op dev waren er géén theme-kleuren instelbaar).

**AANPAK-PIVOT.** Eerst gebouwd als role-meegeleverde module `olvp_login_branding` (platform-ansible). User koos daarna: **alles in `intranet_theme`** (Odoo-module), role-aanpak schrappen ("minder scripting/provisioning-werk"). De 4 platform-ansible-commits + handbook-runbook + memory zijn teruggedraaid (waren unpushed). **Geen `olvp_login_branding`-role meer — niet opnieuw toevoegen.**

**Waar het nu zit:** `/home/demm/SchoolProjects/intranet/extra-addons/intranet_theme` (intranet-repo, branch `dev`, commit `81a0501`). Achtergrond: `intranet_theme_olvp` is `installable:False` (calm-overlay gedropt 2026-06-26); de petrol zit in de **basis `intranet_theme`**, die al de volledige kleur-machinerie + `ms_color_*`-velden had maar **geen settings-UI** → dáárom waren er geen kleuren instelbaar.

**Wat toegevoegd aan intranet_theme:**
- `views/res_config_settings_views.xml` — "OLVP-thema"-tab op `base_setup.res_config_settings_view_form` (werkt ook op de lean app-server). Curated set: logo, favicon, apps-achtergrond + kleuren menu/navbar (`ms_color_brand_1`), zijmenu (`ms_color_sidebar_bg`), achtergrond (`ms_color_bg`), accent (`ms_color_brand_3`) + reset/regen-knoppen. Kleuren-opslag = bestaande live-CSS-editor (`set_values` → `/_custom/`-attachment over layout.scss).
- Logo = `res.company.logo`; favicon = eigen `res.company.theme_favicon` (want `res.company.favicon` zit enkel in de **website**-module, niet op de lean server). `controllers/main.py` serveert `/intranet_theme/favicon` (auth=public, statische fallback); `views/branding_templates.xml` wijst de favicon-link daarheen (backend + login) + zet login-tabtitel "OLVP — ieder1 telt".
- `static/src/scss/login.scss` in `web.assets_frontend` → petrol loginscherm (#007E8C, hover #006A77; witte kaart; knop/links/focus petrol). Login deelt de backend-`--myschool-*`-vars NIET → kleuren hard-coded.
- Defaults ingebakken: `post_init_hook` + migratie `19.0.3.0.0/post-migrate.py` zetten logo/favicon/achtergrond op bedrijven die ze nog niet hebben (overschrijven niets). post_init draait enkel bij install → migratie dekt bestaande DB's bij `-u`.

**Uitrol = normale intranet-deploy** (geen Ansible-role): push branch `dev` → `addons-update` doet `git pull` + `-u intranet_theme` op intranet-dev; test/acc/prod volgen via hun branches (acc-test/test = `test`, acc/prod = `prod`). Verifieer: Instellingen → OLVP-thema-tab; petrol `/web/login`; favicon in tab. Status: **code klaar + gevalideerd (py/xml/manifest/migratie), nog niet gedeployed.**

**Merkkleur:** petrol **#007E8C** (logo-bubble). Logo-bron: `/home/demm/Downloads/OLVP-logo_Transparant.png` (274×240, transparant); favicon = vierkant uitgesneden bubble.

Verwant: [[project-intranet-hosting]], [[feedback-test-from-user-vlan]] (verifieer publiek, niet bare-IP).
