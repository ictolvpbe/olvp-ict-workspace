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

### Incident 2026-08-25 — playbook sloopte de draaiende database (fix `265b724`)
Symptoom bij de user: browser gaf `ERR_QUIC_PROTOCOL_ERROR`, site onbereikbaar. Werkelijke keten, van achter naar voren:
1. De dirs-taak zette `/opt/netxms/db` bij élke run terug naar `root:root 0700`, terwijl postgres in de container als **uid 999** draait (host toont die uid als `systemd-coredump` — puur toeval in de uid-nummering).
2. De draaiende postgres verloor daardoor midden in bedrijf toegang: `FATAL: could not open file "global/pg_filenode.map": Permission denied` → `PANIC: could not open file "global/pg_control"` → WAL-writer `signal 6` → general protection fault in libc → exit 139.
3. Onreine shutdown → **netxmsd-lock bleef in de DB staan** (`Database is already locked by another NetXMS server instance, IP 10.89.0.5`). De nieuwe container heeft een ander IP → weigert te starten, exit 3, 8 herstarts, opgegeven.
4. `Requires=` sleurde web én caddy mee → site volledig dood.

**Herstel**: `nxdbmgr -f unlock` + `systemctl reset-failed` + services starten. **Drie fixes in de playbook**: PGDATA krijgt geen owner/mode meer, stale lock wordt vóór het starten opgeruimd (met guard tegen unlocken bij een draaiende server), en `Requires=` werd `Wants=` tussen web→server en caddy→web zodat een gecrashte netxmsd hoogstens een 502 geeft i.p.v. een dode site.
**Les**: een idempotente `file:`-taak met `owner`/`mode` op een datadir van een container die zelf chownt = tijdbom die pas bij de tweede run afgaat.

## Migratie oude server (beslist 2026-08-25)
Oude server = **`10.10.100.2`** — web-UI `http://10.10.100.2:8080/nxmc` (Tomcat 10 op Debian 13, agents 5.1.3). Poort 4701 is van buiten dicht; 22/4700/8080 open.
**Beslissing user: alleen de CONFIGURATIE overzetten** (templates, drempels, EPP-regels, scripts), géén DB-migratie. Pad = console **Export Configuration** (view `config.export`, Configuration-perspectief) op de oude server → **Import Configuration** (`config.import`) op de nieuwe. Vastgelegd als Fase 8 in het runbook; decommissie schuift op naar Fase 9.
- **Geverifieerd in de bron**: `ImportConfigFromContent` (`src/server/core/import.cpp`, release-6.2.3) detecteert het formaat zelf en heeft een expliciet legacy-XML-pad → een export uit 5.1.x importeert gewoon in 6.2.3 (6.2.3 exporteert zelf JSON).
- **Komt NIET mee**: nodes/objectboom, SNMP-credentials, gebruikers, historische meetdata + alarm-historiek, per-node DCI-aanpassingen. Historiek zou een volledige `nxdbmgr export`/`import`/`upgrade` vergen, wat de hele DB (incl. admin-account) vervangt.
- **Timing**: importeren vóór de nieuwe server ingericht wordt.

## Observability-stack erbij (2026-08-25)
Beslissing user: **volledige stack** Grafana + Prometheus + Loki + log-agent op dezelfde VM, en de config van de oude Grafana-server **als code** overnemen (geen `grafana.db`-kopie).

**Oude server = `10.10.100.1`** (Debian 13). Read-only geïnventariseerd via de API's, geen SSH nodig: Grafana **13.0.1** (draait), Prometheus **2.31.1** (build uit nov 2021), node_exporter + windows_exporter + snmp_exporter (localhost:9116) + unifipoller (localhost:9130), Alertmanager geconfigureerd met **lege targets** (dus geen alerting), **geen Loki**, promtail draait op 9080 met **lege `client.url`** → verstuurt nergens naartoe. Van de **27 scrape-targets stonden er 18 down**.

**Promtail is EOL sinds 2026-03-02** (upstream: "All future feature development will occur in Grafana Alloy") → **Grafana Alloy v1.19.0** i.p.v. promtail.

**Geleverd** (commit `d27f0de` + `1a656b4`): `observability.yml` (eigen podman-netwerk `obs-net`, Quadlets voor grafana/prometheus/loki/alloy/node-exporter/snmp-exporter/unpoller), templates voor alle configs, `vars/monitoring-targets.yml` als SoT met werkende én dode targets gescheiden, runbook **RB-2026-OBS-DEPLOY**, `grafana.md` bijgewerkt.

**Versies gepind**: grafana-oss 13.0.2, prometheus v3.14.0, loki 3.7.6, alloy v1.19.0, node-exporter v1.12.1, snmp-exporter v0.30.1, unpoller v2.7.1.

**Container-uids uitgelezen uit de image-config** (na het postgres-incident geen gok meer): grafana **472**, prometheus **65534**, loki **10001**, alloy/snmp-exporter/unpoller root. Anders dan postgres chownen deze images hun datadir NIET zelf → eigendom hier wél expliciet zetten en laten staan.

**Caddy gesplitst**: hoofd-`Caddyfile` met globale opties + `import /etc/caddy/conf.d/*.caddy`, per dienst een snippet (`caddy.netxms.j2`, `caddy.grafana.j2`). Zonder die splitsing overschrijven `netxms.yml` en `observability.yml` elkaars Caddyfile. **`netxms.yml` moet daarom opnieuw draaien** na `observability.yml`.

**Loki bewust niet gepubliceerd** (geen authenticatie) — agents op andere hosts krijgen later een Caddy-vhost met TLS; dat is meteen de basis voor SEC-3.

**Openstaand**: vault-keys `grafana_admin_password` (+ optioneel `unifi_controller_*`), step-ca cert voor `grafana.olvp.int`, DNS-record, en een **Grafana-service-account-token op 10.10.100.1** om de dashboards te exporteren.

### Observability-uitrol 2026-08-25 — stack draait, 10 van 11 targets up
- **Poortconflict**: Prometheus kon niet starten, `bind: address already in use` op 9090 — **Cockpit** houdt die poort bezet via socket-activation, en Cockpit zit in de Tier-1 baseline → geldt op élke OLVP-VM. Host-kant nu **9091**, container intern 9090.
- **SNMP-community**: exporter viel terug op `public` → time-outs. Opgelost met een **tweede `--config.file`** (`olvp-auth.yml`, auth-naam `olvp_v2`) zodat de ~2 MB meegeleverde module-definities intact blijft; Prometheus geeft `auth: [olvp_v2]` mee als param. Vault-key **`snmp_community`** nodig. Live bevestigd: 86 metrics van 10.10.110.5.
- **unpoller draait maar krijgt geen data**: controller-login faalt (`user: zeus … status 400 authentication failed`) — `zeus` is het device-SSH-account, geen controller-admin. **Let op de meetval**: Prometheus toont de target als UP omdat het `/metrics`-endpoint antwoordt; controleer met `count({__name__=~"unpoller_.+"})`.
- **`10.20.100.1:9100` onbereikbaar** vanaf VLAN 35 (geen ICMP/22/9100). Dat is **srvv-pw001**, de oude SSPR-server in het legacy-DMZ die op de decommissie-lijst staat → bewust naar de down-lijst i.p.v. een firewall-gat naar VLAN 20.
- **Wachtwoord-validatie**: de fail-melding zei "ontbreekt" terwijl de check ook op lengte (≥12, conform eigen password-policy) faalt; melding zegt nu welke van de twee.
- Grafana **13.0.2** healthy, Loki + Alloy draaien, alle 9 units active.

### 2026-08-26 — lock-cascade door mijn eigen restart-handler (fix `1dd8300`)
Symptoom: `netxms.yml` bleef hangen op "Wachten tot netxmsd de client-poort opent", web-UI gaf **"Cannot resolve host name 'server'"**. Oorzaak: de restart-handlers die ik 25/08 aan alle Quadlets toevoegde vuren **aan het einde van de play** — dus ná de unlock-guard. Die guard zag de nog draaiende oude container, sloeg de unlock over, en de herstart erna botste op de achtergebleven lock → exit 3 → `server` niet resolvebaar in `netxms-net` → web-UI stuk.
**Les**: handlers zijn het verkeerde gereedschap wanneer de volgorde ertoe doet. netxmsd wordt nu **expliciet gestopt** zodra unit/config/db-unit wijzigt, dan pas guard → unlock → start; de DB-herstart zit in datzelfde blok met de server gegarandeerd gestopt.
**Tweede oorzaak**: `StopTimeout` ontbrak. Podman kapt na 10 s af — te kort voor netxmsd (geeft lock alleen bij nette shutdown vrij) én voor postgres (verklaart ook de crash-recovery van 25/08). Beide units nu `StopTimeout=60` (< systemd `TimeoutStopSec` 90).

### Stand 2026-08-26
- NetXMS-stack draait, web-UI OK.
- **Observability: 10 van 10 Prometheus-targets up** na het zetten van `snmp_community` in de vault.
- **unpoller v2.7.1 → v4.0.1**: mijn versiecheck las maar één pagina ghcr-tags en gaf v2.7.1 als hoogste terwijl v4.0.1 actueel is. v2.x kan de JSON van een recente controller niet parsen (`cannot unmarshal number … into int64` op `wired-tx_bytes-r`) → nul metrics ondanks geslaagde login. Met v4.0.1: **355 clients, 174 AP's, 62 switches, 58.856 metrics, Err: 0**. Overige pins nagekeken met volledige paginering en correct bevonden.
- **Quadlet-wijziging herstart nu de container** (12 units, beide playbooks); podman-secrets roteren alleen met `-e rotate_secrets=true`; unpoller-unit op 0600 want bevat het controller-wachtwoord in klare tekst.

### Dashboards overgezet 2026-08-26
8 dashboards geëxporteerd van de oude Grafana met een **Viewer**-service-account-token (token in een bestand, niet in de chat). Datasource-mapping opgehaald bij de bron via **`/api/frontend/settings`** — dat endpoint werkt met een Viewer-token, terwijl `/api/datasources` admin vereist. Oude uid's: Prometheus `yLxUejo7k`, Loki `XkVUAqo7k` (+ een ongebruikte InfluxDB `OhXvxIE7z`). Omgezet: **418 → `olvp-prometheus`, 12 → `olvp-loki`**; 4 dangling verwijzingen op row-panelen in windows-exporter-2024 meegenomen.
- **Grafana 13 slaat dashboards op in unified storage** (`resource`-tabel), niet meer in de `dashboard`-tabel — die is leeg. Controleren dus via `select resource,name,folder from resource`.
- **`foldersFromFilesStructure: true` negeert de `folder`-instelling** → alles landde in General. Nu `false` + `folder: OLVP`.
- **Grafana slaat ongewijzigde bestanden over** op basis van `grafana.app/sourceChecksum` in de resource-annotaties. Een gewijzigde provider-config alleen dwingt dus geen herimport af; bestanden kort weghalen en terugzetten wel.
- Alle 8 zijn **community-dashboards**, geen eigen OLVP-werk → er ging niets verloren door géén `grafana.db` te kopiëren.
- Kandidaten om op te ruimen: "Loki stack monitoring (Promtail, Loki)" gaat over Promtail dat we door Alloy vervingen; twee Windows-Exporter-dashboards en twee node-exporter-dashboards overlappen; "Loki - Syslog AIO" verwacht syslog-labels terwijl Alloy `job="systemd-journal"` levert.

### Dashboards opgeschoond + Alloy rechtgezet 2026-08-26
Van 8 naar **5 dashboards**, alle in map `OLVP`: Node Exporter Full, Windows Exporter Dashboard, 2× UniFi-Poller, en het zelfgeschreven **OLVP Logs**. Keuze bij duplicaten onderbouwd met **gemeten metric-dekking** tegen de draaiende Prometheus (Windows 91% vs 88%; Node Exporter Full is het origineel, "Linux Exporter Node" een kopie met dezelfde uid + suffix). Geschrapt: beide Loki-community-dashboards (Promtail-gericht resp. syslog-labels).
**Drie fouten uit deze ronde:**
1. **Alloy job-label**: kwam als `loki.source.journal.system` in Loki, niet `systemd-journal` — het `labels`-argument van `loki.source.journal` wordt door `relabel_rules` overschreven. Nu expliciete relabel-regel `target_label = "job"`. Vóór de fix gaf élke query op een leesbare jobnaam **0 resultaten**; erna 390 regels/5 min.
2. **Alloy bond op 127.0.0.1** → niet te scrapen. `--server.http.listen-addr=0.0.0.0:12345` + `NetworkAlias=alloy` + scrape-job. De log-agent was het enige onbewaakte onderdeel van de keten.
3. **Bind-mount van één bestand overleeft vervanging niet**: Ansible (en `install`) schrijven naar een tijdelijk bestand en verplaatsen dat → nieuwe inode, container houdt de oude. `POST /-/reload` laadt dan de oude inhoud opnieuw. **Config-wijziging vereist container-restart, geen reload** — de playbook doet dat al via de handler.
**Stand: 11 van 11 Prometheus-targets up.**

### Syslog-ingest 2026-08-26 (beslissing user: beide kanten + 30 dagen in NetXMS-DB)
`apparaat --514--> rsyslog --+--> 127.0.0.1:1514/udp netxmsd  +--> /var/log/remote/<host>.log --> Alloy --> Loki`. rsyslog is de voordeur want maar één proces kan 514 binden. Dashboard **OLVP Syslog** toegevoegd.
- **`Syslog.EnableListener` staat standaard op 0** — de ingebouwde syslog-server van NetXMS was volledig uit. Instellingen leven in de **database** (niet netxmsd.conf), gezet via `nxdbmgr set`: NodeMatchingPolicy=**1** (hostnaam eerst; bij een relay is het bron-IP dat van rsyslog), AllowUnknownSources=1, ParseUnknownSourceMessages=1, EnableStorage=1, RetentionTime=**30**.
- **Doorsturen MOET in RFC5424** (`RSYSLOG_SyslogProtocol23Format`). netxmsd checkt na de PRI op `1 ` en probeert anders BSD-formaat; `RSYSLOG_ForwardFormat` levert geen van beide → hele bericht als tekst, `msg_tag=2026`, `hostname=10.89.0.1` (podman-gateway). Na de fix: `hostname=WAP-T001-01`.
- **`nxdbmgr get` geeft `naam=waarde` terug**, niet enkel de waarde → mijn idempotentiecheck meldde elke run een wijziging en herstartte netxmsd. Nu `have=${have#*=}`.
- **De AP's stuurden al syslog naar deze VM** (niet geconfigureerd door ons). Gemeten volume: **3158 msg/min = 4,5 M/dag, 762 MB/dag**. Samenstelling: 65% FWLOG (wifi-firmware-debug), 19% overige kernel, 11% stamgr. **User koos filteren**: rsyslog-filter vóór de vertakking, afgebakend op `re_match($programname, "^[0-9a-fA-F]{12},")` zodat kernelmeldingen van Linux-servers ongemoeid blijven. Resultaat: **625 msg/min**, 0 FWLOG/kernel in 783 nieuwe rijen, stamgr komt door.
- Golden-image-vondst: `/etc/rsyslog.d/to-servers.conf` stuurt de logs van **élke** VM naar de oude NetXMS 10.10.100.2 — fleet-breed te herzien bij de decommissie.
- Meet filtereffect via `max(msg_id)`-markeerpunt, niet via een tijdvenster: rijen van vóór de rsyslog-herstart vallen anders binnen je venster.

### Wifi-kick-diagnose + alarmering 2026-08-26
**Diagnose uit de eigen syslog-data**: 100% van de kicks heeft reden `Low RSSI`. Slechts **12 toestellen** veroorzaken ~6000 kicks, elk aan **één** AP met een **constante** RSSI (7, 8, 10, 11, 14, 15, 16, 21) → vaste toestellen aan de rand van de dekking. 81% gebruikt gerandomiseerde MAC's (telefoons); twee met echt MAC zijn **Intel**-adapters (laptops/desktops). Dit is het bekende **Minimum RSSI kick-loop**-gedrag: de functie werkt alleen als er een bétere AP is om naartoe te roamen; is die er niet, dan verbindt het toestel meteen opnieuw met dezelfde AP.
**Geleverd**: `platform-ansible/files/netxms/syslog-parser.xml` (3 meegeleverde MikroTik-regels + eigen UniFi-regel), regex getoetst op 1974 echte berichten → 1972 match; de 2 missers zijn `ignored kick-sta-on (reason:On other VAP)`, geen echte kicks. Procedure in `netxms.md`.
**Gotcha's**:
- `nxdbmgr get/set` werkt **alleen op de `config`-tabel**; parsers staan in **`config_clob`** en zijn er niet mee te benaderen → console of client-API. Bewust géén SQL-update: de server houdt de parser in geheugen.
- Custom event moet **"Write to event log" UIT** hebben: ~600 kicks/min = 900k events/dag.
- **Parser `repeatCount` telt per regel, niet per node** (`m_matchArray` in `libnxlp/rule.cpp`) → geen drempel per AP. Die maak je met een DCI op de interne metric **`ReceivedSyslogMessages`** (delta per minuut + threshold), in een template op de AP's.
- **AP's moeten eerst NetXMS-nodes zijn**, anders hangt het syslog-bericht nergens aan en kan er geen alarm per AP uit komen.

### ▶ GEPARKEERD 2026-08-26 — wifi-kicks (tracker **NET-6**)
User: "kan hier nu niet mee verder, plaats op de todo en negeer de berichten voorlopig."
- **Berichten gedempt** via `syslog_drop_wifi_kicks: true` (group_vars) → rsyslog-filter vóór de vertakking. Volume 625 → **244 msg/min, 0 kicks**. Op `false` zetten bij het oppakken; de NetXMS-parserregel + procedure liggen klaar.
- **Nog te doen bij oppakken**: Minimum-RSSI-drempel in UniFi herzien + dekking nakijken op de 12 plekken; daarna event `WIFI_CLIENT_KICKED_LOW_RSSI` aanmaken (write-to-log UIT), parser inlezen, EPP-alarm met sleutel `WIFI-KICK-%n`, DCI-drempel op `ReceivedSyslogMessages`.
- **Nieuwe bron gezien**: `srvv-ubiquiti001.olvp.int` stuurt UniFi-controller-events in **CEF-formaat** — aparte parserregels waard.

### Alarmscherm monitoring-TV (werkwijze 2026-08-26)
Runbook **RB-2026-NETXMS-KIOSK** (`hosting/operations/deploy-netxms-kiosk.md`). Ubuntu-machine aan de TV in het ICT-lokaal, automatisch aanmelden, dashboard schermvullend.
- **De web-UI kan dit zelf** via URL-parameters (bevestigd in `nxmc/src/rwt/.../Startup.java`): `auto`, `login`+`password` **of** `token`, `dashboard` (naam **of** ID), `kiosk-mode=true` (geen hoofdvenster, alleen het dashboard).
- **Wachtwoord i.p.v. token**: `WebAPI.AuthTokenMaxLifetime` staat standaard op **86400 s (24 u)** — een token zou dagelijkse vernieuwing vragen. Alleen-lezen account `kiosk-tv`, wachtwoord in een 0600-bestand, niet in het script.
- **Gotcha's**: Chromium gebruikt een **eigen NSS-database** en negeert de systeem-truststore → step-ca root apart toevoegen met `certutil`, anders blijft het scherm op een certificaatwaarschuwing hangen. `auto` werkt **één keer per sessie** → mislukte aanmelding laat een inlogvenster achter; vandaar `Restart=always` + nachtelijke herstart via cron. Geen snap-browser (sandbox hindert autostart + certbeheer).
- Kiosk-account krijgt **geen** recht om alarmen te bevestigen/sluiten — voorkomt dat een voorbijganger iets wegklikt.

### Objectboom + autobind 2026-08-26 — 167 van 181 nodes toegewezen
Discovery vond 181 nodes, maar die stonden enkel onder `Entire Network` (Network-perspectief). **Discovery bindt niet aan containers** — Infrastructure blijft leeg op de management-node na, die netxmsd zelf aan de Service Root hangt. Geen instelling, twee bomen naast elkaar.

Opgelost met **één identiek autobind-script in alle containers**, gestuurd door het **Alias**-veld van de container (`SW-`, `WAP-`, `SRV-`, `SRVV-`, `CAM-`, `GW-`, `PR-`, `TEL-`, `TC-`):
```
prefix = $container->alias;
if ((prefix == null) || (prefix == "")) return false;
pattern = prefix .. "*";
return $node->name ilike pattern;
```
Ouder-containers krijgen geen alias; de guard houdt ze leeg. Resultaat: 111 `WAP-`, 53 `SW-`, 1 elk voor `CAM-`/`SRV-`/`SRVV-`. De 14 overige nodes dragen hun fabrieksnaam (`UBNT`, `HT8XX`, `BRNB…`, vijf `10.10.102.x`) en hebben een hernoeming of een filter op `snmpOID`/`driver` nodig.

**NXSL-valkuilen, alle vier stil falend** (geverifieerd met `nxscript` in de server-container):
- Een mee gekopieerd taallabel `nxsl` als eerste regel → `Error in line 2: syntax error`, container blijft leeg, géén melding in de UI. Zie [[feedback-code-fence-label-in-ui-fields]].
- **Concatenatie is `..`**, `.` is attribuut-toegang → `Error 15: Unknown object's attribute`.
- `ilike` bindt sterker dan `..` → patroon eerst in een eigen variabele zetten.
- Lege alias: `null .. "*"` wordt patroon `*` en matcht **alles** → guard verplicht.
- Verder: `ilike` is hoofdletterongevoelig (vangt `sw-pparking`), onbekende variabele = `null` zonder fout, `trim()`/`length()` deprecated in 6.2.

**Prefix altijd mét streepje**: `SRV*` matcht ook `SRVV-MONITORING-01`, `SRV-*` niet.

**Forceren**: `Objects.AutobindPollingInterval` 3600 en `Objects.AutobindOnConfigurationPoll` 1 → binnen het uur vanzelf. Per node: Poll → Autobind (toetst die node tegen *alle* containers). Massaal: interval tijdelijk op 60, daarna terugzetten. `Objects.AccessPoints.ContainerAutoBind` staat op 0 — telt zodra de UniFi-controller AP's als AccessPoint-objecten aanmaakt i.p.v. Nodes.

**DB-verificatie**: `auto_bind_target.flags` 1=bind, 2=unbind, 3=beide, 0=vinkje uit (script staat er, draait niet). In `container_members` is **`container_id` de ouder en `object_id` het lid** — omgekeerd tellen geeft het aantal ouders.

Vastgelegd in `platform-handbook/management-tools/netxms.md` (hoofdstuk "Objectboom: containers en autobind" + 5 troubleshooting-rijen + Learnings). Containers zitten **niet** in Export Configuration en niet in Ansible — die doc is het enige vangnet.

### Kiosk-scherm: NetXMS-kant AF 2026-08-26
Dashboard **`OLVP`** (object-id 7613) met alarm viewer + SNMP-trap monitor + syslog monitor werkt volledig in kiosk-modus. URL: `https://monitoring.olvp.int/nxmc-light.app?auto&login=kiosk-tv&password=…&dashboard=OLVP&kiosk-mode=true` (**`netxms.olvp.int` staat nog steeds niet in DNS**; cert dekt beide namen).
**Drie rechten-gotcha's, alle drie geverifieerd in de broncode:**
1. **Objectrecht op het dashboard** ontbrak → web-UI meldde `Cannot find dashboard object with name or ID`. Alle root-objecten geven standaard enkel toegang aan `Admins` en een nieuw dashboard erft dat.
2. **In kiosk-modus geeft een niet-gevonden dashboard `Invalid resource ID` in de browser** — er wordt géén hoofdvenster gebouwd, dus zonder dashboard blijft er geen venster over en struikelt RAP. De echte oorzaak staat in `journalctl -u netxms-web`.
3. **Alarmen vragen ZOWEL `VIEW_ALL_ALARMS` (systeem) als `OBJECT_ACCESS_READ_ALARMS` (op het bronobject)** — `SendAlarmsToClient` in `alarm.cpp` eist beide. Alleen het systeemrecht → lege alarmlijst terwijl de DB er 466 bevat. Syslog- en trap-panelen hangen daarentegen puur aan systeemrechten (`VIEW_SYSLOG`, `VIEW_TRAP_LOG`), serverzijde afgedwongen in `onSyslogMessage`.
**SNMP-traps**: `snmp_trap_log` was leeg omdat de container **poort 162 niet publiceerde**. Nu `PublishPort=162:162/udp` + `SNMP.Traps.LogAll=1` + retentie 30 dagen. Let op: UniFi-apparaten pollen wel via SNMP maar sturen doorgaans zelf géén traps — dat paneel kan dus leeg blijven zonder dat er iets stuk is.

### ▶ MORGEN 2026-08-27 — eerste taak: TV-toestel upgraden
Het toestel aan de TV draait **Debian 11**. Uit reguliere ondersteuning sinds **2024-08-14**; Debian 12 sinds **2026-07-11**. Alleen 13 loopt door (tot 2028-08-09). Debian laat geen sprongen toe → **11 → 12 → 13**, release na release. Pas dáárna de kiosk bouwen volgens **RB-2026-NETXMS-KIOSK** (runbook staat inmiddels op Debian, niet Ubuntu: `gdm3` leest `/etc/gdm3/daemon.conf`, en Chromium is een gewone `.deb`).
Het bestaande bash-script op het toestel wordt vervangen door het script uit stap B7 van het runbook (user-keuze).

### 2026-08-27 — "netxms onbereikbaar" was DNS, niet de dienst
Melding: web-UI onbereikbaar. Stack bleek volledig gezond — 12 containers up, `https://monitoring.olvp.int/` gaf end-to-end 200 op `/nxmc-light.app` met geldig step-ca-cert, cert-renewal-timer normaal (24-uurs certs, twee runs per dag), netxmsd stabiel sinds 26/08 14:49 zonder lock-problemen.

Werkelijke oorzaak: **`netxms.olvp.int` bestaat niet in DNS.** NXDOMAIN van alle drie de DC's (10.10.0.10, 10.33.0.10, 10.10.0.11) met identieke SOA-serial, dus geen replicatie-achterstand. Alleen `monitoring.olvp.int` en `srvv-monitoring-01.olvp.int` bestaan. Het stond sinds 25/08 als openstaand punt en is er nooit gekomen; runbook en doc noemden `netxms.olvp.int` wél overal als primaire URL — de valstrik.
- **Les**: bij "dienst onbereikbaar" eerst de naam scheiden van de dienst. `curl --resolve <naam>:443:<ip>` beantwoordt in één commando of het transport of de naamgeving stuk is.
- Runbook + `netxms.md` aangescherpt: beide A-records staan nu apart in de pre-flight én in de verificatie-checklist, met een `dig`-commando tegen de autoritatieve DC.
- Bijvangst: **`*.olvp.be` is een wildcard** bij one.com → elke naam onder dat domein resolvet naar 46.30.213.100. Vastgelegd in [[reference-dns-olvpbe]].

### 2026-08-27 — TV-toestel geüpgraded + opgenomen in beheer
- **Debian 11 → 12 → 13 gelukt** (user). Runbook **RB-2026-DEB1113** vastgelegd (`hosting/operations/upgrade-debian-11-to-13.md`), fleet-breed bruikbaar. Kernpunten: Debian 11 uit reguliere support sinds 2024-08-14, Debian 12 sinds 2026-07-11 → alleen 13 is een zinvolle eindbestemming; geen sprongen mogelijk; `non-free-firmware` is nieuw in 12; `apt modernize-sources` (apt 3.0) zet `.list` om naar deb822.
- **Toestel = `PC-MONITORING-01`, 10.34.0.25, VLAN 34.** Naam volgens `naming.md` (prefix `PC` + functie-patroon). Opgenomen in `inventory.yml` als eigen top-level groep **`kiosk_devices`** (eindtoestel, geen servertier → geen group_vars van servers erven) en in nieuwe doc `network-physical/physical/endpoints.md`.
- **`kiosk.yml` geschreven** (commit `c8f53e3`): Chromium, kiosk-gebruiker, step-ca root in systeem-truststore **én** NSS-database, gdm3-autologin, dconf-systeemdefaults voor schermbeveiliging, env-bestand 0600, startscript met DNS-wachtlus, systemd-gebruikersservice + nachtelijke cron. Vault-key nodig: **`netxms_kiosk_password`**.
- **Netwerkplaatsing herzien 2026-08-27 (user)**: monitor-schermen gaan naar **VLAN 73 `VL-INF-OTHER` (10.70.4.0/23)**, niet de beheerdersband. ⚠️ **User zei eerst "VLAN 72 (INFRA-OTHER)" — dat is een verspreking**: 72 = `VL-INF-VOIP`, 73 = `VL-INF-OTHER`. Bevestigd op 73. `PC-MONITORING-01` → **10.70.4.20**, gateway 10.70.4.1 (VLAN 73 had nog geen IP-plan; nu ingevuld in `vlans.md`). **Verhuizing nog te doen**: toestel stond op 10.34.0.25, vraagt switchpoort-wijziging.
- **IP-plan VLAN 73 (2026-08-27)**: gateway `.1`, **DHCP volledig uitgeschakeld** (user), dus `.2`-`10.70.5.254` volledig statisch. Aanleiding: de pool startte op `10.70.4.11` en overlapte met het geplande `10.70.4.20`. **Gevolg**: toestellen krijgen hier géén adres vanzelf — statisch configureren vóór het aansluiten, anders lijkt het op een kabelprobleem. `endpoints.md` is nu de enige registratie van gebruikte adressen (geen lease-overzicht meer). ⚠️ **User noemde dit netwerk eerst "VLAN 75"** — opnieuw een verspreking, bevestigd 73.
- **Firewall-regel A-012** toegevoegd aan de matrix: `ADMIN-WORKSTATIONS` → nieuwe host-list `KIOSK-DEVICES`, TCP 22, te activeren. ⚠️ De beheerwerkplek staat feitelijk in **VLAN 10 (10.10.150.27)**, niet in 34 — die host moet in `ADMIN-WORKSTATIONS` tot ze verhuist, anders bijt de regel niet.
- **⚠️ NIET UITGEVOERD — het toestel was op 10.34.0.25 onbereikbaar** vanaf het werkstation (10.10.150.27, VLAN 10) én vanaf VLAN 35 (monitoring-VM en jump-01): geen ICMP, geen poort 22. Dat is de bedoelde richting van het beheerpad (VLAN 34 → rest, niet omgekeerd). Ansible moet dus draaien vanaf een werkstation ín VLAN 34, óf er komt een gerichte firewall-uitzondering. Ook nog te doen: `ansible`-account op het toestel bootstrappen (vereist eenmalig een persoonlijk sudo-account daar).
- Codificatie-gotcha's: `gsettings` vereist een sessie → dconf-systeemdefaults; `systemctl --user enable` vereist `XDG_RUNTIME_DIR` → wants-symlink zelf leggen; op Debian is het `daemon.conf`, niet `custom.conf`.

### Toestel in beheer 2026-08-27 — bijna klaar
- Firewall **A-012 actief**, toestel verhuisd naar **10.70.4.20** (VLAN 73). Vanaf het werkstation: ICMP + poort 22 OK, `ansible`-account werkt met de canonical key, sudo → root. Debian **13.6**, gdm actief.
- **Hostnaam was `PC-ITMONITOR`** (weer een afwijking van de documentatie, zoals eerder `SRVV-MONITOR-01`) → hernoemd naar **`PC-MONITORING-01`**, ook in `/etc/hosts` en de FQDN.
- **⚠️ Toestel is hergebruikt en draagt ballast mee** — op ALLE interfaces: Webmin **10000**, Usermin **20000**, xrdp **3389**, postfix **25**; plus postfix én exim4 tegelijk actief, en `/etc/mailname` nog op de oude naam. xrdp geeft toegang tot precies de sessie die straks automatisch aanmeldt. **User: eerst kiosk werkend krijgen** → vastgelegd als tracker **SEC-5** + opruimlijst in `endpoints.md`.
- Chromium stond er nog niet; `kiosk.yml` installeert dat.
- **Nog te doen**: `ansible-playbook kiosk.yml --diff --ask-vault-pass` draaien (vault-key `netxms_kiosk_password` staat er), daarna **reboot** — automatisch aanmelden werkt pas na herstart en de kiosk-service start bij de volgende grafische sessie.

## Volgende stap
1. `netxms.yml` nog één keer draaien ter bevestiging van idempotentie (alles ok, verify groen).
2. Eerste login op `https://netxms.olvp.int/` vanaf VLAN 34/10.x, admin-wachtwoord roteren, persoonsgebonden beheerdersaccount. Management-node is hernoemd naar `SRVV-MONITORING-01`; die hangt nu zowel automatisch onder *Virtuele Servers* als handmatig onder *linux* — dubbeling nog op te ruimen.
3. `netxms-agent.yml` uitrollen (agents rapporteren aan .20 én legacy 10.10.100.2).
4. SNMP op UniFi-devices, alert-routes (e-mail eerst).
5. Oude server `10.10.100.2` inventariseren + uitfaseren; daarna `netxms_legacy_server` leegmaken.
6. Firewall-regels bevestigen: VLAN 34 → .20 (22/443/4701/9090), .20 → agent-hosts :4700 + devices :161/udp.
7. Branches `netxms-fase1` in beide repos mergen na review.

## Gerelateerd
[[project-forgejo-status]] (service-patroon), [[project-infrastructure-params]] (VLAN 35 / IP-plan), [[project-internal-pki-coverage]] (step-ca cert + renewal), [[project-podman-log-hygiene]] (log-hygiene op dezelfde VM), [[project-firewall-strategy]] (nieuwe regels), [[feedback-test-from-user-vlan]] (verificatie vanaf het juiste VLAN).
