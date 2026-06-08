---
name: project-odoo-addons-repo
description: Odoo addons-source = github.com/ictolvpbe/MySchool_addons (private). Branch-conventie case-sensitive — `master` voor prod, `Dev` (hoofdletter D) voor test+dev. Wordt gepulled per-instance door Semaphore "Odoo extra-addons Update" template.
metadata:
  type: project
---

**Repo**: `github.com/ictolvpbe/MySchool_addons` (private, requires PAT)

**Branch-conventie (CASE-SENSITIVE)**:
- `master` → prod-instances (alle SRVV-ODOO-01 + SRVV-ACC-01:prod)
- `Dev` (met hoofdletter D, niet `dev`) → test- en dev-instances

Per-instance mapping in `platform-ansible/vars/instances.yml` veld `addons_branch`.

**Why**: Bij eerste live run van addons-update playbook (Semaphore werf #29, 2026-06-02) failde de pull eerst op `dev` (Git case-sensitive — repo heeft `Dev`), daarna kwam aan het licht dat prod-branch `master` is, niet `main`. Beide conventies vastgelegd na bevestiging door user.

**Pull ≠ actief**: de addons-update playbook zet alleen de code op schijf — geen restart, geen module-upgrade. Sinds 2026-06-05 drie survey-vars om wijzigingen te activeren: `restart_after=true` (herstart `odoo_instance_N` → laadt gewijzigde `.py`, pattern-agnostisch want puur `systemctl restart`), `upgrade_modules=<modulenaam>|all` (`odoo -u` via `podman exec --stop-after-init --no-http` → laadt views/data/security in DB, impliceert restart), `upgrade_db=<db>` (db-override voor instances zonder `db_name`). **upgrade_modules/upgrade_db zijn GEEN booleans** (fail-fast guard; footgun van eerste run was `-u true -d true`). **Upgrade werkt alleen op conf-pattern containers** (role-based, db-creds ín odoo.conf → `odoo -c` verbindt; bv. ACC-01); **env-pattern** VMs (handmatig Docker→Podman gemigreerd — creds via container-env die de entrypoint injecteert) worden overgeslagen-met-SKIP want `odoo -c odoo.conf` vindt daar geen db_host. Restart volgt wel. Upgrade ook overgeslagen waar geen db resolvable is.

**Selector-model (sinds 2026-06-05, channel-refactor)**: 3 complementaire survey-selectors, breed→chirurgisch: `target_limit` (welke VMs, default `webapps`) → `target_channel` (kanaal prod/test/dev, filtert op instance-veld `channel`) → `target_fqdn` (komma/spatie-lijst exacte FQDN's, **wint van channel**). Per-instance veld heet nu **`channel`** (niet meer `env` — `env` op instance-niveau is hernoemd; VM-level `env` blijft = Proxmox-tier, losgekoppeld). Een prod-VM kan een dev-channel instance dragen: bv. **myschool-ict** her-gechanneld naar `dev`/Dev (staging op prod-hardware SRVV-ODOO-01). Survey-var `target_env` → hernoemd naar `target_channel`. **id.olvp.be + id-test.olvp.be verwijderd uit instances.yml** (worden Keycloak op SRVV-ID-01).

**How to apply**:
- Nieuwe Odoo-instance toevoegen aan instances.yml → `channel` (prod/test/dev) + `addons_branch` (prod=master, test/dev=Dev)
- Eén instance updaten dat een channel deelt → `target_fqdn=<fqdn>`
- Survey `addons_branch_override` voor één-shot branch-afwijking (bv. feature-branch test)
- Code echt willen activeren: `restart_after=true` (py) en/of `upgrade_modules=<mods>` (views/data)
- Semaphore-template survey-vars na deze refactor: hernoem `target_env`→`target_channel`, voeg `target_fqdn` toe
- Bij toekomstige Forgejo-mirror ([[project-vcs-strategy]]): zelfde branch-namen behouden om Semaphore-pipelines niet te breken
- Wanneer GitHub-PAT vernieuwd wordt: KeePassXC entry `github-pat-semaphore` updaten + Semaphore Variable Group SECRETS-tab

**Authenticatie**: PAT `github-pat-semaphore` (KeePassXC) staat in Semaphore Variable Group "Default" (project `platform-ansi…`) als secret. Playbook resolveert via ansible-var `github_pat` (Extra Vars) of shell-env `$github_pat` / `$GITHUB_PAT` — beide werken.

Gerelateerd: [[project-vcs-strategy]] (GitHub → Forgejo migratie-pad), [[feedback-semaphore-key-canonical]], [[feedback-semaphore-survey-vs-cli-args]].
