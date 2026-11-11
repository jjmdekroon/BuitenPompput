# Feature Specification: Zoemer verwijderen

**Feature Branch**: `002-zoemer-verwijderen` (spec-map `specs/002-zoemer-verwijderen`)

**Created**: 2026-11-11

**Status**: Implemented (hardware-scenario's open)

**Input**: User description: "de buzer mag weg, deze is niet meer nodig."

**Wijzigt**: [specs/001-pompput-monitor/spec.md](../001-pompput-monitor/spec.md), in het bijzonder
FR-020, FR-027, FR-028, SC-009, US2 scenario 1, de randgevallen en de Assumptions over
signalering ter plaatse.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - De bewaking werkt zonder zoemer (Priority: P1)

Als eigenaar van de pompput wil ik dat de bewaking geen geluid meer maakt en ook geen zoemer meer
nodig heeft, zodat ik het onderdeel kan weglaten. Meldingen krijg ik via Home Assistant, en bij de
put zie ik de toestand op het statuslampje.

**Why this priority**: Dit is de kern van de wijziging. Zonder deze story blijft het apparaat
geluid maken of verwacht het een onderdeel dat er niet meer is.

**Independent Test**: Bouw het apparaat op zonder zoemer en boots achtereenvolgens opstarten, een
hoogwater-alarm, een pompwaarschuwing, een sensorstoring en het herstel daarvan na. Controleer dat
het apparaat bij geen van deze situaties geluid maakt of probeert te maken, dat het statuslampje
elke toestand toont zoals voorheen, en dat de meldingen in Home Assistant ongewijzigd binnenkomen.

**Acceptance Scenarios**:

1. **Gegeven** het apparaat start op, **wanneer** de opstartfase loopt, **dan** klinkt er geen opstartpiep en toont het statuslampje de opstartfase zoals voorheen.
2. **Gegeven** het niveau komt langer dan de wachttijd boven de alarmdrempel, **wanneer** het hoogwater-alarm actief wordt, **dan** ontvang ik het alarm via Home Assistant en knippert het statuslampje rood, en klinkt er geen geluid.
3. **Gegeven** er wordt een pompwaarschuwing of een sensorstoring actief, **wanneer** ik bij de put sta, **dan** toont het statuslampje de waarschuwing of storing en klinkt er geen geluid.
4. **Gegeven** een melding is actief, **wanneer** de oorzaak verdwijnt, **dan** ontvang ik de herstelmelding via Home Assistant en keert het statuslampje terug naar de normale toestand, zonder herstelpiep.
5. **Gegeven** het apparaat heeft geen verbinding met Home Assistant, **wanneer** er een alarm of waarschuwing actief wordt, **dan** toont het statuslampje die nog steeds binnen 5 seconden.

---

### User Story 2 - Geen zoemerbediening meer in Home Assistant en op het apparaat (Priority: P2)

Als eigenaar wil ik in Home Assistant en op het apparaat geen knoppen of schakelaars meer zien
voor een zoemer die er niet is, zodat het dashboard overzichtelijk blijft en er geen bediening is
die niets doet.

**Why this priority**: De bediening voor stilzetten en uitschakelen heeft zonder zoemer geen
functie meer. Laten staan is verwarrend, maar het raakt de bewaking zelf niet.

**Independent Test**: Werk het apparaat en de Home Assistant-configuratie bij en controleer dat
er geen entiteiten, dashboardonderdelen of meldingsteksten meer zijn die naar de zoemer verwijzen,
en dat de instelling voor de helderheid van het statuslampje nog werkt.

**Acceptance Scenarios**:

1. **Gegeven** het bijgewerkte apparaat is gekoppeld aan Home Assistant, **wanneer** ik de entiteiten van het apparaat bekijk, **dan** is er geen schakelaar "Zoemer" en geen knop "Zoemer stil" meer.
2. **Gegeven** het dashboard van de pompput, **wanneer** ik het open, **dan** staan er geen zoemeronderdelen op en zijn er geen onbeschikbare of verweesde entiteiten zichtbaar.
3. **Gegeven** een melding in Home Assistant, **wanneer** ik de tekst lees, **dan** wordt daarin niet meer naar de zoemer verwezen.
4. **Gegeven** het apparaat is in bedrijf, **wanneer** ik de knop op het apparaat kort indruk, **dan** gebeurt er niets: geen bevestiging op het lampje en geen verandering in de meldingen.
5. **Gegeven** ik pas de helderheid van het statuslampje aan in Home Assistant, **wanneer** de nieuwe waarde is ingesteld, **dan** volgt het lampje die binnen 1 seconde, zoals voorheen.

---

### User Story 3 - Documentatie en aansluitschema zonder zoemer (Priority: P3)

Als degene die het apparaat bouwt of onderhoudt, wil ik dat het aansluitschema, de
bouwbeschrijving en de testprocedure de zoemer niet meer noemen, zodat ik het apparaat zonder
zoemer kan bouwen en controleren zonder tegenstrijdige aanwijzingen.

**Why this priority**: Nodig om het apparaat later correct te kunnen nabouwen of repareren, maar
de werking van de bewaking hangt er niet van af.

**Independent Test**: Lees de bouwbeschrijving, het aansluitschema en de testprocedure door en
controleer dat de zoemer, de aansluiting ervan en de zoemerbediening nergens meer als onderdeel
van het apparaat worden genoemd, en dat de vrijgekomen aansluiting als vrij staat aangegeven.

**Acceptance Scenarios**:

1. **Gegeven** het aansluitschema, **wanneer** ik het doorlees, **dan** komt de zoemer er niet meer in voor en staat de aansluiting waar hij zat als vrij aangegeven.
2. **Gegeven** de testprocedure op het apparaat, **wanneer** ik die uitvoer, **dan** bevat die geen stappen die geluid verwachten of de zoemer stilzetten, en controleert die wel dat het apparaat stil blijft.
3. **Gegeven** de beschrijving van de signalering ter plaatse, **wanneer** ik die lees, **dan** beschrijft die alleen nog het statuslampje.

---

### Edge Cases

- **Er zit nog een zoemer aangesloten** op een bestaand apparaat dat wordt bijgewerkt: de aansluiting wordt niet meer aangestuurd en blijft vanaf het opstarten in een veilige, stille toestand, zodat de zoemer niet gaat brommen of piepen.
- **Bestaande instelling "zoemer uit" in het apparaat**: na het bijwerken heeft die opgeslagen waarde geen effect meer en mag die de opslag van de overige instellingen niet verstoren. De andere bewaarde instellingen (put-maten, drempels, helderheid) blijven behouden.
- **Oude zoemer-entiteiten in Home Assistant**: na het bijwerken mogen er geen onbeschikbare zoemer-entiteiten op het dashboard of in automatiseringen achterblijven.
- **Knop op het apparaat tijdens het opstarten ingedrukt houden**: dit blijft de chip in de programmeermodus zetten (eigenschap van het board). De knop heeft verder geen functie meer.
- **Iemand staat bij de put zonder op het lampje te letten**: er is geen hoorbare waarschuwing meer ter plaatse. Dit is een bewuste keuze van de gebruiker (zie Assumptions).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Het apparaat MAG op geen enkel moment geluid maken: niet bij opstarten, niet bij een alarm, waarschuwing of storing, niet bij herstel en niet als bevestiging van een bediening.
- **FR-002**: De signalering ter plaatse MOET uitsluitend via het statuslampje verlopen. De kleuren, knipperpatronen, prioriteiten en de minimale helderheid bij een alarm blijven gelijk aan wat spec 001 voorschrijft.
- **FR-003**: Het statuslampje MOET een nieuwe melding binnen 5 seconden tonen, ook zonder verbinding met Home Assistant (vervangt SC-009 van spec 001 voor het lampje; het hoorbare deel vervalt).
- **FR-004**: Het apparaat MAG in Home Assistant geen zoemer-gerelateerde bediening meer aanbieden: de schakelaar om de zoemer uit te schakelen en de knop om de zoemer stil te zetten vervallen.
- **FR-005**: De knop op het apparaat MAG geen functie meer hebben tijdens normaal bedrijf. Een druk op de knop verandert niets aan de meldingen, het statuslampje of de bewaking.
- **FR-006**: De gebruiker MOET de helderheid van het statuslampje vanuit Home Assistant kunnen blijven instellen, zoals voorheen.
- **FR-007**: De aansluiting waarop de zoemer zat, MOET vanaf het vroegste moment bij het opstarten in een veilige, niet-aangestuurde toestand staan en daarna niet meer worden gebruikt.
- **FR-008**: Het verwijderen van de zoemer MAG geen gevolgen hebben voor de overige bewaking: niveaumeting, pompdetectie, alle meldingen in Home Assistant, herinneringen, herstelmeldingen en het bewaren van instellingen werken ongewijzigd.
- **FR-009**: Na het bijwerken van een bestaand apparaat MOETEN alle eerder bewaarde instellingen, behalve de vervallen zoemerinstelling, behouden blijven.
- **FR-010**: Meldingsteksten, het dashboard en de Home Assistant-configuratie MOGEN niet meer naar de zoemer of het stilzetten ervan verwijzen.
- **FR-011**: De bouwbeschrijving, het aansluitschema, de beschrijving van de signalering ter plaatse en de testprocedure op het apparaat MOETEN de zoemer, de zoemerbediening en de knopfunctie niet meer als onderdeel van het apparaat noemen. De vrijgekomen aansluiting staat als vrij aangegeven.

### Key Entities *(include if feature involves data)*

- **Signalering ter plaatse**: wat het apparaat bij de put laat zien. Bestaat na deze wijziging alleen nog uit het statuslampje met zijn toestanden (opstarten, normaal, pomp draait, geen verbinding, storing, waarschuwing, alarm) en een instelbare helderheid.
- **Bewaarde instellingen**: de instellingen die het apparaat over een herstart heen bewaart. De zoemerinstelling vervalt; de overige blijven ongewijzigd.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Bij het nabootsen van opstarten, hoogwater-alarm, elke pompwaarschuwing, sensorstoring en herstel maakt het apparaat in 100% van de gevallen geen geluid.
- **SC-002**: In dezelfde situaties toont het statuslampje de juiste toestand binnen 5 seconden, ook zonder verbinding met Home Assistant.
- **SC-003**: Het apparaat werkt volledig zonder aangesloten zoemer: alle testscenario's van de bewaking slagen met een apparaat waarop geen zoemer is gemonteerd.
- **SC-004**: In Home Assistant zijn na het bijwerken 0 entiteiten, dashboardonderdelen en meldingsteksten over de zoemer te vinden, en 0 onbeschikbare entiteiten van het apparaat.
- **SC-005**: Na het bijwerken van een bestaand apparaat zijn alle eerder ingestelde waarden (put-maten, drempels, wachttijden, helderheid) ongewijzigd aanwezig.
- **SC-006**: In de bouwbeschrijving, het aansluitschema en de testprocedure komt de zoemer niet meer als onderdeel van het apparaat voor.

## Assumptions

- De gebruiker kiest bewust voor geen hoorbaar signaal bij de put. Meldingen op afstand via Home Assistant en het statuslampje ter plaatse zijn voldoende. Als Home Assistant niet beschikbaar is, signaleert alleen nog het statuslampje; dat is acceptabel.
- De knop op het apparaat had alleen een functie voor de zoemer en vervalt daarom als bediening. Er komt geen andere functie voor in de plaats (geen nieuwe functionaliteit in deze wijziging).
- De bevestiging op het statuslampje bij het stilzetten van de zoemer vervalt mee, omdat er niets meer stil te zetten is.
- De vrijgekomen aansluiting krijgt geen nieuwe functie; die blijft vrij voor later gebruik.
- Een eventueel nog gemonteerde zoemer mag fysiek blijven zitten of worden verwijderd; het apparaat werkt in beide gevallen.
- Het bevestigen (acknowledge) van meldingen in Home Assistant staat los van de zoemer en blijft ongewijzigd.
- **Afhankelijkheid (afgehandeld)**: de constitutie noemde de zoemer bij Principle III en bij "Independent operation". Dat is bijgewerkt in constitutie 2.0.1 (2026-11-11); de constitutie veronderstelt geen zoemer meer.
- **Afhankelijkheid**: de ontwerpdocumenten van spec 001 (signalering ter plaatse, hardware-aansluitingen, koppeling met Home Assistant, testprocedure) en de README worden in de uitvoering van deze feature bijgewerkt.
