---
name: feedback-terminal-paste-long-lines
description: "Vermijd heredocs + lange echo/printf-strings bij copy-paste van runbook-commando's in een terminal — terminal-wrap voegt echte newlines in waar de bron `\\n`-escapes had. Use nano voor multi-line content en split lange strings in korte stukken."
metadata:
  type: feedback
---

**Regel:** bij elk shell-runbook dat door een mens via copy-paste in een terminal wordt uitgevoerd, vermijd:
- **Heredocs** (`<<'EOF' … EOF`) — paste-modus eet de EOF terminator op, of de terminal wraps tussenliggende regels, bash ziet onverwacht "regel 49 eindigt met einde van bestand (verwachtte 'EOF')".
- **Lange `echo`/`printf`-strings met `\n`-escapes** die langer zijn dan de terminal-width — terminal-wrap injecteert een echte newline middenin → wat als `\n`-escape was bedoeld wordt een literal newline + indent. Voorbeeld: `printf 'Defaults:ansible logfile=/var/log/sudo-ansible.log\n'` wraps in een 80-cols-terminal tot `Defaults:ansible \n  logfile=...` → sudoers parse-error.
- **Lange pubkey-`echo`-regels** (>100 chars) — wraps midden in de base64-blob → `authorized_keys` blijft leeg of corrupt.

**Use these instead:**
- **Heredoc-vrije multi-line files**: split in losse `echo … | sudo tee -a <file>` regels, één per logische lijn. Geen `\n`-escapes nodig.
- **Lange single-line content (pubkeys, certificates)**: open `sudo nano <file>` en paste in de editor. Nano accepteert wrap-vrije paste. Daarna `chown`/`chmod`/`cat <file>` om te verifiëren.
- **Sudo + commando-flags**: vermijd `sudo CMD -c -f <arg>` — sudo's eigen argument-parser kan `-c` opslokken als z'n eigen flag (`-c <count>`). Schrijf als `sudo CMD -cf <arg>` of `sudo -- CMD -c -f <arg>`. Concreet ervaren met `sudo visudo -c -f /etc/sudoers.d/ansible` → "usage: sudo -h | …".

**Why:** 2026-06-01 — bij `ansible`-user provisioning op TST-ODOO-01 (volgens provision-ansible-account.md runbook) gingen ~5 minuten verloren aan terminal-paste-issues. Heredoc gaf "regel X eindigt onverwacht", lange printf gaf sudoers-parse-error door wrap-injected newline, lange pubkey-echo liet authorized_keys leeg. Drie symptomen, één onderliggend probleem.

**How to apply:**
- Bij elke nieuwe runbook of operations-doc: **kort regels**, vermijd heredoc voor multi-line content, gebruik `nano` voor "paste-this-into-a-file" patronen.
- Bij hands-on-fix-script in een chat-context: dezelfde regels — geen heredoc, geen lange escape-strings, geen multi-flag `sudo` commando's met spatie tussen.
- Wanneer runbooks via Ansible-automation gerund worden (niet copy-paste): heredocs zijn dan WEL veilig — Ansible `copy:`/`template:` modules schrijven correct.

Gerelateerd: [[feedback-yaml-colon-space]] (eveneens copy-paste-eenduidigheid van docs), [[feedback-no-preview-destructive]] (samen het thema "runbook-content moet exact-copy-paste-safe zijn").
