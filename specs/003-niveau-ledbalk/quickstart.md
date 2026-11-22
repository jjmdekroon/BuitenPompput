# Quickstart: Niveau-ledbalk valideren

**Feature**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md) | **Datum**: 2026-11-11

Deze gids controleert dat de zes leds het niveau en de toestand tonen zoals in
[contracts/ledbalk.md](contracts/ledbalk.md), en dat de rest van de bewaking niet is veranderd.
Installeren, bouwen en flashen gaan zoals in
[001 quickstart §2](../001-pompput-monitor/quickstart.md#2-installeren-bouwen-en-de-eerste-keer-flashen).
De uitkomsten komen in `specs/003-niveau-ledbalk/verificatie.md`.

## 1. Benodigdheden

- Het bestaande, ingestelde apparaat met de **zes leds** aangesloten volgens
  [contracts/ledbalk.md §4](contracts/ledbalk.md#4-aansluiting-vervangt-de-led-in-001-hardwaremd),
  met een **1N4001** als diode.
- Home Assistant met de ESPHome-integratie.
- **Noteer vóór de update** de 14 instellingen in HA (voor §3 stap 3).
- Een testbak of een plank die je op bekende hoogtes onder de afstandsmeter kunt houden, en
  een rolmaat. Met de put-maten in HA reken je uit welke afstand bij welke vulgraad hoort, of
  lees je de Vulgraad direct af in HA.

## 2. Compile- en resource-gate

1. **Compile-gate** (gate 1):
   ```powershell
   .venv\Scripts\esphome config esphome/pompput.yaml
   .venv\Scripts\esphome compile esphome/pompput.yaml
   ```
   **Verwacht**: geen fouten en geen waarschuwingen uit de eigen lambdas.
2. **Resource-gate** (gate 2): noteer RAM en flash. **Verwacht**: RAM ≤ 75 %, flash ≤ 90 %
   (research R7; vorige meting 0.3.0: 34,7 % en 47,3 %).
3. **Geen groene led en geen losse getallen meer**:
   ```powershell
   Select-String -Path esphome/pompput.yaml -Pattern 'led_groen|led_normaal_aan_ms'
   ```
   **Verwacht**: geen uitvoer (research R3).

## 3. Bijwerken van een bestaand apparaat

1. Flash via WiFi: `.venv\Scripts\esphome run esphome/pompput.yaml --device pompput.local`.
2. **Verwacht**: bij het opstarten zijn alle leds uit, dan is led 1 wit en blijft de balk uit.
   Na de eerste geldige meting is led 1 continu blauw en toont de balk het niveau. In HA toont
   de apparaatinfo versie `0.4.0`.
3. **Instellingen behouden** (FR-016): de 14 instellingen in HA zijn gelijk aan de genoteerde
   waarden.
4. **Geen nieuwe entiteiten** (research R5): het apparaat "Pompput" in HA heeft geen lamp of
   andere nieuwe entiteit.

## 4. Scenario's op het apparaat (hardware-gate)

**Eerst de bank-controles uit [research.md](research.md#te-controleren-op-de-bank)**:
- **B1 Partities**: bij "normaal" brandt led 1 blauw terwijl er niets anders aan de strip is
  ingeschakeld.
- **B2 Gamma**: laat een storing van de niveausensor ontstaan (#12); led 1 is paars, niet blauw
  (gewijzigd door 004: het oranje van een waarschuwing bestaat niet meer).
- **B3 Kleurvolgorde**: led 1 is bij "normaal" blauw, niet groen of wit.
- **B4 Ketenvolgorde**: bij een stijgend niveau gaat de led naast led 1 als eerste aan.

Stel de alarmdrempel in op **80 %** (standaard), tenzij anders vermeld.

| # | Scenario | Handeling | Verwacht | Spec |
|---|---|---|---|---|
| 1 | Normaal, leeg | Niveau op het leeg-niveau, HA verbonden | Led 1 continu blauw; led 2–6 uit | US1 scen. 1, FR-002, FR-013 |
| 2 | Balk per grens | Breng de vulgraad achtereenvolgens op 0, 10, 20, 35, 48, 50, 66 en 85 % (lees af in HA; 48 % precies op een grens) | Aantal brandende leds (incl. led 1): 1, 1, 2, 3, 4, 4, 5, 6. De leds gaan van onder naar boven aan. Samen met #18 zijn dat 10 controleniveaus | US1 scen. 1, 2, 4, FR-004, FR-006, SC-001 |
| 3 | Balk zakt | Ga terug van 85 % naar 10 % | De leds gaan van boven naar onder uit; de balk is steeds aaneengesloten | US1 scen. 5, FR-006 |
| 4 | Led 6 vóór het alarm | Breng het niveau op 82 % en kijk binnen de wachttijd (30 s) | Led 6 brandt binnen 5 s; led 1 nog blauw (geen alarm) | US1 scen. 3, FR-005, FR-009 |
| 5 | Alarm | Houd 82 % langer dan 30 s vast | Led 1 snel rood binnen 5 s na het actief worden; led 2–6 blijven blauw | US2 scen. 2, FR-003 |
| 6 | Alarm, niveau zakt | Laat het niveau zakken tot 77 % (onder de drempel, boven de herstelgrens) | Led 6 uit; led 1 blijft rood tot het alarm bij 75 % herstelt, daarna continu blauw | randgeval "alarm actief, niveau zakt", FR-005 |
| 7 | (vervallen) | — | Waarschuwing en pomp draait zijn vervallen met specs/004-stroomsensor-verwijderen | — |
| 8 | (vervallen) | — | Waarschuwing en pomp draait zijn vervallen met specs/004-stroomsensor-verwijderen | — |
| 9 | (vervallen) | — | Waarschuwing en pomp draait zijn vervallen met specs/004-stroomsensor-verwijderen | — |
| 10 | Geen verbinding | Stop HA of zet de WiFi uit; daarna #5 | Led 1 wit flits per 2 s, bij het alarm snel rood binnen 5 s; de balk toont het niveau door | US2 scen. 6, FR-009, SC-002 |
| 11 | Opstarten | Onderbreek de voeding en zet hem terug | Eerst alles uit, dan led 1 wit en balk uit; na de eerste geldige meting led 1 blauw en de balk aan | US3 scen. 1, FR-007, FR-012 |
| 12 | Niveau onbekend en sensorstoring | Laat Wachttijd sensorstoring op de standaard 5 min staan en koppel de ECHO-draad los; noteer het moment. Wacht daarna tot "Niveausensor storing" actief wordt (zoals 001 §4 #28) | De eerste 30 s: geen verandering (laatste geldige niveau). Uiterlijk 35 s na het loskoppelen: led 1 paars knipperend (500/1500 ms) en led 2–6 uit, terwijl HA nog status `normaal` en Vulgraad onbekend toont. Bij de storingsmelding: led 1 blijft ononderbroken paars knipperen. Na het aansluiten: binnen 5 s weer de balk en led 1 blauw | US3 scen. 2, 4, 5, FR-007, FR-017, SC-005 |
| 13 | Alarm en meting weg | Tijdens #5: maak meten onmogelijk (001 §4 #7) | Led 1 blijft rood; led 2–6 uit; na herstel van de meting binnen 5 s weer de balk | US3 scen. 3, 4, FR-007 |
| 14 | Niet ingesteld | Alleen bij een apparaat zonder bewaarde put-maten (eerste start, 001 quickstart §2 stap 5). Bij een ingesteld apparaat overslaan en dat in de verificatie noteren | Led 1 paars; led 2–6 uit | FR-007 |
| 15 | Schommeling rond een grens | Houd het niveau 10 min rond 48 % (± 1 procentpunt), bijvoorbeeld met een plank die licht beweegt | Led 4 verandert hoogstens één keer van toestand | FR-008, SC-003 |
| 16 | Andere alarmdrempel | Zet de alarmdrempel op 60 % en houd de vulgraad op 50 % | Binnen 1 min branden led 1–5 (grenzen 12/24/36/48 %); bij 61 % ook led 6. Zet daarna terug op 80 % | US1 scen. 6, FR-010 |
| 17 | Helderheid | Zet Led-helderheid op 80 %, dan 10 %, dan 0 %; bij 0 % ook #5 | Alle zes leds volgen binnen 1 s. Bij 0 % alles uit; bij een alarm alleen led 1 rood op 10 % | FR-011 |
| 18 | Onder leeg en boven vol | Breng het niveau iets onder het leeg-niveau, daarna boven het vol-niveau | Onder leeg: alleen led 1. Boven vol: alle zes leds | randgevallen |
| 19 | Overige bewaking | Voer 001 §4 #4 uit en kijk in HA | Meldingen en herstelmeldingen in HA zoals voorheen | FR-016 |
| 20 | Verbruik | Meet de stroom uit de adapter bij #5 (82 %, alarm actief) met helderheid 100 %. Sinds 004 is dat het zwaarste geval: led 1 paars met een volle balk komt niet meer voor, want paars betekent een ongeldig niveau en dan is de balk uit | Alle zes leds aan (led 1 rood, led 2–6 blauw); hoogstens ongeveer 0,6 A (research R4) | FR-015, research R4 |

## 5. Documentatie (FR-015)

```powershell
Select-String -Path README.md, specs/001-pompput-monitor/contracts/*.md, specs/001-pompput-monitor/quickstart.md, specs/001-pompput-monitor/data-model.md -Pattern 'groen|1N4148|num_leds: 1|één led' -CaseSensitive:$false
```

**Verwacht**: geen treffers die de led nog als groen bij "normaal", als enkele led of met een
1N4148 beschrijven. De README en het aansluitschema tonen zes leds in een keten, led 1 het
dichtst bij het apparaat, en de nieuwe ledtabel.

## 6. Traceerbaarheid: requirement → scenario

| Requirement | Scenario's |
|---|---|
| FR-001 | §1; §4 B4, #2 |
| FR-002 | §4 #1 |
| FR-003 | §4 #5, #10, #12 |
| FR-004 | §4 #2 |
| FR-005 | §4 #4, #6 |
| FR-006 | §4 #2, #3 |
| FR-007 | §4 #11–#14 |
| FR-008 | §4 #15 |
| FR-009 | §4 #4, #5, #10 |
| FR-010 | §4 #16 |
| FR-011 | §4 #17 |
| FR-012 | §3 stap 2; §4 #11 |
| FR-013 | §2 stap 3; §4 #1 |
| FR-014 | vervallen met 004 |
| FR-015 | §5; §4 #20 |
| FR-016 | §3 stap 3; §4 #19 |
| FR-017 | §4 #12, #13; 001 quickstart §4 #27 (kort paars na een herstart van de niveaumeting) |
| SC-001 | §4 #2, #18 |
| SC-002 | §4 #4, #5, #10 |
| SC-003 | §4 #15 |
| SC-004 | §4 #5, #10, #11, #12 |
| SC-005 | §4 #12–#14 |
| SC-006 | §4 #2 (een tweede persoon zegt bij 35, 50 en 85 % binnen 5 s hoeveel stappen het niveau van de alarmdrempel af is) |
