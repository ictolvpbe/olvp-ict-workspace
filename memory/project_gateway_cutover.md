---
name: project-gateway-cutover
description: "Nieuwe UniFi-gateway parallel opgezet (juni 2026); doel = volledige cutover (re-IP alles) later. NU draait netwerk via OUDE gw (WAN+DNAT op .82) en dat moet zo blijven. Incident 2026-06-16: DHCP op nieuwe gw + .2-IP-conflict met jump-01 brak externe bereikbaarheid; opgelost (nieuwe gw VLAN35-interface -> 10.35.0.3)."
metadata:
  type: project
---

**Status 2026-06-17.** Een **nieuwe UniFi-gateway** staat **parallel** naast de oude (opgezet 2026-06-15/16). Intentie = **volledige cutover** (nieuwe vervangt oude) = alle interne IP's + externe adressen omzetten — gepland als aparte werf/window, **nog niet uitgevoerd**.

## Huidige (tijdelijke) toestand — niet wijzigen tot cutover
- **WAN-uplink + DNAT zitten op de OUDE gw** (publiek IP `84.199.147.82`). Netwerk loopt via de oude gw en **moet zo blijven**.
- Nieuwe gw heeft **andere externe adressen** (`84.199.147.84`-blok) en gebruikt per VLAN een `.2`-interface; op **VLAN 35 botste `.2` met jump-01 (10.35.0.2)** -> verplaatst naar **`10.35.0.3`** (transitie-IP; wordt `.1` bij cutover). Op VLAN 21/34/36 is `.2` vrij, dus enkel VLAN 35 aangepast.
- DHCP op de nieuwe gw is **uitgezet** (zie incident hieronder).

## Incident 2026-06-16 (opgelost) — root cause om te onthouden
Publieke Odoo-FQDN's (o.a. **myschool-test**) waren **extern onbereikbaar** hoewel HAProxy/Odoo/DNS gezond waren. Oorzaak: de **DHCP-server op de nieuwe gw deelde zichzelf (`.2`) als default-gateway** uit aan clients, terwijl WAN+DNAT op de oude gw (`.1`) zitten -> **asymmetrische routing** (inbound via oud, antwoord via nieuw zonder NAT-state) -> handshake faalt. **Fix**: DHCP op nieuwe gw uit + clients ge-renewd (terug via oude gw) + nieuwe-gw VLAN35-interface -> `.3`. Daarna alles groen geverifieerd (myschool-test publiek `http=200`, VIP `10.21.0.5` ok, jump-01 ongestoord).
Bewijs van het `.2`-conflict: SSH-hostkey op `10.35.0.2` wisselde tijdens de storing naar een vreemd toestel (de gw-interface) en is na de fix weer de echte jump-01-key.

Vastgelegd in repo: `platform-handbook/network-physical/reference/ip-plan.md` (commit `a326d86`) — tabelregel `10.35.0.3` + waarschuwingsblok "Gateway-cutover-incident".

## Voor de échte cutover (checklist, nog te doen)
- **Publieke DNS** (one.com): `84.199.147.82` -> nieuw `.84`-blok; **TTL vooraf naar 300s** (24u TTL nu). DNAT 80/443->VIP + NAT-hairpin opnieuw op de nieuwe gw.
- **Alle firewall-zones/regels** opnieuw aanbrengen — `firewall-rules-matrix.md` = checklist; let op "Retourverkeer automatisch toestaan"-gotcha.
- **VM-default-gateways + interne DNS** (`olvp.int` A-records) herwijzen.
- **Lessen**: parallelle gw mag NOOIT een host-IP overnemen (kies `.3`/transit, niet `.2`) en NOOIT DHCP serveren zolang de oude gw de WAN draagt.

## Open planning-item (apart, hoort in dezelfde cutover-werf)
**UniFi management-UI lock** = alleen vanaf admin-VLANs (34 + tijdelijk 10). Onderscheid: controller-UI (op `SRVV-UNIFI-01` VLAN 35, gedekt door zone-regels A-002/A-003 + default-deny) vs **de gateway-console zelf** (input/local-chain, NIET door zone-forward-regels gedekt -> aparte "Local"-regel nodig). VLAN-10-temp-bron verwijderen bij NET-2 (VLAN-10-leegmaak). Nog niet in de matrix geschreven.

Gerelateerd: [[project-infrastructure-params]], [[project-firewall-strategy]], [[project-unifi-os-server-migration]], [[project-hosting-fase1-status]].
</content>
