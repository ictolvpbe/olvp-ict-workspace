---
name: feedback-semaphore-survey-vs-cli-args
description: In Semaphore zijn Survey-vars en CLI Arguments aparte concepten. Survey-vars worden auto-doorgegeven als `-e KEY=VALUE`. CLI Arguments-veld verwacht JSON-array, GEEN shell-tekst met Jinja. Symptoom bij verkeerd gebruik: ansible-playbook print --help en exit 2.
metadata:
  type: feedback
---

In Semaphore (versie 3.x, draait sinds 2026-05-23 op 10.35.0.10):

**Survey Variables** worden door Semaphore **automatisch doorgegeven als `-e VAR=VALUE`** aan ansible-playbook. Je hoeft GEEN handmatige `-e ...` in CLI Arguments te zetten.

**CLI Arguments** veld accepteert **JSON-array** van extra-flags, bv. `["-vvv","--check"]`. Multi-line shell-tekst met Jinja-templating (zoals `--limit {{ target_limit }}`) wordt NIET correct gerenderd → de gecomposeerde ansible-playbook-opdracht verliest zijn positionele playbook-argument → exit 2 met volledige `--help`-output in de log.

**Why**: Geconstateerd 2026-06-02 bij eerste run van Addons Update template. Vorige iteratie van `semaphore.md` instrueerde shell-stijl CLI Args, wat resulteerde in een commando dat ansible-playbook niet kon parsen. Symptoom (--help + exit 2) is misleidend omdat het lijkt op een gewone argparse-fout maar de root cause de Semaphore-templating is.

**How to apply**:
- CLI Arguments default = **leeg laten**. Alleen JSON-array gebruiken voor statische flags zoals `["-vvv"]` of `["--check"]`.
- Voor variabele waarden: maak Survey-vars met de exacte var-naam die de playbook verwacht. Semaphore voegt `-e var=value` toe.
- Voor host-pattern (`--limit`-equivalent): stuur het in de playbook zelf via `hosts: "{{ target_limit | default('webapps') }}"`. NIET via CLI Args proberen.
- Pattern werkt voor elke nieuwe template; documenteer in `management-tools/semaphore.md`.

Voorbeeld correct opgezet — `Odoo extra-addons Update`:
- CLI Arguments: leeg
- Survey: `target_env` (default leeg), `addons_branch_override` (default leeg), `target_limit` (default `webapps`)
- Playbook: `hosts: "{{ target_limit | default('webapps') }}"`

Gerelateerd: [[feedback-semaphore-key-canonical]] (eerste run-fout uit dezelfde sessie).
