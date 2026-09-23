---
name: project-frame-frappe-hosting
description: "FRAME = de werfnaam voor het Frappe-spoor bij OLVP (8 eigen Frappe v16-apps van de leverancier) op de OLVP-stack, model B bench-per-env/site-per-tenant met Podman-Quadlet. Vier omgevingen dev/test/acc/acc-test (beslist 2026-09-21) plus de kále Frappe SRVV-FRAPPE-02 als naam-uitzondering. Fase 0 en Fase 1 e2e bewezen (dev-bench, 2026-09-21); code was teruggehaald uit gesloten PR #1."
metadata:
  type: project
---

**Kennisbank staat buiten deze repo:** `/home/demm/OwnProjects/melira-project/hosting/`
(acht genummerde documenten: context, architectuur, image-build, deployment, roadmap, gotcha's,
caddy, app-updates). Die map is het **ontwerpregister**; de uitvoerende code hoort in
`platform-ansible`. Lees bij twijfel eerst die map — ze is uitgebreider dan wat hier past.

## Wat het is

FRAME is de OLVP-werfnaam voor dit spoor. De leverancier levert **8 eigen Frappe v16-apps** op
GitLab (`git@gitlab.com:melira/core/*`, privé, SSH-only), met `melira_core` als verplichte basis
voor alle andere — die repo- en modulenamen zijn feiten over andermans systeem en blijven dus
staan. Bestuursbeslissing B3 (2026-07-04): **MySchool blijft Odoo**, dit wordt greenfield Frappe. Gehost achter dezelfde
HAProxy→Caddy→step-ca-keten als Odoo.

**Model B** (beslist 2026-07-08): één **bench** per omgeving, meerdere **sites** per bench.
Herevaluatie naar model A (volle stack per instance) voorbehouden als security om harde
inter-tenant-isolatie vraagt.

## ⚠️ De code stond niet in de repo

PR #1 (`melira-frappe-fase0`) is op **2026-08-25 gesloten zonder merge** en de branch is
verwijderd, terwijl de kennisbank `platform-ansible` als canonical bron aanwijst. Niemand wist
meer waarom; het lijkt niet opzettelijk. De commits waren nog ophaalbaar via
`git fetch origin refs/pull/1/head` — 28 bestanden, 1796 regels — en staan sinds 2026-09-21 op
branch `melira-frappe-fase1`. **Les:** een gesloten PR met verwijderde branch is niet weg, maar
hij is ook niet vindbaar voor wie er niet naar zoekt.

## Topologie (beslist 2026-09-21)

Vier omgevingen, gespiegeld op [[project-intranet-hosting]]. Blootstelling volgt de suffix:
`.olvp.be` publiek, `.olvp.int` intern.

| VM | bench | kanaal/branch | FQDN | expose |
|---|---|---|---|---|
| `srvv-dev-frame-01` | dev | `dev` | `frame-dev.olvp.be` | publiek |
| `srvv-tst-frame-01` | test | `tst` | `frame-test.olvp.be` | publiek |
| `srvv-acc-frame-01` | acc | **`prod`** | `frame-acc.olvp.int` | intern |
| `srvv-tst-acc-frame-01` | acc-test | `tst` | `frame-acc-test.olvp.int` | intern |
| `srvv-frappe-02` | prod | `prod` | `frappe.olvp.be` | publiek |

**Acc draait het prod-kanaal** omdat de ICT-dienst die omgeving zélf productief gebruikt, onder
meer voor projectbeheer met `melira_projects`. Acc-test is het voorportaal op het test-kanaal.
Exact het intranet-patroon. **Accountbeheer blijft tot nader order in intranet.**

**Naamregel:** noch de productnaam noch `frappe` in server-namen of FQDN's — beide worden `frame`.
Bevestigd 2026-09-21: in code, Podman-namen en mapnamen mág de productnaam blijven staan; FQDN's
niet.
`SRVV-FRAPPE-02` (kále Frappe met ERPNext-stack, eigen image, bestaat al als VMID 401) is een
bewuste uitzondering en behoudt haar naam. Daardoor blijft `frame.olvp.be` vrij voor de latere
publieke productieserver.

## Meerdere instances per server — twee niveaus

| Niveau | Isolatie | Wanneer |
|---|---|---|
| **Sites in één bench** | zwak (gedeelde runtime) | edities of klanten op dezelfde app-versie |
| **Benches op één VM** | sterk (eigen containerset, netwerk, `http_port`) | verschillende branches — het equivalent van Odoo's 8069/8070 |

De role kan beide: `tasks/main.yml` loopt over `benches`, `bench.yml` over `sites` (geverifieerd).

**Waarom acc en acc-test tóch aparte VM's:** een bench draagt **één versie van de app-code** (apps
worden op bench-niveau uit git gehaald op de branch van het kanaal), dus twee omgevingen kunnen
geen twee sites in dezelfde bench zijn. Als twee benches op één host zou het 18 containers en twee
MariaDB's worden — te krap op 8 GB.

## Oude VM's opgeruimd (2026-09-21)

`SRVV-FRAPPE-01` (400), `SRVV-DEV-FRAPPE-01` (10050) en `SRVV-TST-FRAPPE-01` (10051) zijn
**verwijderd** — opnieuw klonen uit de geseald baseline is sneller dan ze bijwerken.
`SRVV-FRAPPE-02` (401, kále Frappe) **blijft bestaan**.

10050 en 10051 hadden elk vier backups in `PROXMO_BU`; **400 had er geen enkele**, dus daar is
vooraf een `vzdump` van gemaakt (3,75 GB) zodat het besluit omkeerbaar blijft.

⚠️ 401 is nu de enige VM in dit spoor die **niet** uit de nieuwe baseline komt en dus nog de oude
template-fouten draagt: gedeelde machine-id, scheve indeling, geen versiemarkering. Neem hem mee
in fase 9 van [[project-template-fleet-defects]] of kloon hem alsnog opnieuw.

## Stand

| Fase | Status |
|---|---|
| **0** — image-build (Frappe 16 + 8 apps, ~2,16 GB) + dev-site | ✅ e2e bewezen 2026-07-08 |
| **1** — `roles/frappe-podman`, SoT, `caddy-frappe.yml`, topologie | ✅ **e2e bewezen 2026-09-21** op de dev-bench |
| **2** — Keycloak-OIDC, MariaDB-backup, branch-sync-wiring | ⬜ open |

**De blocker is weg:** Fase 1 kon niet geverifieerd worden omdat er geen VM's waren en de
Tier-1-master nog in aanbouw was. Die master is geseald op 2026-09-21
([[project-template-fleet-defects]]) — klonen kan.

## ▶ Eerste bench draait (2026-09-21)

`SRVV-DEV-FRAME-01` (VMID 10050, 10.200.14.51): negen containers gezond, site
`frame-dev.olvp.be` met database en db-gebruiker, `/api/method/ping` → `pong`,
`/login` → 200, `developer_mode: 1`.

Bewust met een **runtime-only image** (`melira-frappe:16-runtime`, gebouwd met
`MELIRA_APPS_JSON=/dev/null`) en een **lege app-lijst**: zo testte de run alleen
de role, niet de apps-laag. Geen GitLab-sleutel nodig. Overgezet met
`docker save | ssh 'sudo podman load'` — **let op die `sudo`**: Quadlet-units zijn
systeem-units en gebruiken de rootful store, niet die van de ansible-gebruiker.

## ▶ Gedaan op 2026-09-22

1. **Idempotentie-check verscherpt** (commit `9bf54b2`). De check keek naar
   `site_config.json`, dat bench vóór de database schrijft. Nu spreekt ze de database
   zelf aan: `bench --site X execute frappe.db.get_database_size` — gemeten op de
   dev-bench rc 0 met de grootte, rc 2 op een site die niet bestaat. Draait als
   wegwerpcontainer, dus ook bruikbaar als de backend plat ligt. Een halve site wordt
   **niet** automatisch opgeruimd: dat wist een database, dus de taak stopt met het
   logpad erbij.
2. **Apps-laag gewired** (commit `f8f296a`) — de twee stukken waar `frappe-update.yml`
   zelf op wachtte:
   - `apps`-volume per bench, met dezelfde seeding als `sites/`. ⚠️ Dit is gotcha 2 in
     het kwadraat: **het framework zelf woont in `apps/frappe` in het image** (463 MB,
     geverifieerd), dus een lege bind-mount op dat pad maskeert frappe en er start
     niets meer. Eerst `cp -an` uit het image, dan pas mounten — in alle vijf de
     container-units.
   - read-only GitLab deploy-key per VM in `/opt/melira/ssh` (0600, uid 1000) uit
     `vault_frappe_gitlab_deploy_key`; host-sleutels vooraf vastgelegd met `ssh-keyscan`
     op de controlmachine in plaats van blind `accept-new` op de VM.
   - `frappe-update.yml` draait de sync nu in een **wegwerpcontainer** met de sleutel
     read-only erin, zodat de backend die maanden loopt hem niet draagt. Het image komt
     uit de draaiende backend, dus de sync werkt nooit met een andere runtime.
3. **Units die wijzigen worden herstart.** Stond op `started`, dus een draaiende
   container bleef op zijn oude mounts hangen en de bench liep iets anders dan wat in
   de repo stond. Zonder die fix zou de apps-volume-wijziging stil niet aankomen.
4. **De vier VM's staan nu in `hosting/reference/vm-inventory.md`** (handbook `c87033c`).

**Nog niet gedraaid**: de rol is sinds deze wijzigingen niet uitgevoerd. De dev-bench
draait nog zonder apps-volume. Eerste run doet drie dingen tegelijk — apps-map vullen,
units herschrijven, containers herstarten — dus doe hem met `--limit srvv-dev-frame-01`
en kijk daarna of de negen containers gezond terugkomen en `/login` nog 200 geeft.

**Blokkade voor de echte apps**: er is nog geen deploy-key. Zolang
`vault_frappe_gitlab_deploy_key` leeg is, rolt de stack uit maar kan `frappe-update.yml`
de private Melira-repo's niet ophalen.

## ✅ FRAME draait met de echte apps (2026-09-22)

`SRVV-DEV-FRAME-01`: **acht Melira-apps op branch `dev` geïnstalleerd** op
`frame-dev.olvp.be`, negen containers gezond, `/api/method/ping` → `pong`,
`/login` → 200. Apps 497 MB, venv 615 MB, sites 328 KB.

**Toegang = groeps-deploy-token, geen deploy-key.** GitLab kent deploy-keys enkel per
project; op groepsniveau bestaat alleen een deploy-**token**. Eén token op `melira/core`
met scope `read_repository` dekt alle acht, is met één handeling in te trekken en kan
een vervaldatum dragen. Het token staat in de vault (`vault_frappe_gitlab_deploy_token`
+ `_user`) en belandt in `/opt/melira/git/credentials`, niet in de remote-URL — anders
schrijft `get-app` het in `.git/config` van elke app.

## Stand per host (2026-09-22)

| VM | bench | kanaal | apps | site | Caddy/TLS |
|---|---|---|---|---|---|
| `srvv-dev-frame-01` | dev | `dev` | ✅ 8 | ✅ 200 | ✅ HTTPS 200, cert t/m 29/09, renew-timer |
| `srvv-acc-frame-01` | acc | `prod` | ✅ 8 | ✅ 200 | ✅ HTTPS 200, idem |
| `srvv-tst-frame-01` | test | `test` | ✅ 8 | ✅ 200 | ✅ HTTPS 200, idem |
| `srvv-tst-acc-frame-01` | acc-test | `test` | ✅ 8 | ✅ 200 | ✅ HTTPS 200, idem |

**Fase 1 is af (2026-09-22)**: vier omgevingen, elk negen containers, acht Melira-apps op
hun kanaal, Caddy met een step-ca-cert en een actieve renewal-timer. Alles vanaf de
Tier-1-baseline, volledig uit code.

⚠️ **Gotcha bij het uitgeven van certs**: `step ca certificate` schrijft naar het pad dat je
opgeeft. Drie keer achter elkaar naar `/tmp/cert.crt` laat er één over — en welke merk je pas
als je de subject leest. Geef per FQDN een eigen bestandsnaam. En controleer de FQDN zelf:
test is `.olvp.be` (publiek), acc en acc-test zijn `.olvp.int`.

**Het kanaal heet `test`, niet `tst`** (commit `ed1055a`). De repo's dragen `dev`, `test`,
`prod` en `main`; `test` staat in alle acht op dezelfde commit als `prod`, dus test begint
gelijk aan wat acc draait en elke wijziging erin is een bewuste promotie.

⚠️ **How to apply bij zo'n meting**: mijn eerste controle greppte op `refs/heads/tst$` en
concludeerde "de branch bestaat niet". Dat was letterlijk waar en volstrekt misleidend — de
vraag had moeten zijn wélke branches er staan. Ik heb op grond daarvan bijna acht overbodige
branches in de repo's van de leverancier gemaakt; er staat er nu één teveel op `melira_core`
(`tst`, zelfde commit als `prod`/`test`), te verwijderen.

**Image-distributie**: `ssh <bron> 'sudo podman save <img>' | ssh <doel> 'sudo podman load'`
— 2,46 GB, een paar minuten. Het runtime-image is sinds deze dag de **default**
(`frappe_image`), niet langer een per-VM-override.

**step CLI + cert per FQDN blijven handwerk.** De binary staat op de CA-host
(`/usr/bin/step`, 0.30.2) en gaat er via Ansible naartoe; het initiële cert vraagt het
admin-provisioner-wachtwoord uit KeePassXC — dat staat **niet** in de Ansible-vault.
Laat `--not-after 24h` uit het commando weg: de CA geeft nu 168 u en die marge wil je.

⚠️ **De step-ca draagt een ACME-provisioner** (`{"type":"ACME","name":"acme"}`). Daarmee kan
Caddy zijn certs volledig zelf ophalen én vernieuwen — geen eerste uitgifte met de hand,
geen renew-script, geen provisioner-wachtwoord. Voorwaarde is DNS, want de CA moet de host
op naam bereiken voor de http-01-challenge. Te beslissen vóór test/acc-test gebouwd worden;
het zou de klasse fouten wegnemen uit [[feedback-caddy-cert-renew-fail-isolation]].

## Edge + DNS (2026-09-22)

**frame-test is publiek, frame-dev bewust niet.** Een dev-omgeving met `developer_mode: 1`
hoort niet van buiten bereikbaar te zijn. HAProxy draagt alleen `be_srvv_tst_frame_01`; op
`frame-dev.olvp.be` antwoordt de edge 503. ⚠️ Het publieke A-record voor frame-dev bestaat
nog bij one.com en moet weg; interne toegang tot dev vraagt een AD-zone die naar
10.200.14.51 wijst.

| Laag | Stand |
|---|---|
| HAProxy-backend `be_srvv_tst_frame_01` | UP op beide nodes; Odoo + Keycloak ongemoeid |
| Firewall DMZ → 10.200.14.51-52:443 | ✅ getest vanaf beide HAProxy-nodes |
| Health-check `/api/method/ping` | 200; ook de **SNI-loze** handshake werkt (`default_sni`) |
| LE-cert `frame-test.olvp.be` | ✅ 2026-09-22, t/m 21/12, op beide nodes, extern verify 0 |
| `frame-acc` / `frame-acc-test` in `olvp.int` | ✅ op alle zes de DC's |

**Gotcha die al gedocumenteerd stond en die ik opnieuw ontdekte**: certbot draait de hooks in
`renewal-hooks/deploy/` alleen bij `certbot renew`, niet bij een verse `certonly`. Het cert
belandt dan wél in `/etc/letsencrypt/live/` maar nooit in `/etc/haproxy/certs/`. Handmatig:
`sudo env RENEWED_LINEAGE=/etc/letsencrypt/live/<fqdn> /etc/letsencrypt/renewal-hooks/deploy/haproxy-deploy.sh`.
Staat in `hosting/operations/deploy-sspr.md` — lees dat runbook vóór je een nieuw publiek
cert aanvraagt.

**AD-DNS-latentie**: een nieuw record op één DC staat pas na ~15 minuten op de andere vijf.
En de negatieve TTL van `olvp.int` is 3600 s, dus een te vroege query blijft een uur in de
cache van gateway en clients hangen. Meet met `dig` per DC én let op de `aa`-vlag; een leeg
antwoord is niet hetzelfde als NXDOMAIN.

## De vier volumes — en waarom er precies vier zijn

Bij on-VM git-sync moet **alles wat een build of install aanraakt** op de VM leven, niet in de
image-laag. Er zijn vier zulke plekken, en ze kwamen één voor één boven, elk met een eigen
symptoom:

| Volume | Wat er misgaat zonder | Symptoom |
|---|---|---|
| `sites/` | bench vindt `apps.txt` niet | élk bench-commando weigert |
| `apps/` | app-code verdwijnt | code weg na herstart |
| `env/` | editable pip-install verdwijnt | `ModuleNotFoundError` terwijl de code er staat |
| `assets/` | gebouwde CSS/JS verdwijnt | **404 op CSS, pagina zonder opmaak** |

Die laatste is de subtielste: `sites/assets` is geen map maar een **symlink** naar
`/home/frappe/frappe-bench/assets`, gelegd door de image-entrypoint. `bench build` schrijft
daar, dus zonder volume is het resultaat weg met de container — terwijl de manifest wél naar
de nieuwe bestandsnamen verwijst. De JS-bundle laadde nog omdat die inhoudelijk niet
veranderde en dus dezelfde hash hield; alleen de CSS kreeg een nieuwe hash door de app-styles.
**Half werkend ziet eruit als een toevallig probleem** — dat kostte de meeste tijd.

⚠️ **Gevolg voor image-upgrades**: een nieuw image brengt geen nieuwe `env/` of `assets/` meer
mee. Bij een upgrade moeten die mappen weg zodat de rol ze opnieuw vult, gevolgd door een
`bench build`. Zonder dat draai je nieuwe code op oude assets.

## Zeven fouten die pas een échte app-sync blootlegde

Alle zeven zijn generiek voor "apps uit git in plaats van uit het image":

1. **`bash -lc` wist het PATH.** Een login-shell leest `/etc/profile` en gooit de
   image-PATH weg, inclusief de nvm-node. `bench build` viel over `node: not found`
   terwijl node in het image zit. Met `bash -c` blijft de PATH staan.
2. **De venv zat in de image-laag.** `get-app` doet `uv pip install -e` in
   `frappe-bench/env`; leeft die in het image, dan verdwijnt de installatie met de
   wegwerpcontainer en meldt de backend `ModuleNotFoundError`. `env/` is nu een volume
   op de VM. ⚠️ Gevolg: een nieuw image brengt géén nieuwe venv meer mee — bij een
   upgrade moet die map weg.
3. **`apps/frappe` maskeren.** Het framework zelf woont in `apps/frappe` ín het image;
   een lege bind-mount daarop en er start niets meer. Eerst kopiëren, dan mounten.
4. **Credentials als los bestand mounten kan niet.** Git's store-helper schrijft terug
   via temp + rename, en rename over een bind-gemount *bestand* geeft EBUSY — ook
   read-write. Mount de **map**.
5. **De remote heet `upstream`.** `bench get-app` kloont met `--origin upstream`;
   hardcoded `origin` geeft "does not appear to be a git repository" op een repo die er
   gewoon staat.
6. **`get-app` bouwt te vroeg.** Het bouwt meteen de assets van de app die het binnenhaalt,
   en die build importeert álle apps uit `apps.txt`. Eén app die nog niet in de venv zit
   laat dat stranden, en wélke app je treft hangt af van de volgorde in de lus. Nu:
   `--skip-assets` in de lus, één `bench build` aan het eind.
7. **`bench setup requirements` is onbruikbaar hier.** Het maakt per app een App-object en
   doet `git.Repo()`; `apps/frappe` komt uit het image en heeft geen `.git` (frappe_docker
   stript dat) → `InvalidGitRepositoryError` op frappe zelf. Installeer de apps rechtstreeks.

**En één van mezelf**: het sync-script ging als `bash -c '<script>'` mee. Dan is élk
aanhalingsteken in dat script een tijdbom — `awk '{print $1}'` was genoeg om bash een
afgekapt script te geven. Nu via **stdin** (`bash -s`), zodat er geen buitenste quoting
meer is om stuk te maken. Zie [[feedback-toon-het-echte-commando]].

## ▶ Volgende stappen (stand einde 2026-09-21)

1. **Idempotentie-check verscherpen** — kijkt nu naar `site_config.json`, dat vóór de database
   geschreven wordt; een half mislukte run wordt daardoor overgeslagen. Beter: check op de database
   of op een marker die pas ná `bench new-site` komt.
2. **Apps-laag** volgens `frappe/decisions/0001`: apps-volume per bench, read-only GitLab-deploykey
   op de VM, `get-app` op de channel-branch. Dan pas draait FRAME met de echte Melira-apps; nu
   draait dev op een runtime-only image met een lege app-lijst.
3. **Caddy-edge** (`caddy-frappe.yml`) — nooit gedraaid. Initieel cert per FQDN blijft handwerk met
   de step-ca admin-provisioner; de playbook faalt met instructie als het cert ontbreekt.
4. **DNS** voor de vier FQDN's, en HAProxy voor de twee publieke.
5. **test, acc en acc-test** uitrollen zodra dev met apps draait.
6. **Vloot-inventaris**: de vier VM's staan nog niet in `hosting/reference/vm-inventory.md`.

**PR #2** (`melira-frappe-fase1` → `main`) staat open met elf commits en bevat óók de twee
frappe-vault-sleutels. Nog niet gemerged.

## Vijf vertaalfouten compose → Quadlet (alle gedicht)

De Fase 0-compose was bewezen, de vertaling ernaar niet. Alle vijf kwamen pas bij
de eerste echte run boven, en ze zijn generiek genoeg om bij elke
compose→Quadlet-migratie op te letten:

1. **YAML folded scalar houdt nieuwe regels** zodra een regel dieper inspringt.
   `bench new-site` werd daardoor zónder opties aangeroepen, vroeg interactief om
   het root-wachtwoord en stierf op `EOFError`. **Dit was de echte oorzaak van
   vier mislukte runs.**
2. **Bind-mount ≠ named volume.** Podman vult een named volume bij de eerste
   mount vanuit het image; een bind-mount maskeert de image-inhoud. Zonder
   `sites/apps.txt` weigert élk bench-commando.
3. **Datadirs van sidecars niet chownen.** mariadb en redis chownen hun datadir
   naar uid 999; zet Ansible dat terug op root, dan faalt de database-aanmaak met
   `errno 13` — een bestandssysteemfout die als SQL-rechtenprobleem leest.
4. **Wachtwoorden via Podman-secrets**, niet via de commandoregel en niet via
   `environment:` + `podman run -e VAR` (die bereiken het podman-proces niet
   onder `become`).
5. **`bench new-site` schrijft `site_config.json` vóór de database.** Een half
   mislukte run laat dus een site achter die compleet lijkt. ⚠️ De
   idempotentie-check kijkt nu naar dat bestand en is daarmee nog niet
   waterdicht — open punt.

Zie ook [[feedback-toon-het-echte-commando]] over hoe die vier ronden te
vermijden waren.

Gerelateerd: [[project-frappe-platform]] (de oorspronkelijke aankondiging),
[[project-intranet-hosting]] (het gespiegelde patroon), [[project-containerization]],
[[project-mcp-and-itsm]] (`melira_mcp` raakt aan het MCP-spoor).

## frame-test landt nu op Melira (2026-09-23)

`https://frame-test.olvp.be/` gaf de kale Frappe-login; de toepassing zelf zat op `/melira`. Website
Settings → `home_page` staat nu op **`melira`**, waarna `/` de Melira-SPA rendert (geverifieerd
achter de bench-nginx én achter Caddy: `<title>Melira</title>`).

Gezet zonder de UI, zodat het herhaalbaar is:

```
podman exec melira-test-backend bench --site frame-test.olvp.be execute \
  frappe.client.set_value --kwargs \
  '{"doctype":"Website Settings","name":"Website Settings","fieldname":"home_page","value":"melira"}'
podman exec melira-test-backend bench --site frame-test.olvp.be clear-website-cache
podman exec melira-test-backend bench --site frame-test.olvp.be clear-cache
```

⚠️ Dit is **site-data, geen code**: het zit in de database van deze site, niet in de Ansible-rol en
niet in `melira_core/hooks.py` (waar `home_page` uitgecommentarieerd staat). Een verse site krijgt
het dus niet vanzelf. Wil je dit op dev/acc/acc-test ook, dan moet het daar apart gezet worden — of
beter, in de app als `home_page = "melira"` in `hooks.py`, zodat elke installatie het erft. Zolang
dat niet gebeurt: bij een `bench drop-site` + opnieuw aanmaken is deze instelling weg.

Beide cache-clears zijn nodig: `clear-cache` alleen laat de oude website-routekaart staan.
