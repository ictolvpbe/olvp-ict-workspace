---
name: project-netxms-monitoring
description: "Monitoring-werf (deelproject D): NetXMS 6.2.3 greenfield op SRVV-MONITORING-01 (10.35.0.20, VLAN 35) via Podman/Quadlet, web-UI intern op netxms.olvp.int. Ansible + runbook geschreven 2026-08-25; VM nog te provisionen."
metadata:
  type: project
---

**Start 2026-08-25.** Monitoring krijgt eindelijk een echte basis. Er draait al "deels" een NetXMS (host/versie onbekend) — die wordt **niet gemigreerd**: greenfield 6.2.3, parallel draaien, oude server pas uit na verificatie. Hard gate voor A-go-live (`NetXMS-coverage kritieke services + alerts`, deelproject D) en pre-conditie voor SEC-3.

## Beslissingen (user, 2026-08-25)
- **VM = `SRVV-MONITORING-01`, 10.35.0.20, VLAN 35** (mgmt-appliances — de tier die het legacy VLAN 10 `VL-SD-INFRASTR` vervangt). Was al zo voorzien in `vm-inventory.md`; Grafana/Prometheus/Loki komen later op dezelfde VM ("één VM per logische stack", zie [[project-naming-convention]]).
- **Greenfield**, oude NetXMS later decommissioneren (harvest van drempels/checks als aparte fase).
- **Web-UI intern**: `netxms.olvp.int` (alias `monitoring.olvp.int`), Caddy + step-ca, **geen HAProxy-backend, geen publieke DNS**. Service-patroon = Forgejo ([[project-forgejo-status]]), niet het Odoo-instances-patroon.

## Architectuur
`Beheerder (VLAN 34) → https → Caddy :443 (step-ca) → netxms-web (Jetty 12 + nxmc-6.2.3.war, 127.0.0.1:8080) → netxms-server :4701`. Daarnaast `nxmc` desktop-console rechtstreeks op `10.35.0.20:4701`. Podman-netwerk `netxms-net` met postgres:17 (`db`), netxmsd (`server`, poorten 4701 console + 4703 agent-tunnels) en een in-stack agent (`agent`).
Images: `ghcr.io/netxms/{server,agent,web}:6.2.3` (officiële `netxms/docker`-repo, **linux/amd64-only**) + `docker.io/library/postgres:17`.

## Geleverd 2026-08-25 (branch `netxms-fase1`, beide repos, nog niet gemerged)
- **platform-ansible** commit `4ca24c9`: `netxms.yml` (Tier-2 overlay), `netxms-agent.yml` (native agent-uitrol), templates `netxms.Caddyfile.j2` / `netxmsd.conf.j2` / `nxagentd.conf.j2` / `nxagentd.stack.conf.j2`, `files/netxms-keyring.gpg`, inventory-groepen `monitoring_servers` + view-groep `netxms_agents` (9 hosts), `netxms_version`/`netxms_server_ip` in `group_vars/all/vars.yml`.
- **platform-handbook** commit `30c7968`: runbook `hosting/operations/deploy-netxms.md` (**RB-2026-NETXMS-DEPLOY**, 8 fasen + apart upgradepath-hoofdstuk), `management-tools/netxms.md` volledig herwerkt, `vm-inventory.md` bijgewerkt.
- Syntax-check + YAML + Jinja-render OK. **Nog niet e2e gedraaid** — de VM bestaat nog niet.

## Gotcha's (upstream geverifieerd, niet uit de config af te leiden)
- **Container-aliassen zijn contracten**: de web-container zoekt de server op de naam `server` (JNDI `EnvEntry nxmc/server` in `ROOT.xml` van het image); netxmsd zoekt `db` en `agent`. Niet hernoemen.
- **Web-UI en server moeten exact dezelfde release zijn** (de WAR spreekt het client-protocol van die release) → één `netxms_version` in group_vars, bewust NIET in de playbooks (playbook-vars winnen van group_vars en zouden de gedeelde waarde stil negeren).
- **`nxdbmgr init` genereert het admin-wachtwoord eenmalig** en print het; playbook vangt dat op in `/root/netxms-dbinit-output.txt` (0600). Idempotentie-gate = `nxdbmgr -q get DBLockStatus` → **exit 5 = leeg schema**; alleen dán initialiseren.
- **In-stack agent monitort de host NIET** (ziet enkel de container) — die bestaat als `ManagementAgentAddress` voor web-service/SSH-checks. Host-monitoring = **natieve** nxagentd via apt.
- **Apt-geïnstalleerde agents nooit upgraden via de console-package-manager** — upstream waarschuwt expliciet: overschrijft dpkg-bestanden. Linux-agents gaan via `netxms-agent.yml` (versie-gepind, priority 1001 → downgrade/rollback mogelijk). De console-weg is voor Windows (`.msi`).
- **Agents standaard in `Servers`-modus** (alleen-lezen). `MasterServers` = remote command execution op elke host → gecompromitteerde monitoring-server wordt pad naar alles. Opt-in per host met `-e netxms_agent_master=true`.
- **Apt-repo**: `https://packages.netxms.org/debian trixie main` (http redirect naar https), deb822 + keyring. Sleutel `C72F 5549 8B31 A527 181D 91D9 179C 0A80 CDFA DDB1` **verloopt 2027-02-21** → daarna faalt `apt update` op elke agent-host.
- **`DebugLevel` > 1 met `LogFile={stdout}`** vult /var/log via journald→rsyslog — zelfde klasse als [[project-podman-log-hygiene]].
- **Draai `netxms.yml` vanaf het werkstation, niet via de bastion**: jump-01 → 10.35.0.20 is intra-VLAN-35 en wordt door per-VLAN-isolatie geblokkeerd (zelfde als SRVV-UNIFI-01) → `ansible_ssh_common_args: ""` in de inventory.
- **Upgrade-volgorde**: DB-dump → `nxdbmgr check` (server gestopt) → versie bumpen → playbook → zo nodig `nxdbmgr upgrade` → agents. Image-tag terugrollen helpt niet meer na een schema-migratie; de dump is dan het enige rollback-pad.

## Uitrol 2026-08-25 — server DRAAIT
VM was al geprovisioneerd (Tier-1 clone, `ansible`-account + canonical key zaten er al; hostname stond op `SRVV-MONITOR-01` en is door de user naar `SRVV-MONITORING-01` gezet). Fase 1 t/m 5 gelopen door de user.
- ✅ **Stack live**: `netxms-db` (healthy), `netxms-server` ("NetXMS Server started", init 3,3 s), `netxms-web`, `caddy`. Poorten 443 / 4701 / 4703 / 127.0.0.1:8080 luisteren. Web-UI geeft **302 met geldige step-ca TLS** op `netxms.olvp.int` én `monitoring.olvp.int`.
- ✅ Cert `CN=netxms.olvp.int`, SANs netxms + monitoring, issuer OLVP Internal CA Intermediate.
- ✅ DB geïnitialiseerd; gegenereerd admin-wachtwoord staat in `/root/netxms-dbinit-output.txt` (0600) → **over te zetten naar KeePassXC `netxms-admin` en dan wissen**.
- ⚠️ **DNS `netxms.olvp.int` ontbreekt nog** (alleen `srvv-monitoring-01` + `monitoring` bestaan) — A-record toevoegen.
- 📌 Clone-realiteit: `/opt` = 62 G (daar staat `/opt/netxms`), `/var` = 12 G, vda 200 G waarvan ~100 G ongepartitioneerd. Log-hygiene + step-ca bootstrap ontbraken en zijn via Fase 1/2 rechtgezet.

### Twee fouten die pas bij het echte draaien bovenkwamen (beide gefixt, commit `e0a9f70`)
1. **Unit-naamconflict**: de in-stack agent-Quadlet heette `netxms-agent` — exact de service-naam van het Debian-agentpakket, dat op deze VM óók draait. De Quadlet schaduwde de deb-unit (generator wint van `/lib`), systemd zag het lopende `nxagentd`-proces als active en maakte de container **nooit** aan → `agent` niet resolvebaar in `netxms-net`, `ManagementAgentAddress` stil kapot terwijl de stack gezond oogt. Unit heet nu **`netxms-mgmt-agent`**; de playbook ruimt de oude Quadlet-file op.
2. **Verificatie draaide tegen de oude config**: de restart-handler vuurt pas aan het einde van de play, dus de HTTPS-check testte een Caddy die de nieuwe Caddyfile nog niet geladen had → tweede valse 403. `meta: flush_handlers` staat nu vóór het verify-blok.
3. **Caddy 403 op de eigen verify**: de HTTPS-check draait óp de VM en komt binnen als `127.0.0.1`, wat niet in `internal_networks` zit. Loopback staat nu expliciet in de `@intern`-matcher.

**Eindstand 2026-08-25**: alle 5 containers up (`netxms-db` healthy, `netxms-server`, `netxms-web`, `netxms-mgmt-agent`, `caddy`), `agent` resolvet in `netxms-net` (10.89.0.8) dus `ManagementAgentAddress` werkt, en de HTTPS-check vanaf de VM geeft 302 met geldige step-ca TLS.

## Migratie oude server (beslist 2026-08-25)
Oude server = **`10.10.100.2`** — web-UI `http://10.10.100.2:8080/nxmc` (Tomcat 10 op Debian 13, agents 5.1.3). Poort 4701 is van buiten dicht; 22/4700/8080 open.
**Beslissing user: alleen de CONFIGURATIE overzetten** (templates, drempels, EPP-regels, scripts), géén DB-migratie. Pad = console **Export Configuration** (view `config.export`, Configuration-perspectief) op de oude server → **Import Configuration** (`config.import`) op de nieuwe. Vastgelegd als Fase 8 in het runbook; decommissie schuift op naar Fase 9.
- **Geverifieerd in de bron**: `ImportConfigFromContent` (`src/server/core/import.cpp`, release-6.2.3) detecteert het formaat zelf en heeft een expliciet legacy-XML-pad → een export uit 5.1.x importeert gewoon in 6.2.3 (6.2.3 exporteert zelf JSON).
- **Komt NIET mee**: nodes/objectboom, SNMP-credentials, gebruikers, historische meetdata + alarm-historiek, per-node DCI-aanpassingen. Historiek zou een volledige `nxdbmgr export`/`import`/`upgrade` vergen, wat de hele DB (incl. admin-account) vervangt.
- **Timing**: importeren vóór de nieuwe server ingericht wordt.

## Volgende stap
1. `netxms.yml` nog één keer draaien ter bevestiging van idempotentie (alles ok, verify groen).
2. Eerste login op `https://netxms.olvp.int/` vanaf VLAN 34/10.x, admin-wachtwoord roteren, persoonsgebonden beheerdersaccount.
3. `netxms-agent.yml` uitrollen (agents rapporteren aan .20 én legacy 10.10.100.2).
4. SNMP op UniFi-devices, alert-routes (e-mail eerst).
5. Oude server `10.10.100.2` inventariseren + uitfaseren; daarna `netxms_legacy_server` leegmaken.
6. Firewall-regels bevestigen: VLAN 34 → .20 (22/443/4701/9090), .20 → agent-hosts :4700 + devices :161/udp.
7. Branches `netxms-fase1` in beide repos mergen na review.

## Gerelateerd
[[project-forgejo-status]] (service-patroon), [[project-infrastructure-params]] (VLAN 35 / IP-plan), [[project-internal-pki-coverage]] (step-ca cert + renewal), [[project-podman-log-hygiene]] (log-hygiene op dezelfde VM), [[project-firewall-strategy]] (nieuwe regels), [[feedback-test-from-user-vlan]] (verificatie vanaf het juiste VLAN).
