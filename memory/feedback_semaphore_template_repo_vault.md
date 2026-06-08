---
name: feedback-semaphore-template-repo-vault
description: Twee Semaphore-template-gotchas die de 'Odoo extra-addons Update'-run brak (2026-06-05) — primaire Repository moet platform-ansible zijn (niet de addons-repo), en élke template heeft de vault-key nodig omdat group_vars/all/vault.yml auto-laadt.
metadata:
  type: feedback
---

Bij Semaphore-templates voor `platform-ansible`-playbooks: twee config-fouten gaven cryptische errors (beide 2026-06-05 op template 'Odoo extra-addons Update').

**1. Primaire Repository = `platform-ansible`, NOOIT de addons-repo.**
Semaphore resolved de Playbook-path tegen de **primaire Repository** van de template. Stond die op `ictolvpbe/MySchool_addons` (de eerste repo in het log, met `Get current commit hash/message`), dan zoekt Semaphore `addons-update.yml` dáár → `the playbook: addons-update.yml could not be found`. De playbook (+ inventory + vars/instances.yml) leeft in platform-ansible; de addons-repo wordt **runtime door de playbook zelf** op de targets gepulld (`git clone oauth2:{{github_pat}}@github.com/{{addons_repo}}.git`), dus Semaphore heeft die repo niet als checkout nodig.

**2. Élke template heeft de vault-key nodig** (Key Store `olvp-ansible-vault`, type Login With Password, password = vault-master uit KeePassXC `olvp-ansible-vault-master`). `group_vars/all/vault.yml` wordt door **iedere** playbook auto-geladen (group_vars/all geldt voor alle hosts, ook localhost-plays), óók als de playbook zelf geen vault-var gebruikt. Zonder vault-key → `Attempting to decrypt but no vault secrets found` (exit 4). Dit gold vroeger niet toen het bestand `all_vault.yml` heette (werd niet auto-geladen) — sinds de dir-migratie naar `all/{vars,vault}.yml` wél, zie [[feedback-ansible-vault-loading]].

**3. Na een vault-rekey: update de Semaphore Key Store, en weet dat de fout VERTRAAGD opduikt.** Semaphore cachet de git-repo per template en fast-forwardt 'm pas bij de volgende run. Een gerekeyde `vault.yml` wordt dus pas opgehaald wanneer een run de repo bijwerkt — daardoor kan de taak nog dagen "werken" op de oude cache en **pas later** falen op `Decryption failed (no vault secrets were found that could decrypt)` (exit 4) zodra de nieuwe `vault.yml` binnenkomt. Bovendien zijn er **twee** login_password-keys: `olvp-ansible-vault` (id 7 — de meeste webapp-templates: addons-update, odoo-podman, caddy) en ` ansible-vault-password ` (id 3 — de haproxy-templates). **Beide** moeten naar de nieuwe master, anders faalt de helft. Vastgesteld 2026-06-06: 'Odoo extra-addons Update' faalde nadat run #54 de rekey-commit (`e606621..fe87928`, `vault.yml | 35 +++`) binnenpullde terwijl de keystore nog de oude master had — gediagnosticeerd via de Semaphore REST API ([[feedback-semaphore-rest-api]]).

**Why:** Beide errors zijn cryptisch en wijzen niet naar de template-config; je gaat anders de code/git debuggen terwijl die in orde is.

**How to apply:** Bij een nieuwe Semaphore-template voor platform-ansible: Repository = `platform-ansible`, en koppel de `olvp-ansible-vault` Key Store-entry in de Vault-sectie. Bij `playbook not found`: check eerst of de template-Repository niet naar de verkeerde repo gedrift is. Gedocumenteerd in `platform-handbook/management-tools/semaphore.md` §12.6. Zie ook [[feedback-semaphore-survey-vs-cli-args]], [[feedback-semaphore-key-canonical]], [[project-odoo-addons-repo]].
