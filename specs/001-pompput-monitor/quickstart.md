# Quickstart: Pompput-monitor bouwen en valideren

**Feature**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md)

(Herzien op 2026-11-11 voor ESPHome.)

Met deze gids toon je aan dat de feature van begin tot eind werkt. Omdat er geen unittests
meer zijn (constitutie IV), is dit **de** test: elke beslisregel heeft hieronder een scenario
op het apparaat.

Details staan in de contracten en worden hier niet herhaald:
- [hardware.md](contracts/hardware.md)
- [esphome-homeassistant.md](contracts/esphome-homeassistant.md)
- [lokale-signalering.md](contracts/lokale-signalering.md)
- [data-model.md](data-model.md)

Voer alle commando's uit in PowerShell, vanuit de map van de repository.

## 1. Benodigdheden

- **Software**: Windows met Python 3.13. ESPHome 2026.10.1 komt in een eigen `.venv` (§2).
  PlatformIO is niet nodig.
- **Home Assistant**:
  - de ESPHome-integratie (je gebruikt die al voor de Buitenlamp);
  - een notify-service, bijvoorbeeld de Companion-app (`notify.mobile_app_<telefoon>`);
  - `packages` ingeschakeld in `configuration.yaml`.
- **Hardware** volgens [hardware.md](contracts/hardware.md): XIAO ESP32C3 met antenne,
  HC-SR04, ACS712, RGBW-led, weerstanden, een 5 V USB-C-adapter, een USB-C-datakabel,
  en een testbak of emmer.
- **Voor de bankproef van de stroommeting**: een ACS712 in een **veilig afgesloten**
  testsnoer, met een bekende belasting (bijvoorbeeld een waterkoker of de pomp). Laat dit doen
  of controleren door een vakbekwaam persoon.

## 2. Installeren, bouwen en de eerste keer flashen

1. Installeer ESPHome eenmalig:
   ```powershell
   python -m venv .venv
   .venv\Scripts\python -m pip install -r requirements.txt
   .venv\Scripts\esphome version
   ```
   **Verwacht**: `Version: 2026.10.1`.
2. Kopieer `esphome/secrets.example.yaml` naar `esphome/secrets.yaml`.
   - Vul `wifi_ssid` en `wifi_password` in. Gebruik een netwerk op 2,4 GHz.
   - Maak een sleutel voor `api_encryption_key` met:
     ```powershell
     python -c "import secrets,base64;print(base64.b64encode(secrets.token_bytes(32)).decode())"
     ```
   - Bewaar de sleutel: je hebt hem nodig in HA (§3), en hij beveiligt ook de updates via
     WiFi.
3. **Compile-gate** (constitutie, gate 1):
   ```powershell
   .venv\Scripts\esphome config esphome/pompput.yaml
   .venv\Scripts\esphome compile esphome/pompput.yaml
   ```
   **Verwacht**: geen fouten, en geen compilerwaarschuwingen uit de eigen lambdas.
4. **Resource-gate** (gate 2): noteer RAM en flash uit de uitvoer van `compile`. RAM moet
   ≤ 75 % zijn en flash ≤ 90 % (constitutie II).
5. **De eerste keer flashen, via USB**:
   ```powershell
   .venv\Scripts\esphome run esphome/pompput.yaml --device COM5
   ```
   Vervang COM5 door jouw poort; zie Apparaatbeheer.
   - Lukt de verbinding niet: houd **B** (BOOT) ingedrukt, druk kort op **R** (RESET), laat
     B los, en probeer opnieuw. Druk na het flashen op R.
   - **Verwacht** in het log: het opstarten, de herstartreden en de verbinding met WiFi.
     Bij de put: een witte led, en het apparaat blijft stil. Zolang het apparaat niet is
     ingesteld, knippert de led paars.
6. Latere updates gaan via WiFi:
   `.venv\Scripts\esphome run esphome/pompput.yaml --device pompput.local`. Het log lees je met
   `.venv\Scripts\esphome logs esphome/pompput.yaml`.

## 3. Home Assistant koppelen en instellen

1. In HA, onder *Instellingen → Apparaten en diensten*, verschijnt "Pompput" (ESPHome).
   Voeg het toe met de sleutel uit §2.2.
2. **Verwacht**: het apparaat **Pompput** heeft alle entiteiten uit
   [esphome-homeassistant.md](contracts/esphome-homeassistant.md), met de daar genoemde
   entity-id's. De status is `niet_ingesteld`. De drie verplichte instellingen staan op
   "onbekend".
3. Zet `homeassistant/packages/pompput.yaml` in de map `packages` van HA. Vervang elke
   `REPLACE_ME` door je notify-service, en herstart HA.
4. Vul Afstand leeg, Afstand vol en Inschakelniveau in.
   **Verwacht**: binnen 1 minuut is de status `normaal` en flitst de led groen; het apparaat
   blijft stil. "Niet ingesteld" wordt hersteld (FR-023).
5. Voer een ongeldige combinatie in, bijvoorbeeld een Afstand vol die groter is dan de
   Afstand leeg min 10.
   **Verwacht**: de waarde springt terug, en de gebeurtenis `instelling_afgewezen` komt in het
   logboek. "Laatste gebeurtenis" geeft de reden.
6. Haal de stroom van het apparaat en zet hem er weer op.
   **Verwacht**: de geaccepteerde waarden staan er nog. De afgewezen waarde uit stap 5 is
   **niet** teruggekomen (FR-024, research R11).

## 4. Bankvalidatie (hardware-gate)

**Eerst de hardware controleren** (research, Open punten; leg de uitkomsten vast):
- **TRIG op 3,3 V**: Afstand (ruw) geeft kloppende waarden.
- **Kleurvolgorde van de led**: bij "pomp draait" is de led blauw en niet groen; pas anders
  `channel_colors` aan.
- **Led-data op 3,3 V met diode**: de led reageert betrouwbaar.
- **ACS712 in rust**: de Offset stroomsensor is ongeveer 1,25 V. Noteer ook Pompstroom met en
  zonder belasting.

Stel voor de tests tijdelijk kortere tijden in waar dat staat. Zet ze daarna terug.

| # | Scenario | Handeling | Verwacht resultaat | Spec |
|---|---|---|---|---|
| 1 | Niveau aflezen | Houd een plank op bekende afstanden onder de sensor, of gebruik een gevulde bak | Waterhoogte en vulgraad in HA, ≤ 10 s oud, afwijking ≤ 2 cm | US1, FR-001/002/003, SC-001 |
| 2 | Uitschieter | Haal een hand kort (< 1 s) door de bundel | Geen sprong in het niveau, geen melding | FR-004 |
| 3 | Onmogelijke sprong | Zet tijdelijk Wachttijd sensorstoring op 30 s. Houd iets op 10 cm voor de sensor. Haal het **binnen 5 s** weg nadat Niveausensor storing aan gaat (blijft het staan, dan wordt het ingeleerd, zie #29). Druk daarna op "Niveaumeting opnieuw starten" | Niveau "onbekend" na 30 s, **geen** hoogwater, Niveausensor storing na de wachttijd. Na het wegnemen weer een geldig niveau en de storing hersteld. Na de knop: `niveaumeting_herstart` (knop), en binnen ongeveer 5 s weer een geldig niveau | FR-005, FR-016, R6 |
| 4 | Hoogwater | Breng het niveau boven de alarmdrempel en houd het > 30 s vast. Zet in het pakket tijdelijk `repeat: 2` | Hoogwater aan; HA-melding met de oorzaak, het niveau, de pomp en "sinds"; na 2 min een herinnering; de led snel rood, **binnen 5 s** na het actief worden | US2, FR-012/018/019/020, SC-003, SC-009 |
| 5 | Golf | Niveau < 30 s boven de drempel | Geen alarm | US2 scen. 2 |
| 6 | Herstel en hysterese | Zak eerst tot net onder de drempel, daarna tot 5 procentpunt eronder | Bij net onder de drempel blijft het alarm staan. Bij 5 procentpunt eronder: herstelmelding in HA, groene led | FR-021 |
| 7 | Vasthouden | Maak meten onmogelijk tijdens een actief alarm: koppel de ECHO-draad los, of dek af met iets dat geen echo geeft, zoals zacht schuim. Controleer dat Afstand (ruw) onbekend is | Het alarm blijft actief; na de wachttijd komt er een storing bij; de led blijft rood | FR-022, FR-005 |
| 8 | Pomp aan en uit | Zet de belasting 10 s aan, dan uit | Pomp binnen 5 s aan en uit; `pomp_start` en `pomp_stop`; looptijd ≈ 10 s; "Pompput pompstarts 24 uur" +1; de led blauw tijdens het pompen | US3, FR-006/008/009, SC-002 |
| 9 | Stroompiek | Belasting < 2 s aan | Geen pompstart | FR-011 |
| 10 | Pomp start niet | Niveau boven inschakelniveau + 5 cm, zonder belasting; 2 min wachten | Waarschuwing "pomp start niet"; oranje led; herstel zodra de belasting aangaat | US4 scen. 1/4, FR-013 |
| 11 | Pompt niet af | Belasting aan, niveau gelijk houden, 3 min wachten; daarna in een volgende beurt het niveau ≥ 2 cm laten zakken | Waarschuwing "pomp pompt niet af"; na de goede beurt hersteld | US4 scen. 2/4, FR-014 |
| 12 | Draait te lang | Zet tijdelijk Maximale looptijd pomp op 1 min; belasting > 1 min aan | Waarschuwing "pomp draait te lang"; hersteld als de pomp stopt | US4 scen. 3, FR-015 |
| 13 | Stroomsensor los | Onderbreek de voeding van de ACS712 | Na ongeveer 10 s Stroomsensor storing; Pomp "onbekend"; Offset stroomsensor ver buiten 0,9–1,6 V; herstel 10 s na het terugzetten | FR-016 |
| 14 | (vervallen) | — | De zoemer en het stilzetten zijn vervallen met specs/002-zoemer-verwijderen | — |
| 16 | Geen verbinding | Haal het apparaat uit HA, of zet de WiFi-router uit | De led flitst wit (als er geen meldingen zijn); meten en lokale meldingen werken door. In HA na 5 min "Pompput onbereikbaar" | FR-017, FR-027, SC-009 |
| 17 | Herstel verbinding | Zet de verbinding terug | Binnen ongeveer 1 min zijn de states actueel. Gegevens uit de onderbreking worden **niet** nagezonden | FR-010 |
| 18 | Herstart zonder HA | Verbinding uit; apparaat stroomloos en weer aan | Binnen 1 min meet en bewaakt het apparaat, met de bewaarde instellingen | FR-024/025, SC-005, SC-008 |
| 19 | Pomp onafhankelijk | XIAO stroomloos | De belasting of pomp werkt gewoon door de ACS712 heen | FR-007, SC-007 |
| 20 | Gebeurtenissen | Bekijk na de stappen 3–18 het logboek | Herstart, meldingen, pompbeurten, afgewezen instellingen en de herstarts van de niveaumeting staan erin; "Laatste gebeurtenis" geeft de details | FR-026 |
| 21 | Langdurig netwerkverlies | WiFi > 20 min uit, terwijl er een melding actief is | Geen herstart (Uptime loopt door, Herstartreden is ongewijzigd); de melding en de lokale signalering blijven | constitutie (Independent operation) |
| 22 | Update via WiFi | `esphome run … --device pompput.local` | De update slaagt zonder USB; de instellingen blijven bewaard | R2, R15 |
| 23 | Led-helderheid | Zet Led-helderheid op 0 %, dan een alarm | Zonder melding is de led uit; bij een alarm brandt hij toch, op 10 % | FR-028 |
| 24 | Stroomdip tijdens pompen | Belasting > 5 s aan, dan < 2 s onderbreken, dan weer aan | Pomp blijft aan. Geen extra `pomp_stop` of `pomp_start`. De looptijd telt door | FR-009, FR-011 |
| 25 | Stroomsensor los tijdens waarschuwing | Veroorzaak "pomp start niet" (#10). Onderbreek dan de voeding van de ACS712 | Pomp "onbekend" en Stroomsensor storing erbij. "Pomp start niet" blijft actief, zonder herstelmelding | FR-013, FR-016 |
| 26 | Herstel zonder pomp | Veroorzaak #10. Beweeg zonder belasting de plank langzaam (≤ 2 cm/s) weg, tot onder inschakelniveau + marge − 2 cm | "Pomp start niet" hersteld, met herstelmelding | FR-013, FR-021 |
| 27 | Geometrie wijzigen | Wijzig Afstand leeg met een geldige waarde | `niveaumeting_herstart` met oorzaak `instelling`. Het niveau is kort onbekend en binnen ongeveer 5 s weer geldig | FR-023, FR-026 |
| 28 | Herstel na blinde periode | Zet Wachttijd sensorstoring op 30 s. Koppel de ECHO-draad los (Afstand (ruw) wordt onbekend), zet de plank > 10 cm verder, wacht op Niveausensor storing en sluit de draad weer aan | Binnen ongeveer 5 s een geldig niveau op de nieuwe hoogte, `niveaumeting_herstart` (oorzaak `automatisch`), en de storing hersteld | FR-016, FR-022, R6 |
| 29 | Hindernis bij inleren | Houd iets op 10 cm voor de sensor en druk op "Niveaumeting opnieuw starten" | Na ongeveer 5 s een geldig niveau op die hoogte. Ligt dat boven de alarmdrempel, dan volgt na de wachttijd een hoogwater-alarm (bekende beperking, spec randgeval) | FR-005, randgeval |
| 30 | Buiten het bereik | Zet Wachttijd sensorstoring tijdelijk op 30 s. Zet daarna, **in deze volgorde**, Afstand vol op 10 cm, Inschakelniveau op 5 cm en Afstand leeg op 30 cm (in een andere volgorde weigert het apparaat de combinatie). Houd de plank op 60 cm. Herhaal met een hand plat op de sensor. Zet daarna de oude waarden terug, met Afstand leeg als eerste | Afstand (ruw) is onbekend en het niveau blijft onbekend, want inleren lukt niet. Na 30 s komt Niveausensor storing, en er komt **geen** hoogwater. Met de hand gebeurt hetzelfde | FR-005, FR-016 |
| 31 | Uitschakelgrens pomp | Zet een belasting van 1–10 A aan en noteer de Pompstroom I. Zet de Stroomdrempel pomp achtereenvolgens op 0,9 × I, 1,3 × I en 1,5 × I (afgerond op 0,05 A) | Bij 0,9 × I gaat Pomp aan. Bij 1,3 × I blijft Pomp aan, want de uitgrens is 0,91 × I. Bij 1,5 × I is Pomp na ongeveer 2 s uit, met `pomp_stop` | FR-006, data-model §4 |

**Rekentijd** (constitutie II): het log mag tijdens de tests geen meldingen "took a long time"
bevatten voor de eigen scripts en lambdas. ESPHome geeft die melding pas boven 50 ms; de eis
van hoogstens 10 ms wordt aangetoond met de meting uit tasks T039.

## 5. Installatie in de put en kalibratie

1. Monteer de sensor (zie [hardware.md](contracts/hardware.md)). Het vol-niveau ligt
   minstens 20 cm onder de sensor.
2. Lees bij een **lege** put (net na een pompbeurt) **Afstand (ruw)** af, en vul die waarde in
   als Afstand leeg. Dit werkt ook als het apparaat nog niet is ingesteld.
3. Meet Afstand vol met een meetlint: van de sensor tot het gewenste "vol"-niveau.
4. Bepaal het Inschakelniveau: til de vlotter langzaam op, of laat de put vollopen. Noteer de
   Waterhoogte op het moment dat **Pomp** aan gaat.
5. Lees **Pompstroom** af met de pomp aan en met de pomp uit. Zet de Stroomdrempel pomp op
   ongeveer de helft van de stroom bij een draaiende pomp, en ruim boven de waarde in rust.
6. Doe 10 controlemetingen met een meetlat over het bereik, en vergelijk die met
   **Waterhoogte** (SC-001).

## 6. Veldvalidatie (meetbare uitkomsten)

- **SC-002**: noteer 7 dagen lang steekproefsgewijs pompbeurten, en vergelijk die met de
  historie in HA.
- **SC-004**: houd 30 dagen bij hoeveel onterechte meldingen er komen. Het doel is ≤ 1.
  - **Onterecht** is een melding waarvoor bij nacontrole geen werkelijke oorzaak was,
    bijvoorbeeld een hoogwater-alarm zonder hoog water, of een pompwaarschuwing terwijl de pomp
    normaal werkte.
  - Telt **niet** mee: meldingen tijdens tests of installatiewerk, en een terechte melding
    die de gebruiker al kende.
- **SC-006**: zet het voorbeelddashboard (`homeassistant/dashboard-pompput.yaml`). De
  vulgraad en de pompstatus moeten binnen 10 s af te lezen zijn.
- **Bijhouden voor review** (constitutie, gate 4): leg vast welke scenario's níet op hardware
  zijn uitgevoerd, en waarom.

## 7. Traceerbaarheid: requirement → scenario

Constitutie, gate 3. Het overzicht requirement → YAML-element staat in
[plan.md](plan.md#traceerbaarheid).

| Requirement | Scenario's |
|---|---|
| FR-001, FR-002 | §4 #1 |
| FR-003 | §4 #1, #3 |
| FR-004 | §4 #2 |
| FR-005 | §4 #3, #7, #29, #30 |
| FR-006 | §4 #8, #31 |
| FR-008 | §4 #8 |
| FR-007 | §4 #19 |
| FR-009 | §4 #8, #24 |
| FR-010 | §4 #17 |
| FR-011 | §4 #9, #24 |
| FR-012 | §4 #4, #5 |
| FR-013 | §4 #10, #25, #26 |
| FR-014 | §4 #11 |
| FR-015 | §4 #12 |
| FR-016 | §4 #3, #13, #25, #28, #30 |
| FR-017 | §4 #16 |
| FR-018, FR-019, FR-020 | §4 #4 |
| FR-021 | §4 #6, #10, #11, #12, #26 |
| FR-022 | §4 #7, #28 |
| FR-023 | §3 stappen 4–5, §4 #27 |
| FR-024 | §3 stap 6, §4 #18, #22 |
| FR-025 | §4 #18 |
| FR-026 | §4 #20, #27, #28 |
| FR-027 | §4 #4, #10, #13, #16 |
| FR-028 | §4 #23 |
| Randgeval "niet ingesteld" | §2 stap 5, §3 stappen 2 en 4 |
| Randgeval "ongeldige instellingen" | §3 stappen 5–6 |
| Randgeval "vervuilde afstandsmeter" en "water tot aan de sensor" | §4 #3, #28, #29 |
