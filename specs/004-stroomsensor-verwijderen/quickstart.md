# Quickstart: Stroomsensor verwijderen valideren

**Feature**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md) | **Datum**: 2026-11-18

Deze gids controleert dat het apparaat zonder stroomsensor werkt, dat de pompbewaking overal is
verdwenen en dat de niveaubewaking ongewijzigd is. Installeren, bouwen en flashen gaan zoals in
[001 quickstart §2](../001-pompput-monitor/quickstart.md#2-installeren-bouwen-en-de-eerste-keer-flashen).
De uitkomsten komen in `specs/004-stroomsensor-verwijderen/verificatie.md`.

## 1. Benodigdheden

- Het bestaande, ingestelde apparaat (firmware 0.4.0) en Home Assistant met de
  ESPHome-integratie.
- **Noteer vóór de update** de waarden van de zeven instellingen die blijven: Afstand leeg,
  Afstand vol, Alarmdrempel, Herstelmarge, Wachttijd hoogwater, Wachttijd sensorstoring en
  Led-helderheid (voor §3 stap 3).
- Een testbak of een object onder de afstandsmeter om het niveau te sturen (zoals in 001 §4).
- Voor scenario #7: het apparaat met de ACS712 nog aangesloten. Voor scenario #8: het apparaat
  zonder ACS712 (of met de draad naar A1 los).

## 2. Compile- en resource-gate

1. **Compile-gate** (gate 1):
   ```powershell
   .venv\Scripts\esphome config esphome/pompput.yaml
   .venv\Scripts\esphome compile esphome/pompput.yaml
   ```
   **Verwacht**: geen fouten en geen waarschuwingen uit de eigen lambdas.
2. **Resource-gate** (gate 2): noteer RAM en flash. **Verwacht**: RAM ≤ 75 %, flash ≤ 90 %, en
   beide lager dan de meting van 0.4.0 in
   [003 verificatie.md](../003-niveau-ledbalk/verificatie.md) (RAM 35,1 %, flash 47,4 %;
   research R12).
3. **Geen pomp- of stroomresten in de firmware**:
   ```powershell
   Select-String -Path esphome/pompput.yaml -Pattern 'pomp_|_pomp|pompstroom|looptijd|stroom|acs712|ct_clamp|adc|offset|inschakel|afpomp|start_marge|start_wachttijd|waarschuwing|oranje|ms_per_min' -CaseSensitive:$false | Where-Object { $_.Line -notmatch '004-stroomsensor-verwijderen|stroomonderbreking' }
   ```
   **Verwacht**: geen uitvoer (FR-001 t/m FR-005, FR-012). De verwijzing naar
   `specs/004-stroomsensor-verwijderen/` en het woord "stroomonderbreking" (flash-opslag, FR-024
   van 001) worden bewust overgeslagen.

## 3. Bijwerken van een bestaand apparaat

1. Flash via WiFi: `.venv\Scripts\esphome run esphome/pompput.yaml --device pompput.local`.
2. **Verwacht**: het apparaat start opnieuw; led 1 is wit en daarna continu blauw, en de
   niveaubalk toont het niveau. In HA toont de apparaatinfo versie `0.5.0`.
3. **Instellingen behouden** (FR-011, SC-005): vergelijk de zeven instellingen in HA met de
   genoteerde waarden. **Verwacht**: alle gelijk. "Niet ingesteld" is niet actief.
4. **Entiteiten opgeruimd** (FR-003, SC-004): open in HA het apparaat "Pompput".
   **Verwacht**: precies de 20 entiteiten uit
   [contracts §1](contracts/wijzigingen-firmware-en-ha.md#1-entiteiten-van-het-apparaat); geen
   van de 15 vervallen entiteiten, ook niet als onbeschikbaar. Staan ze er toch, verwijder ze
   dan en noteer dit in de verificatie.
5. **Pakket en dashboard** (FR-010, FR-013): laad het bijgewerkte pakket en dashboard (HA
   opnieuw starten of YAML herladen). **Verwacht**: geen alerts of `history_stats` over de pomp
   in Instellingen → Entiteiten; op het dashboard geen pomponderdelen en geen regels "entiteit
   niet gevonden".
6. **Geen hangende pompmelding**: was er vóór de update een pompwaarschuwing of
   stroomsensorstoring actief, dan staat die er na stap 5 niet meer (randgeval).

## 4. Scenario's op het apparaat (hardware-gate)

| # | Scenario | Handeling | Verwacht | Spec |
|---|---|---|---|---|
| 1 | Rustig bedrijf | Laat het ingestelde apparaat 15 min draaien met het niveau onder de alarmdrempel en een werkende verbinding | Geen melding actief; Status `normaal`; led 1 continu blauw | US1 scen. 1, SC-001 |
| 2 | Hoogwater | Breng het niveau > wachttijd boven de alarmdrempel; laat het daarna zakken tot onder drempel − herstelmarge | Alarm in HA (tekst zonder "Pomp:"); led 1 snel rood binnen 5 s; daarna herstelmelding en led 1 blauw | US1 scen. 2, FR-006, FR-009, SC-002, SC-003 |
| 3 | Niveausensor storing | Dek de afstandsmeter af tot na de wachttijd sensorstoring; haal de afdekking weg | Storing in HA; led 1 paars; balk uit; daarna herstel | US1 scen. 3, FR-006, SC-002 |
| 4 | Hoog water onder de drempel | Houd het niveau 5 min ruim boven het vroegere inschakelniveau, maar onder de alarmdrempel | Geen melding; led 1 continu blauw; de balk toont het niveau | US1 scen. 4, FR-002 |
| 5 | Zonder Home Assistant | Zet de API-verbinding uit (HA stoppen of WiFi weg) en herhaal #2 | Led 1 wit flitsend, bij het alarm snel rood binnen 5 s | US1 scen. 5, SC-003 |
| 6 | Regels zonder inschakelniveau | Zet Afstand vol op Afstand leeg − 5 cm; zet hem daarna terug | De eerste waarde wordt afgewezen (gebeurtenis `instelling_afgewezen`, HA springt terug); "Niet ingesteld" blijft uit; nergens wordt naar een inschakelniveau gevraagd | FR-008, research R3 |
| 7 | ACS712 nog aangesloten | Herhaal #1 met de sensor aangesloten en de pomp een paar keer aan en uit | Geen storing, geen pompstatus; led 1 knippert niet langzaam blauw als de pomp draait | US3 scen. 1, FR-012, randgeval |
| 8 | Zonder ACS712 | Herhaal #1 en #2 met de sensor weg (of A1 los) | Zelfde uitkomst als #1 en #2 | US1, SC-001 |
| 9 | Led-patronen | Doorloop opstarten, #2, #3, niet ingesteld en geen verbinding | Elk patroon zoals in [contracts §5](contracts/wijzigingen-firmware-en-ha.md#5-signalering-ter-plaatse-led-1); led 1 is nooit oranje of langzaam blauw | US3 scen. 2–3, FR-005, FR-007, SC-003 |
| 10 | Instelling wijzigen | Zet Alarmdrempel van 80 naar 60 % en Led-helderheid van 30 naar 80 % | De balkgrenzen volgen binnen 1 min; de leds volgen de helderheid binnen 1 s | US2 scen. 5, FR-008 |
| 11 | Herstart | Onderbreek de voeding en zet hem terug | Binnen 1 min weer actuele waarden in HA; instellingen behouden | FR-006, FR-011 |
| 12 | Veldcontrole 24 uur | Laat het bijgewerkte, ingestelde apparaat 24 uur in de put draaien (met of zonder ACS712), zonder in te grijpen | In het logboek van HA in die 24 uur geen `melding_actief` van een storing; Status steeds `normaal` (of `alarm` bij echt hoog water) | SC-001 |
| 13 | Meldingsteksten | (a) Zet het apparaat > 5 min stroomloos en daarna weer aan. (b) Zet in HA, Ontwikkelhulpmiddelen → Statussen, `binary_sensor.pompput_niet_ingesteld` tijdelijk op `on`, en daarna terug op `off` | (a) Melding "bewaking niet bereikbaar" en de herstelmelding. (b) Melding "niet ingesteld", die alleen Afstand leeg en Afstand vol noemt, en de herstelmelding. In geen van de teksten staat "Pomp:" | US2 scen. 4, FR-009, SC-002 |

Bij #13 (b) wordt de status met de hand gezet: op het apparaat zelf is "niet ingesteld" niet op
te roepen zonder de flash te wissen, omdat een ongeldige waarde wordt geweigerd. Zo wordt de
tekst van de melding toch getest.

## 5. Documentatie (FR-014, FR-015, SC-006)

```powershell
Select-String -Path README.md, specs/001-pompput-monitor/spec.md, specs/001-pompput-monitor/data-model.md, specs/001-pompput-monitor/quickstart.md, specs/001-pompput-monitor/contracts/*.md, specs/003-niveau-ledbalk/spec.md, specs/003-niveau-ledbalk/data-model.md, specs/003-niveau-ledbalk/contracts/ledbalk.md, specs/003-niveau-ledbalk/quickstart.md, homeassistant/packages/pompput.yaml, homeassistant/dashboard-pompput.yaml -Pattern 'acs712|stroomsensor|pompstroom|looptijd|pomp draait|pompstart|pompwaarschuwing|oranje|inschakelniveau' -CaseSensitive:$false
```

**Verwacht**: alleen treffers die zeggen dat de stroomsensor of de pompbewaking is vervallen
(verwijzingen naar 004), en de paragraaf "Een bestaande stroomsensor weghalen" in `hardware.md`
en de README. Geen beschrijving van de sensor of de pompbewaking als onderdeel van het
apparaat. In het aansluitschema staat A1 als vrij. In `specs/001-pompput-monitor/spec.md` en
`specs/003-niveau-ledbalk/spec.md` zijn ook toegestaan: regels met "(vervallen met 004)" of
"(gewijzigd door 004)", de oorspronkelijke **Input** en de Clarifications van vóór 004.

Lees daarnaast de paragraaf "Een bestaande stroomsensor weghalen" en controleer de vijf punten
uit [contracts §4](contracts/wijzigingen-firmware-en-ha.md#4-hardware) (FR-015, US4 scen. 2).

## 6. Traceerbaarheid: requirement → scenario

| Requirement | Scenario's |
|---|---|
| FR-001 | §2 stap 3; §3 stap 4; §4 #7 |
| FR-002 | §2 stap 3; §4 #4 |
| FR-003 | §3 stap 4 |
| FR-004 | §2 stap 3; §3 stap 3 |
| FR-005 | §2 stap 3; §4 #7, #9 |
| FR-006 | §4 #2, #3, #11 |
| FR-007 | §4 #9 |
| FR-008 | §4 #6, #10 |
| FR-009 | §4 #2, #13; §3 stap 5 |
| FR-010 | §3 stap 5 |
| FR-011 | §3 stap 3; §4 #11 |
| FR-012 | §2 stap 3; §4 #7 |
| FR-013 | §3 stap 5, 6 |
| FR-014 | §5 |
| FR-015 | §5 |
| SC-001 | §4 #1, #8, #12 |
| SC-002 | §4 #2, #3, #13 |
| SC-003 | §4 #2, #5, #9 |
| SC-004 | §3 stap 4, 5 |
| SC-005 | §3 stap 3 |
| SC-006 | §5 |
