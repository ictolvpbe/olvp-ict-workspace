---
name: feedback-step-ca-renew-no-password
description: "`step ca renew` gebruikt het bestaande cert als auth — geen provisioner password nodig zolang cert nog geldig is. Hiermee is volledige automation van cert-renewal mogelijk zonder secrets-management."
metadata:
  type: feedback
---

**Regel:** voor periodieke cert-renewal op een host die al een geldig step-ca-cert heeft, gebruik `step ca renew` — **geen provisioner-password vereist**. Het bestaande cert wordt door step-ca als auth geaccepteerd. Initial cert-uitgifte vereist wel een provisioner (admin JWK met password, of ACME, etc.).

**Why:** 2026-06-01 — bij opzet van cron-renewal voor de Caddy-cert op TST-ODOO-02 (24h step-ca cert) dachten we eerst dat we de provisioner-password op de VM zouden moeten plaatsen (KeePassXC entry, secret-tool, of file). Bleek niet nodig: `step ca renew --force --expires-in 12h CERT KEY` werkt volledig non-interactief zolang het cert nog niet expired is. Onze setup:
- systemd-timer 3× per dag (00, 08, 16)
- script: `step ca renew --expires-in 12h --exec "systemctl restart caddy.service" CERT KEY`
- `--exec` triggert ALLEEN bij daadwerkelijke renewal (geen reload bij no-op)
- Verloren cert-window (>24h offline) = handmatige `step ca certificate` met admin provisioner

**How to apply:**
- Voor elke step-ca-cert op een host: schrijf een renewal-script + systemd-timer met `step ca renew --expires-in <kortere-dan-cert-lifetime>h --exec "<reload-cmd>"`. Voor 24h certs: timer 3× per dag, `--expires-in 12h`.
- Initial cert (eenmalig per FQDN/host) blijft met password. Daarna automation hands-off.
- Bij geplande VM-onderbreking >24h: trigger manueel handmatig vóór return, of accept dat next-renewal opnieuw initial-cert-flow nodig heeft (password uit KeePassXC).
- ~~Voor 24h certs: `--expires-in 12h`~~ → **sinds 2026-09-15: certs 168h, renewal `--expires-in 120h`** (5 dagen tolerantie voor step-ca-uitval). Aanleiding: [[project-stepca-cert-incident-202609]].
- **`step ca renew` behoudt de duur van het bestaande cert** (geen `--not-after`-flag). Bewezen 2026-09-15: na de CA-wijziging kwamen vernieuwde certs nog steeds 24h terug → een levensduurwijziging vraagt één nieuwe uitgifte per cert.
- Claims zetten op de CA (per provisioner, niet via ca.json-handwerk): `sudo STEPPATH=/etc/step-ca step ca provisioner update admin --x509-max-dur 168h --x509-default-dur 168h`, dan `systemctl restart step-ca`. Zonder `STEPPATH` zoekt sudo in `/root/.step` en faalt met "requires the '--ca-url' flag". Verifieer via `curl https://stepca.olvp.int:8443/provisioners` (claims per provisioner), niet alleen op het bestand.
- **Verlopen cert = geen auth meer** → opnieuw uitgeven met de `admin`-provisioner, en daarna **caddy herstarten**: de certs zijn losse file-bind-mounts in de container, een reload ziet het nieuwe bestand niet (prod bleef 503 tot de restart).

Gerelateerd: [[project-internal-pki-coverage]] (step-ca coverage strategie), [[project-architecture-haproxy]] (Caddy + step-ca pattern op Odoo-VMs).
