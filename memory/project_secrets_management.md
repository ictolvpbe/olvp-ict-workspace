---
name: project-secrets-management
description: Secrets-management OLVP — KeePassXC voor infra-secrets/SSH-keys (lokaal, gratis, alle features), Vaultwarden self-hosted als langere-termijn-doel (Bitwarden-compatible, in mgmt-tools VLAN 35). Bewuste keuze om kritische keys niet in cloud-store te bewaren.
metadata:
  type: project
---

OLVP secrets-management-strategie (beslist 2026-05-24):

**Beleidskern**: kritische credentials (SSH-keys, Ansible-vault-passwords, DB-master-creds, encryption-keys) worden **niet** in een externe cloud-vault bewaard. Self-hosted of lokale opslag is de norm.

**Korte termijn — KeePassXC:**
- Open-source, mature, gratis, alle features
- SSH-agent-integratie native (encrypted vault serveert keys aan ssh-add)
- `.kdbx`-file met sterk master-password (eventueel + key-file als 2e factor)
- Sync tussen werkstation/laptop via Syncthing, encrypted USB, of gedeelde drive
- Sharing met collega-in-opleiding: gedeelde `.kdbx`-file óf aparte vaults met overlap

**Wat in KeePassXC bewaard wordt:**
- SSH-keys voor `ansible`-service-account + persoonlijke admin-keys
- Ansible-vault-passwords (`vault_keepalived_auth_pass` en latere vault-secrets)
- Database-master-creds (Postgres-admin, MySQL waar relevant)
- Hardware-credentials (Proxmox-root, UniFi-admin, NAS-admin)
- Encryption-keys voor backups (Bacula PKI, eventuele LUKS-keys)

**Lange termijn — Vaultwarden self-hosted:**
- Bitwarden-compatible server (open-source)
- Draait als Podman-container in VLAN 35 (mgmt-tools-stack — `/opt/vaultwarden/` per containers-conventie)
- Alle Premium-features (SSH-key-generation, agent, sharing) gratis omdat self-hosted
- Bitwarden-clients (desktop, browser-extension, mobile) werken er native mee
- Migratie-pad: import vanuit KeePassXC (.kdbx → Bitwarden-format) wanneer Vaultwarden operationeel
- Te plannen na Wazuh/Bacula-deploys — geen kritiek pad

**Wat blijft bij gratis Bitwarden cloud:**
- Reguliere user-wachtwoorden (school-accounts, externe diensten)
- Niet-kritieke secrets (sub-account-toegang, third-party SaaS-creds)
- Bewuste scheiding: low-impact in cloud, high-impact lokaal/self-hosted

**Why:**
- Cloud-vault-compromise raakt potentieel kritieke infrastructure-keys → te hoog blast-radius voor school-context met leerlingen-data
- Self-hosted geeft volledige controle over crypto-policy, backup, retentie
- Open-source-tools (KeePassXC, Vaultwarden) hebben geen vendor-lock-in
- Past in defense-in-depth (zie [[project-security-layers]])

**How to apply:**
- Bij nieuwe SSH-keypair-generatie: bewaar privé-deel direct in KeePassXC, NIET in cloud-store
- Bij nieuwe service-credential (DB-password, API-token voor productie): KeePassXC
- Bij reguliere school-account-wachtwoord: gratis Bitwarden cloud OK
- Bij Vaultwarden-deploy later: migratie-werf documenteren in `management-tools/vaultwarden.md`
- Periodieke backup KeePassXC `.kdbx`-file: encrypted USB op veilige plek (niet alleen werkstation)

**Risico: KeePassXC-bestand verloren**
- Mitigatie: backup-procedure (kwartaalbasis), eventueel meerdere kopieën op verschillende media
- KeePassXC zelf is local-only — vault-master-password verloren = onherstelbaar

**Risico: cloud-vault van Bitwarden gecompromitteerd**
- Mitigatie: kritieke secrets staan er niet in (deze strategie). Reguliere accounts hebben MFA en kunnen herzet worden.

Gerelateerd: [[project-security-layers]] (defense-in-depth), [[project-containerization]] (Vaultwarden past in Podman+Quadlet-patroon), [[feedback-service-accounts]] (Ansible-keys in KeePassXC), [[project-firewall-strategy]] (mgmt-tools VLAN 35 ook voor Vaultwarden).
