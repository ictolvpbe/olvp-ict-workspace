---
name: feedback-semaphore-runner-remote-hang
description: Op de OLVP Semaphore-runner (ansible-core 2.20, 10.35.0.10) hangen playbooks waarvan de EERSTE play remote hosts target (hosts:webapps, hosts:all) vóór de PLAY-banner; alleen localhost-first playbooks (addons-update) werken. Workaround 2026-06-06 — remote-targeting deploys lokaal met --ask-vault-pass draaien. Plus: collections/requirements.yml verplicht voor non-builtin modules.
metadata:
  type: feedback
---

**Vastgesteld 2026-06-06** tijdens de myschool-dev-migratie ([[project-odoo-role-migration]]).

**Symptoom:** een Semaphore-task voor `odoo-podman.yml`, `caddy.yml` of `site.yml` (haproxy) blijft eindeloos op status `running` staan; de output stopt na de galaxy-checks en toont **nooit een `PLAY [...]`-banner**. Geen SSH-sessie vanaf de runner naar de target, geen wijziging op de VM. `--list-tasks` (dat niet eens met hosts verbindt) hangt óók → het is **parse/compile-tijd, vóór de eerste play**.

**Patroon:** playbooks waarvan de **eerste play remote hosts target** (`hosts: webapps`, `hosts: all` op haproxy-hosts) hangen; playbooks met een **`hosts: localhost`-eerste-play** (zoals `addons-update.yml`, dat de groep eerst op localhost opbouwt) werken wél. Zelfde inventory/env/vault-key. Op een **werkstation** draaien dezelfde playbooks probleemloos → het is **runner-specifiek** (vermoedelijk ansible-core 2.20 in de Semaphore-container). Root cause nog niet gevonden.

**Why:** kostte veel debug-tijd omdat de fout cryptisch is (stille hang, geen error). Andere first-guesses bleken vals: niet de image-build (`--skip-tags image` hing ook), niet de runner-gezondheid (addons-update bleef werken), niet de `containers.podman`-collectie (zie hieronder — wel een aparte echte gap).

**How to apply:**
- **Workaround**: draai remote-targeting playbooks (odoo-podman/caddy/site.yml-haproxy) **lokaal vanaf het werkstation** met `--ask-vault-pass` (vault-pw blijft lokaal). Gebruik Semaphore voorlopig enkel voor `addons-update` (localhost-first). DMZ-haproxy-hosts: het werkstation bereikt die enkel via jump-01 (ProxyJump uit group_vars); een lokale haproxy-run die "klaar" lijkt maar de config niet wijzigt (cfg-mtime onveranderd) = de DMZ-hosts werden niet geraakt → check op UNREACHABLE.
- **Later uitzoeken** (WBS-todo): waarom hangt ansible-core 2.20 op de runner bij een remote-first play? Kandidaten: een connection/callback-plugin in de Semaphore-runner-config, fact-gathering-init, of een 2.20-regressie. Test: een minimale `hosts: <een-remote-host>`-playbook via Semaphore.

**Aparte, wél opgeloste gap — `collections/requirements.yml`:** de Semaphore-runner is een schone ansible-omgeving. Playbooks die een **non-builtin collectie** gebruiken (bv. `odoo-podman` → `containers.podman.podman_image/podman_secret`) faalden omdat er geen `requirements.yml` was ("Skip galaxy install process"). Fix: `collections/requirements.yml` met `containers.podman` toegevoegd aan platform-ansible (commit `a6cb7b1`) → Semaphore installeert 'm nu auto vóór elke run. (Dit was NIET de oorzaak van de bovenstaande hang, wel sowieso nodig.)

Gerelateerd: [[feedback-semaphore-rest-api]], [[feedback-semaphore-template-repo-vault]], [[project-odoo-role-migration]].
