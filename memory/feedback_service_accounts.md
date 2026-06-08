---
name: feedback-service-accounts
description: Service-account-strategie OLVP — 'ansible' voor automation, persoonsgebonden accounts voor interactief. Apart key-pair per doel. SSH-key-only + sudo NOPASSWD met audit.
metadata:
  type: feedback
---

OLVP gebruikt een **gescheiden account-strategie** op alle stack-VMs:

- **`ansible`** — service-account voor automation (Ansible-runs vanaf werkstation, Semaphore-jobs). SSH-key-only auth, `NOPASSWD: ALL` sudo. Account leeft op elke stack-VM (jump-01, haproxy-1/2, step-ca, etc.).
- **Persoonsgebonden accounts** (`driesvanmoer`, collega-namen) — voor interactieve SSH, handmatige troubleshooting, ad-hoc commando's. Standaard sudo met wachtwoord.

**Why:**
- **Bus-factor (R-04)**: collega-admin kan Ansible runnen met dezelfde gedeelde service-account-key. Zonder service-account zou elke admin de Ansible-config moeten patchen met zijn eigen username.
- **Audit-trail**: automation-runs lijken op `ansible` in logs (clean), interactieve handelingen op persoonsnaam (traceerbaar).
- **Key-rotatie bij compromise**: ansible-key kan rotaten zonder dat persoonlijke admin-toegang geraakt wordt.
- **Semaphore-integratie**: Semaphore Key Store heeft één key — automatisch passend bij gedeeld service-account.

**Naam-keuze `ansible`**: bewust gangbaar/leesbaar i.p.v. obscure (`olvp-svc`, `srvops`). Security-by-obscurity is geen primary defense — echte verdediging zit in SSH-key-only auth + fail2ban + `AllowUsers`-allowlist + Wazuh-monitoring van sudo-events.

**Sudo-policy `NOPASSWD: ALL`**: acceptabel voor schoolschaal mits mitigatie:
- SSH-key uitsluitend in vault/Semaphore-Key-Store (nooit op disk in plaintext)
- `sudo-ansible.log` voor audit
- Wazuh-agent op host monitort het later
- Strengere policy (per-command-allowlist) is overkill voor 1-2 admins + handvol stack-VMs

**How to apply:**
- Bij **elke nieuwe stack-VM**: doorloop [`platform-handbook/hosting/operations/provision-ansible-account.md`](provision-ansible-account.md) → `ansible`-account + key + sudoers. Tot Ansible baseline-role dit automatiseert.
- `platform-ansible/group_vars/all.yml` heeft `bastion_user: ansible`, `inventory.yml` heeft `ansible_user: ansible`.
- Persoonlijke accounts blijven onafhankelijk daarvan bestaan — geen interferentie.
- Bij `sshd_config` `AllowUsers`: zowel persoonlijke username(s) als `ansible` toelaten.
- Sleutel-management: één centraal exemplaar in vault (KeePassXC/Bitwarden); jaarlijkse rotatie of bij compromise.

Gerelateerd: [[project-bus-factor]] (waarom collega-onboarding nooit Ansible-config-edit mag vereisen), [[project-security-testing]] (sudo-audit-logs voor Wazuh-rules later).
