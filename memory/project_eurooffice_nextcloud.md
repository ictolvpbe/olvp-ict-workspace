---
name: project-eurooffice-nextcloud
description: "Nieuw spoor 2026-09-18: Euro-Office (EU-fork van ONLYOFFICE Document Server) self-hosted als publieke testserver, met twee sporen: Nextcloud (A) en OpenCloud (B) op een gedeelde document-server. Tracker APP-2/APP-4/APP-5. Ontwerp + runbook RB-2026-NC-EO-DEPLOY klaar, niets gebouwd."
metadata:
  type: project
---

**Gestart 2026-09-18 (user-verzoek)** — onderzoeken of en hoe een self-hosted **Euro-Office** +
bestandsplatform-testserver opgezet kan worden, bereikbaar vanaf internet.

## Wat Euro-Office is

AGPLv3-**fork van ONLYOFFICE Document Server**, gelanceerd **9 juni 2026** door een EU-consortium
(Nextcloud, IONOS, XWiki, OpenProject, Eurostack, Abilian, Soverin, BTactic) voor Europese digitale
soevereiniteit. Image `ghcr.io/euro-office/documentserver:9.3.1`, poort 80, JWT-secret verplicht,
min. 4 GB RAM + 5 GB disk, healthcheck `/healthcheck` → `true`.

**Het is géén bestandsplatform** — enkel een editor-backend. Heeft altijd een host-platform nodig.

## Platformkeuze — BESLIST 2026-09-18: twee sporen testen

De user zei "ownCloud", maar dat is vandaag drie dingen. Voorgelegd en beslist:
- ✅ **Spoor A = Nextcloud** — Euro-Office ís het Nextcloud-coalitieproduct, officiële
  `eurooffice`-connector-app, best getest pad. Sluit aan op [[project-moodle-nextcloud-hosting]] (APP-2).
- ✅ **Spoor B = OpenCloud** (opencloud.eu) — toegevoegd 2026-09-18 op user-verzoek na eigen
  opzoekwerk: technisch next-gen (Go-microservices, géén database, Spaces). Eigen VM, eigen FQDN,
  **gedeelde** Euro-Office document-server (die spreekt connector-API én WOPI tegelijk).
- ❌ ownCloud Infinite Scale — eigendom van **Kiteworks (VS)** sinds maart 2025; botst met de
  sovereiniteitsmotivatie.

**How to apply:** Nextcloud koppelt via de connector-app (`occ app:install eurooffice` + gedeeld
JWT-secret); OpenCloud via de aparte `collaboration`-service met `COLLABORATION_APP_PRODUCT=OnlyOffice`.
Twee verschillende paden naar dezelfde document-server — niet door elkaar halen.

## 🟡 Sessielimiet — gedegradeerd van gate naar release-controle

ONLYOFFICE CE had een limiet van **20 gelijktijdige sessies**, pas geschrapt in **9.4**; Euro-Office
staat op **9.3.1**. **Beslist 2026-09-18 (user): niet relevant voor een testserver** → geen blokkade.
Maar wél een go/no-go voor productie op schoolschaal. **Bij élke nieuwe Euro-Office-release
hercontroleren** en noteren in de wijzigingslog van de runbook. Blijft hij op 20 → Collabora Online.

## OpenCloud-vooronderzoek 2026-09-18 — geen dealbreaker, 4 randvoorwaarden

1. **✅ Euro-Office wérkt erop.** Vergelijkingsartikels beweren "alleen Collabora" — achterhaald. De
   docs noemen `COLLABORATION_APP_PRODUCT=OnlyOffice` en issue opencloud-eu#2180 is gesloten. Wel een
   extra bewegend deel (de `collaboration`-service moet je apart starten).
2. **🔴 Géén OCS Provisioning-API** — dit is de echte impact. OpenCloud gebruikt **LibreGraph**
   (MS-Graph-achtig), en users/groepen aanmaken vereist een **schrijfbare LDAP onder eigen beheer**.
   Op prod-AD schrijven wil je niet → praktisch blijft **autoprovisioning** over (Keycloak houdt de
   users, groepen via de `groups`-claim). Dat ís de "scenario 1"-fallback uit
   [[project-moodle-nextcloud-hosting]]. **OpenCloud kiezen = de groeps-SoT-vraag beslissen in het
   voordeel van de AD-keten**; de directe backend-task-sync (optie b) vervalt. Sluit wel mooi aan op
   [[project-ad-member-delegation]].
3. **🟢 Backup — OPGELOST 2026-09-18.** Zie de backup-sectie hieronder; geen blokkade meer.
4. **🟠 PosixFS vs DecomposedFS = eenrichtingsdeur** — géén migratiepad, overstappen = data kopiëren
   naar een verse installatie. Kies meteen **PosixFS non-collaborative** (de huidige default).
   Collaborative mode draagt een expliciete waarschuwing in de docs → niet aanzetten.

## Groupware bij OpenCloud — nagekeken 2026-09-18 (corrigeert een eerdere te stellige uitspraak)

- **Calendar + Contacts: ✅ bestaan al** in de community-editie sinds mei 2025, via **Radicale**
  (CalDAV/CardDAV). Wel vooral een *server* — gebruik vanuit Outlook/Thunderbird/Apple Calendar;
  geen rijke web-UI zoals Nextcloud Calendar.
- **Mail: 🚧 roadmap 2026** — groupware-module (mail+agenda+contacten) op **JMAP** + **Stalwart**-
  backend, Go/TypeScript, open source. FOSDEM 2026-talk, werk loopt.
- **Video/chat: 🚧 roadmap** — **OpenTalk** wordt geïntegreerd; zelfde moederbedrijf (Heinlein Group).
- **Deck: ❌** geen equivalent; ecosysteem-antwoord = **OpenProject** als aparte app.
- ✅ **Editie-vraag BEANTWOORD (user, 2026-09-18):** de gratis **Community Edition bevat de volledige
  functionele basis, groupware inbegrepen**. Enterprise onderscheidt zich op geavanceerde
  beheerdersfuncties, schaalbaarheidsgaranties, enterprise rechtenbeheer en professionele support —
  niet op eindgebruikersfunctionaliteit. Het persbericht ("voor bedrijven, providers en publieke
  sector") wekt ten onrechte de indruk dat het betalend wordt. ⚠️ Deze bevestiging komt van de user,
  niet uit een publieke bron die wij terugvonden — hercontroleren bij de effectieve release.

**Alternatief-patroon = openDesk** (ZenDiS, Duitse overheids-GmbH, v1.0 okt 2024): best-of-breed
achter één IdP i.p.v. één monoliet — Nextcloud (files) + Collabora (office) + Open-Xchange
(mail/agenda/contacten) + Element/Matrix (chat) + Jitsi (video) + OpenProject (≈Deck) + XWiki (wiki),
**gelijmd met Keycloak-SSO**. Past beter bij [[project-identity-architecture]] dan "alles in één app".

**Relativering voor OLVP (gecorrigeerd 2026-09-18 door de user):** de school draait op **Google
Workspace**, NIET op M365 als productiviteitssuite — personeel zit op Gmail, `olvp.be` MX → Google;
de M365 A5-licenties zijn voor leerlingen + Intune, zonder EXO-mailboxen (staat ook zo in
`governance/security-testing-plan.md`). Mail/agenda/contacten/chat/video zijn dus al gedekt →
**geen korte-termijnnood** voor groupware bij OpenCloud.

Wat dat niet wegneemt: Google is even Amerikaans als Microsoft. De soevereiniteitsvraag verschuift
enkel in tijd. Volgorde-denken dus: **welke stukken eerst soeverein** — bestandsopslag + office-suite
eerst, want daar zit de leerlingendata in; agenda en chat veel minder.

## Architectuur (ontworpen, nog niet gebouwd)

Twee publieke FQDN's zijn **verplicht** — de editor laadt in de browser van de eindgebruiker
rechtstreeks vanaf de document-server, dus die kan niet achter de firewall blijven:

```
cloud-test.olvp.be + office-test.olvp.be → DNAT → VIP 10.21.0.5
   → HAProxy (2 LE-certs, be_keycloak-patroon) → Caddy (step-ca, 2 site-blocks)
   → Nextcloud 127.0.0.1:8080 / Euro-Office 127.0.0.1:8081
```

VM `SRVV-TST-CLOUD-01`, VLAN 207, **10.200.14.45** (⚠️ IP nog te bevestigen), 4 vCPU / 8 GB / 120 GB.
Spoor B krijgt een tweede VM `SRVV-TST-OCLOUD-01` op **10.200.14.46** + `ocloud-test.olvp.be`.
Eigen top-level inventory-groep `cloud_servers` (niet onder `webapps` — anders erft het Odoo-vars,
zelfde redenering als `sspr_servers`).

**Server-naar-server hairpin** opgelost met split-horizon `AddHost=`-entries (zelfde patroon als
`odoo_extra_hosts` voor de OIDC-back-channel) + step-ca root in beide container-truststores +
`ALLOW_PRIVATE_IP_ADDRESS=true` op de document-server.

## Backup — BESLIST 2026-09-18: twee lagen

User bevestigde dat **Proxmox-VM-backup geen probleem is**. Daarmee:
- **Proxmox snapshot-backup = de consistentie-laag** (volledig herstel, gegarandeerd consistent)
- **Bacula file-level = de granulaire laag** (één gewiste map terugzetten — wat een VM-image niet kan)

**Nextcloud:** Bacula file-level werkt zonder voorbehoud — data-dir + `config.php` + een `pg_dump`
die vlak vóór de run naar schijf geschreven wordt.

**OpenCloud — 🔴 de valkuil:** het heeft **geen database**; shares, Spaces-lidmaatschap, versies en
tree-info zitten in de **extended attributes van elk bestand**. Bacula neemt xattrs **niet standaard**
mee → zonder `xattrsupport = yes` (+ `aclsupport = yes`) in de FileSet draait de backup groen,
restaureer je de bytes en krijg je bestanden zónder metadata terug. Stille mislukking, pas zichtbaar
wanneer je hem nodig hebt. Past metadata niet in de xattrs, dan schrijft OpenCloud een sidecar-bestand
naast het origineel — dat moet mee in dezelfde FileSet.

**Wat NIET mee hoeft:** de ID-cache (PosixFS houdt die in NATS/JetStream, buiten het filesystem) en
de zoekindex — OpenCloud scant bij opstart de boom en herbouwt beide. Wel één keer écht testen.

**How to apply:** bij élke Bacula-FileSet voor een database-loze store (OpenCloud, maar denk ook aan
andere xattr-afhankelijke apps): `xattrsupport = yes` expliciet zetten, en de restore-test afsluiten
met een controle op de **deelrechten**, niet enkel op de bestanden.

## Status + waar het staat

- Runbook **RB-2026-NC-EO-DEPLOY** → `platform-handbook/hosting/operations/deploy-nextcloud-eurooffice.md`
  (8 fasen, 11 gotcha's, validatie/rollback/troubleshooting).
- Tracker: APP-2 bijgewerkt + nieuwe werf **APP-4** (Euro-Office) in `governance/tracker-additions.md`.
- **Nog niets gebouwd** — geen VM, geen DNS-record, geen cert.

Gerelateerd: [[project-moodle-nextcloud-hosting]], [[project-hosting-fase1-status]],
[[project-identity-architecture]], [[project-gateway-cutover]] (welk WAN-IP draagt de DNAT vandaag?),
[[feedback-caddy-default-sni-for-haproxy-check]], [[feedback-docker-to-podman-migration]],
[[feedback-test-from-user-vlan]].
