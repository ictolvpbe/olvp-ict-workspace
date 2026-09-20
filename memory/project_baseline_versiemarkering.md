---
name: project-baseline-versiemarkering
description: "Elke stack-VM draagt /etc/olvp/baseline.json (gesymlinkt als facts.d/olvp.fact) met baseline-versie + lijst van uitgevoerde saneringen. ADR 0009, geïmplementeerd in platform-ansible en vlootbreed gedraaid 2026-09-19. Bewijs-gestuurd: de inhoud komt uit een meting, niet uit een claim."
metadata:
  type: project
---

[ADR 0009](../platform-handbook/hosting/decisions/0009-baseline-versiemarkering.md), geïmplementeerd
in `platform-ansible` en vlootbreed gedraaid op 2026-09-19. Het mechanisme is schema-migratie, maar
dan voor servers: het playbook leest wat er staat en doet alleen wat ontbreekt.

**Bron van waarheid:** `/etc/olvp/baseline.json`, gesymlinkt als `/etc/ansible/facts.d/olvp.fact`
→ beschikbaar als `ansible_local.olvp`. Rechten **0644**; bij 0755 vóért facts.d het bestand uit
in plaats van het te lezen.

**Twee lagen.** `baseline_version` (`<debian-major>.<revisie>`, nu `13.1`) zegt welke standaard de
host draagt; `applied` zegt welke saneringen effectief gedraaid zijn. Eén nummer volstaat niet voor
een vloot die niet uniform is — een host wordt retroactief nooit in één keer een hele versie.

**Sleutels:** `machine-id-unique`, `disk-profile`, `srv-volume`, `journald-cap`, `log-hygiene`,
`podman-quadlet`, `step-ca-trust`.

## Waarom bewijs-gestuurd

`tasks/baseline-detect.yml` **meet** de werkelijke toestand (machine-id, LV-maten, aanwezige
configs); `tasks/baseline-marker.yml` schrijft dát weg. Een sleutel verdwijnt dus vanzelf weer als
iemand met de hand iets terugdraait, en `machine-id-unique` kan niet gezet worden zolang de
herstart nog moet komen. Daarmee is de regel "alleen automation schrijft" technisch dragend in
plaats van alleen sociaal — een markering die liegt onderdrukt net de controle die ze moet sturen.

De markering is de **laatste** taak van `tier1-baseline.yml`: faalt er iets daarvoor, dan valt de
host uit de play en claimt het bestand niets.

## How to apply

- Inventariseren zonder iets te wijzigen: `ansible-playbook baseline-assess.yml -e assess_write=false`
- Geen bestand = generatie 0, nog te beoordelen. Dat is je werklijst.
- Ruimtegebrek laat `disk-profile`/`srv-volume` **overslaan**, niet falen — anders blokkeert het de
  goedkope saneringen én de markering erna.
- `meta: end_host` in een ingesloten taakbestand beëindigt de **hele** play voor die host. Kostte
  me de markering; vervangen door `when`-condities.
- Leestaken hebben `check_mode: false` nodig, anders is `--check` onbruikbaar.

## Stand van de vloot (2026-09-19, 12 hosts gemeten)

| Sleutel | Ontbreekt op |
|---|---|
| `disk-profile`, `srv-volume`, `journald-cap` | 12/12 — nieuw |
| `machine-id-unique` | 11/12 — alleen `srvv-p-backup-01` is gesaneerd |
| `step-ca-trust` | 5 — sspr, p-backup, forgejo, monitoring, unifi |
| `podman-quadlet` | 4 — haproxy-1/2, p-backup, **dev-odoo-01** |
| `log-hygiene` | 3 — haproxy-1/2, p-backup |

`dev-odoo-01` mist `podman-quadlet` terwijl `tst-odoo-01` het wél heeft: zelfde rol, zelfde
generatie, andere configuratie. Precies de drift die de baseline moest voorkomen.

⚠️ `podman-quadlet` test alleen op `/etc/containers/registries.conf.d/unqualified.conf` — een
zwakke proxy. Op haproxy draaien geen containers, dus "ontbreekt" is daar betekenisloos.
Kandidaat om te hernoemen naar `registries-conf` plus een echte Podman-controle.

Gerelateerd: [[project-template-fleet-defects]], [[project-template-strategy]],
[[feedback-ansible-check-mode-verify]].
