---
name: project-architecture-haproxy
description: "Architectuurkeuzes voor [[project-odoo-public-access]]. HAProxy HA-paar in DMZ, TLS re-encrypt naar Caddy, step-ca interne CA, Cloudflare Free + $5 LB (fase 2), identity-laag met AD als SoT (zie [[project-identity-architecture]])."
metadata: 
  node_type: memory
  type: project
  originSessionId: 59a419f3-43f1-477b-bb2b-196d0c16926a
---

Vastgelegde architectuur voor [[project-odoo-public-access]] (beslist 2026-05-18):

**Edge / DNS (fase 2):**
- DNS-migratie one.com → Cloudflare (geplande zomervakantie 2026, zie [[reference-dns-olvpbe]]).
- Cloudflare Free plan + Load Balancer add-on (~$5/maand): proxy aan (oranje wolk) voor Odoo-records, grijze wolk voor mail/andere subdomeinen.
- Health-checked failover tussen beide ISP-IPs via Cloudflare LB origin pool (~30-60s failover-tijd).

**Reverse proxy laag:**
- HAProxy HA-paar in DMZ-reverse-proxy-VLAN: **10.21.0.0/24, VLAN-tag 21** (deel van DMZ-supernet 10.21.0.0/16; tag-conventie `21 + 3e-octet van /24` — zie [[project-infrastructure-params]]). VLAN-tag-historie: 21/22 → 210/211 (2026-05-21) → 21/22 (2026-05-31).
- IP-toekenning: haproxy-1 = 10.21.0.6, haproxy-2 = 10.21.0.7, VIP = 10.21.0.5 (VRRP via keepalived, actief/passief).
- TLS terminatie op HAProxy + re-encryptie naar Caddy (end-to-end TLS, terminate + re-encrypt model).
- Publieke certs via Let's Encrypt: HTTP-01 in fase 1, DNS-01 (Cloudflare API) met wildcards in fase 2.
- Config-management via Ansible (git repo, parametrische haproxy.cfg uit instances-lijst).

**Backend (Odoo-servers):**
- Caddy blijft staan op Odoo-servers, krijgt intern cert van step-ca.
- Odoo `proxy_mode = True` zodat X-Forwarded-For correct gelezen wordt door applicatie.

**Interne CA:**
- `SRVV-STEPCA-01` op **10.21.1.10** (VLAN 22 `VL-SD-DMZ-CA`), **native Debian-install** (geen container — beslist 2026-05-28, bewuste uitzondering op ADR 0004 wegens long-lived stateful CA-rol; back-up = `/etc/step-ca/`).
- Tweelagige PKI: root + intermediate. Root-key na init **offline** op LUKS-USB + password in KeePassXC; intermediate-key online voor signing.
- ACME-provisioner `acme` voor Caddy automatic enrollment + renewal. JWK-provisioner `admin` voor manuele admin-acties.

**Netwerksegmentatie:**
- DMZ → test-VLAN / prod-VLAN: alleen TCP 443 naar specifieke Caddy-backend-IPs (allowlist per host).
- DMZ → management: dicht. Odoo-VLAN's → DMZ: dicht (Caddy initieert nooit naar HAProxy).
- test-VLAN ↔ prod-VLAN onderling: dicht.

**Naamgeving:**
- Prod: `<naam>.olvp.be` (vb. myschool.olvp.be)
- Test: `<naam>-test.olvp.be` (vb. myschool-test.olvp.be)

**Identity-laag (zie [[project-identity-architecture]] voor detail):**
- Aparte IdP-instance (`SRVV-ID-01`, FQDN `id.olvp.be`, expose `public-restricted` met path-allowlist) als auth-broker tussen apps en AD; OAuth2-tokens uitgeven.
- Aparte account-admin-instance (`SRVV-ACC-01`, FQDN `myschool-acc.olvp.be`, expose `internal`) — staat **niet** in HAProxy/DMZ, gebruikt direct LDAP naar AD voor zijn eigen login (resilience tegen IdP-uitval).
- App-instances (myschool, myschool-ict, myschool-test, myschool-dev) zijn OAuth-clients van de IdP, met JIT user-provisioning. Alleen minimale token-claims (`sub`, `email`, `name`, `groups`).
- AD = single source of truth voor credentials. IdP en acc binden **onafhankelijk** naar AD.

**HAProxy-routering per expose-klasse:**
- `public` → volledige routering naar Caddy:443 backend op de VM.
- `public-restricted` → path-allowlist (alleen auth-paden); rest = 403.
- `internal` → **niet in HAProxy-config**, alleen interne DNS-resolver verwijst ernaar.

**Niet voor fase 1/2:**
- WAF (Coraza op HAProxy, of Cloudflare Pro Managed Rules) — overweging na fase 2 afgerond.

**How to apply:** Verwijs naar deze keuzes bij ieder voorstel rond reverse proxy, certificaten, netwerkstructuur, DNS of identity in dit project. Wijk er enkel van af na expliciete bespreking. Cloudflare-config kan pas opgezet worden in fase 2.
