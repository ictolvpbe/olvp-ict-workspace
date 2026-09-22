---
name: feedback-unifi-override-inform-two-fields
description: "UniFi classic controller: Override Inform Host = 2 aparte settings (super_mgmt.override_inform_host + super_identity.hostname) — hostname EERST zetten, anders pusht de provision-golf de oude/korte naam naar de hele vloot."
metadata:
  type: feedback
---

Op de klassieke UniFi Network-controller (10.0.162) bestaat "Override Inform Host" uit **twee aparte settings-records**:

- `super_mgmt.override_inform_host` (boolean) — de aan/uit-knop; een `inform_host`-veld in super_mgmt bestaat hier NIET (blijft null).
- `super_identity.hostname` — de eigenlijke hostnaam die gepusht wordt ("Controller Hostname/IP" in de UI).

**Why:** bij het aanzetten via API op 2026-09-22 stond `super_identity.hostname` nog op de korte naam `unifi`. Het enkel omzetten van de boolean triggerde meteen een provision-golf over álle 236 devices met `http://unifi:8080/inform` — devices zonder DNS-suffix in hun fixed-config strandden op "Unable to resolve" en werden controller-loos (herstel = per-device `set-inform` via SSH, zie [[project-unifi-os-server-migration]]).

**How to apply:** volgorde is heilig — **eerst** `set/setting/super_identity {"hostname":"<fqdn>"}`, **dán** `set/setting/super_mgmt {"override_inform_host":true}`. Verifieer beide via `get/setting`. En check vooraf dat élk device de FQDN kan resolven (verse sweep) én dat de hostname een FQDN is (kale namen breken devices zonder search-domain). Gerelateerd: [[feedback-unifi-set-inform-not-persistent]], [[feedback-unifi-device-noninteractive-cli]].
