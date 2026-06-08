---
name: feedback-semaphore-rest-api
description: Claude kan Semaphore (10.35.0.10:3000) volledig via de REST API aansturen met een wegwerp-API-token — templates aanmaken/lezen, task-historie + output ophalen, runs starten en monitoren. Bespaart UI-klikwerk, houdt vault/PAT in de keystore (buiten de chat) en maakt debuggen van faalde taken direct mogelijk. Methode-keuze 2026-06-06.
metadata:
  type: feedback
---

**Nieuwe, tijdsbesparende werkwijze (afgesproken 2026-06-06):** Claude bestuurt Semaphore via de **REST API** i.p.v. de gebruiker alles in de UI te laten klikken of secrets in de chat te laten plakken.

**Why:**
- **Secret-hygiëne**: vault-master + GitHub-PAT blijven in Semaphore's **Key Store**; ze raken de chat nooit. Het enige chat-secret is een **Semaphore API-token** — laag-risico (enkel UI-acties), in seconden te roteren (Settings → API Tokens → delete). Veel beter dan vault/PAT plakken (die anders in het sessie-transcript op schijf belanden, zie [[feedback-no-preview-destructive]]-redenering rond blootstelling).
- **Tijdwinst**: templates aanmaken/aanpassen, task-historie + output ophalen, en faalde runs debuggen kan Claude zelf — geen heen-en-weer met screenshots.
- **Debugbaar**: foutoorzaak van een falende taak direct opvraagbaar (bv. de vault-rekey-mismatch in [[feedback-semaphore-template-repo-vault]] werd zo gevonden).

**How to apply:**
- **Bereikbaarheid**: `http://10.35.0.10:3000/api` is bereikbaar vanaf het werkstation (VLAN 34 → MGMT-APPLIANCES:3000, firewall A-002). Vanuit Claude's Bash-omgeving werkt het met `dangerouslyDisableSandbox: true` (netwerk-egress).
- **Auth**: header `Authorization: Bearer <token>`. Token schrijven naar een `0600`-bestand, gebruiken, en na afloop shredden.
- **Project**: heet `platform-ansible` (id 1), NIET "OLVP Platform" zoals de oude doc zei.
- **Kern-endpoints**: `GET /api/projects`; `GET /api/project/1/{templates,inventory,repositories,environment,keys,tasks}`; `GET /api/project/1/templates/{id}` (toont o.a. `vaults:[{vault_key_id,type}]`, `arguments` als JSON-string-array, `inventory_id`, `environment_id`); `POST /api/project/1/templates` (body: project_id, name, playbook, inventory_id, repository_id, environment_id, app:"ansible", arguments, vaults:[{vault_key_id,type:"password",name:""}], allow_override_args_in_task); `GET /api/project/1/tasks/{id}/output` voor de run-logs.
- **Mirror bewezen configs** bij nieuwe templates: webapp-targets → zoals 'Odoo extra-addons Update' (inventory 2, env "OLVP Default" id 1 met ProxyJump, vault 7); haproxy → zoals 'haproxy: dry-run' (inventory 1 = haproxy-hosts, env 0, vault 3). `--limit`/`--tags` via de `arguments` JSON-array (CLI-args), survey-vars gaan als `-e` ([[feedback-semaphore-survey-vs-cli-args]]).

**Verwijs naar een template via NAAM, niet het volgnummer.** Het numerieke template-id is in de Semaphore-UI niet/moeilijk zichtbaar; gebruik in communicatie met de gebruiker altijd de templatenaam (bv. *"Odoo extra-addons Update"*, *"Caddy Deploy — dev-odoo-01"*).

**API-token-levenscyclus**: een token dat de gebruiker plakt blijft door Claude bewaard zolang nodig voor monitoring, en wordt na afloop door de gebruiker in de UI verwijderd + door Claude geshred. Bij langer gebruik: Claude vraagt actief om verwijdering (bv. binnen enkele dagen).

Gerelateerd: [[feedback-semaphore-template-repo-vault]], [[feedback-semaphore-survey-vs-cli-args]], [[feedback-semaphore-key-canonical]], [[feedback-semaphore-proxyjump-prod-webapps]].
