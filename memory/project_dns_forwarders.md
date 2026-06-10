---
name: project-dns-forwarders
description: "Externe DNS-forwarder-beslissing (2026-06-10): waar mogelijk DNS4EU (joindns4.eu) als upstream. Standaard = Protective (86.54.11.1); basisscholen/leerlingen = Protective + Child Protection (86.54.11.12). Implementeren voor schooljaar 2026-2027."
metadata:
  type: project
---

**Beslissing 2026-06-10:** voor **externe** naamresolutie (de upstream waar onze interne resolvers naar forwarden) gebruiken we **zoveel mogelijk en waar het kan** de publieke DNS van **DNS4EU** (EU-gefinancierd, https://joindns4.eu/for-public). Vervangt/standaardiseert de huidige forwarders (bv. ISP-/Google-/Cloudflare-DNS).

**Standaardkeuzes:**
- **Standaard (staf/algemeen): Protective** — `86.54.11.1` (malware/phishing-blocking, geen content-filter).
- **Basisscholen / leerlingen: Protective + Child Protection** — `86.54.11.12`.

**DNS4EU "for public" varianten (IPv4, ter referentie):**

| Optie | IPv4 |
|---|---|
| Protective | `86.54.11.1` |
| Protective + Child Protection | `86.54.11.12` |
| Protective + Ad Blocking | `86.54.11.13` |
| Protective + Child Protection + Ad Blocking | `86.54.11.11` |
| Unfiltered | `86.54.11.100` |

> Vul vóór uitrol de **secundaire + IPv6-endpoints** aan vanaf joindns4.eu (redundantie); DoH/DoT-endpoints bestaan ook. Ad-Blocking-varianten blijven een optie maar zijn nu niet gekozen.

**Why:** EU-resolver = past bij onze GDPR-/data-minimalisatie-houding (geen US-cloud-DNS), met ingebouwde security-filtering zonder eigen filterinfra. Child Protection geeft de basisscholen DNS-niveau-bescherming bovenop [[project-client-identity]] (NXfilter).

**How to apply (architectuur — design-punt, te bevestigen bij uitrol):** een Windows-AD-DC-forwarder is **globaal per DC** → kan staf vs leerling niet differentiëren. Differentiatie dus per client-segment:
- **Staf/algemeen/domain-clients** → interne AD-DC's (`SRVV-SERV-01` 10.33.0.10 e.a., zie [[project-ad-serv01-migration]]) forwarden naar **Protective `86.54.11.1`**.
- **Leerling-VLAN's (basisscholen)** → via **NXfilter** (zit al op alle student-VLANs, [[project-client-identity]]) upstream naar **Child Protection `86.54.11.12`**; of DHCP-pushed DNS4EU rechtstreeks waar geen interne namen nodig zijn.
- Firewall: forwarders (DC's + NXfilter-host) → `86.54.11.0/24:53` (UDP+TCP) egress openen.

**Timing:** opnemen/implementeren voor **schooljaar 2026-2027** (start ~sep 2026). Tracker: **NET-5**. Doc: `platform-handbook/network-physical/network/dns.md`. Gerelateerd: [[project-client-identity]] (NXfilter), [[project-ad-serv01-migration]] (interne DC's/forwarders), [[reference-dns-olvpbe]] (publieke olvp.be-zone — los hiervan).
