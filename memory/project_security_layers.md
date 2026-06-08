---
name: project-security-layers
description: "Defense-in-depth strategie voor [[project-odoo-public-access]] verdeeld over Fase 1b (prio 1, vóór go-live), Fase 3 (prio 2, najaar 2026) en Fase 4 (prio 3, 2027+). Inclusief host-firewall (nftables) baseline en gedifferentieerde anti-malware. Bijgewerkt 2026-05-31."
metadata: 
  node_type: memory
  type: project
  originSessionId: 59a419f3-43f1-477b-bb2b-196d0c16926a
---

Security-aanpak voor [[project-odoo-public-access]] volgt defense-in-depth: DMZ + service-VLAN-segmentatie alleen is **onvoldoende** voor productie met persoonsgegevens van minderjarigen (GDPR-relevant). Aanvullende lagen zijn verspreid over de projectfasen (zie [[project-phasing]]).

**Why:** User vroeg expliciet of de DMZ-architectuur op zich veilig genoeg was. Antwoord: solide fundering, maar één laag uit een meerlaags model. Cruciaal residueel risico: aanvaller komt via legitiem HTTPS-pad bij Odoo, exploiteert applicatielaag, zit dan op prod-VLAN. Zonder bijkomende lagen: vrije egress (C2/exfil), lateral spread binnen VLAN, backups bereikbaar (ransomware), DB direct toegankelijk.

**Prio 1 — Fase 1b, vóór eerste publieke gebruiker:**
- Egress-filtering prod/test-VLAN (default-deny + allowlist) — perimeter via UniFi
- **Host-firewall (nftables) per-rol via Ansible-baseline** — defense-in-depth naast perimeter, dezelfde rules-matrix als source-of-truth (zie [ADR 0005](platform-handbook/hosting/decisions/0005-host-firewall-nftables.md))
- 2FA op Odoo staff
- Admin-endpoints afgeschermd (HAProxy ACL + `list_db=False`)
- Automatische OS-security-updates
- Backups encrypted, off-site, immutable, getest restore
- **FIM-paden + uitzonderingen per rol gedefinieerd** (Ansible-role klaar, activatie in Fase 3 bij Wazuh-deploy)

**Prio 2 — Fase 3, najaar 2026:**
- Inter-instance isolatie binnen prod-VLAN
- Management-plane scheiden met jump-host
- PostgreSQL-architectuur evalueren (aparte DB-VM)
- Centrale logging + alerting (Wazuh manager + Loki + Grafana)
- **Wazuh-features activeren**: FIM op `/etc /usr/bin /usr/sbin` + container-image-paden, rootkit-detection, auditd-rules (sudo/privesc/ssh-key-modify), SCA (CIS-benchmark per host)
- **Anti-malware gedifferentieerd** (zie sectie hieronder)
- HAProxy rate-limiting
- GeoIP-blokkering

**Prio 3 — Fase 4, 2027+:**
- WAF (Coraza op HAProxy met OWASP CRS, of Cloudflare Pro)
- UniFi IDS/IPS
- Disk-encryption (LUKS) op data-VM's
- Externe pen-test
- Security Incident Response Plan + jaarlijkse oefening

**Anti-malware-strategie (gedifferentieerd, Fase 3):**
ClamAV-overal is **slechte ROI** voor onze stack. Threats per server-type verschillen — toepassing is daarom rol-gebonden:

| Server-type | Primaire dreiging | Mitigatie |
|---|---|---|
| Odoo / HAProxy / step-ca / mgmt-tools | App-exploit, RCE via Python-deps, supply-chain | **FIM + SCA + CVE-scan** via Wazuh. Géén signature-AV — signatures matchen geen runtime app-exploits. |
| Fileserver (SMB, user-uploads) | User uploadt malware-doc, deelt door | **ClamAV on-access** zinvol; Wazuh-ingest van scan-logs |
| Mail-gateway (indien later) | Inbound attachments | ClamAV + rspamd, klassiek |
| Backup-target (Bacula) | Geïnfecteerd bestand restoren | Optioneel scan-on-restore |

Centraal beheer: Wazuh manager ingest **alle** logs (FIM + ClamAV waar wel actief) → één pane-of-glass. Geen aparte AV-management-console nodig.

**Host-firewall vs Proxmox-firewall:**
Proxmox VE heeft eigen 3-niveau firewall (datacenter/node/VM) die out-of-band werkt — VM kan haar eigen FW niet uitschakelen om Proxmox-laag te bypassen. **Verworpen als baseline** wegens drift-risico tussen 3 lagen (perimeter + Proxmox + host) en bus-factor. Heroverwegen indien centrale rule-generation komt (één source-of-truth → genereert alle 3). Zie [ADR 0005](platform-handbook/hosting/decisions/0005-host-firewall-nftables.md) sectie "Alternatieven".

**How to apply:** Bij elke security-vraag verwijzen naar deze lagenstructuur. Prio 1 is een hard-gate vóór publieke ingebruikname — niet onderhandelbaar. Bij escalatie of incident-vragen check eerst welke lagen al actief zijn en welke gepland. Wijk niet af van de fasering tenzij er een concrete nieuwe dreiging is die rechtvaardigt iets uit prio 2/3 naar voren te halen.
