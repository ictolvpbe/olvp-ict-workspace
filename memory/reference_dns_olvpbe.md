---
name: reference-dns-olvpbe
description: "DNS-zones: extern olvp.be (registratie one.com, hosting migreert naar Cloudflare zomer 2026), intern olvp.int (interne resolver, aparte zone — niet split-DNS). Volledige record-inventaris nog te maken vóór externe migratie."
metadata: 
  node_type: memory
  type: reference
  originSessionId: 59a419f3-43f1-477b-bb2b-196d0c16926a
---

OLVP gebruikt **twee aparte DNS-zones**:

## Extern: `olvp.be`

- **Registrar (WHOIS-houder):** one.com — blijft daar, niet wijzigen.
- **DNS-hosting nu:** one.com (basic DNS, geen API, geen health checks).
- **DNS-hosting toekomstig:** Cloudflare — geplande migratie zomervakantie 2026, zie [[project-phasing]] fase 2.
- **Inhoud:** alleen records die publiek bereikbaar moeten zijn (mail, publieke Odoo-FQDN's, etc.).

Bekende subdomeinen / records:
- `baple.olvp.be` — niet-Odoo service, exacte functie nog te documenteren (relevant voor grijs/oranje-wolk-keuze).
- Geplande Odoo-FQDN's na fase 1: `<naam>.olvp.be` (prod) en `<naam>-test.olvp.be` (test).
- Mail-records (MX → Google Workspace, SPF, DKIM-selectoren, DMARC) — volledige inventaris nog op te halen uit one.com vóór fase 2.

Migratie-aandachtspunten:
- Cloudflare's auto-import is niet 100% betrouwbaar voor lange DKIM TXT-waarden of SRV-records — manuele verificatie nodig.
- DNSSEC-status nog te checken (`dig olvp.be DNSKEY +short` of dnsviz.net) — indien actief, eerst uitschakelen bij one.com, wachten op TTL, dan migreren, dan Cloudflare-DNSSEC opnieuw aanzetten.
- TTL alle records 24u vooraf op 300s zetten.
- API-token in Cloudflare aanmaken met scope `Zone:DNS:Edit` op zone `olvp.be` voor ACME DNS-01 challenge.

## Intern productie: `olvp.int`

- **Hosting:** interne resolver (AD-DC of dedicated — te documenteren). Niet publiek gepubliceerd.
- **Inhoud:** alle hostnames van interne productie-services die niet publiek bereikbaar mogen zijn. Voorbeelden: `semaphore.olvp.int`, `myschool-acc.olvp.int` (account-admin Odoo), beheers-appliances, jump-01, etc.
- **Geen split-DNS** — totaal aparte zone van `olvp.be`. Geen records met dezelfde naam in beide zones.

## Intern test: `olvp.test`

- **Hosting:** interne resolver, test-VLANs (200-209).
- **Inhoud:** test-equivalenten van productie-services. Bv. test-Odoo-VMs (`SRVV-TST-ODOO-01`) hebben hier hun records.
- **Volledig gescheiden** van `olvp.int` — geen overlap, geen split-DNS tussen beide.

**FYI over `.int` als TLD:** `.int` is door IANA gereserveerd voor intergouvernementele organisaties (NATO, ITU, EU-instellingen). In de praktijk werkt het ongestoord voor intern gebruik omdat publieke `.int`-records uitsluitend bij die internationale organisaties zitten. Sinds 2024 heeft ICANN officieel `.internal` gereserveerd voor private use, wat de "RFC-conforme" keuze zou zijn voor nieuwe deployments. OLVP houdt voorlopig `.olvp.int` — wijziging is een toekomstige doc-werf met breed impact (AD-config, GPO, certs, scripts). Geen acute actie.

**How to apply:**
- Nieuwe interne **productie**-service → FQDN `<naam>.olvp.int`, record toevoegen aan interne resolver.
- Nieuwe interne **test**-service → FQDN `<naam>.olvp.test`, record toevoegen aan interne resolver.
- Nieuwe publieke service → FQDN `<naam>.olvp.be`, record toevoegen aan one.com (Fase 1) / Cloudflare (Fase 2).
- **Voor host-aliassen binnen olvp.int**: gebruik **dubbele A-records** (canonical + alias, beide met hetzelfde IP). Geen CNAMEs — UniFi-DNS-resolver serveert die niet betrouwbaar (NXDOMAIN bij CNAME-lookup binnen eigen zone). Bij IP-wijziging: alle aliassen mee-updaten. Zie [[project-naming-convention]] voor details.
- Cert-strategie: interne `.olvp.int`-services krijgen step-ca-cert (intern getekend). Publieke `.olvp.be`-services krijgen LE-cert via HTTP-01 (Fase 1) of DNS-01 op Cloudflare (Fase 2). Zie [[project-architecture-haproxy]].
- Onthoud: `expose: internal` Odoo-instances (zoals `myschool-acc`) staan in `.olvp.int`, niet in `.olvp.be`. Bij `[[project-infrastructure-params]]`-tabel zijn die als "internal" gemarkeerd — interpreteer als `.olvp.int`-FQDN.
