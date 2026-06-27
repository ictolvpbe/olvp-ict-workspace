---
name: project-ad-member-delegation
description: "AD-delegatie: schrijfrecht op het member-attribuut van group-objecten voor het MySchool-Odoo LDAP-bind-account, zodat de Odoo-sync groepslidmaatschap kan schrijven. Fix voor INSUFF_ACCESS_RIGHTS (50/4003). TEST + PROD gedaan 2026-06-17."
metadata:
  type: project
---

**Symptoom**: Odoo-sync kon users wel aanmaken maar GEEN groepslidmaatschap schrijven. `add_group_member` (`conn.modify(group_dn, {'member':[(MODIFY_ADD,[member_dn])]})` in `myschool_core/models/ldap_service.py` ~regel 1499) faalde met `LDAPInsufficientAccessRightsResult (50) — 00002098: SecErr: DSID-03151648, problem 4003 (INSUFF_ACCESS_RIGHTS)`.

**Oorzaak**: bind-account gedelegeerd op de user-OU's (create/modify users), maar NIET op group-objecten. Schrijven naar `member` op een group vereist een ACE op het group-object zelf.

> ⚠️ **NetBIOS-prefix (let op!):** test-domein `olvp.test` = **`OLVPTEST`**, prod `olvp.int` = **`OLVP`**. Trustee in dsacls dus `OLVPTEST\svc-odoo-ldap` (test) vs `OLVP\myschool` (prod). De eerste documentversie gebruikte abusievelijk `OLVP\` voor test — gecorrigeerd 2026-06-17. Verifieer met `Get-ADDomain` (NetBIOSName). Check of de eerder gedraaide member-grant op test een resolvende ACE opleverde (`dsacls "DC=olvp,DC=test" | findstr svc-odoo-ldap` — geen unresolved SID); zo niet, herdraai met `OLVPTEST\`.

**Recept** (`dsacls` op de Windows-DC, als Domain Admin; `/I:S` = inheritance naar descendant group-objecten; `RPWP;member;group` = Read+Write enkel op `member` van group-objecten):

- **TEST — olvp.test** (DC SRVV-TST-WIN-01 / 10.200.0.10, base DN `DC=olvp,DC=test`, bind `svc-odoo-ldap@olvp.test`):
  ```
  dsacls "DC=olvp,DC=test" /I:S /G "OLVPTEST\svc-odoo-ldap:RPWP;member;group"
  ```
- **PROD — olvp.int** (DC SRVV-INFRA002.olvp.int, base DN `DC=olvp,DC=int`, bind `myschool@olvp.int`):
  ```
  dsacls "DC=olvp,DC=int" /I:S /G "OLVP\myschool:RPWP;member;group"
  ```

**Status**:
- ✅ **TEST: gedaan + gevalideerd 2026-06-17** — grant gedraaid op SRVV-TST-WIN-01, geen foutmeldingen; `INSUFF_ACCESS_RIGHTS` weg, Odoo voegt nu succesvol leden toe aan AD-groepen.
- ✅ **PROD: gedaan 2026-06-17** — `dsacls "DC=olvp,DC=int" /I:S /G "OLVP\myschool:RPWP;member;group"` uitgevoerd op de prod-DC. Delegatie volledig uitgerold (test + prod).

**Caveats** (volledig in runbook): AdminSDHolder/SDProp zet ACL op protected groups elke ~60 min terug → sync nooit naar protected groups richten; domein-brede impact → bind-bron-restrictie (firewall/userWorkstations) + monitoring events 4728/4732 (en 4729/4733); schrijf naar `member` (forward link op group), NIET naar `memberOf` (read-only backlink); `dsacls` werkt want Windows-DC, fallback `samba-tool dsacl set` bij Samba-DC.

**Precedent**: zelfde delegatie-patroon als de Keycloak `dsacls`-attribuut-grants (`RP`/`RPCA` op `employeeID`, per domein olvp.test + olvp.int) — zie [[project-identity-architecture]] (update 2026-06-09). Daar lezend (`RP`), hier schrijvend (`RPWP`).

**Runbook**: `platform-handbook/hosting/operations/delegate-ad-group-member-write.md` (RB-2026-AD-GRPMEMBER) — symptoom/oorzaak, exact commando (test ✓ / prod ✓), verificatie (`dsacls ... | findstr` + wegwerp-testgroep via `ldapmodify`/`Add-ADGroupMember`), caveats.

---

## Tweede delegatie: user-object CREATE + MANAGE lifecycle (2026-06-17)

**Symptoom**: sync kan groepslidmaatschap schrijven (delegatie hierboven) maar kan GEEN user-objecten AANMAKEN. `create_user_at_dn` (`conn.add(dn,['user','person','organizationalPerson','top'],attrs)` in `ldap_service.py:624`) faalt met `LDAPInsufficientAccessRightsResult (50) — 00000005: SecErr: DSID-03152FCC, problem 4003 (INSUFF_ACCESS_RIGHTS) — addResponse`. Doel-DN bv. `CN=esther.decauwer,OU=pers,OU=so,DC=olvp,DC=test`.

**Oorzaak**: bind-account mist "Create Child: user"-recht op de OU-boom. Bij bawa viel het niet op want die users bestonden al → `create_user_at_dn` doet idempotente pre-check (`_find_user_dn`) en logt "already exists — skipping create" zonder `conn.add`. Domein-breed probleem, NIET SO-specifiek.

**Lifecycle-ops in code (bron-geverifieerd)**: create (`conn.add` user) · attr-write (`update_user`/`_build_user_changes`: displayName/givenName/sn/mail/employeeID) · password (`_set_user_password`: `unicodePwd` REPLACE, UTF-16LE, LDAPS verplicht) · enable/disable (`_enable_user_account`/`deactivate_user`: `userAccountControl` REPLACE) · move (`deactivate_user` → `conn.modify_dn` naar disabled-container) · delete (`delete_user` → `conn.delete`) · OU-create (`_ensure_ou_path`, alleen verse test-env).

**Recept** (`dsacls`, domein-breed, `/I:S` inheritance naar `user`-objecten; gekozen: ruime `WP;;user` i.p.v. attribuut-lijst voor onderhoudbaarheid):
- **TEST — olvp.test** (`svc-odoo-ldap@olvp.test`):
  ```
  dsacls "DC=olvp,DC=test" /I:S /G "OLVPTEST\svc-odoo-ldap:CC;user"
  dsacls "DC=olvp,DC=test" /I:S /G "OLVPTEST\svc-odoo-ldap:DC;user"
  dsacls "DC=olvp,DC=test" /I:S /G "OLVPTEST\svc-odoo-ldap:WP;;user"
  dsacls "DC=olvp,DC=test" /I:S /G "OLVPTEST\svc-odoo-ldap:CA;Reset Password;user"
  # optioneel verse test-env: /G "OLVPTEST\svc-odoo-ldap:CC;organizationalUnit"
  ```
- **PROD — olvp.int** (`myschool@olvp.int`, pas ná test): zelfde 4 regels met `DC=olvp,DC=int` + `OLVP\myschool`.

Reset Password = extended right GUID `00299570-246d-11d0-a768-00aa006e0529` (dsacls accepteert friendly name "Reset Password"). `CC`=Create Child, `DC`=Delete Child, `WP;;user`=Write Property alle attrs (dekt userAccountControl + unicodePwd-write); move/rename komt mee via CC+DC op descendant-OU's.

**Status**:
- ✅ **TEST: gedaan 2026-06-17** — 4 grants met `OLVPTEST\svc-odoo-ldap`, geen errors.
- ✅ **PROD: gedaan 2026-06-17** — 4 grants met `OLVP\myschool` op SRVV-INFRA002, geen errors. Beide delegaties (member-write + user-lifecycle) volledig uitgerold test+prod.

**Caveats**: KRACHTIG account (kan elke user aanmaken/wijzigen/verwijderen/wachtwoord resetten = account-overname mogelijk) → bind-bron-restrictie + monitor events 4720/4722/4725/4726/4738/4724; AdminSDHolder/SDProp → protected users niet via sync; per-`OU=pers`-scope ware veiliger maar gebruiker koos domein-breed (consistent + niet SO-specifiek), genoteerd als toekomstige hardening; `WP;;user` ruimer dan attribuut-lijst (bewuste keuze, onderhoudbaarheid); LDAPS verplicht voor unicodePwd.

**Runbook**: `platform-handbook/hosting/operations/delegate-ad-user-create-manage.md` (RB-2026-AD-USERLIFECYCLE) — sibling van RB-2026-AD-GRPMEMBER, kruislings gelinkt.
