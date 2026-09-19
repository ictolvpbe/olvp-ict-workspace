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
Gerelateerd: [[feedback-ansible-check-mode-verify]], [[feedback-same-subnet-test-proves-nothing]].
