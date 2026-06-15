---
name: project-workstation-ansible-key
description: "Werkstation LPT-SCHK (/home/kristof) HEEFT de canonical ansible-key wél (~/.ssh/ansible_olvp, comment ansible-service-account). De eerdere 'verkeerde key'-diagnose was een valse gevolgtrekking. CLAUDE.md/historie verwijst naar oudere demm-machine."
metadata:
  type: project
---

Dit werkstation is **`LPT-SCHK`** (`/home/kristof/SchoolProjects/olvp-ict/`, git-user `ictolvpbe`, `kristof.schalckens@olvp.be`). De CLAUDE.md en eerdere deploy-historie verwijzen naar `/home/demm/` = een **ouder/ander werkstation**.

**CORRECTIE (2026-06-15) — de "verkeerde key"-diagnose van 2026-06-12 was FOUT.** Het werkstation HEEFT de canonical key wél: **`~/.ssh/ansible_olvp`** (fp `SHA256:DpQLouNSmge8uyhCapNKeOoD51pOpsiDmrIgNbuq3Zg`, comment `ansible-service-account`). Bewezen werkend: authenticeert als `ansible@` op **jump-01 (10.35.0.2)** én **srvv-unifi-01 (10.35.0.15)** op 2026-06-15. Deze key zit ook in mijn ssh-agent.

**Waarom de oude diagnose fout was:** srvv-unifi-01 weigerde de key niet omdat hij verkeerd was, maar omdat **die VM toen nog geen `/home/ansible/.ssh/authorized_keys` had** (verse golden-image-clone; `ansible`-account aanwezig uid 1001 maar `.ssh` nooit gevuld + een transiënte NSS/hostname-glitch maakte `getent passwd ansible` even leeg). Een VM zónder ansible-bootstrap weigert *élke* key → daaruit "key is verkeerd" afleiden is een valse gevolgtrekking.

**Naam-val (belangrijk):** de comment `ansible-service-account` ≠ de lege/kapotte Semaphore-keystore-credential met diezelfde naam. Het is net de canonical key die Semaphore opslaat onder credential-naam **`ansible-targets-ssh`** ([[feedback-semaphore-key-canonical]]). Key-*comment* ≠ Semaphore-credential-*naam* — niet verwarren.

**How to apply / les:** om te bepalen of een service-key canonical is, **test 'm tegen een VM die het `ansible`-account al heeft** (bv. `ssh -i <key> ansible@10.35.0.2 whoami`), nooit tegen een verse/ongeprovisionede VM. Bij een nieuwe VM die SSH weigert: check eerst `getent passwd ansible` + `/home/ansible/.ssh/authorized_keys` op de target (account-bootstrap, RB `provision-ansible-account.md`) vóór je de key verdenkt.

**Why:** bus-factor ([[project-bus-factor]]) + secrets-hygiëne ([[project-secrets-management]]) — de canonical key is betrouwbaar terugvindbaar (KeePassXC + lokaal `~/.ssh/ansible_olvp`).

**Los-eindjes hygiëne op LPT-SCHK:**
- `~/.ssh/known_hosts` is **root-owned** → `sudo chown kristof:kristof ~/.ssh/known_hosts` (anders "Failed to add the host", host-key-pinning werkt niet).
- Dubbele kopie `~/Downloads/ansible_olvp(.pub)` (mode 664, private key!) → verwijderen na gebruik.
