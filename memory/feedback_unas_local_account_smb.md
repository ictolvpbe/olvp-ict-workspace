---
name: feedback-unas-local-account-smb
description: "UniFi UNAS Pro: een gebruiker aangemaakt met de optie 'local account' krijgt geen SMB-toegang. Mount faalt met mount error(13) Permission denied terwijl account en wachtwoord kloppen. Opnieuw aanmaken zonder die optie lost het op."
metadata:
  type: feedback
---

Op de **UniFi UNAS Pro** breekt de optie **"local account"** bij het aanmaken van een gebruiker de SMB-toegang, ongeacht welke rol (editor/owner) je toekent.

**Symptoom**: `mount error(13): Permission denied` bij `mount -t cifs`, terwijl gebruikersnaam en wachtwoord aantoonbaar kloppen en een ánder account (bv. een bestaande admin) op dezelfde NAS wél mount. `smbclient -L` faalt eveneens.

**Oplossing**: gebruiker verwijderen en opnieuw aanmaken **zonder** "local account" aan te vinken. Daarna werkt SMB meteen.

**Why**: vastgesteld 2026-09-17 bij het aanmaken van `svc-clbu-rw` voor de Teamdrive-backup. Het kostte een uur omdat `error(13)` de vertaling is van meerdere NT-statuscodes en dus niet vertelt of het aan het wachtwoord, de share-rechten of het accounttype ligt. De foutmelding wijst je richting credentials, terwijl het probleem in het accounttype zit.

**How to apply**: bij `mount error(13)` op de UNAS Pro met credentials waarvan je zeker bent — controleer eerst het **accounttype** in de UI, vóór je wachtwoorden gaat roteren of mount-opties uitkamt. En test authenticatie los van mounten met `smbclient -L //<nas> -U <user>`: die geeft een leesbare NT-statuscode (`LOGON_FAILURE` vs `ACCESS_DENIED`) in plaats van de dichtgetimmerde `error(13)`. Zie [[project-teamdrive-backup-outage-202609]] en [[project-srvv-p-backup-01]].
