---
name: feedback-unifi-device-noninteractive-cli
description: "UniFi AP/switch scripten via SSH: info/set-inform zijn interactieve aliassen → gebruik mca-cli-op; kale mca-cli-op loopt oneindig; ssh -n verplicht in read-loop."
metadata:
  type: feedback
---

Bij het **non-interactief scripten** tegen UniFi-devices (UAP/USW, BusyBox `ash`) via SSH:

- `info` en `set-inform` werken enkel **interactief** (shell-aliassen uit het login-profiel). In `ssh host 'info'` → `ash: info: not found`. Gebruik de echte CLI: **`mca-cli-op info`** (status + inform-URL) en **`mca-cli-op set-inform http://host:8080/inform`**.
- ⚠️ **Káál `mca-cli-op` (zonder subcommando) dropt naar een interactieve prompt** en leest met `-n`/EOF oneindig door → genereerde **154 MB** "UniFi# "-output. **Altijd mét subcommando** + `timeout` + `| head -N` als vangnet.
- **`ssh` in een `while read`-loop slurpt de loop-stdin** (de inputfile) op → loop stopt na 1 iteratie. Fix: **`ssh -n`** (stdin uit /dev/null).
- **Password-automation zonder sshpass**: `SSH_ASKPASS=<script-dat-pw-echoot> SSH_ASKPASS_REQUIRE=force setsid -w ssh -n ...`. Device-SSH-user op de OLVP-estate = **`zeus`** (zelfde creds als de Network-controller).
- Status-regel parsen: `Status:  Connected (http://<host>:8080/inform)` → resolutie + connectie in één.

**Why:** zonder dit faalt elke scripted device-actie stil (not found / lege output / 1 device) of ontspoort (154MB).
**How to apply:** zie het werkende script `platform-handbook/hosting/operations/unifi-fase0b-set-inform.sh`. Gerelateerd: [[project-unifi-os-server-migration]].
