---
name: feedback-verify-memory-against-repo
description: "Memory-regels kunnen verouderd of ronduit fout zijn — de index-regel van project_security_testing beweerde het tegenovergestelde van zijn eigen memory. Cross-check een uit memory afgeleid feit tegen het handbook vóór je er een conclusie op bouwt."
metadata:
  type: feedback
---

**Vastgesteld 2026-09-18** tijdens de Euro-Office-werf ([[project-eurooffice-nextcloud]]).

Ik schreef drie keer dat OLVP **M365 A5** als productiviteitssuite gebruikt, en bouwde daar een
strategisch argument op ("de groupware-achterstand van OpenCloud is theoretisch, want M365 dekt
mail/agenda/Teams al"). De user corrigeerde: **OLVP draait op Google Workspace.** De M365 A5-licenties
zijn voor leerlingen + Intune, zonder EXO-mailboxen voor personeel.

**Waarom het misging:** ik leidde het af uit één memory-regel (`project_security_testing` noemt
"Defender Attack Simulator (M365 A5)") en verifieerde niet. Het handbook had het wél correct staan —
`governance/security-testing-plan.md` zegt letterlijk dat DAS **niet bruikbaar** is omdat het
personeel op Gmail zit. Eén grep had het opgelost.

**Erger nog:** die MEMORY.md-index-regel sprak zijn eigen memory-bestand tegen. De index zei
"Defender Attack Simulator i.p.v. GoPhish", het bestand zelf zei dat DAS verworpen is en GoPhish
gekozen. Een index-regel wordt élke sessie in context geladen — een foute regel daar vergiftigt
structureel. (Rechtgezet 2026-09-18.)

**How to apply:**
- Een feit uit memory is een **aanwijzing, geen bron** — zeker wanneer het de premisse wordt onder
  een aanbeveling. Grep het handbook vóór je erop voortbouwt.
- Let extra op wanneer het feit uit een memory komt die over een **ander onderwerp** gaat: de
  M365-vermelding stond in een security-testing-memory, niet in een memory over de productiviteitssuite.
- Ziet een index-regel er samenvattend uit, lees dan het onderliggende bestand vóór je hem citeert;
  index en inhoud kunnen uiteenlopen.
- Corrigeer een foute memory-regel meteen wanneer je hem tegenkomt, ook al gaat de taak er niet over —
  past bij [[feedback-zijsporen-welkom]].

In dezelfde sessie ook te stellig geweest over "OpenCloud heeft geen Calendar/Contacts" (die bestaan
al sinds mei 2025). Zelfde onderliggende fout: een plausibele aanname als vaststelling presenteren.

## Het omgekeerde geval — 2026-09-23: het handbook had ongelijk

Hierboven staat "grep het handbook" als de correctie. Dat is niet de hele les, want het handbook is
zelf ook maar een document. Bij de cloud-werf bleek:

- `network-physical/reference/ip-plan.md` zette de Odoo-testservers op **`10.200.0.40/.41`** in
  "VLAN 200", en `hosting/reference/vm-inventory.md` herhaalde dat.
- `platform-ansible/inventory.yml` zei **`10.200.14.40/.41`**.
- Op `10.200.0.40` antwoordt **niets**. De echte band is `10.200.14.0/23`, gateway `10.200.14.1`,
  dus VLAN **207**. Eén `ssh` besliste het. (Rechtgezet 2026-09-23; `SRVV-TST-ODOO-02` stond er
  bovendien nog, terwijl die VM al sinds juni `SRVV-DEV-ODOO-01` heet.)
- Ook `management-tools/ansible.md` droeg nog `group_vars/all_vault.yml` als conventie, terwijl
  [[feedback-ansible-vault-loading]] al vastlegt dat dat pad **niet** geladen wordt.

**How to apply — de rangorde van bronnen, van hard naar zacht:**

1. **De draaiende machine.** `ssh`, `ip -4 addr`, `dig`, `curl`, `podman ps`. Dit is de enige bron
   die niet kan verouderen.
2. **De automation-repo** (`inventory.yml`, de SoT-bestanden in `vars/`). Die wordt bij elke run
   gebruikt, dus een fout valt er snel op.
3. **Het handbook.** Wordt gelezen, niet uitgevoerd — een fout kan er maanden blijven staan.
4. **Memory.** Een aanwijzing, geen bron.

Spreken twee lagen elkaar tegen, geloof dan de hardere — en **schrijf de zachtere meteen bij**.
Documentatie die ongecorrigeerd naast een gemeten feit blijft staan, is erger dan geen documentatie:
de volgende lezer weet niet welke van de twee hij moet geloven.

Gerelateerd: [[feedback-ansible-check-mode-verify]], [[feedback-same-subnet-test-proves-nothing]],
[[feedback-ontwerp-runbook-niet-geverifieerd]], [[feedback-deployed-branch-not-main]].
