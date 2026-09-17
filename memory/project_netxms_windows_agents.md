---
name: project-netxms-windows-agents
description: "NetXMS-agent op de Windows-servers: 6 DC's van olvp.int (elk ander VLAN) + SRVV-TOEGANG-01 (Win10, Net2). 6 van 7 hebben al een oude agent. Stand 2026-09-17: klaar voor share + test op INFRA002. Aanpak: GPO-opstartscript op DC's i.p.v. Ansible (geen Tier-0-account in Semaphore); TOEGANG-01 manueel."
metadata:
  type: project
---

**Inventaris (bevestigd door user 2026-09-15)** — alle DC's zijn van het domein `olvp.int`, elk in een ander VLAN:

| Server | IP | 4700 vanaf monitoring (10.35.0.20) | WinRM 5985 vanaf VLAN 34 |
|---|---|---|---|
| SRVV-INFRA002 (DC) | 10.10.0.10 | open (agent draait) | open |
| SRVV-EDU001 (DC) | 10.12.0.10 | open (agent draait) | dicht |
| SRVV-ADM001 (DC) | 10.11.0.10 | open (agent draait) | dicht |
| SRVV-BSP001 (DC) | 10.4.0.10 | open (agent draait) | dicht |
| SRVV-BSW001 (DC) | **192.168.1.10** (niet .0.10 zoals de oude exporterlijst) | dicht (agent draait wel, open vanaf VLAN 34) | open |
| SRVV-SERV-01 (DC) | 10.33.0.10 | dicht, ook vanaf VLAN 34 → geen agent of geblokkeerd | open |
| SRVV-TOEGANG-01 (Win10, Net2-toegangscontrole) | 10.10.102.2 | open (agent draait) | dicht |

- User noemde 10.10.0.10 eerst "SRVV-INFA001"; rootDSE `dnsHostName` en PTR zeggen **SRVV-INFRA002** — user bevestigde.
- WinRM biedt enkel **Negotiate/Kerberos** (geen Basic).
- Geen enkele Windows-machine staat als node in de nieuwe NetXMS; de bestaande agents rapporteren vermoedelijk nog aan de oude server `10.10.100.2`.
- Windows-installer: `nxagent-6.2.3-x64.exe` (Inno Setup; server op 6.2.3, niet 6.2.5 nemen). Stil: `/VERYSILENT /SUPPRESSMSGBOXES /SERVER=<ip>`. **Installer negeert /SERVER en /CONFIGENTRY als er al een config-bestand is** → bij upgrade de config zelf wegschrijven.

**Beslissing (voorstel Claude, 2026-09-15; user corrigeerde feiten zonder bezwaar):** geen Ansible voor de DC's. Dat vraagt een Domain-Admin-account in de vault + Semaphore → 5985 op zes DC's = Semaphore wordt Tier 0 ([[feedback-service-accounts]]). In de plaats: **GPP Immediate Task (SYSTEM) op OU Domain Controllers** — niet een opstartscript, want DC's herstarten zelden; de taak draait bij elke policy-refresh. Script + installer + config in `\\olvp.int\NETLOGON\netxms`, idempotent (config-hash vergelijken, enkel installeren bij versieverschil, firewallregel 4700, service herstarten bij wijziging). SRVV-TOEGANG-01 manueel met hetzelfde script. Code: `platform-ansible/files/netxms/windows/` (`Install-NetXMSAgent.ps1`, `nxagentd.conf`), runbook **RB-2026-NETXMS-WINAGENT** (`deploy-netxms-agent-windows.md`), nog niet getest op een DC.

**Stand 2026-09-17** (4700 getest vanaf 10.35.0.20): BSW001 (192.168.1.10) nu **open** → M-005-deel voor BSW001 is gebeurd. **SERV-01 (10.33.0.10) nog dicht** → UniFi M-005 uitbreiden + vermoedelijk geen agent (script maakt Windows-firewallregel zelf). Rest open zoals voorheen. `nxget` staat niet op monitoring; oude agents laten wellicht enkel 10.10.100.2 toe → agentversie niet remote uit te lezen.

**Volgende stap (bij user):** runbook stap 2 (share `\\olvp.int\NETLOGON\netxms` vullen met installer 6.2.3 + script + conf) en stap 3 (manuele run op SRVV-INFRA002, 2× draaien). User plakt log, `C:\NetXMS\etc\` listing, inhoud oude `nxagentd.conf.bak-*` en service-naam → nakijken exitcode, oude server/subagents (geen metingen verliezen), installatiemap. Daarna GPO, TOEGANG-01, nodes in NetXMS, runbook + firewall-matrix (M-005-rij: BSW001 open) bijwerken. Let op: afgedwongen ExecutionPolicy (AllSigned via GPO) overschrijft `-ExecutionPolicy Bypass`.

**Open:** firewall M-005 uitbreiden met 10.33.0.10; nodes aanmaken in NetXMS; oude config op een DC nakijken (wijst die naar 10.10.100.2?). Zie [[project-netxms-monitoring]], [[project-ad-serv01-migration]].
