---
name: project-linux-werkplekken
description: "Langetermijnwerf gestart 2026-09-24: eindgebruikerswerkplekken van Windows (leerkrachten/administratie) en Chromebooks (leerlingen) naar Linux waar het kan, Windows waar het echt moet. Chromebooks uitfaseren of met een anoniem Google-ID tegen cloud@olvp. Project K in de programmastructuur."
metadata:
  type: project
---

**Gestart 2026-09-24 (user-verzoek)**, als tegenhanger van [[project-cloud-olvp]].

## Doel

Op termijn overschakelen naar **Linux-toestellen**, met Windows alleen waar het echt niet anders
kan. Chromebooks worden ofwel uitgefaseerd, ofwel gebruikt met een **anoniem Google-ID** waarbij al
het werk in cloud@olvp gebeurt.

Huidige stand: leerkrachten en administratie op Windows, leerlingen op Chromebooks.

## 🔴 De volgorde-afhankelijkheid die alles bepaalt

**Werf J (cloud@olvp) moet eerst bruikbaar zijn.** Een Linux-werkplek is pas verdedigbaar als
bestanden, office, mail en agenda in de browser werken. Doe je werf K eerst, dan vraag je gebruikers
om Windows op te geven zonder dat er iets voor in de plaats staat — en dan is één slechte ervaring
genoeg om het hele spoor politiek dood te maken.

Omgekeerd geldt: is J er, dan wordt K aanzienlijk makkelijker, want dan is het besturingssysteem nog
maar een schil rond een browser en een sync-client.

## De drie groepen zijn drie verschillende projecten

| Groep | Moeilijkheid | Waarom |
|---|---|---|
| **Leerlingen** (Chromebooks) | laag–midden | werken nu al vrijwel volledig in de browser |
| **Leerkrachten** (Windows) | **hoog** | **digibordsoftware** — ~145 installaties, veruit de zwaarste post |
| **Administratie** (Windows) | **laag–midden** | kernsoftware is **webbased**; resteert randapparatuur |

> ⚠️ **Deze tabel is op 2026-09-24 omgekeerd na meting.** De eerste versie zette de administratie op
> *hoog* wegens vermoede Windows-only vakapplicaties. GLPI vond er over 11.159 titels **geen enkele**,
> en de user bevestigde: Questi, Informat en Smartschool zijn alle drie webbased; de
> `Smartschool Management Tool` is enkel voor ICT. **Gevolg:** een administratieve werkplek is niet
> de moeilijkste pilot maar een van de kansrijkste. Laat deze correctie staan — ze herinnert eraan
> dat de moeilijkste groep niet noodzakelijk zit waar je hem verwacht.

**Begin dus met een inventaris, niet met een distributiekeuze.** De vraag "welke Linux" is de minst
interessante van deze werf; de vraag "welke toepassing houdt ons tegen" is de enige die telt.

**Te inventariseren, per toepassing: draait het in een browser, onder Wine, in een VM, of niet?**
- ~~**Administratie:** de Vlaamse schooladministratiepakketten~~ ✅ **beantwoord 2026-09-24**:
  Questi, Informat en Smartschool zijn webbased. Geen blokkade. Resteert randapparatuur —
  eID-lezers, badge-/volgafdrukken.
- **Leerkrachten:** digibord-/smartboardsoftware. Dit is in het onderwijs stelselmatig de
  hardnekkigste Windows-binding, en ze hangt aan het merk van de borden — dus het antwoord verschilt
  per lokaal en kan een vervangingsinvestering betekenen in plaats van een softwarekeuze.
- **Examens/toetsen:** afnamesoftware met lockdown-browser is vaak Windows- of ChromeOS-only.

## Chromebooks — de "anoniem Google-ID"-route heeft een addertje

Het idee is goed: het toestel wordt een domme browser, het werk gebeurt in cloud@olvp. Maar:

- **Beheerde Chromebooks blijven Google-beheerd.** Enrollment loopt via Google Workspace /
  Chrome Education Upgrade. Een anoniem ID haalt de persoonsgegevens uit het account, maar het
  toestelbeheer, de telemetrie en de afhankelijkheid van Google blijven. Als het doel
  *BigTech-vrij* is, is dit een tussenstap en geen eindstation — noem het ook zo.
- **ChromeOS Flex is géén ontsnapping** — dat is nog steeds Google.
- **De AUE-datum geeft je een gratis tijdlijn.** Elk Chromebook-model heeft een *Auto Update
  Expiration*-datum; daarna geen beveiligingsupdates meer. **Inventariseer de AUE-datums van de
  vloot**: dat bepaalt vanzelf wanneer welk toestel vervangen of geherinstalleerd moet worden, en
  het is een argument dat ook buiten ICT begrepen wordt.
- **Herinstalleren naar Linux kan**, maar hangt aan de hardware (firmware-write-protect, model).
  Per modelreeks uitzoeken; voor een deel van de vloot zal vervangen goedkoper zijn dan omzetten.

## Wat er technisch bij komt kijken — en waar OLVP al sterk staat

Het goede nieuws: het beheer-fundament ligt er grotendeels al.

- **Configuratiebeheer:** Ansible + Semaphore draaien al vlootbreed. Een Linux-werkplek past daar
  natuurlijker in dan Windows ooit deed. Zie [[project-workspace-repo-onboarding]] en de bestaande
  `kiosk_devices`/`backup_devices`-groepen als patroon voor eindtoestellen in de inventory.
- **Netwerkidentiteit:** 802.1X met dynamische VLAN-toewijzing werkt al via NPS, met FreeRADIUS als
  doel ([[project-client-identity]]). Linux-clients moeten daar gewoon in meelopen — testen, niet
  aannemen.
- **Gebruikersidentiteit:** aanmelden tegen AD via SSSD/Kerberos, of via Keycloak. Dit is de meest
  onderschatte stap: een werkplek zonder centrale login is een beheerlast per toestel.
- **Wat nog ontbreekt en vaak vergeten wordt:** printen (CUPS + de kopieerdervloot),
  schijfversleuteling, patchbeheer, en een herinstallatieprocedure die een leerkracht niet blokkeert.

## Aanpak-advies

1. **Inventaris eerst** — toepassingen per groep, AUE-datums van de Chromebooks, digibordmerken.
2. **Pilot met vrijwilligers**, niet met een hele klas. Een handvol leerkrachten die het zélf willen,
   plus één administratieve werkplek. ⚠️ **Bijgesteld 2026-09-24:** de administratie stond hier
   eerst als "niet mee beginnen". Nu blijkt haar kernsoftware webbased, is ze juist een van de
   kansrijkste startpunten — al blijft een schaduwopstelling naast de bestaande werkplek verstandig
   zolang eID en badge-printen niet bewezen zijn.
3. **Windows houden waar het moet**, en dat expliciet opschrijven in plaats van het als mislukking
   te zien. Een restgroep Windows is een normale uitkomst, geen nederlaag.
4. **Meet de echte last:** hoeveel tijd kost een Linux-werkplek per maand tegenover een
   Windows-werkplek? Dat cijfer beslist de opschaling, niet de voorkeur van ICT — en het raakt
   dezelfde bus-factor-grens als werf J ([[project-bus-factor]]).

## 🔧 Openstaand opruimpunt in GLPI — modelveld van de Chromebooks

**Vastgesteld 2026-09-24, te doen wanneer het uitkomt.** In GLPI staan **117 computers zonder OS en
zonder software**. Dat zijn géén kapotte records: het zijn **Chromebooks**, en die dragen geen
GLPI-agent.

| Status | Aantal | Wat het is |
|---|---|---|
| `NIEUW` | 107 | nog niet uitgeleverd, of geleverd maar nog niet in gebruik — **ca. 20 daarvan zijn nieuwe toestellen die nog op de plank liggen** (bevestigd door de user) |
| `IN PRODUCTIE` | 10 | in gebruik |

Model: vrijwel allemaal **HP Fortis 11 G10 Chromebook**, met één Lenovo ThinkBook 15 G4 ertussen.

**De actie:** bij **105 van de 107** is het **modelveld leeg** (alleen fabrikant `HP` ingevuld).
Daardoor kan de vloot niet per model gegroepeerd worden — en dat is net wat nodig is om
**AUE-datums** te plannen (WP-2). Het modelveld invullen maakt van deze records meteen de
WP-2-basis, in plaats van dat daar een aparte export voor nodig is.

⚠️ **Niet verwijderen.** De verleiding is deze records op te ruimen omdat ze "leeg" lijken. Ze zijn
niet leeg, ze zijn correct geregistreerd; alleen het modelveld ontbreekt. Wat wél opruimwerk is:
nakijken of de tien op `IN PRODUCTIE` werkelijk in gebruik zijn.

*(Ik noemde dit eerst zelf "lege records om op te ruimen" — te snel geconcludeerd op basis van een
ontbrekend OS-veld, zonder naar de status te kijken.)*

## CxLogon — 500 toestellen dragen iets dat niet draait

GLPI vond `CxLogon` (Jahastech) op **500 van de 605 toestellen**. De user: het werkt samen met
NXfilter — het koppelt een aanmelding aan webverkeer zodat filtering per gebruiker kan in plaats van
per IP — maar het is **niet actief in gebruik**.

- **Geen blokkade** voor werf K; het hoeft niet mee naar Linux.
- **Wel opruimwerk met een veiligheidskant**: software die niemand gebruikt, wordt door niemand
  bijgewerkt.
- ⚠️ **Niet opruimen vóór de opvolger bepaald is.** Komt er een andere oplossing voor afscherming en
  registratie van webverkeer per host of gebruiker, dan vervalt CxLogon definitief — en pas dan weet
  je of die opvolger iets op de werkplek nodig heeft. Dat raakt rechtstreeks de tier-ladder voor
  NXfilter↔RADIUS-correlatie uit [[project-client-identity]]: lukt het om gebruikersidentiteit uit
  802.1X/RADIUS te halen, dan is er op de werkplek niets meer nodig. **Bekijk die twee werven
  daarom samen** — de uitkomst bepaalt of de Linux-werkplek een agent moet dragen.

Gerelateerd: [[project-cloud-olvp]] (werf J, randvoorwaarde), [[project-client-identity]] (802.1X),
[[project-it-workstation-hardening]] (beheerderswerkplekken, andere scope),
[[project-programma-structuur]].
