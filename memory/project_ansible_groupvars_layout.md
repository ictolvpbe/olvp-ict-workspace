---
name: project-ansible-groupvars-layout
description: "platform-ansible group_vars gebruikt directory-stijl: group_vars/all/{vars,vault}.yml worden auto-geladen voor de 'all'-groep. Nooit expliciet als vars_files opnemen."
metadata:
  type: project
---

Sinds ~2026-06-02 gebruikt `platform-ansible` de **directory-stijl** group_vars: `group_vars/all/vars.yml` (plain) + `group_vars/all/vault.yml` (ansible-vault encrypted, bevat o.a. `vault_keepalived_auth_pass`, `vault_odoo_db_password`, `vault_odoo_admin_passwd`, `forgejo_db_password`). Vervangt de oude losse bestanden `group_vars/all.yml` + `group_vars/all_vault.yml`.

**Why:** Ansible laadt `group_vars/all/*` **automatisch** voor elke play die hosts in de `all`-groep target (= alle plays, want elke host zit onder `all`). De migratie brak stilletjes 3 playbooks (`site.yml`, `forgejo.yml`, `odoo-podman.yml`) die nog `vars_files: group_vars/all_vault.yml` hadden → `ansible-vault edit`/run faalt met *"Bestand of map bestaat niet"*. Gefixt 2026-06-03.

**How to apply:**
- Vault bewerken: `ansible-vault edit group_vars/all/vault.yml` (NIET `all_vault.yml`).
- In playbooks: **geen** expliciete `vars_files`-entry voor de vault — die laadt automatisch. Wel nog `vars_files: vars/instances.yml` (dat is een custom pad, geen auto-load-locatie).
- Vault-password bij run: `--ask-vault-pass` lokaal, of Semaphore Key Store op de runner. Zie [[feedback-semaphore-key-canonical]].

Gerelateerd: [[project-hosting-fase1-status]], role `odoo-podman` ([[project-infrastructure-params]]).
