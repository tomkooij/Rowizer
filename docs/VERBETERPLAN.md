# Rowizer – analyse en verbeterplan

Opgesteld op 10-11-2026 op basis van commit `4793264` (master). Dit document is bedoeld
als werkinstructie: elke stap is los uit te voeren, heeft een eigen acceptatiecriterium
en hoort in een eigen commit.

## 1. Wat de app doet

Rowizer is een statische webpagina (geen build, geen dependencies, ES-modules) die op een
groot scherm in de hal de roosterwijzigingen van vandaag uit Zermelo toont.

| Laag | Bestanden | Rol |
|---|---|---|
| Start | `index.html`, `js/script.js`, `js/clock.js` | URL-parameters lezen, drie verversloops starten, herlaad-waakhond |
| API | `js/zermelo/**` | `Session` (fetch + time-out + retry), één `Manager` per endpoint |
| Connector | `js/connectors/zermeloConnector.js` | Basisgegevens bij opstarten: vestiging, afdelingen, groepen, leerlingen, lesuren, dagrooster |
| Controllers | `js/controllers/{changes,absences,outofoffice}` | Data ophalen en bewerken |
| Views | `js/views/**`, `styles.css` | DOM opbouwen: raster jaarlaag × lesuur, afwezigen, externe activiteiten |

Gedrag: elke 5 minuten verversen (na een fout na 1 minuut), om middernacht de pagina herladen.
Instellen gaat volledig via URL-parameters (`portal`, `token`, `branch`, `date`, `external`,
`departmentsIgnore`, `mergeAppointments`).

Verhouding tot upstream (`Vlietland-College/Rowizer`, laatste commit `e39dc3b`): deze fork loopt
28 commits voor en 0 achter. De verschillen zitten in stabiliteit (time-outs, retries,
statusmelding, herlaad-waakhond), toetsen, keuzelessen en de versiestempel. Upstream heeft geen
open issues of PR's.

## 2. Bevindingen

Gemeten door de app lokaal te draaien tegen de demo-portal uit de README
(`?portal=j9qeq&date=23-9-2026&branch=a`, token staat in `README.md`) op 1920×1080.

### 2.1 Fouten die nu op het bord zichtbaar zijn

| # | Bevinding | Bewijs |
|---|---|---|
| B1 | Afwezigen worden niet samengevoegd. Een docent die per lesuur is afgemeld staat er acht keer, ongesorteerd: `krh (3) - krh (8) - krh (5) - krh (2) - krh (6) - krh (t/m 1) - krh (4) - krh (7)`. | `js/views/absences/absencesUiManager.js:14-23`; `merge()` op regel 26 is leeg |
| B2 | Bij meer dan twee docenten klopt de teller niet: drie docenten wordt `a,b+2` in plaats van `a,b+1`. | `js/views/changes/changesUIRecordClass.js:41-44` |
| B3 | De tijd van een afspraak zonder lesuur wordt aan `locations[0]` geplakt; daarna sorteert de view de locaties op lengte en toont alleen de kortste. Bij twee lokalen kan de tijd dus verdwijnen, en de sortering wijzigt de brongegevens bij elke render. | `js/controllers/changes/changes.js:124`, `js/views/changes/changesUIRecordClass.js:50` |
| B4 | Zonder `portal`/`token` vraagt de app `https://null.zportal.nl` op en blijft elke minuut herladen, met alleen een titel in beeld. Een verlopen token (401) geeft dezelfde algemene melding. | `js/script.js:32-36`, `js/script.js:103-106` |
| B5 | Een dag zonder wijzigingen of een weekenddag geeft een leeg raster zonder enige tekst. Niet te onderscheiden van een storing. | Gemeten met `date=26-9-2026` |

### 2.2 Leesbaarheid op een groot scherm

| # | Bevinding | Bewijs |
|---|---|---|
| L1 | Alle tekst is vast 16 px, ook titel en klok. Op een full-HD-scherm is een regel 19 px hoog; dat is vanaf een paar meter niet te lezen. De README vraagt daarom om 4K + 50 inch, maar ook dan schaalt er niets mee. | `styles.css:1-6` (geen `font-size`) |
| L2 | Alle lesuurnamen van het schooljaar krijgen een kolom, ook als er die dag niets in staat. Op de demodag zijn u09 en u10 leeg: 20% van de breedte ongebruikt, terwijl 15 van de 62 items over twee regels afbreken. | `js/views/changes/changesUiManager.js:22-27` |
| L3 | Geen markering van het huidige lesuur; voorbije uren blijven even prominent. | – |
| L4 | Geen legenda. Groen (activiteit), blauw (toets) en de rode gloed (nieuw sinds vannacht 01:00) moet je maar weten. | `styles.css:128-138` |
| L5 | "Vervalt" is alleen tekst; uitval ziet er hetzelfde uit als een lokaalwijziging. | `styles.css:122-126` |
| L6 | Geen `lang`, geen `<title>`, muiscursor blijft zichtbaar op het scherm. | `index.html:2-6` |

### 2.3 Verspilling en kwetsbaarheden

| # | Bevinding | Bewijs |
|---|---|---|
| V1 | De categorie `invalid` wordt elke ronde opgehaald en daarna nergens getoond. Op de demodag is dat 49 van de 93 afspraken. Ze tellen wel mee in de wijzigingsdetectie, dus het bord wordt onnodig herbouwd (en scrolt dan terug naar boven). | `js/controllers/changes/changes.js:55-57`, `js/views/changes/changesUiManager.js:162-172` |
| V2 | Er wordt dag en nacht en in het weekend elke 5 minuten gepold: 7 verzoeken per ronde, ruim 2000 per dag, waarvan het grootste deel buiten schooltijd. | `js/script.js:13`, `js/script.js:80-92` |
| V3 | Query-parameters worden niet ge-encodeerd. Een `external`-locatienaam met spatie of `&` breekt het verzoek. | `js/zermelo/utils/session.js:20` |
| V4 | Kolommen gaan uit van lesuur-rangnummers die bij 1 beginnen en aaneengesloten zijn. De koppen worden op volgorde geplaatst, de afspraken op rangnummer, en afspraken zonder lesuur op index. Met een nulde uur of een gat in de nummering verschuift alles. | `js/views/changes/changesUiManager.js:27,41-47`, `js/controllers/changes/changes.js:126-136` |
| V5 | Na een deploy draait het scherm tot middernacht de oude code; de versiestempel wordt met de hand gezet en loopt per definitie één commit achter. | `scripts/stamp-version.sh:3`, `js/version.js` |
| V6 | Het token staat blijvend in de adresbalk. Er is geen `referrer`-beleid. | `js/script.js:21-36` |
| V7 | De sorteerfunctie is niet consistent (activiteit tegenover toets geeft beide kanten `-1`), waardoor de volgorde per browser kan verschillen. | `js/views/changes/changesUiManager.js:174-189` |
| V8 | `arraysEqual` sorteert de arrays van de afspraak zelf. Docent- en lokaalvolgorde veranderen daardoor als bijwerking van een vergelijking. | `js/zermelo/utils/mergeAppointments.js:77` |

### 2.4 Onderhoudbaarheid

- Geen tests, geen CI, geen lint. De lastigste logica (afspraken samenvoegen, lesuur bepalen,
  afwezigheid naar lesuren) is alleen met de hand op een scherm te controleren.
- Dode code: `changedRecord.js` en `changedRecordHolder.js` (wel geïmporteerd in
  `changes.js:1`, nooit gebruikt), de lege `connector.js`, lege `merge()`-methodes,
  `Number(r.id)` zonder effect (`js/zermelo/lib/manager.js:11`), en managers die nergens
  worden aangeroepen (`schools`, `weektimetables`, `timetableoverrides`).
- Debug-globals `window.ad`, `window.cm`, `window.zc`, `window.dc` staan altijd aan; elke ronde
  worden complete afspraakobjecten naar de console gelogd ("Skipping: lesson vv").
- Bedrijfslogica in de view: het afleiden van afdelingen uit leerlingen voor keuzelessen en
  toetsen staat in `changesUiManager.js:108-149` (in de code zelf gemarkeerd als hack).
- CSS: `overflow-y: overlay` is vervallen (`styles.css:99`); de regel die de scrollbalk
  verbergt richt zich op `.year-row` terwijl `.schedule-content` scrolt (`styles.css:89`).
- Bekende beperking, buiten dit plan: één dagrooster voor de hele vestiging
  (`zermeloConnector.js:244-246`, FIXME).

## 3. Werkafspraken voor de uitvoering

1. **Niet pushen naar `live`.** De remote `live` (`Coornhert-Gymnasium/Rowizer`) is de
   productie-installatie; een push komt op het scherm in de hal. Alleen lokaal committen,
   tenzij Tom uitdrukkelijk om een push vraagt.
2. Werk op een branch per fase (`plan/fase-0-tests`, `plan/fase-1-fouten`, ...), één commit
   per stap, commitberichten in dezelfde stijl als de bestaande (Engels, gebiedende wijs).
3. Geen buildstap en geen runtime-dependencies toevoegen. De pagina moet rechtstreeks vanaf
   GitHub Pages blijven werken. Ontwikkel-tooling mag alleen wat Node zelf meelevert.
4. Het scherm draait op een signage-speler (GoBright; `window.caches` ontbreekt daar, zie
   `session.js:15`). Gebruik geen webfuncties die nieuwer zijn dan wat de code nu al gebruikt
   (private velden, `Array.prototype.at`, `??`, CSS subgrid). Ga er niet van uit dat
   `localStorage` of service workers beschikbaar zijn.
5. Controleer elke zichtbare wijziging in de browser: start de server uit
   `.claude/launch.json` (`rowizer-static`, poort 8765) en open de demo-URL uit de README met
   `localhost:8765` als host. Testdagen: `date=23-9-2026` (drukke dag, 62 items, toetsen,
   activiteit, afwezigen) en `date=26-9-2026` (zaterdag, leeg). Controleer op 1920×1080 én
   3840×2160.
6. Bestaand gedrag dat niet in een stap genoemd wordt, blijft gelijk. Bij twijfel over wat een
   school wil zien: niet gokken maar de vraag in §6 laten beantwoorden.
7. Commentaar in de code is Nederlands, namen en logmeldingen Engels – houd dat aan.

## 4. Stappenplan

Volgorde: fase 0 eerst (vangnet), daarna 1 en 2 (grootste winst voor wie naar het scherm
kijkt). Fase 3 bevat keuzes die Tom eerst moet maken. Fase 4 kan altijd tussendoor.

### Fase 0 – Vangnet

**0.1 Testomgeving zonder dependencies**
- Voeg `package.json` toe met alleen `"type": "module"`, `"private": true` en
  `"scripts": {"test": "node --test"}`.
- Maak `test/` met een eerste test voor `js/zermelo/utils/date.js` (weeknummer rond de
  jaarwisseling: 30-12-2024 → `202501`, 1-1-2027 → `202653`; `getSchoolYear` voor juli/augustus).
- Klaar als: `npm test` slaagt met Node 20+ en de pagina ongewijzigd laadt.

**0.2 Fixtures van de demo-portal**
- Schrijf `scripts/record-fixtures.mjs` dat voor `23-9-2026` de antwoorden ophaalt van alle
  endpoints die de app gebruikt en ze als JSON in `test/fixtures/2026-09-23/` zet. Portal en
  token komen uit argumenten, niet hardgecodeerd.
- Maak `test/helpers/fakeSession.js`: zelfde `request(path, options)` als `Session`, maar leest
  uit de fixtures (filtert `appointments` op de meegegeven `type`/`cancelled`/`modified`/`valid`).
- Klaar als: een test `ZermeloConnector` + `Changes.loadData()` op de fixtures kan draaien en
  62 getoonde items oplevert, gelijk aan wat nu op het scherm staat.
- Let op: `ZermeloConnector` start het laden in de constructor en `Changes` schrijft naar
  `window.ad`. Maak dat laatste voorwaardelijk (`typeof window`), verander verder niets.

**0.3 Karakteriseringstests voor de kernlogica**
- `mergeSubsequentAppointments`: aaneengesloten uren worden één afspraak; een gat niet.
- Lesuurbepaling voor afspraken zonder lesuur (`changes.js:119-139`).
- Afwezigheid naar lesuren (`absences.js:46-70`): hele dag, "t/m", "va", één uur.
- Tekst per item (`ChangesUIRecordClass.getInnerText`): les, uitval, activiteit, toets,
  keuzeles zonder vak. Haal hiervoor de tekstopbouw uit de DOM-klasse naar een pure functie
  in `js/views/changes/appointmentText.js` en laat de klasse die aanroepen.
- Leg het huidige gedrag vast, ook waar het fout is (B2, B3); markeer die tests met een
  verwijzing naar de stap die ze gaat omdraaien.

**0.4 CI**
- `.github/workflows/test.yml`: bij push en PR `npm test` op Node 20. Geen deploy-stappen.

### Fase 1 – Zichtbare fouten

**1.1 Afwezigen samenvoegen (B1)**
- Nieuw: `js/controllers/absences/mergeAbsences.js` – groepeer per `employee`, sorteer op
  begintijd, voeg aaneengesloten of overlappende lesuren samen tot één bereik. Dekt het
  resultaat de hele lesdag, dan geen toevoeging.
- `AbsencesUiManager.render()` gebruikt het resultaat; verwijder de lege `merge()`.
- Verwacht op de demodag: `bee - krh`. Met een gat: `krh (1-3, 6-8)`.
- Tests: acht losse uren → hele dag; uren 1-3 en 6-8; één hele-dagmelding plus een losse.

**1.2 Docententeller (B2)**
- `+ (length - 2)` in de tekstfunctie uit 0.3. Pas de test aan.

**1.3 Tijd en locatie scheiden (B3)**
- In `changes.js` de tijd niet meer aan `locations[0]` plakken maar als
  `appointment.timeLabel = "14:00-14:50"` bewaren.
- In de tekstfunctie: kies de locatie met een kopie (`[...locations].sort(...)`), toon `+n`
  bij meerdere, en zet `timeLabel` erachter.
- Vervang `arraysEqual` door een variant die op kopieën sorteert (V8).

**1.4 Duidelijke meldingen bij ontbrekende of ongeldige instellingen (B4)**
- In `script.js`: ontbreekt `portal` of `token`, toon dan schermvullend
  "Rowizer is nog niet ingesteld" met de twee benodigde parameters, start geen API-verzoeken
  en zet `window.rowizerStarted = true` zodat de waakhond niet blijft herladen.
- `ZermeloAuthorizationError` bij het opstarten: melding "Token ongeldig of verlopen",
  opnieuw proberen na 15 minuten in plaats van elke minuut.
- Onbekende vestiging of meerdere vestigingen zonder `branch`: toon de foutmelding uit de
  connector op het scherm in plaats van alleen in de console.

**1.5 Lege toestand (B5)**
- Geen items: toon gecentreerd in het raster "Geen roosterwijzigingen".
- Geen dagrooster (`getTodayTimeSlots()` leeg): "Vandaag geen lessen" en verberg de
  afwezigenregel.

**1.6 Query-encoding (V3)**
- Bouw de URL in `Session.request` met `URLSearchParams`. Arrays (`fields`) blijven
  komma-gescheiden. Test met een locatienaam met spatie.

**1.7 Stabiele sortering (V7)**
- Eén vergelijkfunctie: type (activiteit, toets, les) → groepsnaam
  (`localeCompare` met `numeric: true`) → beginuur. Test op de fixtures dat de volgorde bij
  herhaald renderen identiek is.

### Fase 2 – Leesbaarheid

**2.1 Schalende typografie (L1)**
- Zet op `html` een basisgrootte die met het scherm meegroeit, bijvoorbeeld
  `font-size: clamp(14px, 1.25vw, 48px)`, en reken marges, kolomtussenruimte en de
  `max-height: 20px`/`max-width: 15px` van de koppen om naar `em`/`rem`.
- Titel en klok 1,6× de basisgrootte.
- Nieuwe URL-parameter `scale` (standaard `1`) als vermenigvuldiger, zodat een school kan
  bijstellen zonder browserzoom. Documenteer in de README.
- Klaar als: op 1920×1080 en 3840×2160 ziet het bord er verhoudingsgewijs gelijk uit en past
  de demodag zonder scrollen bij `scale=1`.

**2.2 Alleen de lesuren van vandaag als kolom (L2, V4)**
- Bepaal de kolommen uit `connector.getTodayTimeSlots()` in plaats van uit alle
  lesuurnamen; laat lege kolommen aan het eind van de dag weg (nooit in het midden).
- Maak één plek die rangnummer → kolomindex vertaalt en gebruik die voor koppen, voor
  `--start-hour`/`--end-hour` en voor afspraken zonder lesuur. Daarmee vervalt de aanname
  dat rangnummers bij 1 beginnen.
- Test met een fixturevariant met een nulde uur.

**2.3 Huidig lesuur markeren (L3)**
- Elke minuut: geef de kolomkop van het lopende lesuur een klasse `current` en items die
  volledig voorbij zijn een klasse `past` (lagere dekking, niet verbergen).
- Alleen als de getoonde datum vandaag is. Geen herbouw van de tabel; alleen klassen wisselen.

**2.4 Legenda en tijdstip van verversen (L4)**
- Eén regel onderaan: gekleurde blokjes "activiteit", "toets", "vervalt", "nieuw vandaag",
  rechts "bijgewerkt 10:35". De bestaande statusmelding bij storing blijft.

**2.5 Uitval herkenbaar maken (L5)**
- Geef `.appointment.cancelled` een eigen stijl (bijvoorbeeld een rode rand links en het
  woord "vervalt" vet). Houd het contrast van zwarte tekst op de achtergrond minimaal 7:1.
- De uitzondering `div.appointment.activity.cancelled` in `styles.css:140` gaat hierin op.

**2.6 Pagina-basis (L6)**
- `<html lang="nl">`, `<title>Roosterwijzigingen</title>`, `cursor: none` op `body`,
  lege `<style>` en uitgecommentarieerde script-tag weg.
- Vervang `overflow-y: overlay` en corrigeer de scrollbalk-selector (zie §2.4).

### Fase 3 – Functionele uitbreidingen (eerst §6 beantwoorden)

**3.1 Niet-getoonde gegevens niet meer ophalen (V1)**
- Verwijder de categorie `invalid` uit `changes.js:55-57`, tenzij stap 3.3 wordt uitgevoerd.
- Klaar als: 4 in plaats van 5 `appointments`-verzoeken per ronde en hetzelfde bord.

**3.2 Rusttijden (V2)**
- Buiten een instelbaar venster (parameter `active`, standaard `06:30-18:00`, alleen op
  dagen met een dagrooster) niet pollen; het bord blijft staan met de laatste stand.
- Eerste verversing direct bij het begin van het venster.

**3.3 Verplaatste lessen op het oude uur tonen**
- Nu verschijnt een verplaatste les alleen op het nieuwe uur; op het oude uur staat niets.
- Gebruik de `invalid`-afspraken om op het oorspronkelijke uur "verplaatst naar u5" te tonen,
  mits er voor die groep op dat uur niets anders op het bord staat.
- Koppel oud en nieuw via `appointmentInstance`. Eerst op de fixtures uitzoeken of dat veld
  daarvoor betrouwbaar is; zo niet, deze stap melden en overslaan.

**3.4 Morgen tonen na de laatste les**
- Parameter `tomorrowAfter` (bijvoorbeeld `16:30`, standaard uit). Daarna toont het bord de
  eerstvolgende dag met een dagrooster, met de datum duidelijk in de titel
  ("Roosterwijzigingen morgen – donderdag 24 september").
- De connector laadt het dagrooster nu één keer per week; controleer dat de weekwissel
  (vrijdagmiddag → maandag) het juiste `realizedweektimetable` ophaalt.

**3.5 Zelf bijwerken na een deploy (V5)**
- Elke verversronde `js/version.js` ophalen met `cache: "no-store"`; wijkt de versie af van
  de geladen versie, dan `window.rowizerReload()`.
- Laat de stempel door een GitHub Action zetten bij een push naar `master` in plaats van met
  de hand, zodat de hash klopt. Overleg eerst: deze workflow moet ook in de `live`-repo werken.

**3.6 Token minder zichtbaar (V6)**
- `<meta name="referrer" content="no-referrer">` in `index.html` (altijd doen).
- Optioneel, achter parameter `rememberToken=1`: token in `localStorage` bewaren en met
  `history.replaceState` uit de adresbalk halen. Niet standaard aanzetten: het is onbekend of
  de signage-speler opslag bewaart.

### Fase 4 – Opruimen

**4.1 Dode code verwijderen**
- `changedRecord.js`, `changedRecordHolder.js` en de import in `changes.js:1`; `connector.js`
  (laat `ZermeloConnector` niets meer uitbreiden, verwijder `super(options)`); lege
  `merge()`-methodes; `Number(r.id)` in `manager.js:11`; de ongebruikte velden `#date`,
  `#schoolInYear` in `Changes`.
- Ongebruikte managers alleen verwijderen als Tom geen upstream-merges meer verwacht; anders
  laten staan (ze kosten niets tijdens het draaien).

**4.2 Debug achter een schakelaar**
- `window.ad/cm/zc/dc` en het loggen van complete afspraken alleen met `?debug=1`.
  `window.pretendLikeIts` hoort daar ook bij.

**4.3 Keuzeles-logica naar de controller**
- Verplaats `changesUiManager.js:108-149` naar `Changes`: elke afspraak krijgt een lijst
  `targets` (`{id, name, extendedName, isMainGroup, department}`), ook de afspraken met
  gewone groepen. De view loopt alleen nog over `targets`.
- Geen zichtbaar verschil; de fixturetest uit 0.2 moet identiek blijven.

**4.4 README bijwerken**
- Nieuwe parameters, de lokale testinstructie, `npm test`, en een verwijzing naar dit plan.
- Corrigeer "Highlights new changes (made after 8AM)": de code gebruikt 01:00
  (`changesUIRecord.js:24-29`).

## 5. Aanbevolen volgorde en omvang

| Blok | Stappen | Waarom eerst |
|---|---|---|
| A | 0.1 – 0.4 | Zonder tests is elke volgende stap alleen op het oog te controleren |
| B | 1.1, 1.2, 1.3, 1.5 | Fouten die leerlingen nu zien |
| C | 2.1, 2.2 | Grootste winst in leesbaarheid; 2.2 lost V4 mee op |
| D | 1.4, 1.6, 1.7, 2.3 – 2.6 | Afwerking |
| E | 3.1, 3.2, 3.5, 3.6 (meta-tag) | Minder verzoeken, sneller uitrollen |
| F | 3.3, 3.4 | Alleen na akkoord op §6 |
| G | 4.1 – 4.4 | Kan tussendoor, 4.3 pas na blok A |

Blok A t/m D raken samen ongeveer 12 bestanden en voegen ongeveer 6 nieuwe toe.

## 6. Open vragen voor Tom

1. **Verplaatste lessen (3.3):** moet op het oude uur "verplaatst naar u5" komen, of is het
   nieuwe uur tonen genoeg?
2. **Morgen tonen (3.4):** gewenst, en zo ja vanaf welk tijdstip?
3. **Rusttijden (3.2):** is 06:30-18:00 een goed standaardvenster voor het Coornhert?
4. **Uitrol (3.5):** mag er een GitHub Action in de `live`-repo komen die de versie stempelt?
5. **Upstream:** moeten de algemene fixes (1.1, 1.2, 1.6, 2.1) als PR naar
   `Vlietland-College/Rowizer`? Dat bepaalt hoe voorzichtig 4.1 met verwijderen moet zijn.
6. **Afwezigen (1.1):** bij deels afwezig het bereik tonen (`krh (1-3)`), of alleen de naam?
