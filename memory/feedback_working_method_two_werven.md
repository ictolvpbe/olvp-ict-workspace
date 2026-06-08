---
name: feedback-working-method-two-werven
description: "Werkmethode voor 2 parallelle werven (infra: netwerk+server; Odoo-dev). Subagents persisteren NIET — continuïteit komt van memory + per-repo CLAUDE.md + status-memory + claude --resume. Draai Claude per werf vanuit de juiste repo-dir."
metadata:
  type: feedback
---

Twee parallelle werven de komende dagen: **(1) netwerk + server/infra** (repo `~/SchoolProjects/olvp-ict`: platform-handbook + platform-ansible) en **(2) Odoo-ontwikkeling** (repo `~/PyCharm/odoo-myschool`, [[project-myschool-repo]]). Vraag: hoe elke sessie snel her-instappen zonder alles opnieuw in te lezen.

**Kernpunt:** Subagents (Agent-tool) zijn **ephemeer binnen één sessie** — ze persisteren niet en delen geen state over sessies. "2 agents maken" lost het her-inlezen dus NIET op.

**Wat continuïteit wél geeft (persistentie-levers):**
1. **`memory/` + `MEMORY.md`** (auto-geladen index) — al in gebruik. Bij sessiesluit bijwerken.
2. **Per-repo `CLAUDE.md`** — auto-geladen bij sessiestart; korte oriëntatie (architectuur, conventies, "waar staat wat", pointer naar de status-memory). Nog aan te maken voor beide repos.
3. **Per-werf status-memory als her-instap-punt**: infra = [[project-hosting-fase1-status]]; Odoo-dev = [[project-myschool-repo]]. Eén "status + volgende acties"-doc per werf, elke sessie geüpdatet.
4. **`claude --continue` / `--resume`** — zelfde gesprek hervatten zonder her-inlezen (vooral binnen dezelfde dag).

**Aanbevolen opzet:**
- Draai Claude **per werf vanuit de eigen repo-dir** → aparte context + eigen `CLAUDE.md` per werf; minder ruis dan één gemengde sessie.
- Odoo-dev-specifieke memories (bv. [[project-myschool-repo]], [[project-odoo-addons-repo]]) horen logisch bij de odoo-repo; infra-memories bij olvp-ict. Cross-cutting feiten (image, [[project-odoo-image-ldap3]]) in één plek houden en vanuit de andere ernaar verwijzen.
- Optioneel: herbruikbare **custom subagent-definities** in `.claude/agents/*.md` per repo (bv. "ansible-role-author", "odoo-module-reviewer") — consistente scoping voor terugkerende deeltaken, maar secundair t.o.v. de levers hierboven.

**How to apply:** Bij sessiestart: lees de status-memory van die werf + CLAUDE.md, niet alle docs. Bij sessiesluit: werk de status-memory bij. Stel niet voor om "agents" te maken voor persistentie — wijs naar memory/CLAUDE.md/resume.
