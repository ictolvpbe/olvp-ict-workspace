---
name: project-programma-structuur
description: Project is gesplitst in programma met 6 deelprojecten + governance-laag, niet één monolitisch project. Hard gates per project expliciet.
metadata:
  type: project
---

OLVP-infrastructuur-modernisering is sinds 2026-05-21 georganiseerd als **programma** met 6 deelprojecten + 1 governance-laag, na security/sysadmin/ITIL-review.

**Projectstructuur:**
- Project A — Hosting & Identity (huidige Fase 1-2 scope)
- Project B — Security baseline & hardening (HARD GATE voor A)
- Project C — Backup & DR (HARD GATE voor A)
- Project D — Monitoring & Observability (minimaal voor A, volledig Fase 3)
- Project E — Network & physical infrastructure
- Project F — PostgreSQL HA / Patroni (Fase 3)
- Governance — Service-management-laag (loopt parallel)

**15 hard gates** gedefinieerd voor A-go-live, gedocumenteerd in `platform-handbook/governance/programma-structuur.md`.

**Why:** verschillende cadansen (uitvoering vs. ontwerp vs. proces), verschillende stakeholders (directie vs. intern-technisch), bus-factor-mitigatie, ITIL-conformiteit. Eén monolithisch project maakt afhankelijkheden onzichtbaar en is niet schaalbaar bij groei van team.

**How to apply:** bij elk nieuw werkitem eerst project-toewijzing bepalen, dan tracker-rij in juiste tracker. Hard gates blokkeren A-go-live — geen productieverklaring zonder alle 15 op GROEN. Governance-werf altijd parallel updaten bij architectuur-wijzigingen (risk-register, SLA, runbooks).

Gerelateerd: [[project-odoo-public-access]], [[project-phasing]], [[project-bus-factor]].
