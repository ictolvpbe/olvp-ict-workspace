---
name: project-vpn-mfa
description: "Admin-VPN met MFA op de UniFi EFG — hard gate Fase 1b. Startanalyse 2026-09-21: huidige stand (OpenVPN zonder MFA), drie routes (UniFi Identity Enterprise / eigen RADIUS / VPN van de gateway af), quick wins en de drie feiten die nog uit de controller moeten komen."
metadata:
  type: project
---

Werf gestart **2026-09-21**: de admin-VPN afschermen met MFA. Is volgens `platform-handbook/network-physical/network/vpn.md` een **hard gate vóór publieke go-live** (Fase 1b, zie [[project-phasing]]).

**Why:** vanaf go-live moeten ICT-admins remote kunnen ingrijpen; de VPN is het enige pad naar jump-01 en de beheer-UI's. Vandaag staat dat pad op enkel een wachtwoord.

## Huidige stand (uit vpn.md, niet opnieuw gemeten)
- **UniFi built-in OpenVPN**, pool `192.168.3.0/24` = host-list `VPN-ADMIN-CLIENTS`, firewall-rules **W-002** (→ management) en **A-011** (→ acc-instance `10.36.0.42:443`).
- Split-DNS beslist 2026-06-10: push `10.33.0.10` + `10.10.0.10` + search-domain `olvp.int`; géén publieke resolver ernaast. Rule **D-001** is voorwaarde voor de `.33`-helft.
- **Geen MFA.** De EFG is in dienst sinds 2026-08-28 ([[project-gateway-cutover]]), dus de voorwaarde "na de Fortress-upgrade" uit vpn.md is nu vervuld.

## Nog te meten in de controller (bepaalt de keuze)
1. Welke **VPN Server-types** biedt deze Network-versie op de EFG (WireGuard / OpenVPN / L2TP / Identity Endpoint)? Nieuwere versies duwen OpenVPN weg richting WireGuard — beslissend, want **WireGuard kan geen RADIUS/MFA** (puur sleutel-gebaseerd).
2. Staat **RADIUS** als auth-bron in het VPN-server-profiel (Settings → VPN → server → Authentication)?
3. Is **UniFi Identity Enterprise (UID)** beschikbaar/licentieerbaar op dit account? Enige native MFA in de UniFi-stack, per-gebruiker-abonnement.

## Drie routes (analyse 2026-09-21)
| | A — UniFi Identity Enterprise | B — UniFi VPN + eigen RADIUS | C — VPN van de gateway af |
|---|---|---|---|
| Endpoint | EFG (Identity/WireGuard-client) | EFG (OpenVPN of L2TP) | eigen VM, EFG doet enkel UDP-forward |
| MFA | native push/TOTP | TOTP als `wachtwoord+code` in één veld | echte OIDC-flow (TOTP/WebAuthn) |
| Identity-bron | UID-directory (AD-sync te verifiëren) | AD via FreeRADIUS of privacyIDEA | Keycloak → AD (bestaande LDAPS-federatie) |
| Extra infra | geen | 1 VM (past in FreeRADIUS-doel, Tier 3 van [[project-client-identity]]) | 1 VM (Defguard / Firezone / NetBird) |
| Kost | abonnement per gebruiker | tijd | tijd |
| Risico | **auth loopt via Ubiquiti-cloud** | klungelige UX, L2TP gedateerd | Keycloak is nog PoC-grade ([[feedback-keycloak-ldaps-truststore]]) |

**Aanbeveling:** is (3) ja en blijft het aantal admin-accounts klein (2-3), dan **A** — snelst, geen nieuw onderhoud, precies waarvoor de Fortress-upgrade in vpn.md genoemd staat. Harde voorwaarde: een **break-glass-pad buiten de cloud om** (tweede WireGuard-peer, sleutel-only, in KeePassXC + op papier in de kluis), anders hangt remote ingrijpen bij een incident af van een externe dienst die mee uit kan liggen. Is (3) nee of te duur: **C** boven B — C is meteen de lange-termijnkeuze uit vpn.md ("WireGuard + Firezone of Defguard met OIDC"), B geeft MFA met de slechtste UX op een verouderd protocol.

## Quick wins, losstaand van die keuze
- **MFA op jump-01 zelf** (`libpam-google-authenticator` via Ansible-rol): VPN levert bereikbaarheid, jump-host levert de tweede factor → geen pad naar de stack zonder MFA zolang het VPN-spoor loopt.
- **Per-persoon VPN-accounts**, geen gedeeld admin-account; wachtwoorden uit KeePassXC ([[project-secrets-management]]).
- **W-002 aanscherpen**: VPN-pool → alleen jump-01 (TCP/22) + expliciet toegestane beheer-UI's i.p.v. het hele management-VLAN.
- **VPN-sessies naar NetXMS/Loki** met alarm op elke login ([[project-netxms-monitoring]]) — nu ziet niemand wanneer er ingelogd wordt.
- Testen vanaf een **écht extern net**, niet vanaf VLAN 34 ([[feedback-test-from-user-vlan]]).

## Vast te leggen bij de beslissing
ADR voor de VPN-MFA-keuze (naast ADR 0006), update van `vpn.md` (sluit de open vragen Fortress-timing / MFA-app standaardiseren / backup-codes / always-on) en een tracker-item onder Fase 1b, want het is een hard gate.

**How to apply:** bij hervatten eerst de drie meetpunten uit de controller halen, dán pas de route kiezen. De quick wins kunnen ondertussen door.
