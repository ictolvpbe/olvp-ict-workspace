---
name: feedback-deployed-branch-not-main
description: "platform-ansible draaide op 2026-09-23 niet vanaf main: melira-frappe-fase1 stond 36 commits voor en droeg de frappe_vms-blokken die wél in de live haproxy.cfg staan. Uitrollen vanaf main had de FRAME-backend gewist. Check altijd welke branch de uitgerolde staat draagt vóór je een gedeelde template aanraakt."
metadata:
  type: feedback
---

**Vastgesteld 2026-09-23** bij het toevoegen van backends aan `templates/haproxy.cfg.j2`
([[project-eurooffice-nextcloud]]).

De werkkopie van `platform-ansible` stond op **`melira-frappe-fase1`**, niet op `main`. Die branch
liep **36 commits voor** en `main` één commit apart. Het verschil is niet cosmetisch: de hele
FRAME-werf zit erin, inclusief de `frappe_vms`-lus in `haproxy.cfg.j2` en `vars/frappe-instances.yml`.

**Waarom dat gevaarlijk is:** de live `haproxy.cfg` op `SRVV-HAPROXY-01A` bevat `be_srvv_tst_frame_01`.
De uitgerolde staat komt dus van de **feature-branch**, niet van `main`. Had ik mijn wijziging
netjes "volgens de regel" vanaf `main` vertakt en uitgerold, dan had `site.yml` een `haproxy.cfg`
gerenderd **zonder** de FRAME-blokken — en was `frame-test.olvp.be` publiek stukgegaan door een
wijziging die er niets mee te maken had.

**Why:** een gedeelde template als `haproxy.cfg.j2` rendert de **volledige** configuratie uit alle
SoT-bestanden. Er is geen gedeeltelijke uitrol: wat niet in jouw branch staat, verdwijnt uit de
gerenderde output. Bij een per-host rol valt dat mee; bij een centrale edge-config is het een storing.

**How to apply:**

- Vóór je een gedeelde template aanraakt: `git branch --show-current` en
  `git rev-list --left-right --count main...<branch>`. Staat de werkkopie niet op `main`, ga er dan
  van uit dat die branch de uitgerolde staat draagt tot je het tegendeel hebt **gemeten**.
- Meten doe je op de doelhost, niet in de repo: `grep backend /etc/haproxy/haproxy.cfg` op
  haproxy-1 vertelt je in één regel welke SoT's er in de live config zitten.
- Vertak van de branch die de uitgerolde staat draagt, niet van `main`, ook al voelt dat fout. De
  regel "geen rechtstreekse commits op prod-config zonder branch" gaat over reviewbaarheid, niet
  over de vraag wélke basis correct is.
- Een langlopende feature-branch die in productie draait maar niet gemerged is, is zelf het echte
  probleem — signaleer het in plaats van er alleen omheen te werken. Hier: FRAME draait publiek
  sinds 2026-09-22 ([[project-frame-frappe-hosting]]) terwijl `main` het niet kent.

Gerelateerd: [[feedback-verify-memory-against-repo]] (zelfde onderliggende fout: een plausibele
aanname als vaststelling nemen), [[project-hosting-fase1-status]].
