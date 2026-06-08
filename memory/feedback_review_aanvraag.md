---
name: feedback-review-aanvraag
description: Bij review-vragen ("analyseer als security expert/sysadmin/ITIL") verwacht user grondige extractie eerst, dan eerlijke gap-analyse, ook strategisch advies (bv. project opsplitsen).
metadata:
  type: feedback
---

Wanneer de user vraagt om review/analyse vanuit een expert-rol (security-expert, sysadmin, ITIL-auditor, code-reviewer), volg deze aanpak:

1. **Eerst gestructureerde extractie** van wat er werkelijk in de repo/code staat — niet op samenvatting werken. Bij grotere corpora: Explore-agent gebruiken met expliciete extractie-opdracht (geen oordeel, alleen feiten).
2. **Eerlijke gap-analyse** — benoem zowel sterktes als zwaktes. Niet alleen ja-knikken.
3. **Strategische aanbevelingen** toevoegen, ook als de user er niet expliciet om vraagt — bv. "dit zou efficiënter zijn als programma met deelprojecten" of "dit hoort eigenlijk in een aparte werf". De user heeft dit expliciet als waardevol bestempeld op 2026-05-21.
4. **Concrete verbeterpunten** met prioriteit, owner en deadline-suggestie.
5. **Bijvragen aan einde** via AskUserQuestion bij echte beslismomenten (max 4) — niet om scope te bevestigen, wel om onbekende info op te halen.

**Why:** user is ICT-coördinator, alleen primary, en wil tegenspel om blinde vlekken te vinden. Zachte adviezen worden gemist; harde adviezen worden gewaardeerd. Bevestigd op 2026-05-21 review-sessie.

**How to apply:** bij elke "analyseer/review/audit"-vraag — investeer eerst in volledige read-pass, dan analyse. Niet vertrouwen op samenvattingen uit eerdere context. Bij grote corpora: budget 1-2 grondige sub-agent-extracties voor analyse-kost ipv samenvatting-uit-context risico.
