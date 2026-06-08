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
- Voor langere cert-validity (bv. 7d, 30d): pas step-ca `claims.maxTLSCertDuration` aan in `/etc/step-ca/config/ca.json` — vereist step-ca restart.

Gerelateerd: [[project-internal-pki-coverage]] (step-ca coverage strategie), [[project-architecture-haproxy]] (Caddy + step-ca pattern op Odoo-VMs).
