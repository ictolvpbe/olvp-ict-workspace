---
name: feedback-unifi-set-inform-not-persistent
description: "Per-device `mca-cli-op set-inform` op UniFi is NIET duurzaam: bij elke controller-provision herpusht de controller de inform-host uit super_mgmt; alleen Override Inform Host houdt stand."
metadata:
  type: feedback
---

Per-device **`mca-cli-op set-inform http://host:8080/inform`** op UniFi-devices (AP/switch) verandert de inform-host enkel **tot de volgende controller-*provision***. Bij provision (config-push, reboot, adoptie-refresh, firmware) herpusht de controller de inform-host uit zijn systeeminstelling **`super_mgmt.override_inform_host`** en **overschrijft de per-device-flip**.

- Staat `override_inform_host = False` (default), dan pusht de controller zijn eigen verbindings-IP terug → devices vallen geleidelijk terug naar het hard-IP-inform.
- Empirisch op OLVP (2026-06-29): estate dreef 197→194 FQDN ondanks +12 nieuwe flips; eerder-geflipte switches stonden device-side weer op hard IP. `super_mgmt.override_inform_host=False` bevestigd via Network-API (`/api/s/default/get/setting`).

**Why:** een per-device set-inform-migratie naar een FQDN/nieuwe controller lekt over tijd — onbetrouwbaar voor een fleet over een niet-triviaal venster.
**How to apply:** duurzame inform-host = controller-side **Override Inform Host** (`super_mgmt.override_inform_host=True` + host-FQDN). Per-device set-inform enkel voor adoptie/eenmalige nood, of vlak vóór een onmiddellijke cut-over. PREREQ: de FQDN moet voor *alle* devices naar de juiste controller resolven (geen gateway-DNS-kaping) vóór je de globale override aanzet. Zie [[project-unifi-os-server-migration]] + [[feedback-unifi-device-noninteractive-cli]].
