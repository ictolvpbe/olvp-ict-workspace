---
name: project-template-fleet-defects
description: "Twee fouten in het golden image die op alle 12 klonen zitten: /etc/machine-id niet leeggemaakt (12 VM's delen dezelfde id) en een partitie-indeling met /var 2,9 GB tegenover /home 27,5 GB. Tracker TPL-1 en TPL-2."
metadata:
  type: project
---

Vastgesteld 2026-09-17 tijdens de Teamdrive-backup-migratie ([[project-teamdrive-backup-outage-202609]]). Beide fouten komen uit het golden image van [[project-template-strategy]] en zijn dus twaalf keer uitgerold.

## TPL-1 — `/var` te klein, `/home` te groot

Het template geeft `/home` **27,5 GB** (werkelijk in gebruik: 128 KB) en `/var` **2,9 GB**, terwijl `/var` de logs, container-images en databases draagt. Op een server hoort die verhouding omgekeerd.

Concreet gemeten op `SRVV-P-BACKUP-01`: VG 49,52 GB met **9,91 GB ongebruikt**, `/var` op 79% met nog een volledige backup-run te gaan. Op `SRVV-CLOUDBACKUP-01` liep `/var` in juni-september volledig vol, wat de backup mee om zeep hielp.

**Per host online op te lossen** — `/var` is ext4 en er staat ruimte vrij in de VG, dus geen herstart nodig:

    sudo lvextend -r -L +8G /dev/SRVV-DEBIAN-TEMPL-vg/var

`-r` groeit meteen het bestandssysteem mee. Krimpen van `/home` is niet nodig zolang de VG vrije ruimte heeft.

## TPL-2 — dezelfde machine-id op 12 VM's

`/etc/machine-id` is niet leeggemaakt vóór het wegschrijven van het template, dus elke kloon erft `24a516d46f714444a68e97843c763d1a`. Gemeten op haproxy-1/2, tst-odoo-01, dev-odoo-01, srvv-odoo-01, srvv-acc-01, forgejo-01, id-01, monitoring-01, unifi-01, sspr-01 en p-backup-01.

Impact nu beperkt — de VM's hebben statische adressen (geen DHCP-DUID-botsing) en NetXMS gebruikt een eigen agent-id — maar het raakt journal- en loglabeling en software die er installatie-id's uit afleidt. **Eerst nakijken of Alloy/Loki op machine-id labelt** ([[project-netxms-monitoring]]); dat bepaalt de urgentie.

Saneren vraagt per VM een herstart. `SRVV-ODOO-01` en `SRVV-ACC-01` zijn productie en krijgen een eigen venster. **Let op**: `/var/lib/dbus/machine-id` is op deze VM's een echt bestand, geen symlink, en moet mee.

Gedaan op `SRVV-P-BACKUP-01` en `SRVV-CLOUDBACKUP-01`.

## How to apply

De sanering per host is dweilen: zolang het template ongewijzigd blijft, erft elke nieuwe VM beide fouten. Repareer eerst het template — `/etc/machine-id` leegmaken (niet verwijderen; systemd vult hem bij de eerste boot) en de partitie-indeling omkeren — en saneer daarna de bestaande hosts. Zie [[project-srvv-p-backup-01]].
