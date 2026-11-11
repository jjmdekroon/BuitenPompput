# Feature Specification: Niveau-ledbalk met zes leds

**Feature Branch**: `003-niveau-ledbalk` (spec-map `specs/003-niveau-ledbalk`)

**Created**: 2026-11-11

**Status**: Implemented (hardware-scenario's open)

**Input**: User description: "Ik heb de RGBW led vervangen door 6 RGBW leds. Deze leds moeten het waternivo weergeven waarbij de eerste led altijd blauw is behalve als 1 van de voorgedefinieerde kleuren waar is. De voorgedefinieerde kleur overruled de blauwe kleur. Als het waternivo het maximum bereikt voor het alarm moet de zesde led blauw kleuren."

**Wijzigt**: [specs/001-pompput-monitor/spec.md](../001-pompput-monitor/spec.md) FR-027, FR-028 en
SC-009 (zoals gewijzigd door [specs/002-zoemer-verwijderen](../002-zoemer-verwijderen/spec.md)), en
de beschrijving van de signalering ter plaatse en het aansluitschema van spec 001.

## Clarifications

### Session 2026-11-11

- Q: "Pomp draait" was continu blauw en valt samen met de nieuwe normale kleur van led 1. Moet de pompstatus bij de put herkenbaar blijven? → A: Ja, met een eigen patroon op led 1: langzaam blauw knipperen (1 s aan / 1 s uit).
- Q: Bij welk waterniveau moeten led 2 t/m led 5 aangaan? → A: In gelijke stappen tussen het leeg-niveau en de alarmdrempel (bij 80%: 16, 32, 48 en 64%; led 6 op 80%). Geen vaste stappen van 20% en geen instelling per led.
- Q: Bij een weggevallen meting is het niveau na 30 s "verouderd", maar de storingsmelding komt pas na de wachttijd (standaard 5 min). Led 1 zou dan blauw zijn met de balk uit, net als bij een lege put. Wat moet led 1 dan tonen? → A: Het storingspatroon (paars knipperen), ook als de storingsmelding nog niet actief is; buiten de opstartfase, met een prioriteit direct onder opstarten.

## Begrippen

- **Led 1 t/m led 6**: de zes leds van de ledbalk bij de put. Led 1 is de onderste led, led 6 de
  bovenste. Led 1 zit het dichtst bij het apparaat.
- **Statuslampje**: led 1. Toont de toestand van het apparaat, zoals voorheen de enkele led.
- **Niveaubalk**: led 2 t/m led 6. Toont hoe hoog het water staat ten opzichte van de
  alarmdrempel.
- **Statuspatroon**: een van de vooraf gedefinieerde kleuren en knipperpatronen uit spec 001:
  alarm (rood), waarschuwing (oranje), storing (paars), opstarten (wit) en geen verbinding (wit),
  plus "pomp draait" met het nieuwe patroon uit FR-014 (langzaam blauw knipperen) en "niveau
  onbekend" met het patroon van storing (FR-017).
- **Vulgraad**: het niveau in procenten, met 0% = leeg-niveau en 100% = vol-niveau (spec 001
  FR-002). De alarmdrempel is ook een vulgraad (standaard 80%).

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Waterniveau aflezen op de ledbalk (Priority: P1)

Als eigenaar wil ik bij de put in één oogopslag zien hoe hoog het water staat, zonder Home
Assistant te openen. Hoe meer leds er blauw branden, hoe hoger het water. Brandt de bovenste led,
dan staat het water op of boven de alarmdrempel.

**Why this priority**: Dit is de nieuwe functie waarvoor de zes leds zijn aangebracht. Ook zonder
telefoon of netwerk ziet de gebruiker hoe dicht het water bij het alarm is.

**Independent Test**: Breng het water (of een testbak) achtereenvolgens op 0%, 10%, 20%, 40%,
60%, 70% en 85% vulgraad bij een alarmdrempel van 80%. Controleer per niveau hoeveel leds blauw
branden en vergelijk dat met de tabel in FR-004.

**Acceptance Scenarios**:

1. **Gegeven** het water staat op het leeg-niveau en er is geen statuspatroon actief, **wanneer** ik naar de ledbalk kijk, **dan** brandt alleen led 1, continu blauw.
2. **Gegeven** de alarmdrempel staat op 80%, **wanneer** de vulgraad 50% is, **dan** branden led 1 t/m led 4 blauw en zijn led 5 en led 6 uit.
3. **Gegeven** de alarmdrempel staat op 80%, **wanneer** de vulgraad 80% of hoger wordt, **dan** brandt led 6 blauw, direct, ook als de wachttijd van het hoogwater-alarm nog niet voorbij is.
4. **Gegeven** het water stijgt geleidelijk van leeg tot boven de alarmdrempel, **wanneer** ik de ledbalk volg, **dan** gaan de leds één voor één van onder naar boven aan, en nooit in een andere volgorde.
5. **Gegeven** het niveau zakt weer, **wanneer** het onder de grens van een led komt, **dan** gaat die led uit, van boven naar onder.
6. **Gegeven** ik verander de alarmdrempel in Home Assistant van 80% naar 60%, **wanneer** de nieuwe instelling op het apparaat van kracht is, **dan** gaat led 6 aan bij 60% en zijn de grenzen van led 2 t/m led 5 daarop aangepast, binnen 1 minuut.

---

### User Story 2 - Toestand van het apparaat op led 1 (Priority: P1)

Als eigenaar wil ik dat led 1 normaal blauw brandt, maar een alarm, waarschuwing, storing,
opstartfase of verbroken verbinding toont met de kleur en het knipperpatroon die ik al ken. Zo zie
ik bij de put zowel het niveau als een eventueel probleem.

**Why this priority**: De bestaande signalering ter plaatse mag niet verloren gaan; zonder deze
story zou de gebruiker bij de put geen alarm of storing meer zien.

**Independent Test**: Boots met de ledbalk aangesloten achtereenvolgens opstarten, een
hoogwater-alarm, een pompwaarschuwing, een sensorstoring en het wegvallen van de verbinding met
Home Assistant na. Controleer dat led 1 telkens het bijbehorende statuspatroon toont en daarna
weer continu blauw wordt.

**Acceptance Scenarios**:

1. **Gegeven** er is geen statuspatroon actief, **wanneer** ik naar led 1 kijk, **dan** brandt die continu blauw.
2. **Gegeven** het hoogwater-alarm wordt actief, **wanneer** ik naar led 1 kijk, **dan** knippert die snel rood zoals beschreven in spec 001, in plaats van blauw.
3. **Gegeven** er is tegelijk een waarschuwing en een storing actief, **wanneer** ik naar led 1 kijk, **dan** toont die de waarschuwing (oranje), volgens de bestaande prioriteit.
4. **Gegeven** een statuspatroon is actief, **wanneer** de oorzaak verdwijnt, **dan** wordt led 1 binnen 5 seconden weer continu blauw.
5. **Gegeven** led 1 toont een statuspatroon, **wanneer** ik naar led 2 t/m led 6 kijk, **dan** tonen die gewoon het niveau (behalve bij een ongeldige meting, zie User Story 3).
6. **Gegeven** de verbinding met Home Assistant is weg, **wanneer** er een alarm actief wordt, **dan** toont led 1 het alarm toch binnen 5 seconden.
7. **Gegeven** er is geen ander statuspatroon actief, **wanneer** de pomp gaat draaien, **dan** knippert led 1 langzaam blauw (1 s aan / 1 s uit), en wordt die binnen 5 seconden na het stoppen van de pomp weer continu blauw.
8. **Gegeven** de pomp draait en er is een waarschuwing actief, **wanneer** ik naar led 1 kijk, **dan** toont die de waarschuwing (oranje), omdat die een hogere prioriteit heeft dan "pomp draait".

---

### User Story 3 - Geen misleidende niveauweergave bij een ongeldige meting (Priority: P2)

Als eigenaar wil ik dat de niveaubalk uit gaat als het apparaat het niveau niet betrouwbaar kent,
zodat ik nooit een niveau zie dat niet klopt.

**Why this priority**: Een verkeerd getoond niveau is gevaarlijker dan geen niveau. Led 1 laat in
die gevallen al een storing of de opstartfase zien.

**Independent Test**: Start het apparaat op, dek de afstandsmeter af tot er een sensorstoring
komt, en wis de put-maten zodat het apparaat "niet ingesteld" meldt. Controleer dat led 2 t/m
led 6 in al die gevallen uit zijn en dat led 1 het juiste statuspatroon toont.

**Acceptance Scenarios**:

1. **Gegeven** het apparaat start op en heeft nog geen geldige niveaumeting, **wanneer** ik naar de ledbalk kijk, **dan** toont led 1 het opstartpatroon (wit) en zijn led 2 t/m led 6 uit.
2. **Gegeven** er is een sensorstoring van de niveaumeting, **wanneer** ik naar de ledbalk kijk, **dan** toont led 1 storing (paars) en zijn led 2 t/m led 6 uit.
3. **Gegeven** het hoogwater-alarm is actief en de niveaumeting valt uit, **wanneer** ik naar de ledbalk kijk, **dan** blijft led 1 het alarm tonen (spec 001 FR-022) en zijn led 2 t/m led 6 uit.
4. **Gegeven** de meting was ongeldig, **wanneer** er weer een geldige meting is, **dan** toont de niveaubalk binnen 5 seconden weer het actuele niveau en wordt led 1 weer blauw (of het patroon dat dan geldt).
5. **Gegeven** de opstartfase is voorbij en de meting valt weg, **wanneer** het niveau na 30 seconden als verouderd geldt maar de storingsmelding nog niet actief is, **dan** knippert led 1 paars zoals bij een storing en zijn led 2 t/m led 6 uit, zodat de ledbalk niet op een lege put lijkt.

---

### User Story 4 - Bouwbeschrijving en testprocedure voor zes leds (Priority: P3)

Als degene die het apparaat bouwt of onderhoudt, wil ik dat het aansluitschema, de voeding, de
beschrijving van de signalering ter plaatse en de testprocedure uitgaan van zes leds, zodat ik de
ledbalk correct kan aansluiten en controleren.

**Why this priority**: Nodig om het apparaat later na te bouwen of te repareren; de werking zelf
hangt er niet van af.

**Independent Test**: Lees de bouwbeschrijving, het aansluitschema en de testprocedure door en
controleer dat ze zes leds beschrijven, de volgorde (led 1 onder, het dichtst bij het apparaat)
aangeven, het stroomverbruik van zes leds meenemen en per led-grens een testscenario bevatten.

**Acceptance Scenarios**:

1. **Gegeven** het aansluitschema, **wanneer** ik het doorlees, **dan** staat erin hoe de zes leds achter elkaar worden aangesloten en welke led led 1 is.
2. **Gegeven** de beschrijving van de voeding, **wanneer** ik die lees, **dan** is het stroomverbruik van zes leds meegenomen en volstaat de voorgeschreven adapter.
3. **Gegeven** de testprocedure op het apparaat, **wanneer** ik die uitvoer, **dan** controleert die elke grens van de niveaubalk en elk statuspatroon op led 1.

---

### Edge Cases

- **Niveau schommelt rond de grens van een led** (golfslag, ruis): de led mag niet steeds aan en uit gaan. Een led gaat pas weer uit als het niveau duidelijk onder zijn grens is gezakt (herstelmarge, zie FR-008).
- **Niveau onder het leeg-niveau** (negatieve vulgraad): alleen led 1 brandt; de niveaubalk is uit.
- **Niveau boven het vol-niveau** (meer dan 100%): led 1 t/m led 6 branden blauw.
- **Niveau boven de alarmdrempel, maar de wachttijd van het alarm loopt nog**: led 6 brandt al blauw, led 1 is nog blauw (geen alarm). Dit is een vroege waarschuwing ter plaatse.
- **Alarm actief, niveau zakt onder de alarmdrempel maar nog niet onder de herstelgrens**: led 6 gaat uit zodra het niveau onder zijn grens (met herstelmarge) komt; led 1 blijft het alarm tonen tot het alarm volgens spec 001 is hersteld.
- **Helderheid op 0% gezet**: alle zes leds zijn uit, behalve led 1 bij een alarm; die brandt dan op de minimale alarmhelderheid (spec 001).
- **Alarmdrempel of put-maten veranderd in Home Assistant**: de grenzen van de niveaubalk worden opnieuw bepaald zodra de nieuwe instelling op het apparaat van kracht is.
- **Meting valt weg, storingsmelding nog niet actief**: het niveau is na 30 s verouderd, de storingsmelding komt pas na de wachttijd. In die tijd knippert led 1 al paars en is de balk uit (FR-017). In Home Assistant is de status dan nog "normaal" en het niveau "onbekend"; dat verandert niet.
- **Niveaumeting opnieuw gestart** (met de knop "Niveaumeting opnieuw starten" of na een gewijzigde afstand leeg of vol): het niveau is enkele seconden onbekend terwijl het opnieuw wordt ingeleerd. Led 1 knippert dan kort paars en de balk is uit (FR-017); daarna wordt led 1 weer blauw en toont de balk het niveau. Dit is geen storing.
- **Een led in het midden van de balk is defect**: led 1 blijft werken, omdat die het dichtst bij het apparaat zit. Leds na de defecte led kunnen ook uitvallen; dat wordt niet gedetecteerd of gemeld (zie Assumptions).
- **Pomp draait**: het oude patroon "pomp draait" (continu blauw) zou samenvallen met de nieuwe normale kleur van led 1. Daarom krijgt "pomp draait" een eigen patroon: langzaam blauw knipperen (FR-014). Een pomp die kort aan- en uitgaat, laat led 1 wisselen tussen knipperen en continu blauw; dat is gewenst, omdat het de werkelijke pompstatus volgt.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: De signalering ter plaatse MOET bestaan uit zes leds in een vaste volgorde: led 1 (onder) t/m led 6 (boven). Led 1 is het statuslampje, led 2 t/m led 6 vormen de niveaubalk.
- **FR-002**: Led 1 MOET continu blauw branden zolang er geen statuspatroon actief is.
- **FR-003**: Zodra een statuspatroon actief is (alarm, waarschuwing, storing, opstarten, niveau onbekend, geen verbinding of pomp draait), MOET led 1 dat patroon tonen in plaats van blauw. Kleuren, knipperpatronen en de onderlinge prioriteit blijven gelijk aan spec 001, behalve FR-013, FR-014 en FR-017; de volledige tabel staat in [contracts/ledbalk.md §1](contracts/ledbalk.md#1-led-1-statuslampje). Bij meerdere actieve toestanden wint de toestand met de hoogste prioriteit.
- **FR-004**: Led 2 t/m led 6 MOETEN het niveau tonen als een balk van continu blauwe leds. Led *n* (n = 2 t/m 6) brandt wanneer de vulgraad minstens (n − 1) / 5 × de alarmdrempel is: vijf gelijke stappen tussen het leeg-niveau en de alarmdrempel. De grenzen zijn niet per led instelbaar. Bij de standaard-alarmdrempel van 80% betekent dat:

  | Led | Brandt vanaf vulgraad |
  |---|---|
  | led 2 | 16% |
  | led 3 | 32% |
  | led 4 | 48% |
  | led 5 | 64% |
  | led 6 | 80% (de alarmdrempel) |

- **FR-005**: Led 6 MOET blauw branden zodra de vulgraad de alarmdrempel bereikt, ongeacht of het hoogwater-alarm al actief is (de wachttijd van het alarm speelt hier geen rol).
- **FR-006**: De niveaubalk MOET altijd aaneengesloten zijn: als led *n* brandt, branden ook alle leds daaronder.
- **FR-007**: Als er geen geldige niveaumeting is (opstarten, sensorstoring van de niveaumeting, niet ingesteld, of meting uitgevallen tijdens een alarm), MOETEN led 2 t/m led 6 uit zijn. Led 1 toont dan een statuspatroon en nooit continu blauw: opstarten (wit) tijdens de opstartfase, alarm als dat actief is, en anders storing (paars), ook vóór de storingsmelding (FR-017).
- **FR-008**: Een led van de niveaubalk MOET pas uitgaan als de vulgraad minstens de herstelmarge van de balk onder zijn grens is gezakt (standaard 2 procentpunt, vast in de firmware), zodat de balk bij schommelingen rond een grens niet knippert.
- **FR-009**: Een verandering van het niveau over een grens MOET binnen 5 seconden zichtbaar zijn op de niveaubalk, en een nieuw statuspatroon binnen 5 seconden op led 1 (vervangt SC-009 van spec 001). Dit geldt ook zonder verbinding met Home Assistant of WiFi.
- **FR-010**: De grenzen van de niveaubalk MOETEN de actuele alarmdrempel en put-maten volgen; een gewijzigde instelling is binnen 1 minuut zichtbaar op de balk (zoals spec 001 FR-023).
- **FR-011**: De helderheid die de gebruiker in Home Assistant instelt, MOET voor alle zes leds gelden. De minimale helderheid bij een alarm (spec 001) blijft gelden voor led 1.
- **FR-012**: Alle zes leds MOETEN vanaf het vroegste moment bij het opstarten uit zijn en daarna de opstartfase tonen volgens FR-003 en FR-007.
- **FR-013**: Het statuspatroon "normaal" (groene flits) vervalt; de normale toestand is voortaan led 1 continu blauw.
- **FR-014**: Het statuspatroon "pomp draait" MOET op led 1 te herkennen zijn als langzaam blauw knipperen: 1 seconde aan, 1 seconde uit. Het houdt zijn plaats in de prioriteit van spec 001: onder "geen verbinding" en boven de normale toestand.
- **FR-015**: Het aansluitschema, de beschrijving van de voeding, de beschrijving van de signalering ter plaatse, de testprocedure op het apparaat en de README MOETEN de ledbalk met zes leds beschrijven, inclusief de volgorde van de leds, het stroomverbruik en een testscenario per grens van de niveaubalk.
- **FR-016**: Deze wijziging MAG geen gevolgen hebben voor de overige bewaking: niveaumeting, pompdetectie, de meldingen in Home Assistant, herinneringen, herstelmeldingen en het bewaren van instellingen werken ongewijzigd.
- **FR-017**: Is na de opstartfase het niveau niet bruikbaar (onbekend, verouderd, storing of niet ingesteld), dan MOET led 1 het patroon van storing tonen (paars, 500 ms aan / 1500 ms uit), ook als de storingsmelding van spec 001 nog niet actief is. Dit "niveau onbekend" staat in de prioriteit direct onder opstarten en boven geen verbinding; alarm en waarschuwing gaan voor. De meldingen en de status in Home Assistant veranderen hierdoor niet.

### Key Entities *(include if feature involves data)*

- **Ledbalk**: de zes leds bij de put. Led 1 is het statuslampje, led 2 t/m led 6 de niveaubalk. Heeft één instelbare helderheid.
- **Grens van de niveaubalk**: per led 2 t/m 6 de vulgraad waarbij die led aangaat. Afgeleid van de alarmdrempel; geen eigen instelling.
- **Statuspatroon**: de bestaande toestanden met kleur, knipperpatroon en prioriteit uit spec 001, zonder "normaal", met een nieuw patroon voor "pomp draait" (FR-014) en met de nieuwe toestand "niveau onbekend" (FR-017).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Bij 10 controleniveaus verspreid van leeg tot boven de alarmdrempel klopt het aantal brandende leds in 100% van de gevallen met de tabel in FR-004.
- **SC-002**: Een niveauverandering over een grens en een nieuw statuspatroon zijn binnen 5 seconden zichtbaar, ook zonder verbinding met Home Assistant.
- **SC-003**: Bij een niveau dat 10 minuten binnen 1 procentpunt rond een grens schommelt, verandert de betreffende led hoogstens één keer van toestand.
- **SC-004**: Bij het nabootsen van opstarten, alarm, elke pompwaarschuwing, sensorstoring, geen verbinding en een draaiende pomp toont led 1 in 100% van de gevallen het juiste statuspatroon, en is led 1 binnen 5 seconden na herstel weer continu blauw.
- **SC-005**: Bij een ongeldige niveaumeting zijn led 2 t/m led 6 in 100% van de gevallen uit en is led 1 nooit continu blauw, uiterlijk 35 seconden nadat de meting is weggevallen.
- **SC-006**: Een gebruiker die de uitleg van de ledbalk één keer heeft gelezen, kan bij de put binnen 5 seconden zeggen hoeveel stappen (van de vijf) het niveau van de alarmdrempel af is, en of het de alarmdrempel heeft bereikt.

## Assumptions

- De zes leds zijn van hetzelfde type als de oorspronkelijke led en zitten achter elkaar op één aansluiting van het apparaat. Er komt geen extra aansluiting bij.
- De ledbalk hangt verticaal met led 1 onder en led 6 boven, zodat de balk meestijgt met het water. Led 1 zit het dichtst bij het apparaat in de keten.
- Alle leds van de niveaubalk branden in dezelfde blauwe kleur als led 1 en knipperen niet. Led 6 wordt bij de alarmdrempel ook blauw (niet rood); het alarm zelf is te zien aan led 1.
- De grenzen van led 2 t/m led 5 liggen op gelijke afstanden tussen het leeg-niveau en de alarmdrempel. Er komen geen aparte instellingen per led in Home Assistant.
- De herstelmarge van de niveaubalk (standaard 2 procentpunt) is een interne filtertijd/-marge en ligt vast in de firmware, net als andere interne filters (spec 001 FR-023).
- Led 6 volgt alleen het niveau, niet de toestand van het hoogwater-alarm. Of een alarm actief is, blijkt uit led 1.
- De bestaande voeding (5 V-adapter van minimaal 1 A) is voldoende, omdat de firmware de helderheid begrenst en de leds in normaal bedrijf alleen blauw branden. Dit wordt in de uitvoering nagerekend en vastgelegd.
- Een defecte led in de keten wordt niet gedetecteerd of gemeld; dat valt buiten scope.
- Home Assistant krijgt geen nieuwe entiteiten voor de ledbalk; de bestaande helderheidsinstelling blijft de enige bediening.
- **Afhankelijkheid**: de ontwerpdocumenten van spec 001 (signalering ter plaatse, hardware-aansluitingen, testprocedure) en de README worden in de uitvoering van deze feature bijgewerkt.
