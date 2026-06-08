---
name: feedback-ansible-check-mode-verify
description: Bij ansible-runs altijd target-host state verifiëren (dpkg/systemctl) wanneer iets na `changed=true` niet werkt zoals verwacht — `--check`-mode kan onbedoeld actief blijven en ansible rapporteert dan misleidend changed-status zonder werkelijke wijziging.
metadata:
  type: feedback
---

**Regel:** wanneer een ansible-task `changed=true` rapporteert maar opvolgende tasks falen alsof er niets gewijzigd is, verifieer eerst op de target-host dat de wijziging écht doorgevoerd is (`dpkg -l <pkg>`, `systemctl is-active`, `ls -la <pad>`). Vertrouw niet blind op de PLAY RECAP.

**Why:** 2026-05-31 — bij HAProxy-deploy verloren we 30+ minuten doordat de eerste twee playbook-runs in check-mode liepen (één expliciet, één per ongeluk via shell-history-recall). Beide runs rapporteerden `changed: [host]` voor de package-install-task, maar de packages stonden NIET op de hosts. Pas toen ik via SSH `dpkg -l haproxy` runde was duidelijk dat ansible's apt-module enkel het simulatie-resultaat had gelogd. Symptoom dat naar deze valkuil wijst: opvolgende tasks falen met "service/file not found" terwijl ansible enthousiast "changed" rapporteerde.

**Bijkomende observatie:** ansible's apt-module rapporteert `changed=true` zelfs in pure-cache-update gevallen (`cache_updated: true, changed: false` op de _ene_ package-actie). Het outer `changed`-label is gebaseerd op de OR van cache + packages — kan misleiden.

**How to apply:**
- Na elke "echte" run waar packages/configs/services nieuw zijn: SSH naar één target en quick-verify (`dpkg -l <pkg>`, `systemctl is-active <svc>`). Kost 5 seconden, voorkomt half-uur troubleshooting later.
- Bij playbook-runs die failen NA een "successful" apt-task: eerst dpkg-check vóór andere oorzaken zoeken.
- In shell-history: gebruik `Ctrl+R` zoekfunctie i.p.v. blind pijl-omhoog wanneer wisselend tussen `--check` en echte runs — een vergeten `--check` is een dure error.
- Overweeg: ansible-task taggen met een `validate:`-step die WERKELIJK op de host valideert (zoals haproxy's `validate: 'haproxy -c -f %s'` doet) — vroege fail in plaats van late mysterie.

Gerelateerd: [[feedback-risk-aware-changes]] (waarom we überhaupt dry-runs doen vóór live).
