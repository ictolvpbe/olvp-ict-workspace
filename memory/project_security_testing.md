---
name: project-security-testing
description: 5-cyclus security testing plan (continuous/hygiëne/audit/drill/extern-jaarlijks). Tooling-keuze open-source-first behalve Defender Attack Simulator (M365 A5 reeds inbegrepen, vervangt GoPhish). VLAN 38 gereserveerd voor zelf-pen-test. Externe mini-pen-test 1x/jaar, vanaf Jaar 2 zelf-pen-test-leerpad.
metadata:
  type: project
---

OLVP security-testing-strategie (vastgelegd 2026-05-22):

**Aanpak**: 5 parallelle snelheden i.p.v. 1x/jaar externe pen-test:
- **Continuous**: Wazuh (vuln + SCA + FIM) + Ansible --check + Loki alert-rules — dagelijks geautomatiseerd
- **Hygiëne**: testssl + nuclei + checkdmarc + patch-status — wekelijks via Semaphore
- **Audit**: nmap + ZAP-baseline + lynis + Bacula restore-test + access-rights review — maandelijks
- **Drill**: OpenVAS + Defender Attack Simulator phishing + tabletop + risk-register-walk — per kwartaal
- **Audit-extern**: jaarlijkse mini-pen-test (max budget €2-4k) door BE-vendor (NVISO/Toreon/Securitymind)

**Key keuzes:**
- **Phishing-simulatie via GoPhish in ad-hoc-modus** met strikte hardening. DAS (M365 A5) verworpen want OLVP-personeel zit op Google Workspace (Gmail), geen EXO-mailboxen. Google's eigen tool is uit Admin Console verwijderd (vóór 2026). Geen native "gratis en veilig" pad meer beschikbaar — GoPhish-keuze is pragmatisch met mitigatie via isolation.
- **GoPhish-hardening (verplicht bij elk gebruik):** VLAN 38 isolated, alleen booted-up tijdens campagne (~1 dag/kwartaal), dashboard intern IP only, eigen mail-relay (postfix lokaal, tijdelijk), source-code/CVE-check vóór elk gebruik, snapshot+power-off na campagne, DMARC/SPF-uitzondering tijdelijk en gericht.
- **Quarterly review:** bij elke drill één-zin-evaluatie of GoPhish blijft of overschakelen naar Lucy/KnowBe4. Beslissing in drill-rapport noteren.
- **VLAN 38 gereserveerd voor zelf-pen-test**: VL-SD-PENTEST 10.38.0.0/24 — Kali + lab-VMs (DVWA/Juice Shop/Metasploitable). Default-deny naar alle andere zones, opt-in per scan-sessie.
- **Zelf-pen-test leerpad voor ICT-coör**: Maand 1-6 PortSwigger Academy + TryHackMe (gratis/€10/m). Maand 6-12 praktijk op eigen test-stack. Vanaf Jaar 2 quarterly zelf-pen-tests. Optioneel eJPT (€200). OSCP pas overwegen na 2j ervaring.
- **Externe audit elke 2-3 jaar** voor "second pair of eyes" — schoolbudget = mini-scope.

**Why:**
- Eén-keer-per-jaar audit = 51 weken aanvalsoppervlak ongedekt.
- M365 A5 reeds in bezit = phishing-tooling gratis route.
- Bus-factor verlagen door zelf-capability op te bouwen, externe audit als verificatie behouden.
- Open-source-first past in stack-keuzes (Wazuh, Ansible, Loki).

**How to apply:**
- Bij elke nieuwe stack-component (HAProxy, Odoo-instance, ...): check of het in continuous-mode (Wazuh-agent) en hygiëne-mode (publieke endpoint dan in testssl/nuclei-scope) zit.
- Quarterly drills strict inplannen tijdens schoolvakanties — niet tijdens schooluren of examen-periodes.
- Bij elk Wazuh/SCA-finding > High of nieuwe CVE relevant voor stack-component: triage binnen 1 werkdag.
- VLAN 38 alleen activeren tijdens scan-sessies; daarbuiten hosts down. Geen Wazuh-agent op Kali (anders fouttraps eigen scans).
- Pen-test-output naar aparte log-stream (security-scans/), retentie 90 dagen, ACL beperkt tot ICT-coör + collega.

**Risk-entries:**
- R-27: phishing-tool-landschap fragiel (mitigatie: GoPhish ad-hoc + hardening, quarterly review of tool-keuze nog acceptabel is, Lucy/KnowBe4 als commerciële fallback, eigen Odoo-module als lange-termijn-optie post-Project G+H)
- R-28: zelf-pen-test skill-rot (mitigatie: doorlopend leerpad + quarterly drills + externe audit als second opinion)

**Te checken vóór go-live:**
- GoPhish-VM voorbereiden in VLAN 38 met hardening-checklist (zie [[security-testing-plan]])
- Externe pen-test-vendor-shortlist + quotes (~3 maanden vóór go-live)
- Scope-document-template + jaarlijkse directie-autorisatie (formele basis)
- Eerste pilot phishing-campagne kleine groep, daarna breder

Gerelateerd: [[project-security-layers]] (defense-in-depth), [[project-mcp-and-itsm]] (Project H — pen-test-findings → Odoo-tickets), [[feedback-risk-aware-changes]] (drills tijdens schoolvakanties), [[project-bus-factor]] (skill-onderhoud + collega-in-opleiding).
