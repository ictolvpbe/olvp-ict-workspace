---
name: project-vcs-strategy
description: Voorlopig GitHub (private) als VCS voor olvp-ict-repos; migratie naar self-hosted Forgejo gepland wanneer Forgejo-stack opgeleverd is.
metadata:
  type: project
---

Sinds 2026-05-21: GitHub wordt **tijdelijk** gebruikt als VCS voor `platform-handbook` en `platform-ansible`. Beide repos zijn private en staan onder GitHub-user `ictolvpbe` (gedeeld ICT-account, e-mail `ict@olvp.be` — niet persoonlijk, dus bus-factor-veilig). SSH-key voor dit account staat al op deze werkplek. Eindbestemming is self-hosted Forgejo conform [[project-programma-structuur]] (zie `platform-handbook/management-tools/git-forgejo.md`).

**Why:** Forgejo-stack staat nog niet (actief/passief multi-site nog te bouwen — zie git-forgejo.md). Tot dan is GitHub de pragmatische keuze want OLVP gebruikt het al voor myschool-development; geen extra account-overhead voor de ICT-coördinator. Bewust een shared account i.p.v. persoonlijk om [[project-bus-factor]] niet te verslechteren.

**How to apply:**
- Repo's krijgen voorlopig GitHub-remotes (private). Voor publiek-bereikbaarheid: standaard private.
- Bij Forgejo-go-live: migratie via `git remote set-url` + history-push, daarna GitHub-repo's archiveren (niet verwijderen — als rollback en historisch artefact).
- Niet vooruitlopen: geen GitHub-specifieke features (Actions, Projects, Issues) hard-koppelen in docs als die niet 1-op-1 naar Forgejo te dragen zijn. Issues/Project-management gaat sowieso naar Odoo via [[project-mcp-and-itsm]] (Project H).
- CI/CD: Semaphore koppelt aan de actieve VCS-remote; bij migratie remote-URL omschakelen.

Gerelateerd: [[project-programma-structuur]], [[project-mcp-and-itsm]] (Odoo wordt PM/ITSM-tracker, niet GitHub).
