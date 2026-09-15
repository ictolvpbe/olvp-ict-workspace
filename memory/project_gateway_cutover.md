---
name: project-gateway-cutover
description: "EFG (Enterprise Fortress Gateway) in dienst sinds 2026-08-28. Cutover liet gaten: VLAN 21/22 zonder gateway-IP (step-ca+HAProxy weken onbereikbaar, gefixt 2026-09-14) en DHCP nog actief op de OUDE gw in VLAN 34 (gefixt 2026-09-15). Incident 2026-06-16 (parallelle gw) blijft de les."
metadata:
  type: project
---

## Stand 2026-09-15 — EFG in dienst, cutover-gaten opgeduikt
De **EFG draait sinds 2026-08-28** (zie [[project-netxms-monitoring]]). De secties hieronder van vóór die datum beschrijven de parallelle fase en zijn historisch. Twee gaten bleken pas weken later:
- **VLAN 21 (HAProxy) + 22 (step-ca) hadden geen gateway-IP op de EFG.** Vanaf 29/08 bereikte niets step-ca of HAProxy → alle 24u step-ca-certs verlopen, publieke sites 503. User zette de IP's op 2026-09-14 (volgens user voor "de 3 VLANs"). Zie [[project-stepca-cert-incident-202609]].
- **DHCP stond nog aan op de oude gateway in VLAN 34**: `10.34.0.7`, MAC `0c:ea:14:19:e4:11`, deelde zichzelf uit als gateway + DNS → vanaf het beheerwerkstation was niets meer bereikbaar. Uitgezet 2026-09-15. **Nog na te kijken**: DHCP van de oude gw op de andere VLANs.
- **De oude gateway staat nog op `.7` in meerdere VLANs** (MAC `0c:ea:14:19:e4:11`, bevestigd 2026-09-15): `10.34.0.7` (DHCP, gefixt), `10.35.0.7`, **`10.21.0.7` = IP-conflict met haproxy-2** → failover stuk (✅ oude gw van `.7` gehaald 2026-09-15; haproxy-2 weer gezond — zie [[project-stepca-cert-incident-202609]]). Na de ingreep antwoorden ook `10.34.0.7` en `10.35.0.7` niet meer (0/3 pings, 13:27); de oude MAC die nog in ARP-tabellen staat is cache. `10.36.0.7` antwoordt niet. **Verdacht, niet getest**: de NAS staat op `10.19.0.7` + `10.10.100.7` — het "antwoordt alleen in eigen subnet"-symptoom van de backup-storing (31/08, [[project-offline-backup-chain]]) past bij een zelfde conflict. Les van 16/06 herhaalt zich: een gateway mag nooit een host-IP innemen.
- EFG VLAN 34: gateway `10.34.0.1`, MAC `58:d6:1f:4f:cb:9d` (zelfde MAC op VLAN 35/36). Beheer VLAN 34 → 35/36/200 gaat **rechtstreeks**; DMZ (21/22) enkel via jump-01 (A-004).
- **Diagnose-truc**: bij "plots niets bereikbaar" eerst `nmcli -f DHCP4 device show <if>` (server_identifier/routers) en `ip neigh` op de gateway — een vreemde MAC wijst meteen een tweede DHCP-server aan.
- Open: publieke DNS-omzetting, SEC-5 en de UI-lock uit de checklist hieronder zijn niet geverifieerd; haproxy-2 ziet backends flappen (mogelijk EFG-gerelateerd).

**Status 2026-06-17 (historisch).** Een **nieuwe UniFi-gateway** staat **parallel** naast de oude (opgezet 2026-06-15/16). Intentie = **volledige cutover** (nieuwe vervangt oude) = alle interne IP's + externe adressen omzetten — gepland als aparte werf/window, **nog niet uitgevoerd**.

## Huidige (tijdelijke) toestand — niet wijzigen tot cutover
- **WAN-uplink + DNAT zitten op de OUDE gw** (publiek IP `84.199.147.82`). Netwerk loopt via de oude gw en **moet zo blijven**.
- Nieuwe gw heeft **andere externe adressen** (`84.199.147.84`-blok) en gebruikt per VLAN een `.2`-interface; op **VLAN 35 botste `.2` met jump-01 (10.35.0.2)** -> verplaatst naar **`10.35.0.3`** (transitie-IP; wordt `.1` bij cutover). Op VLAN 21/34/36 is `.2` vrij, dus enkel VLAN 35 aangepast.
- DHCP op de nieuwe gw is **uitgezet** (zie incident hieronder).

## Incident 2026-06-16 (opgelost) — root cause om te onthouden
Publieke Odoo-FQDN's (o.a. **myschool-test**) waren **extern onbereikbaar** hoewel HAProxy/Odoo/DNS gezond waren. Oorzaak: de **DHCP-server op de nieuwe gw deelde zichzelf (`.2`) als default-gateway** uit aan clients, terwijl WAN+DNAT op de oude gw (`.1`) zitten -> **asymmetrische routing** (inbound via oud, antwoord via nieuw zonder NAT-state) -> handshake faalt. **Fix**: DHCP op nieuwe gw uit + clients ge-renewd (terug via oude gw) + nieuwe-gw VLAN35-interface -> `.3`. Daarna alles groen geverifieerd (myschool-test publiek `http=200`, VIP `10.21.0.5` ok, jump-01 ongestoord).
Bewijs van het `.2`-conflict: SSH-hostkey op `10.35.0.2` wisselde tijdens de storing naar een vreemd toestel (de gw-interface) en is na de fix weer de echte jump-01-key.

Vastgelegd in repo: `platform-handbook/network-physical/reference/ip-plan.md` (commit `a326d86`) — tabelregel `10.35.0.3` + waarschuwingsblok "Gateway-cutover-incident".

## Update 2026-08-28 — indienststelling gepland
De nieuwe gateway is een **UniFi Enterprise Fortress Gateway (EFG)** (productnaam geverifieerd bij Ubiquiti — *Enterprise*, niet Edge). User plant de **indienststelling binnen twee weken** (uiterlijk 2026-09-10), omdat **SEC-5** daaraan opgehangen is: de meegekomen diensten op `PC-MONITORING-01` (Webmin/Usermin/xrdp op `0.0.0.0`) worden bij die cutover afgeschermd.
⚠️ Dat koppelt een securitydeadline aan de volledige cutover uit de checklist hieronder — publieke DNS, alle firewall-zones opnieuw, default-gateways en interne DNS. Schuift de cutover, dan schuift SEC-5 mee.

## Voor de échte cutover (checklist, nog te doen)
- **Publieke DNS** (one.com): `84.199.147.82` -> nieuw `.84`-blok; **TTL vooraf naar 300s** (24u TTL nu). DNAT 80/443->VIP + NAT-hairpin opnieuw op de nieuwe gw.
- **Alle firewall-zones/regels** opnieuw aanbrengen — `firewall-rules-matrix.md` = checklist; let op "Retourverkeer automatisch toestaan"-gotcha.
- **VM-default-gateways + interne DNS** (`olvp.int` A-records) herwijzen.
- **Lessen**: parallelle gw mag NOOIT een host-IP overnemen (kies `.3`/transit, niet `.2`) en NOOIT DHCP serveren zolang de oude gw de WAN draagt.

## Open planning-item (apart, hoort in dezelfde cutover-werf)
**UniFi management-UI lock** = alleen vanaf admin-VLANs (34 + tijdelijk 10). Onderscheid: controller-UI (op `SRVV-UNIFI-01` VLAN 35, gedekt door zone-regels A-002/A-003 + default-deny) vs **de gateway-console zelf** (input/local-chain, NIET door zone-forward-regels gedekt -> aparte "Local"-regel nodig). VLAN-10-temp-bron verwijderen bij NET-2 (VLAN-10-leegmaak). Nog niet in de matrix geschreven.

Gerelateerd: [[project-infrastructure-params]], [[project-firewall-strategy]], [[project-unifi-os-server-migration]], [[project-hosting-fase1-status]].
</content>
