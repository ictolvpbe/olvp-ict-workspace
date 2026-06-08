---
name: feedback-addons-branch-switch-risico
description: Een addons-branch-switch op een env-pattern Odoo-instance (zonder DB-upgrade) brak myschool-ict met 500; rollback legde bloot dat master HEAD van MySchool_addons zelf stuk is (org_students-import zonder bestand). Lessen voor addons-update op draaiende prod-instances.
metadata:
  type: feedback
---

**Incident 2026-06-05**: myschool-ict.olvp.be (odoo_instance_2 op SRVV-ODOO-01, DB **ICT-PROD**) van channel prod→dev gezet en Dev gepulld via addons-update. Restart gaf **500 op alles**: `UndefinedColumn res_users.myschool_sap_sync_always_review` — Dev-code definieert een veld waarvan de DB-kolom niet bestaat, want **env-pattern instances slaan de `odoo -u` module-upgrade over** (zie [[project-odoo-addons-repo]]). Een branch-switch = nieuwe code tegen oude schema = stuk.

**Bij de rollback naar master nóg een 500**, andere oorzaak: master HEAD (`d1217c4`) van `ictolvpbe/MySchool_addons` is **zelf stuk** — `activiteiten/models/__init__.py` doet `from . import org_students` maar `org_students.py` is nooit in master gecommit (botched merge/commit; import erin sinds `114b233`, bestand niet). Elke verse master-pull + restart → `Couldn't load module activiteiten` → `Failed to load registry` → 500. **Dit is een app-repo-bug, te fixen in de MySchool_addons-werf** (odoo-dev, `/home/demm/PyCharm/odoo-myschool/extra-addons/`): org_students.py toevoegen óf de import verwijderen, dan pushen.

**Herstel uitgevoerd**: addons2 handmatig `git reset --hard` op de laatste schone master-commit `3c9d01f "improvements devhub"` (= `114b233^`) + `clean -fdx` (stale Dev-`__pycache__` weg) + restart → health/login 200. myschool-ict SoT terug op channel:prod/master, addons2 **gepind**; NIET addons-update draaien tot master gefixt is.

**Why / How to apply:**
- Een **branch-switch op een draaiende instance** is alleen veilig als de DB mee-geüpgraded kan worden. Op env-pattern VMs kan de playbook dat niet → doe het niet op zo'n instance, of migreer eerst naar de odoo-podman-role (conf-pattern).
- **Pull nooit blind latest master/Dev op een prod-instance**: een kapotte branch-tip sloopt een draaiende instance bij restart. Test branch-tips eerst op een wegwerp-DB.
- **Diagnose-volgorde bij Odoo-500**: `journalctl -u odoo_instance_N` → kolom-fout = schema/upgrade-mismatch; ImportError/Couldn't load module = code-bug of vuile tree. Bij vuile tree na branch-switch: `git reset --hard <commit>` + `clean -fdx` (verwijdert stale `.pyc`).
- myschool.olvp.be (real prod, addons1) draait een **andere addons-set zonder activiteiten** en was niet getroffen — maar verifieer dat altijd apart.
- SRVV-ODOO-01 is env-pattern → `git`/restart kan vanaf het werkstation (VLAN 34 direct), maar dit was een **prod-VM-actie** met expliciete user-go ([[feedback-no-preview-destructive]], CLAUDE.md prod-confirmation). Zie ook [[project-hosting-fase1-status]].
