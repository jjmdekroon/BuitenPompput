# Feature Specification: Stroomsensor verwijderen

**Feature Branch**: `004-stroomsensor-verwijderen` (spec-map `specs/004-stroomsensor-verwijderen`)

**Created**: 2026-11-18

**Status**: Implemented (hardware-scenario's open)

**Input**: User description: "de stroomsensor die kijkt of de motor aan slaat kan eruit. Deze gaan we niet gebruiken."

**Wijzigt**:

- [specs/001-pompput-monitor/spec.md](../001-pompput-monitor/spec.md): User Story 3 en 4, FR-006,
  FR-008, FR-009, FR-010, FR-011, FR-013 t/m FR-016, FR-019, FR-023, FR-026, SC-002, SC-003,
  SC-006, de randgevallen en de Assumptions over de pomp en de stroommeting.
- [specs/003-niveau-ledbalk/spec.md](../003-niveau-ledbalk/spec.md): FR-003, FR-014, User Story 2
  scenario 3, 7 en 8, SC-004 en het patroon "pomp draait".

## Clarifications

### Session 2026-11-18

- Q: Zonder stroomsensor weet het apparaat niet meer of de pomp draait. Wat blijft er over van de pompbewaking? → A: Niets. De pompstatus, de looptijd, de pompstarts, de drie pompwaarschuwingen, het patroon "pomp draait" op led 1 en alle instellingen die alleen daarvoor bestonden, vervallen. Er komt geen vervangende waarschuwing op basis van het niveau, en de pompactiviteit wordt ook niet uit het niveau afgeleid.

## Begrippen

- **Stroomsensor**: de sensor in de voedingsdraad van de pomp die meet of de pompmotor stroom
  trekt. Daarmee bepaalt het apparaat nu of de pomp draait.
- **Pompbewaking**: alles wat op de stroomsensor leunt: de pompstatus (draait / staat stil), de
  looptijd van de laatste pompbeurt, het aantal pompstarts, de waarschuwingen "pomp start niet",
  "pomp pompt niet af" en "pomp draait te lang", de storing van de stroomsensor, het patroon
  "pomp draait" op led 1 en de instellingen voor dit alles.
- **Niveaubewaking**: wat overblijft: de niveaumeting (waterhoogte en vulgraad), het
  hoogwater-alarm, de storing van de niveausensor, de melding "niet ingesteld", de melding
  "bewaking niet bereikbaar" in Home Assistant en de ledbalk.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - De bewaking werkt zonder stroomsensor (Priority: P1)

Als eigenaar van de pompput wil ik dat het apparaat geen stroomsensor meer nodig heeft en ook
geen storing meldt omdat die ontbreekt, zodat ik de sensor kan weglaten. Het niveau, het
hoogwater-alarm en de ledbalk blijven gewoon werken.

**Why this priority**: Dit is de kern van de wijziging. Zonder deze story meldt een apparaat
zonder stroomsensor een storing die niet weg te krijgen is, en ziet de gebruiker bij de put
steeds een storingspatroon.

**Independent Test**: Bouw het apparaat op zonder stroomsensor en laat het opstarten. Breng het
niveau achtereenvolgens naar leeg, half vol en boven de alarmdrempel, en dek daarna de
afstandsmeter af. Controleer dat het apparaat geen storing over de stroommeting meldt, dat het
niveau, het hoogwater-alarm, de storing van de niveausensor en de ledbalk werken zoals voorheen,
en dat het apparaat geen pompwaarschuwing geeft.

**Acceptance Scenarios**:

1. **Gegeven** het apparaat is ingesteld en er is geen stroomsensor aangesloten, **wanneer** het apparaat opstart en daarna 15 minuten in bedrijf is, **dan** is er geen storingsmelding actief en brandt led 1 continu blauw (bij een geldig niveau en een verbinding met Home Assistant).
2. **Gegeven** het niveau komt langer dan de wachttijd boven de alarmdrempel, **wanneer** het hoogwater-alarm actief wordt, **dan** ontvang ik het alarm via Home Assistant en knippert led 1 rood, zoals voorheen.
3. **Gegeven** de afstandsmeter is afgedekt, **wanneer** de wachttijd voor een sensorstoring voorbij is, **dan** ontvang ik de storingsmelding van de niveausensor en knippert led 1 paars, zoals voorheen.
4. **Gegeven** het water staat ruim boven het niveau waarop de pomp normaal aanslaat, maar onder de alarmdrempel, **wanneer** dat langer duurt dan de vroegere wachttijd van "pomp start niet", **dan** krijg ik geen waarschuwing en blijft led 1 continu blauw. De niveaubalk toont het niveau.
5. **Gegeven** het apparaat heeft geen verbinding met Home Assistant, **wanneer** het hoogwater-alarm actief wordt, **dan** toont led 1 dat nog steeds binnen 5 seconden.

---

### User Story 2 - Geen pompgegevens en pompbediening meer in Home Assistant (Priority: P2)

Als eigenaar wil ik in Home Assistant geen pompstatus, pompstroom, looptijden, pompstarts,
pompwaarschuwingen of pompinstellingen meer zien, zodat het dashboard alleen toont wat het
apparaat echt weet en er geen waarden of instellingen staan die niets betekenen.

**Why this priority**: Zonder stroomsensor zijn die gegevens altijd leeg of fout, en hebben de
instellingen geen effect meer. Ze laten staan is verwarrend, maar raakt de niveaubewaking niet.

**Independent Test**: Werk het apparaat en de Home Assistant-configuratie bij en controleer dat
er geen entiteiten, dashboardonderdelen, meldingen of meldingsteksten meer zijn over de pomp of
de stroomsensor, en dat de overgebleven instellingen nog werken.

**Acceptance Scenarios**:

1. **Gegeven** het bijgewerkte apparaat is gekoppeld aan Home Assistant, **wanneer** ik de entiteiten van het apparaat bekijk, **dan** zijn er geen entiteiten meer voor de pompstatus, de pompstroom, de spanning of offset van de stroomsensor, de looptijd, de pompwaarschuwingen of de storing van de stroomsensor.
2. **Gegeven** de instellingen van het apparaat in Home Assistant, **wanneer** ik die bekijk, **dan** staan er alleen nog: afstand leeg, afstand vol, alarmdrempel, herstelmarge, wachttijd hoogwater, wachttijd sensorstoring en led-helderheid.
3. **Gegeven** het dashboard van de pompput, **wanneer** ik het open, **dan** staan er geen pomponderdelen (pompstatus, looptijd, pompstarts, pompwaarschuwingen) op en zijn er geen onbeschikbare of verweesde entiteiten zichtbaar.
4. **Gegeven** een melding in Home Assistant (hoogwater, niveausensor storing, niet ingesteld of bewaking niet bereikbaar), **wanneer** ik de tekst lees, **dan** staat er geen pompstatus meer in, maar wel het soort melding, de vermoedelijke oorzaak, het actuele niveau en sinds wanneer de situatie speelt.
5. **Gegeven** ik pas een van de overgebleven instellingen aan in Home Assistant, **wanneer** de nieuwe waarde is ingesteld, **dan** is die binnen 1 minuut van kracht, zoals voorheen.

---

### User Story 3 - Led 1 zonder pomp- en waarschuwingspatroon (Priority: P2)

Als eigenaar wil ik dat led 1 alleen nog toestanden toont die het apparaat kan vaststellen, zodat
ik bij de put niet op een patroon wacht dat nooit meer verschijnt.

**Why this priority**: Zonder pompstatus kan "pomp draait" nooit meer verschijnen, en het
waarschuwingspatroon (oranje) werd alleen door de pompwaarschuwingen gebruikt. De uitleg van de
ledbalk moet kloppen met wat de gebruiker ziet.

**Independent Test**: Boots opstarten, een hoogwater-alarm, een storing van de niveausensor,
"niet ingesteld" en het wegvallen van de verbinding na. Controleer dat led 1 telkens het
bijbehorende patroon toont en daarna weer continu blauw wordt, en dat led 1 nooit oranje is of
langzaam blauw knippert.

**Acceptance Scenarios**:

1. **Gegeven** er is geen statuspatroon actief, **wanneer** ik naar led 1 kijk, **dan** brandt die continu blauw, ook als de pomp op dat moment draait.
2. **Gegeven** een van de overgebleven statuspatronen wordt actief, **wanneer** ik naar led 1 kijk, **dan** toont die het patroon met dezelfde kleur, hetzelfde knipperpatroon en dezelfde onderlinge prioriteit als voorheen.
3. **Gegeven** het apparaat is in bedrijf, **wanneer** ik bij de put kijk, **dan** toont led 1 nooit het waarschuwingspatroon (oranje) en nooit het patroon "pomp draait" (langzaam blauw knipperen).

---

### User Story 4 - Documentatie en aansluitschema zonder stroomsensor (Priority: P3)

Als degene die het apparaat bouwt of onderhoudt, wil ik dat het aansluitschema, de
bouwbeschrijving, de beschrijving van de ledbalk en de testprocedure de stroomsensor en de
pompbewaking niet meer noemen. Wil ik een bestaande stroomsensor weghalen, dan moet duidelijk
zijn hoe dat veilig gaat en dat de pomp daarna weer stroom krijgt.

**Why this priority**: Nodig om het apparaat later goed na te bouwen of te repareren. De
werking van de niveaubewaking hangt er niet van af, maar de veiligheid bij het weghalen wel.

**Independent Test**: Lees de bouwbeschrijving, het aansluitschema, de beschrijving van de
ledbalk en de testprocedure door. Controleer dat de stroomsensor, de aansluiting ervan en de
pompbewaking nergens meer als onderdeel van het apparaat staan, dat de vrijgekomen aansluiting
als vrij staat aangegeven, en dat er een aanwijzing is voor het veilig weghalen van een bestaande
sensor.

**Acceptance Scenarios**:

1. **Gegeven** het aansluitschema, **wanneer** ik het doorlees, **dan** komt de stroomsensor er niet meer in voor en staat de aansluiting waar hij zat als vrij aangegeven.
2. **Gegeven** een bestaand apparaat met een stroomsensor in de voedingsdraad van de pomp, **wanneer** ik de documentatie lees, **dan** staat erin dat het weghalen werk aan netspanning is, dat een bevoegd persoon het doet of erop toeziet, en dat de voedingsdraad van de pomp daarna weer doorverbonden moet zijn, zodat de pomp op zijn eigen vlotter blijft werken.
3. **Gegeven** de testprocedure op het apparaat, **wanneer** ik die uitvoer, **dan** bevat die geen stappen meer voor de pompstatus, de looptijd, de pompwaarschuwingen of de stroomsensor.
4. **Gegeven** de uitleg van de ledbalk, **wanneer** ik die lees, **dan** staan "pomp draait" en "waarschuwing" niet meer in de lijst van statuspatronen.

---

### Edge Cases

- **De stroomsensor zit nog aangesloten** op een bestaand apparaat dat wordt bijgewerkt: het apparaat leest de sensor niet meer, meldt er niets over en de aansluiting doet geen kwaad. De sensor mag blijven zitten of worden weggehaald.
- **De stroomsensor wordt weggehaald**: dit is werk aan de netspanning van de pomp. Na het weghalen moet de voedingsdraad van de pomp weer correct doorverbonden zijn; anders werkt de pomp niet meer (spec 001 FR-007).
- **Bestaande pompinstellingen in het apparaat** (inschakelniveau, marges, wachttijden en maximale looptijd van de pompwaarschuwingen, stroomdrempel): na het bijwerken hebben die opgeslagen waarden geen effect meer en mogen ze de opslag van de overige instellingen niet verstoren. De overige bewaarde instellingen blijven behouden.
- **Er is een pompwaarschuwing of stroomsensorstoring actief op het moment van bijwerken**: na het bijwerken bestaat die melding niet meer. In Home Assistant blijft er geen actieve pompmelding hangen.
- **Oude pomp-entiteiten en de historie in Home Assistant**: na het bijwerken mogen er geen onbeschikbare pomp-entiteiten op het dashboard of in automatiseringen achterblijven. De historie van vóór de wijziging hoeft niet bewaard of verwijderd te worden; die verdwijnt volgens de gewone bewaartermijn van Home Assistant.
- **De pomp start niet (vlotter vast, zekering eruit)**: dit wordt niet meer apart gemeld. Het water stijgt dan, en het hoogwater-alarm komt zodra het niveau langer dan de wachttijd boven de alarmdrempel staat. Dat is later dan de vroegere waarschuwing "pomp start niet".
- **De pomp pompt niet af (verstopping, terugslagklep)**: dit wordt niet meer apart gemeld. Stijgt het niveau daardoor tot boven de alarmdrempel, dan komt het hoogwater-alarm.
- **De pomp loopt droog of blijft draaien** (vlotter blijft in de aan-stand hangen terwijl de put leeg is): dit wordt niet meer gedetecteerd. Het niveau blijft laag, dus er komt geen melding.
- **Pendelen** (de pomp slaat vaak kort aan en af): dit is niet meer te zien in de historie van Home Assistant. Alleen het verloop van het niveau laat het mogelijk zien als een zaagtand.

## Requirements *(mandatory)*

### Functional Requirements

**Wat vervalt**

- **FR-001**: Het apparaat MAG de stroomsensor niet meer gebruiken en MAG niet meer vaststellen of de pomp draait. De pompstatus, de looptijd van de laatste pompbeurt en de pompstarts vervallen (vervangt spec 001 FR-006, FR-008, FR-009 en FR-011).
- **FR-002**: De waarschuwingen "pomp start niet", "pomp pompt niet af" en "pomp draait te lang" en de storingsmelding van de stroomsensor MOETEN vervallen (vervangt spec 001 FR-013, FR-014, FR-015 en het stroomdeel van FR-016). Er komt geen vervangende melding.
- **FR-003**: Het apparaat MAG in Home Assistant geen entiteiten meer aanbieden voor de pompstatus, de pompstroom, de meetwaarden van de stroomsensor, de looptijd, de pompwaarschuwingen of de storing van de stroomsensor.
- **FR-004**: De instellingen die alleen voor de pompbewaking bestonden, MOETEN vervallen: het inschakelniveau van de dompelschakelaar, de marge en de wachttijd van "pomp start niet", de controletijd en de minimale daling van "pomp pompt niet af", de maximale looptijd van de pomp en de stroomdrempel van de pomp.
- **FR-005**: Het statuspatroon "pomp draait" op led 1 MOET vervallen (vervangt spec 003 FR-014). Het statuspatroon "waarschuwing" (oranje) vervalt ook, omdat er geen waarschuwingen meer zijn.

**Wat blijft**

- **FR-006**: De niveaubewaking MOET ongewijzigd blijven werken: de niveaumeting, de vulgraad en waterhoogte, het hoogwater-alarm, de storingsmelding van de niveausensor, de melding "niet ingesteld", de melding "bewaking niet bereikbaar", de herinneringen en herstelmeldingen, de ledbalk en het bewaren van instellingen.
- **FR-007**: De overgebleven statuspatronen op led 1 (alarm, storing, opstarten, niveau onbekend, geen verbinding en normaal) MOETEN hun kleur, knipperpatroon en onderlinge prioriteit uit spec 003 houden.
- **FR-008**: De gebruiker MOET vanuit Home Assistant de overgebleven instellingen kunnen blijven instellen: afstand leeg, afstand vol, alarmdrempel, herstelmarge, wachttijd hoogwater, wachttijd sensorstoring en led-helderheid. Een gewijzigde instelling is binnen 1 minuut van kracht (zoals spec 001 FR-023).
- **FR-009**: Elke melding MOET in begrijpelijke taal vermelden: het soort melding, de vermoedelijke oorzaak, het actuele niveau en sinds wanneer de situatie speelt. De pompstatus vervalt uit de meldingstekst (vervangt spec 001 FR-019).
- **FR-010**: Home Assistant MOET het verloop van het niveau en de meldingen blijven opslaan en tonen. Pompstarts en -stops vervallen uit de historie en uit de gebeurtenissen (vervangt het pompdeel van spec 001 FR-010 en FR-026).

**Bijwerken van een bestaand apparaat**

- **FR-011**: Na het bijwerken van een bestaand apparaat MOETEN alle eerder bewaarde instellingen, behalve de vervallen pompinstellingen, behouden blijven.
- **FR-012**: Het apparaat MAG een nog aangesloten stroomsensor niet uitlezen en er geen melding over geven. De aansluiting waar de sensor zat, MOET vrij blijven en geen nieuwe functie krijgen.
- **FR-013**: Na het bijwerken MOGEN er in Home Assistant geen actieve meldingen, dashboardonderdelen, automatiseringen of meldingsteksten meer zijn die naar de pomp of de stroomsensor verwijzen.

**Documentatie**

- **FR-014**: De bouwbeschrijving, het aansluitschema, de beschrijving van de voeding, de uitleg van de ledbalk, de koppeling met Home Assistant, de testprocedure op het apparaat en de README MOETEN de stroomsensor en de pompbewaking niet meer als onderdeel van het apparaat noemen. De vrijgekomen aansluiting staat als vrij aangegeven.
- **FR-015**: De documentatie MOET beschrijven hoe een bestaande stroomsensor veilig wordt weggehaald: het is werk aan netspanning, een bevoegd persoon doet het of ziet erop toe, en de voedingsdraad van de pomp wordt daarna weer doorverbonden, zodat de pomp op zijn eigen vlotter blijft werken.

### Key Entities *(include if feature involves data)*

- **Melding**: blijft zoals in spec 001, maar er zijn er nog drie op het apparaat: hoogwater (alarm), niveausensor storing (storing) en niet ingesteld (storing). Home Assistant voegt daar "bewaking niet bereikbaar" aan toe. De soort "waarschuwing" komt niet meer voor.
- **Pompput-configuratie**: de instellingen die het apparaat bewaart. Na deze wijziging: afstand leeg, afstand vol, alarmdrempel, herstelmarge, wachttijd hoogwater, wachttijd sensorstoring en led-helderheid.
- **Statuspatroon**: de toestanden op led 1 uit spec 003, zonder "waarschuwing" en zonder "pomp draait".
- **Pompstatus / pompcyclus** (spec 001): vervalt.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Een ingesteld apparaat zonder stroomsensor, met een werkende niveaumeting, geeft in 24 uur bedrijf 0 storingsmeldingen, en led 1 toont in die tijd geen storingspatroon.
- **SC-002**: Bij het nabootsen van hoogwater, een storing van de niveausensor, "niet ingesteld" en het wegvallen van de verbinding komt in 100% van de gevallen dezelfde melding binnen als vóór de wijziging, binnen 1 minuut na afloop van de ingestelde wachttijd.
- **SC-003**: In dezelfde situaties toont led 1 binnen 5 seconden het juiste patroon, ook zonder verbinding met Home Assistant. Led 1 is in 0 gevallen oranje of langzaam blauw knipperend.
- **SC-004**: In Home Assistant zijn na het bijwerken 0 entiteiten, dashboardonderdelen, meldingen en meldingsteksten over de pomp of de stroomsensor te vinden, en 0 onbeschikbare entiteiten van het apparaat.
- **SC-005**: Na het bijwerken van een bestaand apparaat zijn alle zeven overgebleven instellingen ongewijzigd aanwezig.
- **SC-006**: In de bouwbeschrijving, het aansluitschema, de uitleg van de ledbalk en de testprocedure komt de stroomsensor niet meer als onderdeel van het apparaat voor, en er staat een aanwijzing voor het veilig weghalen van een bestaande sensor.

## Assumptions

- De gebruiker kiest bewust voor bewaking van alleen het niveau. Een pomp die niet start, niet afpompt of droog loopt, wordt niet meer apart gemeld. Het hoogwater-alarm vangt de gevolgen op zodra het water te hoog komt. Dat is later dan de vroegere pompwaarschuwingen, en dat is acceptabel.
- Er komt geen vervangende functie: geen waarschuwing op basis van het niveau alleen, en geen schatting van de pompactiviteit uit het verloop van het niveau.
- De vrijgekomen aansluiting krijgt geen nieuwe functie en blijft vrij voor later gebruik.
- Een nog gemonteerde stroomsensor mag blijven zitten of worden weggehaald; het apparaat werkt in beide gevallen. Weghalen gebeurt door of onder toezicht van een bevoegd persoon.
- De voeding van het apparaat staat los van de voeding van de pomp en verandert niet.
- De historie van pompgegevens in Home Assistant van vóór de wijziging hoeft niet actief verwijderd te worden.
- Het bevestigen van meldingen in Home Assistant en de herinneringsinstellingen in Home Assistant blijven ongewijzigd.
- **Afhankelijkheid (open)**: de constitutie noemt de stroomsensor (ACS712) bij "Mains", en de pompstatus en pompdetectie bij Principle II en IV. De constitutie moet via `/speckit-constitution` worden bijgewerkt, met goedkeuring van de maintainer, vóór of tijdens de uitvoering van deze feature.
- **Afhankelijkheid**: de ontwerpdocumenten van spec 001 en spec 003 (hardware-aansluitingen, signalering ter plaatse en ledbalk, koppeling met Home Assistant, testprocedure) en de README worden in de uitvoering van deze feature bijgewerkt.
