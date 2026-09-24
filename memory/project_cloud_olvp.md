---
name: project-cloud-olvp
description: "Langetermijnwerf gestart 2026-09-24: cloud@olvp — een soevereine digitale werkplek op eigen infrastructuur. Bestandsplatform + online office + mail/agenda/contacten + chat + foto's + ELO, alles BigTech-vrij, open source, één account via Keycloak. Ook Europese externe hosting onderzoeken. Project J in de programmastructuur."
metadata:
  type: project
---

**Gestart 2026-09-24 (user-verzoek)**, na de geslaagde PoC van
[[project-eurooffice-nextcloud]]. Die PoC is de eerste bouwsteen; deze werf is het geheel.

## Doel

Eén digitale werkplek voor OLVP op eigen infrastructuur, waar een gebruiker met **één account**
bij alles komt. Harde randvoorwaarden van de user: **BigTech-vrij, open source, zelf te hosten**.
Offline/desktopvarianten zijn een pluspunt en worden getest.

| Functie | Kandidaat | Stand |
|---|---|---|
| Bestanden | OpenCloud **of** Nextcloud | beide draaien als PoC, keuze open |
| Online office | Euro-Office | draait, gedeeld door beide sporen |
| Mail | Open-Xchange | te onderzoeken |
| Agenda + contacten | Open-Xchange | te onderzoeken |
| Chat | nog te kiezen (Element/Matrix ligt voor de hand) | open |
| Foto's | Immich | te onderzoeken |
| ELO | Moodle | al belegd als APP-1 |
| Hosting | eigen Proxmox **of** Europese partij | te onderzoeken |

## ⚠️ Dit bestaat al als referentiearchitectuur — begin daar

**openDesk** (ZenDiS, Duitse overheids-GmbH onder het Bundesinnenministerium, v1.0 sinds oktober
2024) is vrijwel exact deze stapel: Nextcloud (bestanden) + Collabora (office) + **Open-Xchange**
(mail/agenda/contacten) + Element/Matrix (chat) + Jitsi (video) + OpenProject + XWiki, **gelijmd met
Keycloak-SSO**. Dat is geen toevallige gelijkenis: het is hetzelfde probleem, opgelost door een
overheidsorganisatie met een onderhoudsverplichting.

**How to apply:** neem openDesk als vertrekpunt en wijk er bewust van af, in plaats van de stapel
opnieuw uit te vinden. Wat OLVP extra wil (Immich, Moodle) hangt er los naast en kan er zonder
conflict bij. Wat openDesk anders doet (Collabora i.p.v. Euro-Office) is een meting waard, geen
principekwestie — zie de sessielimiet-vraag in [[project-eurooffice-nextcloud]].

## 🔑 De vier dingen die deze werf laten slagen of stranden

### 1. Eén account = elke app moet OIDC spreken. Dat is het selectiecriterium, niet een detail.

Keycloak draait al (`SRVV-ID-01`, realm `olvp`, AD-federatie met 3075 users — zie
[[project-identity-architecture]]). **Toets elke kandidaat eerst op OIDC** vóór je naar functies
kijkt. Een app die alleen LDAP of een eigen gebruikersdatabase kent, kost je het hele
"één account"-doel. Voor zover nu bekend spreken OpenCloud, Nextcloud, Open-Xchange, Immich, Moodle
en Element allemaal OIDC — **te verifiëren per app en per versie**, niet aannemen.

### 2. 🔴 De groeps-SoT-vraag moet NU beslist worden, niet per app.

Wie is de bron van waarheid voor klassen, vakgroepen en lidmaatschappen: **AD** of **MySchool**?
Die vraag staat al open in [[project-moodle-nextcloud-hosting]] en is bij OpenCloud al eens
beantwoord "in het voordeel van de AD-keten" omdat het geen OCS-provisioning-API heeft.

Met één app is dat een implementatiekeuze. Met **zeven** apps die allemaal groepen consumeren is het
een architectuurbeslissing, en hem laat beslissen door de toevallige beperking van de eerst
uitgerolde app is de duurste variant. Elke app die je erbij zet vóór dit vastligt, moet je later
opnieuw aansluiten.

### 3. 🔴 Mail is de zwaarste en hoort als LAATSTE.

OLVP draait op Google Workspace; personeel zit op Gmail, `olvp.be` MX → Google. Mail overnemen is
niet "nog een app": het is postvakmigratie, deliverability (SPF/DKIM/DMARC, IP-reputatie),
spamfiltering en een hersteltijd van minuten in plaats van uren. Het is ook het meest zichtbare: een
uur mailstoring merkt iedereen, een uur Immich-storing niemand.

**Aanbeveling:** doe mail bewust laatst, en overweeg voor juist dit onderdeel een **Europese
managed-mailpartij** in plaats van zelf hosten. Dat is geen concessie aan de soevereiniteitsdoelstelling
— het verplaatst de afhankelijkheid van een Amerikaanse naar een Europese partij — maar wel aan de
werklast, en die is hier de schaarse grootheid (zie punt 4).

### 4. 🔴 De bus-factor is het grootste praktische risico, niet de techniek.

Zeven zelfgehoste diensten voor 3000+ gebruikers, beheerd door één persoon plus een collega in
opleiding ([[project-bus-factor]]). Elke dienst brengt updates, certificaten, backups, storingen en
een DPIA mee. De techniek is haalbaar; de **exploitatielast** is het echte schaalprobleem.

Dat pleit voor drie dingen, in deze volgorde: **minder apps tegelijk**, **managed Europese hosting
voor de moeilijkste onderdelen**, en **automatisering waar OLVP al sterk in is** (Ansible +
Semaphore + NetXMS). Reken er niet op dat het vanzelf meevalt omdat de eerste PoC vlot ging — die
ging vlot omdat het één dienst was met een runbook eromheen.

## Externe Europese hosting — scherpere criteria dan "staat in Europa"

De user wil dit onderzoeken, mits het **in hoge mate integreerbaar** blijft met de eigen werking
(automatisering, monitoring). Dat is een goed criterium; het sluit meer uit dan het lijkt.

**Waar op te toetsen:**

1. **Eigendom en jurisdictie, niet datacenterlocatie.** Een Europees datacenter van een
   VS-moederbedrijf valt nog steeds onder de CLOUD Act. "EU-region" is geen soevereiniteit. Toets de
   **aandeelhoudersstructuur**, niet de landkaart.
2. **API + Ansible + SSH.** Alles wat je niet met je eigen automatisering kan aansturen, wordt een
   handmatig eiland. Pure SaaS zonder API valt daarmee af voor alles wat je zelf wil beheren.
3. **Monitoring van binnenuit.** Kan de NetXMS-agent of een Prometheus-exporter erop draaien
   ([[project-netxms-monitoring]])? Zo niet, dan zie je storingen pas wanneer een gebruiker belt.
4. **Backup onder eigen regie.** Kan je er zelf uit halen wat je nodig hebt, of ben je afhankelijk
   van hun herstelproces? Sluit aan op [[project-offline-backup-chain]].
5. **Verwerkersovereenkomst + DPIA.** Leerlingendata bij een derde partij: juridisch verplicht, en
   het is de kern van de motivatie onder deze hele werf.

Kandidaten om te bekijken (niet geverifieerd, startpunt): IONOS (zit in het Euro-Office-consortium),
OVHcloud, Hetzner, Scaleway, Exoscale, en de Nextcloud-partnerlijst.

## Verder mee te nemen

- **Opslag en backup schalen niet vanzelf mee.** Immich voor een school is een fotoarchief dat
  jaarlijks groeit en nooit krimpt. De huidige offline keten is 1,1 TB op roterende USB-schijven
  ([[project-offline-backup-chain]]) — dat past niet op een fotoarchief. Capaciteitsplan vóór
  Immich, niet erna.
- **Immich is volop in ontwikkeling** en kondigt zelf breaking changes aan. Voor een testopstelling
  prima; voor een archief waar herinneringen in gaan een reden om het versiebeleid strak te zetten
  en restores echt te oefenen.
- **Offline/desktop hangt vast aan werf K.** "Offline beschikbaar" betekent in de praktijk:
  desktop-sync-client (Nextcloud/OpenCloud) plus LibreOffice als lokale editor. Dat is precies de
  werkplek uit [[project-linux-werkplekken]]. De twee werven delen dit stuk; behandel ze niet los.
- **Volgorde-advies:** bestanden + office (draait) → ELO (Moodle, APP-1 bestaat al) → foto's (Immich;
  weinig integratie, laag risico, goede oefening) → chat → agenda/contacten → **mail als laatste**.
- **Per app een DPIA-entry** vóór er echte leerlingendata op komt, niet erna. `governance/dpia.md`.
- **Naamgeving:** de user schreef "eurocloud"; het product heet **Euro-Office**. Voor de werf zelf is
  `cloud@olvp` de gekozen naam — let bij VM- en FQDN-naamgeving op de conventie uit
  [[project-naming-convention]].

Gerelateerd: [[project-eurooffice-nextcloud]] (de PoC), [[project-moodle-nextcloud-hosting]] (APP-1/2),
[[project-identity-architecture]] (Keycloak), [[project-linux-werkplekken]] (werf K),
[[project-bus-factor]], [[project-programma-structuur]].
