---
name: feedback-ansible-vault-loading
description: Ansible laadt `group_vars/<groupname>.yml` of `group_vars/<groupname>/` directory automatisch, MAAR niet `group_vars/<groupname>_vault.yml`. Vault-vars verschijnen dan als "undefined". Gebruik split-file pattern `group_vars/all/{vars,vault}.yml`.
metadata:
  type: feedback
---

**Regel**: Voor split-tussen-encrypted-en-plain group_vars **altijd het directory-pattern** gebruiken:

```
group_vars/
├── all/
│   ├── vars.yml      ← onversleuteld
│   └── vault.yml     ← ansible-vault encrypted
```

NIET `group_vars/all_vault.yml` (vaak gezien in oudere snippets) — Ansible interpreteert die filename als groep "all_vault", die niet bestaat → file wordt nooit geladen → vault-vars zijn "undefined" ondanks dat `--ask-vault-pass` succesvol decrypteren werkt.

**Why**: Geconstateerd 2026-06-02 bij Forgejo deploy. `forgejo_db_password` werd `is undefined` ondanks correcte vault-content en vault-pw. Diagnose via `ansible -m debug -a 'var=forgejo_db_password'` toonde het probleem direct. Restructure naar `group_vars/all/{vars,vault}.yml` loste het op.

**How to apply**:
- Bij elke nieuwe Ansible-repo: direct directory-pattern toepassen
- Bestaande `group_vars/<name>_vault.yml` migreren met `git mv` naar `group_vars/<name>/vault.yml` (en de plain `<name>.yml` naar `<name>/vars.yml`)
- Vault- en plain-files kunnen verschillende encryption-state hebben; Ansible behandelt ze hetzelfde voor variabele-resolution

Gerelateerd: [[project-secrets-management]] (KeePassXC + ansible-vault strategie).
