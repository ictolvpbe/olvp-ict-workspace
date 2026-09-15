---
name: project-stepca-cert-incident-202609
description: "Incident 2026-09-14/15: alle step-ca-certs sinds 18/08 verlopen (step-ca uit, daarna geen gateway-IP op EFG voor VLAN 21/22) → publieke sites 503. Opgelost: certs 168h, renewal 120h, NetXMS-lock-fix, step-ca-bewaking gecodificeerd. Open: bewaking uitrollen, haproxy-2 flapt, wees-cert, repos pushen."
metadata:
  type: project
---

**Tijdlijn.**
- 18/08 ~16:00: step-ca onbereikbaar (`no route to host` = VM uit); VMs op de hypervisor gingen 18–20/08 down. Alle 24u-certs verlopen binnen 12 u.
- 28/08: EFG in dienst. 29/08 00:08 renew-fouten wisselen naar `connection timed out` → **VLAN 21/22 hadden geen gateway-IP op de EFG** ([[project-gateway-cutover]]).
- 10/09 13:57: alle VMs abrupt gestopt; 14/09 10:25 herstart. NetXMS kwam niet op (achtergebleven DB-lock).
- 14/09: user zette de gateway-IP's → step-ca/HAProxy bereikbaar. 15/09: DHCP op oude gw in VLAN 34 uitgezet; certs opnieuw uitgegeven met 168h; playbooks uitgerold; caddy op SRVV-ODOO-01 herstart → **myschool + myschool-ict weer 200 via VIP**.

**Wat structureel veranderde** (commits platform-ansible `c565c95`, `4091a99`; handbook `8ca33ce`, `eae2683`):
- step-ca provisioner `admin`: claims 168h (max + default). Renewal `--expires-in 120h`. Zie [[feedback-step-ca-renew-no-password]].
- netxmsd ruimt de stale lock bij **elke** start op (`ExecStartPre=-/usr/local/sbin/netxms-unlock-stale.sh`), niet alleen bij een playbook-run. Zie [[project-netxms-monitoring]].
- NetXMS-agent levert `OLVP.StepCA.CertHoursLeft` + `OLVP.StepCA.Reachable` (script `/usr/local/lib/olvp-stepca-check.sh`); drempels/EPP via console, RB-2026-NETXMS-DEPLOY gotcha 19 (drempel `< 96` u, unreachable 3 samples).
- step-ca.md rechtgezet: hosts gebruiken JWK `admin` + renew-timer, **niet** ACME.

**Lessen.**
- De renew-timer logde weken enkel WARN in de journal; niemand keek. Dashboards/lokale 200 verbergen dit — alleen HAProxy-backendstatus of publieke test toont het.
- Caddy leest certs als file-bind-mounts → na handmatige heruitgifte **restart**, geen reload.
- Bij "X onbereikbaar": eerst `ip route get`, gateway-MAC en DHCP-server nakijken ([[feedback-same-subnet-test-proves-nothing]]).
- Een curl-fout 77 (`--cacert` onleesbaar zonder sudo) leek op netwerkfout — lees de foutcode voor je conclusies trekt.
- Claude mag via de auto-mode geen remote writes (unlock, restarts) of git push/merge: die stappen gaan via de user.

**Open (stand 2026-09-15).**
1. `netxms-agent.yml` **nooit effectief gedraaid**: `/etc/nxagentd.conf` op monitoring is het pakket-bestand van 2023. Eerst `-e target_limit=srvv-monitoring-01`, dan console-stappen gotcha 19.
2. **haproxy-2 (BACKUP) flapt door een IP-conflict met de OUDE gateway** (bewezen 2026-09-15): haproxy-2 = MAC `bc:24:11:d2:4c:1d`, maar haproxy-1 resolvet `10.21.0.7` consequent naar `0c:ea:14:19:e4:11` (oude gw). TCP vanaf haproxy-2 werkt enkele seconden, valt dan volledig weg → checks L4TOUT. VIP op haproxy-1 dus geen gebruikersimpact, maar **geen werkende failover**. Fix = interface van de oude gw van `.7` af (of oude gw uit die VLANs), daarna ARP flushen. Verklaart ook 22/443 dicht vanaf VLAN 34 op 14/09. **✅ Opgelost 2026-09-15 13:25**: user haalde de oude gw van `10.21.0.7`; haproxy-1 ziet nu `bc:24:11:d2:4c:1d`, TCP-reeks vanaf haproxy-2 10/10 ok, alle 4 backends UP (L7OK) op beide nodes → failover weer bruikbaar.
3. Wees-cert `/etc/caddy/id.olvp.be.*` op SRVV-ODOO-01 (niet in Caddyfile of container) → WARN bij elke run; verwijderen.
4. Repos niet gepusht: platform-ansible ahead 33; platform-handbook ahead 57/behind 3 (overlap `migrate-unifi-controller.md`); workspace ahead/behind 4 (overlap `MEMORY.md`, `project_unifi_os_server_migration.md`).
5. `forgejo.olvp.int` heeft twee A-records (`10.35.0.11` + oud `10.36.0.11`) → oude weghalen op AD-DNS.
6. Firefox mist de step-ca-root; Chrome-melding "niet beveiligd" was sessie-uitzondering van het verlopen cert.
