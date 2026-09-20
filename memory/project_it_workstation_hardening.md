---
name: project-it-workstation-hardening
description: "Nieuw werfje (user, 2026-09-20): de Linux-werkstations van de IT-dienst extra beveiligen en bewaken. Nog niets beslist. Raakt aan secrets-opslag, VLAN 34, NetXMS/Wazuh en de bus-factor."
metadata:
  type: project
---

Aangekondigd door de user op 2026-09-20, samen met het voornemen om de KeePassXC-`.kdbx` naar een
externe disk te verhuizen ([[project-secrets-management]]).

**Waarom dit zwaarder weegt dan het lijkt.** De IT-werkstations staan in **VLAN 34**, het segment
dat volgens [[project-mgmt-segmentatie-vlan30]] als enige bij de beheerinterfaces mag. Ze dragen
de ansible-service-key, het Ansible-vault-wachtwoord en de KeePassXC-database. Een gecompromitteerd
beheerwerkstation levert dus in één klap de sleutels tot de hele vloot — alle segmentatie eromheen
ten spijt. Dit is de laag waar de defense-in-depth van [[project-security-layers]] vandaag het
dunst is.

**Meer dan één toestel.** [[project-workstation-ansible-key]] documenteert `LPT-SCHK`
(`/home/kristof`), terwijl deze sessie op `/home/demm` draait. Eerste stap is dus simpelweg
vaststellen **welke** werkstations er zijn en wat er op staat.

## Te bekijken (nog niets van beslist)

- **Inventaris** — welke toestellen, welke distributie, wie gebruikt ze.
- **Schijfversleuteling** — LUKS full-disk; staat als Fase 4 in [[project-phasing]], maar voor
  een beheerwerkstation is dat laat.
- **Secrets** — `.kdbx` naar externe disk, en de afweging bruikbaarheid tegenover air-gap.
- **Bewaking** — NetXMS-agent zoals op de servers ([[project-netxms-monitoring]]), en Wazuh voor
  FIM/auditd zodra die er is. Een werkstation is geen server: andere ruis, andere drempels.
- **Toegang** — schermvergrendeling, sudo-beleid, SSH-agent-timeout, 2FA op de vault.
- **Updates** — onbeheerd of via dezelfde Ansible-weg als de vloot.
- **802.1X** — werkstations horen bij het bestaande spoor [[project-client-identity]].

## Aandachtspunt

Bewaking van een persoonlijk werkstation raakt aan de privacy van de gebruiker, ook als dat de
ICT-verantwoordelijke zelf is. Bij uitbreiding naar toestellen van collega's hoort dat in de DPIA
— zie de redenering in ADR 0007 over netwerkmonitoring.
