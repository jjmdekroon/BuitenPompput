# Feature Specification: Pompput-monitor

**Feature Branch**: `001-pompput-monitor` (aangemaakt op 2026-11-11; spec-map `specs/001-pompput-monitor`)

**Created**: 2026-11-09

**Status**: Draft

**Input**: User description: "Ik heb een pompput buiten en wil deze in de gaten houden. Ik wil met een afstandsmeter kunnen zien hoe vol de put loopt, met een stroommeter controleren wanneer de pomp aanspringt omdat de pomp een eigen dompel-schakelaar heeft. Ik wil ook een waarschuwing krijgen als de pomp niet goed afpompt en het nivo van het water te hoog wordt."

## Clarifications

### Session 2026-11-09

- Q: Hoe wil de gebruiker de stand aflezen en meldingen ontvangen? → A: Via Home Assistant. Status, meldingen en grafieken lopen via het bestaande Home Assistant-systeem. (Later aangevuld met signalering ter plaatse, zie hieronder.)
- Q: Hoeveel historie van pompcycli en waterniveau is gewenst, en waar wordt die bewaard? → A: Home Assistant slaat het waterniveau en alle pompstarts op. Op het apparaat zelf worden geen meetgegevens bewaard.
- Q: Wat moet het apparaat met de gegevens doen als de verbinding met Home Assistant wegvalt? → A: Niets bewaren; de gegevens van die periode gaan verloren. (Een tijdelijke opslag van maximaal 1 dag op het apparaat is overwogen en bewust geschrapt.)
- Q: Waar worden de instellingen aangepast, en mag het apparaat er een kopie van bewaren? → A: In Home Assistant. Het apparaat bewaart de laatst ontvangen instellingen, zodat die een herstart overleven.
- Q: Moet het apparaat bij de put zelf ook de status en meldingen laten zien of horen? → A: Ja. Een RGBW-led toont de status en meldingen, en een piëzo-zoemer met PWM-geluid geeft alarmen en waarschuwingen. Dit komt naast Home Assistant, niet in plaats daarvan. (zoemer vervallen met 002)

### Session 2026-11-11 (naar aanleiding van `/speckit-analyze`)

- Q: Wat doet de niveaumeting als die na een storing terugkomt terwijl het niveau intussen sterk veranderd is? → A: Het apparaat leert het niveau vanzelf opnieuw in. Een hindernis die blijft hangen, kan daardoor een onterecht hoogwater-alarm geven; liever een onterecht alarm dan een gemist alarm.
- Q: Geldt de beperking "hoogstens eens per herinneringsinterval" ook voor de zoemer bij de put? → A: Nee. Die geldt alleen voor meldingen op afstand via Home Assistant (FR-020); de signalering ter plaatse volgt FR-027 en FR-028. (zoemer vervallen met 002)
- Q: Welke waarden moeten vanuit Home Assistant instelbaar zijn? → A: De put-maten en de drempels, marges en wachttijden van de meldingen op het apparaat. Het herinneringsinterval en de wachttijd voor "bewaking niet bereikbaar" staan in de Home Assistant-configuratie; interne filtertijden liggen vast in de firmware (FR-023).

### Session 2026-11-11 (002-zoemer-verwijderen)

- De zoemer vervalt; ter plaatse signaleert alleen de statusled. Zie [specs/002-zoemer-verwijderen](../002-zoemer-verwijderen/spec.md).

### Session 2026-11-11 (003-niveau-ledbalk)

- De enkele led is vervangen door zes leds: led 1 toont de toestand, led 2–6 het niveau. Zie [specs/003-niveau-ledbalk](../003-niveau-ledbalk/spec.md).

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Vulgraad van de put aflezen in Home Assistant (Priority: P1)

Als eigenaar van de pompput wil ik in Home Assistant op elk moment kunnen zien hoe vol de put is en
hoe het niveau in de tijd verloopt, zodat ik weet of alles normaal verloopt zonder het deksel van de
put te hoeven halen.

**Why this priority**: Het waterniveau is de basis van alle andere functies: zowel het hoogwater-alarm
als de controle of de pomp goed afpompt zijn ervan afhankelijk. Alleen al het kunnen aflezen van het
niveau levert direct waarde op.

**Independent Test**: Volledig te testen door de put (of een testbak) tot verschillende bekende
hoogtes te vullen en de waterhoogte en vulgraad die Home Assistant toont te vergelijken met een
handmatige meting.

**Acceptance Scenarios**:

1. **Gegeven** het water staat op het ingestelde leeg-niveau, **wanneer** ik de status in Home Assistant bekijk, **dan** zie ik een vulgraad van 0% en de bijbehorende waterhoogte in centimeters.
2. **Gegeven** het water staat op het ingestelde vol-niveau, **wanneer** ik de status in Home Assistant bekijk, **dan** zie ik een vulgraad van 100%.
3. **Gegeven** het waterniveau stijgt of daalt, **wanneer** ik de status in Home Assistant opnieuw bekijk, **dan** is de getoonde waarde niet ouder dan 30 seconden.
4. **Gegeven** het apparaat kan geen geldige afstandsmeting doen, **wanneer** ik de status in Home Assistant bekijk, **dan** zie ik dat het niveau onbekend is, in plaats van een verouderde waarde zonder vermelding.
5. **Gegeven** het systeem draait al enkele dagen, **wanneer** ik de historie in Home Assistant bekijk, **dan** zie ik het verloop van het waterniveau over die dagen.

---

### User Story 2 - Alarm bij te hoog waterniveau (Priority: P2)

Als eigenaar wil ik via Home Assistant een alarm krijgen zodra het water in de put te hoog komt,
zodat ik kan ingrijpen voordat de put overloopt, ongeacht wat de oorzaak is.

**Why this priority**: Een overlopende put is het grootste risico (wateroverlast). Dit alarm werkt
puur op het waterniveau en biedt daarom al bescherming, ook zonder pompbewaking.

**Independent Test**: Te testen door het water (of een object onder de afstandsmeter) boven de
alarmdrempel te brengen en te controleren dat binnen de ingestelde wachttijd een alarm binnenkomt,
en dat een herstelmelding volgt wanneer het niveau weer voldoende is gedaald.

**Acceptance Scenarios**:

1. **Gegeven** de alarmdrempel staat op 80%, **wanneer** het niveau langer dan 30 seconden boven 80% blijft, **dan** ontvang ik via Home Assistant een hoogwater-alarm met het actuele niveau, en geeft de statusled bij de put alarm. (gewijzigd door 002)
2. **Gegeven** het niveau komt door een golf heel kort boven de drempel, **wanneer** het binnen 30 seconden weer onder de drempel zakt, **dan** ontvang ik geen alarm.
3. **Gegeven** er is een actief hoogwater-alarm, **wanneer** het niveau zakt tot 5 procentpunt onder de drempel, **dan** ontvang ik een herstelmelding en is het alarm niet meer actief.
4. **Gegeven** er is een actief hoogwater-alarm, **wanneer** het een uur later nog steeds actief is, **dan** ontvang ik een herinnering.

---

### User Story 3 - Pompactiviteit volgen (Priority: P3)

Als eigenaar wil ik in Home Assistant kunnen zien wanneer de pomp aanspringt en hoe lang die draait,
zodat ik kan controleren of de eigen dompelschakelaar van de pomp goed werkt.

**Why this priority**: Geeft inzicht in de werking van de pomp en is nodig om storingen van de pomp
te herkennen (User Story 4), maar is zonder die storingsdetectie vooral informatief.

**Independent Test**: Te testen door de pomp een aantal keren te laten aanslaan (via de put of door
de dompelschakelaar handmatig op te tillen) en te controleren dat Home Assistant elke start en stop
met de juiste looptijd registreert en toont.

**Acceptance Scenarios**:

1. **Gegeven** de pomp staat stil, **wanneer** de dompelschakelaar de pomp inschakelt, **dan** toont Home Assistant binnen 5 seconden dat de pomp draait.
2. **Gegeven** de pomp heeft een cyclus afgerond, **wanneer** ik de status in Home Assistant bekijk, **dan** zie ik wanneer de laatste start was en hoe lang de pomp toen heeft gedraaid.
3. **Gegeven** de pomp is vandaag meerdere keren aangeslagen, **wanneer** ik de historie in Home Assistant bekijk, **dan** zie ik alle starts van vandaag en het aantal starts in de afgelopen 24 uur.
4. **Gegeven** de pomp staat stil, **wanneer** er een korte stroompiek optreedt die korter duurt dan 2 seconden, **dan** telt het systeem dit niet als pompstart.

---

### User Story 4 - Waarschuwing als de pomp niet goed afpompt (Priority: P4)

Als eigenaar wil ik gewaarschuwd worden als de pomp niet aanslaat terwijl dat wel zou moeten, of
als de pomp draait maar het water niet zakt, zodat ik een defect (vastzittende vlotter,
verstopping, uitgevallen zekering) al opmerk voordat het water te hoog staat.

**Why this priority**: Combineert niveau en pompactiviteit tot een vroege waarschuwing. Het
hoogwater-alarm (User Story 2) vangt de gevolgen al op; deze story waarschuwt eerder en vertelt
wat de oorzaak waarschijnlijk is.

**Independent Test**: Te testen door storingen na te bootsen: de pomp afkoppelen terwijl het niveau
stijgt, de pomp laten draaien met een afgesloten uitlaat, en de pomp langdurig laten draaien. Bij
elke situatie moet de juiste waarschuwing via Home Assistant binnenkomen.

**Acceptance Scenarios**:

1. **Gegeven** het verwachte inschakelniveau van de dompelschakelaar is ingesteld, **wanneer** het niveau meer dan 5 cm boven dat inschakelniveau komt en de pomp langer dan 2 minuten niet draait, **dan** ontvang ik de waarschuwing "pomp start niet".
2. **Gegeven** de pomp draait, **wanneer** het niveau na 3 minuten pompen niet minstens 2 cm is gedaald, **dan** ontvang ik de waarschuwing "pomp pompt niet af".
3. **Gegeven** de pomp draait, **wanneer** de pomp langer dan de maximale looptijd van 10 minuten onafgebroken draait, **dan** ontvang ik de waarschuwing "pomp draait te lang".
4. **Gegeven** er is een actieve pompwaarschuwing, **wanneer** de pomp weer normaal werkt (aanslaat en het niveau daalt), **dan** ontvang ik een herstelmelding.

---

### Edge Cases

- **Golven en opspattend water** bij instromen of tijdens het pompen geven afwijkende metingen. Een enkele afwijkende meting mag geen melding of sprong in het getoonde niveau veroorzaken.
- **Vervuilde of beslagen afstandsmeter** (condens, spinnenweb, blad) kan een onrealistisch korte afstand meten, wat op een volle put lijkt. Een niveausprong die sneller is dan fysiek mogelijk wordt als onbetrouwbare meting behandeld en leidt tot een storingsmelding, niet tot een hoogwater-alarm. Bij het inleren (na een herstart, na een reset van de niveaumeting of na een langdurige storing) is er geen eerdere waarde om mee te vergelijken. Een hindernis die stil blijft hangen, wordt dan als waterniveau gezien en kan een hoogwater-alarm geven. Dit is een bekende beperking: liever een onterecht alarm dan een gemist alarm.
- **Water tot aan de afstandsmeter** (put overstroomt) maakt meten onmogelijk. Als de meting uitvalt terwijl het laatst bekende niveau boven de alarmdrempel lag, blijft het hoogwater-alarm actief en komt er een storingsmelding bij; het alarm wordt niet stilzwijgend opgeheven. Komt de meting terug terwijl het niveau intussen sterk veranderd is, dan leert het apparaat het niveau vanzelf opnieuw in. Pas een geldige meting onder de herstelgrens heft het alarm op.
- **Pomp krijgt geen stroom** (zekering of aardlekschakelaar eruit, stekker los): er wordt geen stroom gemeten terwijl het niveau stijgt, wat tot de waarschuwing "pomp start niet" leidt.
- **Pomp loopt droog** (vlotter blijft hangen in de aan-stand terwijl de put leeg is): dit wordt gevangen door de waarschuwing "pomp draait te lang".
- **Hevige regen**, waarbij de toestroom groter is dan wat de pomp kan afvoeren: het niveau stijgt ondanks een draaiende pomp. Dit leidt eerst tot "pomp pompt niet af" en zo nodig tot het hoogwater-alarm.
- **Pomp slaat vaak kort aan en af** (pendelen, bijvoorbeeld door een lekkende terugslagklep): dit is zichtbaar via de pompstarts in de historie van Home Assistant.
- **Apparaat niet bereikbaar** (geen verbinding, apparaat zonder stroom of vastgelopen): Home Assistant toont het niveau en de pompstatus dan als onbeschikbaar en geeft de melding "bewaking niet bereikbaar". Het apparaat bewaart geen meetgegevens, dus metingen, pompstarts en meldingen uit die periode gaan verloren. Na herstel wordt direct de actuele toestand doorgegeven; een melding die dan nog actief is, wordt alsnog afgeleverd.
- **Stroomuitval of herstart van het apparaat**: na herstel gaat het apparaat zelfstandig weer meten en gegevens doorgeven. Een melding die vóór de herstart actief was, wordt opnieuw beoordeeld en zo nodig opnieuw gemeld.
- **Herstart terwijl Home Assistant niet bereikbaar is**: het apparaat meet en bewaakt verder met de laatst bewaarde instellingen.
- **Eerste start zonder instellingen**: het apparaat meldt dat het nog niet is ingesteld. Het toont geen vulgraad en geeft geen pompwaarschuwingen op basis van onbekende waarden.
- **Ongeldige instellingen** (bijvoorbeeld een vol-niveau dat lager ligt dan het leeg-niveau): het apparaat weigert de nieuwe waarden, blijft de laatst geldige instellingen gebruiken en meldt dat de wijziging is afgewezen.
- **Home Assistant zelf is niet beschikbaar**: er worden dan geen gegevens opgeslagen en er komen geen meldingen op afstand binnen. De statusled bij de put signaleert wel gewoon. Dit is een geaccepteerde beperking (zie Assumptions). (gewijzigd door 002)
- **Niveau schommelt rond een drempel**: door de herstelmarge (hysterese) wisselt een melding niet steeds tussen actief en hersteld.
- **Vorst**: ijs op het wateroppervlak wordt als waterniveau gemeten. Dit is een bekende beperking (zie Assumptions).

## Requirements *(mandatory)*

### Functional Requirements

**Waterniveau**

- **FR-001**: Het systeem MOET het waterniveau in de put bepalen door van bovenaf de afstand tot het wateroppervlak te meten, zonder contact met het water.
- **FR-002**: Het systeem MOET het niveau in Home Assistant beschikbaar maken als waterhoogte in centimeters boven het leeg-niveau én als vulgraad in procenten (0% = leeg-niveau, 100% = vol-niveau).
- **FR-003**: Het niveau in Home Assistant MOET altijd een meting zijn die niet ouder is dan 30 seconden, of anders als onbekend of onbeschikbaar worden weergegeven.
- **FR-004**: Het systeem MOET losse afwijkende metingen (golven, druppels) uitfilteren, zodat een enkele uitschieter geen melding en geen sprong in het getoonde niveau veroorzaakt.
- **FR-005**: Het systeem MOET een meting als ongeldig aanmerken als die buiten het fysiek mogelijke bereik van de put valt, als het niveau sneller verandert dan fysiek mogelijk is, of als er geen meting lukt.

**Pompactiviteit**

- **FR-006**: Het systeem MOET aan de hand van het stroomverbruik van de pomp vaststellen of de pomp draait of stilstaat.
- **FR-007**: Het systeem MOET de pomp en de dompelschakelaar ongemoeid laten. Het systeem bewaakt alleen, schakelt de pomp nooit, en de pomp MOET normaal blijven werken als de bewaking uitvalt of gestoord is.
- **FR-008**: Het systeem MOET de actuele pompstatus (draait / staat stil) in Home Assistant beschikbaar maken.
- **FR-009**: Het systeem MOET elke pompstart en -stop binnen 5 seconden aan Home Assistant doorgeven, zodat daar per cyclus het startmoment en de looptijd te zien zijn, en ook het aantal starts per periode (minimaal de afgelopen 24 uur).
- **FR-010**: Home Assistant slaat het verloop van het waterniveau, de pompstarts en -stops en de meldingen op en toont ze. Het apparaat bij de put MAG geen historie van metingen, pompcycli of meldingen bewaren, ook niet tijdelijk wanneer de verbinding met Home Assistant wegvalt.
- **FR-011**: Het systeem MAG een stroompiek die korter dan 2 seconden duurt NIET als pompstart tellen.

**Meldingen**

- **FR-012**: Het systeem MOET een **hoogwater-alarm** geven wanneer het niveau langer dan de wachttijd (standaard 30 seconden) boven de alarmdrempel (standaard 80%) staat.
- **FR-013**: Het systeem MOET de waarschuwing **"pomp start niet"** geven wanneer het niveau meer dan een marge (standaard 5 cm) boven het verwachte inschakelniveau van de dompelschakelaar staat en de pomp langer dan een wachttijd (standaard 2 minuten) niet draait.
- **FR-014**: Het systeem MOET de waarschuwing **"pomp pompt niet af"** geven wanneer de pomp draait en het niveau binnen een controletijd (standaard 3 minuten) niet met een minimale daling (standaard 2 cm) is gezakt.
- **FR-015**: Het systeem MOET de waarschuwing **"pomp draait te lang"** geven wanneer de pomp langer dan de maximale looptijd (standaard 10 minuten) onafgebroken draait.
- **FR-016**: Het systeem MOET een **storingsmelding** geven wanneer langer dan 5 minuten geen geldige niveaumeting beschikbaar is, of wanneer de stroommeting een onmogelijke waarde geeft.
- **FR-017**: Het systeem MOET de melding **"bewaking niet bereikbaar"** geven wanneer Home Assistant langer dan 5 minuten geen gegevens van het apparaat heeft ontvangen.
- **FR-018**: Alle meldingen MOETEN via Home Assistant worden afgeleverd en daar als actieve melding zichtbaar zijn. Daarnaast signaleert het apparaat ter plaatse (FR-027). Het apparaat heeft geen display.
- **FR-019**: Elke melding MOET in begrijpelijke taal vermelden: het soort melding (alarm, waarschuwing of storing), de vermoedelijke oorzaak, het actuele niveau, de pompstatus en sinds wanneer de situatie speelt.
- **FR-020**: Zolang een melding actief is, MOET Home Assistant die zichtbaar houden en de gebruiker op afstand hoogstens eens per herinneringsinterval (standaard 1 uur) opnieuw waarschuwen. Daartussen wordt de melding op afstand niet herhaald. De signalering ter plaatse (led) valt hier niet onder; die volgt FR-027. (gewijzigd door 002)
- **FR-021**: Het systeem MOET een herstelmelding geven zodra de oorzaak van een melding is verdwenen. Bij niveaumeldingen geldt daarvoor een herstelmarge (standaard 5 procentpunt onder de drempel), zodat een melding niet steeds wisselt tussen actief en hersteld.
- **FR-022**: Valt de niveaumeting uit terwijl het hoogwater-alarm actief is, dan MOET het alarm actief blijven totdat een geldige meting aantoont dat het niveau weer gezakt is.

**Instellingen en betrouwbaarheid**

- **FR-023**: De gebruiker MOET het leeg-niveau, het vol-niveau, het verwachte inschakelniveau van de dompelschakelaar, de drempels, marges en wachttijden uit FR-012 t/m FR-016 en de herstelmarge uit FR-021 vanuit Home Assistant kunnen instellen, zonder de put te openen en zonder het apparaat opnieuw te programmeren. Een gewijzigde instelling MOET binnen 1 minuut op het apparaat van kracht zijn. Het herinneringsinterval (FR-020) en de wachttijd voor "bewaking niet bereikbaar" (FR-017) worden in de Home Assistant-configuratie ingesteld. Interne filtertijden, zoals de bevestigingstijd van een storing van de stroomsensor en de herstelmarge van "pomp start niet", liggen vast in de firmware.
- **FR-024**: Het apparaat MOET de laatst ontvangen geldige instellingen bewaren en na een stroomonderbreking of herstart direct weer gebruiken, ook als Home Assistant op dat moment niet bereikbaar is. Dit zijn de enige gegevens die het apparaat blijvend bewaart.
- **FR-025**: Na een stroomonderbreking of herstart MOET het apparaat binnen 1 minuut zelfstandig weer meten en de actuele toestand aan Home Assistant doorgeven.
- **FR-026**: Het systeem MOET belangrijke gebeurtenissen (pomp start/stop, begin en einde van meldingen, sensorstoringen, herstart van het apparaat) in Home Assistant zichtbaar maken, zodat de gebruiker achteraf kan nagaan wat er is gebeurd en welke sensor of oorzaak bij een storing hoort.

**Signalering ter plaatse**

- **FR-027**: Het apparaat MOET met een statuslampje (led 1 van de ledbalk) via verschillende kleuren en knipperpatronen tonen: opstarten, normaal, pomp draait, geen verbinding met Home Assistant, storing, waarschuwing en alarm. Deze signalering MOET ook werken als de verbinding of Home Assistant niet beschikbaar is. (gewijzigd door 002) (gewijzigd door 003)
- **FR-028**: De gebruiker MOET de helderheid van de led vanuit Home Assistant kunnen instellen. (gewijzigd door 002) (gewijzigd door 003)

### Key Entities *(include if feature involves data)*

- **Pompput-configuratie**: de vaste gegevens van de put en de bewaking: leeg-niveau, vol-niveau, verwacht inschakelniveau van de dompelschakelaar, alarmdrempel, marges, wachttijden, maximale looptijd en herinneringsinterval. Wordt beheerd in Home Assistant; het apparaat bewaart een kopie van de laatst geldige instellingen.
- **Niveaumeting**: een waarneming van het waterniveau op een bepaald moment: gemeten afstand, afgeleide waterhoogte (cm), vulgraad (%) en of de meting geldig is. Alleen de actuele waarde staat op het apparaat; het verloop wordt in Home Assistant opgeslagen.
- **Pompstatus / pompcyclus**: of de pomp draait. Elke overgang van stil naar draaien en weer terug vormt een cyclus met een startmoment en een looptijd. De cycli worden in Home Assistant opgeslagen.
- **Melding**: een gesignaleerde afwijking: soort (alarm, waarschuwing, storing), oorzaak, begin, einde, status (actief of hersteld) en het moment van de laatste herinnering. Meldingen worden in Home Assistant getoond en opgeslagen. Het apparaat houdt alleen de actuele toestand van elke melding bij, geen historie.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Bij 10 controlemetingen verspreid over het volledige bereik van de put wijkt de waterhoogte in Home Assistant maximaal 2 cm af van een handmatige meting.
- **SC-002**: Gedurende een testperiode van 7 dagen met een werkende verbinding staat 100% van de pompstarts en -stops in Home Assistant, met een afwijking van maximaal 5 seconden ten opzichte van het werkelijke moment.
- **SC-003**: Bij het nabootsen van elk van de vijf storingssituaties (niveau boven alarmdrempel, pomp zonder stroom, afgesloten uitlaat, langdurig draaiende pomp, apparaat losgekoppeld) ontvangt de gebruiker via Home Assistant binnen 1 minuut na afloop van de ingestelde wachttijd een melding met de juiste oorzaak.
- **SC-004**: In 30 dagen normaal gebruik, inclusief regenbuien, komt er hoogstens 1 onterechte melding voor.
- **SC-005**: Na een stroomonderbreking geeft het apparaat binnen 1 minuut weer actuele gegevens aan Home Assistant door, zonder handmatige handeling en zonder verlies van instellingen.
- **SC-006**: De gebruiker kan de actuele vulgraad en pompstatus binnen 10 seconden aflezen in Home Assistant.
- **SC-007**: Als de bewaking wordt losgekoppeld of uitvalt, blijft de pomp in 100% van de testgevallen normaal aanslaan en afslaan op de eigen dompelschakelaar.
- **SC-008**: Een in Home Assistant gewijzigde instelling is binnen 1 minuut van kracht. Na een herstart terwijl Home Assistant onbereikbaar is, meet en bewaakt het apparaat binnen 1 minuut weer met de laatst ingestelde waarden.
- **SC-009**: Een nieuwe melding is binnen 5 seconden na activering zichtbaar op de statusled. Dit geldt ook zonder verbinding met Home Assistant. (gewijzigd door 002) (gewijzigd door 003)

## Assumptions

- Het gaat om één put met één pomp, bewaakt voor één huishouden.
- De gebruiker heeft een werkend Home Assistant-systeem op het thuisnetwerk, met meldingen naar de telefoon of een ander kanaal al ingericht. Home Assistant installeren en onderhouden valt buiten scope. Het beschikbaar maken van de gegevens in Home Assistant en de benodigde dashboard- en meldingsconfiguratie horen wel bij deze feature.
- Bij de put is een stabiele netwerkverbinding naar Home Assistant beschikbaar.
- Home Assistant is het enige kanaal voor historie en voor meldingen op afstand. Als Home Assistant niet beschikbaar is, signaleert alleen de led bij de put. (gewijzigd door 002) Een tweede meldkanaal op afstand valt buiten scope.
- Omdat het apparaat geen meetgegevens bewaart, gaan gegevens uit een periode zonder verbinding verloren. Dat is acceptabel.
- De pomp heeft een eigen dompel-/vlotterschakelaar die leidend blijft. Het systeem is uitsluitend bewakend en stuurt de pomp niet aan; automatisch schakelen van de pomp of een reservepomp valt buiten scope.
- De stroommeting wordt zo aangebracht dat die de pomp niet beïnvloedt en elektrisch veilig is voor de netspanning van de pomp.
- De afstandsmeter hangt boven in de put met vrij zicht op het wateroppervlak. De put heeft over de meethoogte een gelijkmatige doorsnede, zodat waterhoogte en vulgraad recht evenredig zijn. Het volume in liters is niet nodig.
- Het apparaat heeft een eigen voeding die niet wegvalt wanneer de beveiliging van de pomp afslaat. Bij een volledige stroomuitval in huis is bewaken niet mogelijk; noodstroom valt buiten scope voor deze versie.
- Alle standaardwaarden (alarmdrempel 80%, wachttijd hoogwater 30 s, marge "pomp start niet" 5 cm / 2 min, controle "pomp pompt niet af" 2 cm daling in 3 min, maximale looptijd 10 min, sensorstoring na 5 min, "bewaking niet bereikbaar" na 5 min, herinnering elk uur, herstelmarge 5 procentpunt) zijn uitgangspunten die tijdens de installatie op de eigen put worden afgestemd.
- Vaak aan- en afslaan (pendelen) is zichtbaar via de pompstarts in Home Assistant, maar levert in deze versie geen automatische melding op.
- Gebruik bij vorst (ijs op het wateroppervlak) wordt in deze versie niet apart gedetecteerd.
- Buiten scope: meerdere putten, energieverbruik in kWh, regenvoorspelling, en het aansturen van externe apparatuur.
