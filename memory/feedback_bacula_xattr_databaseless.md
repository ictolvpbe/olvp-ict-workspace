---
name: feedback-bacula-xattr-databaseless
description: "Bacula neemt extended attributes NIET standaard mee. Bij database-loze stores (OpenCloud/OCIS) zit alle metadata in xattrs → zonder `xattrsupport = yes` in de FileSet draait de backup groen en restaureer je bestanden zonder shares, versies of rechten."
metadata:
  type: feedback
---

**Vastgesteld 2026-09-18** bij het ontwerp van de Euro-Office/OpenCloud-testserver
([[project-eurooffice-nextcloud]]).

Bacula's FileSet-optie **`xattrsupport = yes` staat niet standaard aan**. Voor de meeste
applicaties maakt dat niet uit. Voor een store **zonder database** is het fataal: daar zit de
metadata — shares, groeps-/Spaces-lidmaatschap, versies, tree-info — in de **extended attributes
van elk bestand**. Zonder die optie:

1. draait de backup **groen**,
2. restaureer je de **bytes** correct,
3. en krijg je bestanden terug **zonder enige metadata**.

Je ontdekt het pas op het moment dat je de restore nodig hebt. Zet `aclsupport = yes` erbij;
Bacula ontdubbelt dat zelf zodat ACL's niet twee keer opgeslagen worden.

```
Options {
  signature    = MD5
  aclsupport   = yes
  xattrsupport = yes
}
```

**Bijkomend:** past de metadata niet in de xattrs, dan schrijft de applicatie een
**sidecar-bestand** naast het origineel. Die moeten mee in dezelfde FileSet — dus geen
include-filter op documenttypes.

**How to apply:** bij élke Bacula-FileSet voor een database-loze of xattr-afhankelijke store
(OpenCloud/OCIS, maar denk ook aan Samba-shares met NT-ACL's) expliciet `xattrsupport = yes` zetten.
En sluit de restore-test **altijd af met een controle op de deelrechten**, niet enkel op de vraag of
de bestanden er staan — anders heb je de instelling niet bewezen, enkel de bytes.

Past in het patroon van [[feedback-ansible-check-mode-verify]] (vertrouw geen groen scherm) en
[[project-teamdrive-backup-outage-202609]] (een backup die stil niets deed, 87 dagen lang).
Gerelateerd: [[project-srvv-p-backup-01]], [[project-offline-backup-chain]].
