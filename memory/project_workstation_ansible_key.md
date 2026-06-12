---
name: project-workstation-ansible-key
description: "Werkstation LPT-SCHK (/home/kristof) mist de canonical ansible-targets-ssh private key; deploys vanaf deze PC werken pas na correcte key-install. CLAUDE.md/historie verwijst naar oudere demm-machine."
metadata:
  type: project
---

Dit werkstation is **`LPT-SCHK`** (`/home/kristof/SchoolProjects/olvp-ict/`, git-user `ictolvpbe`, `kristof.schalckens@olvp.be`). De CLAUDE.md en eerdere deploy-historie verwijzen naar `/home/demm/` = een **ouder/ander werkstation** dat de canonical ansible-key wél had.

**Knelpunt (ontdekt 2026-06-12):** lokale Ansible-deploys (remote-first playbooks die niet via Semaphore kunnen, [[feedback-semaphore-runner-remote-hang]]) lukken niet vanaf LPT-SCHK omdat de **canonical `ansible-targets-ssh` private key** hier niet stond. Stack-VM's weigeren `ansible@`-SSH (`Permission denied (publickey)`).

**Gotcha's bij het oplossen:**
- Het werkstation heeft enkel een **persoonlijke** `~/.ssh/id_ed25519` (`kristof.schalckens@olvp.be`, aangemaakt 2026-06-11) — niet de service-key.
- KeePassXC-entry-verwarring: er bestaan keys met comment **`ansible-service-account`** (= leeg/problematisch volgens [[feedback-semaphore-key-canonical]]) én de echte canonical **`ansible-targets-ssh`**. Een geëxporteerde `ansible_olvp` (comment `ansible-service-account`, fp `SHA256:DpQLou…`) werd door een verse golden-image-clone GEWEIGERD → exporteer gericht de `ansible-targets-ssh`-entry. Private key zit in die entry mogelijk als **attachment** (het `ssh-ed25519`-veld toont enkel de publieke helft).
- Bij export-fouten: bestand kan als **root** + **publieke** key belanden (`chmod`/`ssh-add` falen dan). Installeer als eigenaar kristof, `chmod 600`, `ssh-add` (of KeePassXC SSH-Agent-integratie).
- `ansible.cfg` definieert géén `private_key_file` (gedeeld, committed — niet wijzigen voor demm-machine) → gebruik **ssh-agent** of `--private-key`.

**Verifieer welke key een VM vertrouwt** via console: `sudo ssh-keygen -lf /home/ansible/.ssh/authorized_keys`, vergelijk met `ssh-add -l`.

**Why:** bus-factor ([[project-bus-factor]]) + secrets-hygiëne ([[project-secrets-management]]) — de canonical service-key hoort betrouwbaar terugvindbaar in KeePassXC én op elk admin-werkstation. Dat hij hier ontbrak/verwarrend opgeslagen was, is een te dichten gat.

**How to apply:** controleer bij een nieuw/ander OLVP-admin-werkstation eerst of `ssh-add -l` de `ansible-service-account`/`ansible-targets`-service-key toont vóór je lokale deploys plant; zo niet, eerst de key-install regelen.
