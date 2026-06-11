---
name: project-acc-ldap-config
description: acc-test (myschool-acc-test.olvp.int, SRVV-ACC-01 odoo_instance_2) auth_ldap-config compleet + werkend 2026-06-11. Zentyal/Samba test-AD onderhoudt memberOf NIET → OU-gebaseerde filter i.p.v. groep. Prod-AD olvp.int is echte Windows-AD (memberOf kan daar wél).
metadata:
  type: project
---

**acc-test auth_ldap volledig geconfigureerd + geverifieerd (2026-06-11).** DB `myschool_acc_test`, container `odoo_instance_2` op SRVV-ACC-01 (10.36.0.42). Runbook: [[RB-2026-ACC-CONFIG]] (`config-account-admin.md`).

**Werkende config** (`res_company_ldap`):
- server `srvv-tst-win-01.olvp.test:389`, TLS=true (StartTLS) — bind service-account `CN=svc-odoo-ldap,OU=Managed Service Accounts,DC=olvp,DC=test` geverifieerd OK.
- base `OU=ict,OU=od,DC=olvp,DC=test`, filter `(&(objectClass=user)(sAMAccountName=%s))` → exact de 5 ICT-team (mark.demeyer, jana.vandender, lotte.soens, kim.geleyn, nele.dewit).
- create_user=true, template-user `__ldap_template_ict__` (inactief, **base.group_system** = Administrator).

**⚠️ KERN-GOTCHA — Zentyal/Samba test-AD onderhoudt `memberOf` NIET.** `olvp.test` (DC `srvv-tst-win-01`) is Zentyal/Samba, geen echte Windows-AD. Custom-groep-leden (`grp-ict-od` distributie / `bgrp-ict-od` security) hebben een **lege `memberOf`-backlink** → een `(memberOf=CN=...)`-login-filter matcht **0 users** (ook `LDAP_MATCHING_RULE_IN_CHAIN` = 0). Lidmaatschap leeft enkel op het groep-`member`-attribuut, niet per-user bevraagbaar. **Daarom OU-filter i.p.v. groep.** (martine.verbeke, 6e groeplid in `OU=adm,OU=pers,OU=bawa`, valt buiten de OU → bewust uitgesloten.)

**➡️ PROD verschilt**: `myschool-acc.olvp.int` (DB `myschool_acc`, odoo_instance_1) bindt tegen de **échte Windows-AD `olvp.int`** (DC `SRVV-INFRA002`/`SERV-01` 10.33.0.10) waar `memberOf` normaal **wél** werkt → daar mag de oorspronkelijke **groep-filter** (security-groep) gebruikt worden i.p.v. de OU-workaround. **Verifieer `memberOf` op prod eerst** (bind+search-test, zelfde patroon).

**Odoo-19-detail**: het groepen-veld op `res.users` heet **`group_ids`** (niet meer `groups_id`); `all_group_ids` = computed incl. implied.

**Verificatie-techniek (herbruikbaar)**: bind+search-test via `podman exec -i odoo_instance_2 python3` met de creds uit `res_company_ldap` via stdin (pw blijft op de VM, nooit in agent-context). De container heeft `LDAPTLS_CACERT=/etc/ssl/certs/ca-certificates.crt` → StartTLS-CA-trust werkt.

**Login-test ✅ (2026-06-11)**: user logde in met een ICT-AD-account, JIT-`res.users` aangemaakt met admin-rechten.

**MySchool-apps geïnstalleerd op acc-test (2026-06-11)**: `myschool_admin` + `myschool_sync` + `myschool_servermanager` (+ deps `myschool_core`, `myschool_theme`) op branch **`test`** (instances.yml acc-test `addons_branch: Dev`→`test`; addons-update gepulld in `addons2`). 47 modules installed. **muk_web_\*** zijn gitignored/extern → door addons-update gearchiveerd, géén van de gekozen modules hangt ervan af. **Demo-data**: de install bracht demo-users mee (`medewerker@test.com`, `directie@test.com`, `boekhouding@test.com`, `vervangingen@test.com`, `beheerder@test.com`) — OK op test; **bij PROD `-i ... --without-demo=all`** gebruiken.

**Nog te doen**: (a) 2FA (stap 2 runbook); (b) audit-logging (stap 3); (c) herhalen op **prod** (olvp.int, memberOf-groep-filter mogelijk + `--without-demo`). Gerelateerd: [[project-hosting-fase1-status]], [[project-ad-serv01-migration]], [[project-identity-architecture]], [[project-odoo-addons-repo]].
