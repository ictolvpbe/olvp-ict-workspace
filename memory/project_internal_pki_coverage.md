---
name: project-internal-pki-coverage
description: Backlog/scope-keuze (2026-05-28) — step-ca dekt niet alleen Odoo-backends maar uiteindelijk alle interne mgmt-UIs (UniFi, Proxmox, Grafana, NetXMS, ...). Doel: geen browser-warnings meer op mgmt-vlakken.
metadata:
  type: project
---

**Status 2026-05-28**: step-ca is **live** op `SRVV-STEPCA-01` (10.21.1.10, VLAN 22 — VLAN-tag verhuisd 211 → 22 op 2026-05-31, IP-subnet ongewijzigd, service uptime onaangetast). PKI ECDSA P-256, root + intermediate beide valid tot 2036-05-25. Provisioners actief: `admin` (JWK, voor handmatige cert-issuance) en `acme` (voor Caddy automatic enrollment). Root-key offline op LUKS-encrypted USB; alleen intermediate-key + intermediate-pw op de server. Root fingerprint + USB-locatie + 3 passwords (root / intermediate / provisioner) in KeePassXC. Caddy-cohort + UniFi/Proxmox-rollout uit te voeren in volgende sessies. Zie [install-stepca.md](platform-handbook/hosting/operations/install-stepca.md) voor de bijgewerkte runbook met **alle reality-gaps** die tijdens deze bringup naar voren kwamen (deb is minimaal — handmatige user + systemd-unit; 2-niet-3 prompts; --acme bij init; USB-backup verplicht vóór change-pass + shred).

Vastgelegd 2026-05-28: scope van [[project-architecture-haproxy]]/step-ca breidt expliciet uit naar **alle interne mgmt-UIs**, niet alleen Odoo-Caddy-backends.

**Drijfveer (Why):** Cert-warnings op UniFi-controller, Proxmox, etc. zijn nu de norm. Dat traint admins om `Doorgaan ondanks waarschuwing` als reflex te klikken — precies in de zone waar MITM het pijnlijkst zou zijn. Eindbeeld: nul browser-warnings op enig intern UI binnen OLVP.

**Strategie per device-categorie** (volledige matrix in `platform-handbook/management-tools/step-ca.md` sectie "Coverage"):

| Categorie | Aanpak |
|---|---|
| Linux + Caddy | ACME → step-ca, automatic. Triviaal zodra step-ca live. |
| Linux native HTTPS | `step ca certificate` via systemd-timer + service-reload. |
| Appliance met cert-upload UI (UniFi controller, Proxmox) | mgmt-host pulled leaf-cert uit step-ca → keystore-import-script → service-restart. |
| Appliance zonder cert-mgmt | Meerijden via parent (UniFi switch volgt controller), of niet direct exposen (achter Caddy/HAProxy met TLS daar). |
| Niet-cert-baar (legacy IPMI, etc.) | Geen cert; toegang alleen via VPN/bastion, gedocumenteerde acceptatie. |

**Volgorde-aanbeveling:**
1. Caddy-cohort (out-of-the-box bij stappen Fase 1).
2. Proxmox (één keer scripten, hoge waarde — kritieke admin-UI).
3. UniFi controller (zelfde inspanning, hoge waarde — frequent gebruikt).
4. Long-tail per kost-baat.

**Backlog-runbook (nog te schrijven):** `platform-handbook/hosting/operations/cert-rollout-internal-uis.md`. Te starten zodra step-ca operationeel is en de Caddy-cohort werkt.

**How to apply:**
- Bij elke nieuwe Linux-VM met web-UI: standaard Caddy ervoor + ACME tegen step-ca, geen self-signed-fallback laten staan.
- Bij elke nieuwe appliance: check of cert-upload-API/CLI bestaat. Zo ja, agendeer voor het rollout-runbook. Zo nee, beoordeel "achter reverse proxy zetten" als alternatief.
- Bij security-risk-register: bij een MITM-incident eerst checken of de admin een cert-warning heeft weggeklikt — train tegen die reflex.

Gerelateerd: [[project-architecture-haproxy]] (TLS terminate+re-encrypt-model), [[project-secrets-management]] (root-key offline), [[feedback-zijsporen-welkom]] (proactief vastleggen).
