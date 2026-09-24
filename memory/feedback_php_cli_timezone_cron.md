---
name: feedback-php-cli-timezone-cron
description: "Taak draait op het verkeerde uur? Vergelijk date('H') van PHP-CLI met timedatectl — DB-timestamps kunnen correct zijn terwijl de CLI in UTC rekent."
metadata:
  type: feedback
---

Bij een geplande taak die **wel draait maar op het verkeerde moment**: meet het uur zoals de
**uitvoerende interpreter** het ziet, niet zoals de logs het tonen.

```
php -r 'echo ini_get("date.timezone"), " ", date("H:i"), "\n";'
timedatectl
```

**Why**: op SRVV-GLPI-01 haalde GLPI's mailgate pas vanaf 09:00 mail op terwijl het venster op
07:00–19:00 stond. `date.timezone` was niet gezet in de **CLI**-php.ini, dus PHP-CLI viel terug
op UTC terwijl de host op `Europe/Brussels` stond — twee uur verschil, precies het waargenomen
gat. Debian levert php.ini met `;date.timezone =` uitgecommentarieerd; de CLI-variant wordt bij
handmatige installaties vaak vergeten terwijl de Apache-variant wél ingevuld raakt.

De valstrik: **MariaDB stond op `time_zone=SYSTEM`** en schreef dus correcte lokale tijdstempels
weg. In `glpi_crontasklogs` en in `cron.log` zag alles er daardoor gezond uit — de runs stonden
netjes op 09:00 lokale tijd. Niets in de logs wees naar de tijdzone; het patroon werd pas
zichtbaar door de runs **per uur te tellen** en de grenzen te vergelijken met de ingestelde
`hourmin`/`hourmax`.

**How to apply**:
- Tel bij "draait op het verkeerde moment" eerst de runs per uur en leg de waargenomen grenzen
  naast de geconfigureerde. Een constante verschuiving van een heel aantal uren = tijdzone.
- Controleer dat **web en cron dezelfde PHP-versie én dezelfde tijdzone** gebruiken. Op deze host
  draaide Apache op 8.2 en de cron op 8.4 — dat maakte het verschil onzichtbaar, want manueel
  starten via de webinterface wérkte altijd.
- Zet `date.timezone` expliciet in **elke** php.ini die in gebruik is, en neem dat op in de
  baseline-role in plaats van het per host te repareren.
- Algemener: dit is dezelfde les als [[feedback-verify-memory-against-repo]] — de hardere laag
  (de draaiende interpreter) wint van de zachtere (het logbestand).
