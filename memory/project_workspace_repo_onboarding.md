---
name: project-workspace-repo-onboarding
description: "Gedeelde Claude-context staat sinds 2026-06-08 in private repo ictolvpbe/olvp-ict-workspace (CLAUDE.md + memory/ + .claude/). Onboarding-runbook RB-2026-ONBOARD-WS dekt een tweede admin op zijn werkstation. Drie repos vormen samen de werkmap."
metadata:
  type: project
---

**Sinds 2026-06-08** — om een tweede admin (bus-factor, zie [[project-bus-factor]]) mee te laten werken is de Claude-werkcontext geversioneerd.

## Drie repos = één werkmap

De werkmap `~/SchoolProjects/olvp-ict/` (bij driesvanmoer: `/home/demm/SchoolProjects/olvp-ict/`) bestaat uit drie GitHub-repos onder org `ictolvpbe`, allemaal SSH + branch `main`:

- **`olvp-ict-workspace`** (privé, NIEUW) — top-niveau: `CLAUDE.md` + `memory/` + `.claude/`. `.gitignore` negeert `platform-handbook/`, `platform-ansible/`, `*.jsonl`, `.idea/` (de twee geneste clones + transcripts + IDE-config). Lost op dat het top-niveau vroeger géén git-repo was → CLAUDE.md/memory stonden buiten versiebeheer.
- **`platform-handbook`** — docs (subdir-clone).
- **`platform-ansible`** — playbooks (subdir-clone).

Klonen in volgorde: eerst workspace-repo als `olvp-ict`, dan de twee werk-repos eráchter erin.

## Onboarding-runbook

`platform-handbook/hosting/operations/onboarding-werkstation.md` (**RB-2026-ONBOARD-WS**): GitHub-org-uitnodiging (Write) → eigen SSH-key → drie repos klonen → vault-pw via KeePassXC (`olvp-ansible-vault-master`, copy-paste niet typen) → verificatie (`ansible-galaxy collection install -r collections/requirements.yml` + `ansible-vault view` + `claude`). Verwijst naar [[feedback-service-accounts]]/provision-ansible-account voor de VM-deploy-key (apart, buiten git).

## Memory-sync gotcha bij parallel werken

`memory/` + `CLAUDE.md` zitten nu in de workspace-repo → bij gelijktijdig Claude-gebruik door beide admins kunnen merge-conflicts op memory-bestanden ontstaan. **How to apply:** start sessie met `git pull` in de workspace-repo, push aan het eind; of spreek per sessie af wie de memory beheert. Sluit aan op [[feedback-working-method-two-werven]] (continuïteit via memory + per-repo CLAUDE.md).

**Secrets blijven buiten git**: vault-pw (KeePassXC) en `ansible-targets-ssh` deploy-key apart en beveiligd delen — nooit in een repo.
