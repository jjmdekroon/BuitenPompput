---

description: "Takenlijst voor het verwijderen van de zoemer"
---

# Tasks: Zoemer verwijderen

**Input**: Design documents from `specs/002-zoemer-verwijderen/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md),
[data-model.md](data-model.md), [contracts/wijzigingen-signalering-en-ha.md](contracts/wijzigingen-signalering-en-ha.md),
[quickstart.md](quickstart.md)

**Tests**: geen geautomatiseerde tests (constitutie IV). Controle gebeurt met de compile- en
resource-gate en de scenario's op het apparaat in [quickstart.md](quickstart.md).

**Organization**: taken per user story. Alle firmwarewerk zit in US1, omdat de zoemerscripts,
de schakelaar, de knop "Zoemer stil" en de BOOT-knop elkaar aanroepen: de firmware compileert
pas weer als ze allemaal weg zijn. Het pinnencontract `hardware.md` zit ook in US1, omdat de
constitutie eist dat het `substitutions`-blok en `hardware.md` altijd gelijk lopen. US2 doet
de Home Assistant-kant, US3 de overige documentatie.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: kan parallel (ander bestand, geen afhankelijkheid van een open taak)
- **[Story]**: US1, US2 of US3 uit [spec.md](spec.md)

## Path Conventions

- Firmware: `esphome/pompput.yaml` (genummerde secties 1–8)
- Home Assistant: `homeassistant/packages/pompput.yaml`, `homeassistant/dashboard-pompput.yaml`
- Documentatie: `README.md`, `specs/001-pompput-monitor/`, `specs/002-zoemer-verwijderen/`
- Commando's draaien in PowerShell vanuit de root van de repository met `.venv\Scripts\esphome`

---

## Phase 1: Setup

**Purpose**: vastleggen hoe het apparaat er vóór de wijziging voor staat, zodat FR-009, SC-005
en de resource-gate te controleren zijn.

- [X] T001 Maak `specs/002-zoemer-verwijderen/verificatie.md` met de koppen: "Samenvatting van de gates" (tabel gate 1–5 met kolommen Gate, Uitkomst, Datum), "Uitgangssituatie", "Compile- en resource-gate", "Bijwerken" (quickstart §3), "Scenario's op het apparaat" (tabel #1–#11 uit quickstart §4 met kolommen #, Uitgevoerd ja/nee, Uitkomst, Reden als niet uitgevoerd), "Documentatie" (quickstart §5), "Review- en traceerbaarheidsgate" en "Afwijkingen"
- [X] T002 Noteer onder "Uitgangssituatie" in `specs/002-zoemer-verwijderen/verificatie.md`: de laatste RAM- en flashwaarden uit `specs/001-pompput-monitor/verificatie.md` (sectie "T039: laatste compile- en resource-gate": RAM 35,3 %, flash 48,5 %), en de actuele waarden van de 14 `number`-instellingen van het apparaat in Home Assistant. Kan het apparaat niet worden uitgelezen, schrijf dat op met de reden (constitutie IV)

---

## Phase 2: Foundational

Geen blokkerende voorbereiding nodig: er komt geen nieuwe infrastructuur bij. US1 kan direct
na Phase 1 beginnen; US2 en US3 ook (zie Dependencies).

---

## Phase 3: User Story 1 - De bewaking werkt zonder zoemer (Priority: P1) 🎯 MVP

**Goal**: firmware 0.3.0 zonder zoemer, stilzet-logica en BOOT-knopfunctie; de statusled
signaleert precies zoals voorheen; het pinnencontract klopt met de firmware (FR-001 t/m
FR-009).

**Independent Test**: compile- en resource-gate ([quickstart.md](quickstart.md) §2), bijwerken
via WiFi (§3 stap 1–3) en scenario's §4 #1–#9: nooit geluid, led ongewijzigd, instellingen
behouden.

De taken T003–T010 en T012–T013 wijzigen `esphome/pompput.yaml` en lopen dus na elkaar. T011
wijzigt `hardware.md` en hoort direct na T010, zodat pinlijst en contract samen veranderen.

- [X] T003 [US1] Verwijder in sectie 2 (`esphome:` → `on_boot`) van `esphome/pompput.yaml` stap 4 "De opstartpiep" (het `script.execute` van `zoemer_eenmalig` met `altijd: true`) en nummer de volgende stap van 5 naar 4. Pas het commentaar boven `on_boot` aan: "De led staat al uit (restore_mode ALWAYS_OFF) en TRIG is laag" (zonder "de zoemer-output begint uit"), en haal `FR-028` uit de FR-regel daarboven (FR-025, FR-027 blijven)
- [X] T004 [US1] Vervang in sectie 5, script `status_bijwerken` in `esphome/pompput.yaml`, de laatste actie `script.execute: signalering_bijwerken` door `script.execute: led_patroon_kiezen` (research R1; FR-003)
- [X] T005 [US1] Verwijder in sectie 7 (`signalering:`) van `esphome/pompput.yaml` de scripts `signalering_bijwerken`, `zoemer_herhalen`, `zoemer_eenmalig`, `zoemer_patroon_kiezen`, `zoemer_stilzetten` en `led_bevestiging`, met hun commentaarblokken (FR-001, FR-005)
- [X] T006 [US1] Verwijder in sectie 7 van `esphome/pompput.yaml` het `output:`-blok met `zoemer_output` (`platform: ledc`), het `binary_sensor:`-blok met `boot_knop` en het `button:`-blok met `zoemer_stil`, met hun commentaar (FR-001, FR-004, FR-005, FR-007)
- [X] T007 [US1] Verwijder in sectie 7, `globals:` van `esphome/pompput.yaml`, de globals `g_led_bevestiging_bezig`, `g_zoemer_modus`, `g_zoemer_herstart`, `g_eenmalig_bezig`, `g_stil`, `g_stil_set` en `g_vorige_set` met hun commentaar. `g_led_patroon`, `g_led_helderheid`, `g_opstarten` en `g_verbonden` blijven (data-model §2)
- [X] T008 [US1] Haal in script `led_patroon_kiezen` (sectie 7) van `esphome/pompput.yaml` de `if`-voorwaarde `return !id(g_led_bevestiging_bezig);` weg, zodat de lambda altijd draait. Laat de keuze, de kleuren, `led_min_helderheid_alarm` en de aanroepen van `led_knipperen` letterlijk gelijk (FR-002). Zet de FR-regel boven het script op `# FR-027; 002 FR-002, FR-003; contracts/lokale-signalering.md "Statusled"`
- [X] T009 [US1] Verwijder in sectie 6 (`instellingen:`) van `esphome/pompput.yaml` het `switch:`-blok met `zoemer` (`restore_mode: RESTORE_DEFAULT_ON`) en het commentaar "Uit = de zoemer klinkt nooit…" (FR-004). Raak de id's van de overige bewaarde `number`s en `globals` niet aan (research R3; FR-009)
- [X] T010 [US1] Verwijder in sectie 1 (`substitutions:`) van `esphome/pompput.yaml`: `pin_zoemer`, `pin_knop`, `knop_debounce`, `led_bevestiging_aantal`, `led_bevestiging_aan_ms`, `led_bevestiging_uit_ms`, het commentaar "Zoemer: passieve piëzo…", alle sleutels die met `zoemer_` beginnen (frequentie, niveau, de patronen alarm/waarschuwing/storing/hersteld/bevestiging/opstarten, `zoemer_pauze_ms`) en het blok "Zoemermodus" (`zoemer_modus_stil`, `zoemer_modus_alarm`, `zoemer_modus_waarschuwing`). Zet `project_version` op `"0.3.0"` (research R6)
- [X] T011 [US1] Werk `specs/001-pompput-monitor/contracts/hardware.md` bij, tegelijk met T010 (constitutie, Embedded Platform Constraints: het `substitutions`-blok MOET met dit contract overeenkomen): de inleiding noemt `pin_zoemer` en `pin_knop` niet meer en meldt "D5 is vrij sinds 002-zoemer-verwijderen"; in de pintabel wordt de rij D5 `| D5 | 7 | — | vrij (was piëzo-zoemer, vervallen met 002) | — |` en de rij D9 `| D9 | 9 | — | BOOT-knop op het board, geen functie in de firmware | alleen voor de downloadmodus bij het opstarten |`; verwijder de paragraaf "Piëzo-zoemer (passief, PWM)"; in "Veilige begintoestand" vervalt de regel over de zoemer en komt "GPIO7 wordt niet geconfigureerd; een nog aangesloten passieve zoemer blijft stil (002 research R2)"; de alinea over GPIO9 zegt dat de firmware de pin niet meer leest
- [X] T012 [US1] Werk de kopcommentaar van `esphome/pompput.yaml` bij: "Ter plaatse signaleert een RGBW-statusled; Home Assistant krijgt alles via de versleutelde native API", sectie 7 in de inhoudsopgave wordt "signalering: led, patroonscripts", en de regel "Spec, plan, contracten en validatie" noemt ook `specs/002-zoemer-verwijderen/`. Pas de FR-regel boven het pakket `signalering:` aan naar `# FR-027, FR-028 (led-helderheid); 002 FR-001–FR-007; contracts/lokale-signalering.md, 001 research R9`
- [X] T013 [US1] Controleer dat er geen resten zijn ([quickstart.md](quickstart.md) §2 stap 3): `Select-String -Path esphome/pompput.yaml -Pattern 'zoemer|ledc|pin_knop|boot_knop|g_stil|bevestiging|signalering_bijwerken|g_vorige_set|g_eenmalig' -CaseSensitive:$false | Where-Object { $_.Line -notmatch '002-zoemer-verwijderen' }` geeft geen uitvoer. De verwijzing naar `specs/002-zoemer-verwijderen/` uit T012 telt niet mee. Verwijder wat er nog staat
- [X] T014 [US1] Draai de compile-gate: `.venv\Scripts\esphome config esphome/pompput.yaml` en `.venv\Scripts\esphome compile esphome/pompput.yaml`. Los fouten op tot er geen fouten en geen waarschuwingen uit eigen lambdas zijn. Noteer RAM en flash in `specs/002-zoemer-verwijderen/verificatie.md` onder "Compile- en resource-gate" en vergelijk met de uitgangssituatie (verwacht: RAM ≤ 75 %, flash ≤ 90 %, beide gelijk of lager; research R7)
- [ ] T015 [US1] Werk het apparaat bij via WiFi (`.venv\Scripts\esphome run esphome/pompput.yaml --device pompput.local`) en voer [quickstart.md](quickstart.md) §3 stap 1–3 en §4 #1–#9 uit. Noteer per scenario de uitkomst in `specs/002-zoemer-verwijderen/verificatie.md`; is het apparaat niet beschikbaar, noteer "niet uitgevoerd" met de reden. **Faalt #9** (de oude zoemer maakt toch geluid), voeg dan in `esphome/pompput.yaml` een `output` op GPIO7 toe die vanaf het opstarten laag staat (het verworpen alternatief uit research R2, met een substitution `pin_d5_vrij` en een commentaar met de reden), werk `hardware.md` mee bij, herhaal T014 en #9, en noteer dit onder "Afwijkingen"

**Checkpoint**: firmware 0.3.0 maakt geen geluid, de led werkt als voorheen, de instellingen
zijn behouden en het pinnencontract klopt. US1 is als MVP bruikbaar.

---

## Phase 4: User Story 2 - Geen zoemerbediening meer in Home Assistant en op het apparaat (Priority: P2)

**Goal**: geen zoemer-entiteiten, dashboardregels of meldingsteksten meer in Home Assistant;
de BOOT-knop doet niets; de led-helderheid blijft instelbaar (FR-004, FR-005, FR-006, FR-010).

**Independent Test**: [quickstart.md](quickstart.md) §3 stap 4–5 en §4 #10–#11. De
bestandswijzigingen (T016, T017) kunnen los van US1 gebeuren; de controle op het apparaat
(T018) vraagt firmware 0.3.0 uit US1.

- [X] T016 [P] [US2] Verwijder in `homeassistant/dashboard-pompput.yaml`, kaart "Bediening", de twee regels `- entity: button.pompput_zoemer_stil` (name: Zoemer stil) en `- entity: switch.pompput_zoemer` (name: Zoemer aan). `input_boolean.pompput_toon_instellingen` blijft (FR-010)
- [X] T017 [P] [US2] Vervang in `homeassistant/packages/pompput.yaml`, alert "Pompput onbereikbaar", de zin "Bij de put signaleren de led en de zoemer nog wel." door "Bij de put signaleert de led nog wel." Pas in de kopcommentaar de contractverwijzing aan zodat die ook `specs/002-zoemer-verwijderen/contracts/wijzigingen-signalering-en-ha.md` noemt (FR-010)
- [ ] T018 [US2] Laad het pakket en het dashboard opnieuw in Home Assistant en voer [quickstart.md](quickstart.md) §3 stap 4–5 en §4 #10–#11 uit (na T015). Verwijder een eventueel achtergebleven `switch.pompput_zoemer` of `button.pompput_zoemer_stil` onder Instellingen → Entiteiten (research R4). Noteer de uitkomsten in `specs/002-zoemer-verwijderen/verificatie.md`

**Checkpoint**: Home Assistant toont 35 entiteiten van het apparaat, zonder zoemer en zonder
onbeschikbare entiteiten (SC-004).

---

## Phase 5: User Story 3 - Documentatie en aansluitschema zonder zoemer (Priority: P3)

**Goal**: bouwbeschrijving, aansluitschema, contracten en testprocedure noemen de zoemer niet
meer als onderdeel; D5 staat als vrij (FR-011, SC-006). Historische documenten van 001
(`research.md`, `tasks.md`, `verificatie.md`) blijven ongewijzigd (research R5). Het
pinnencontract `hardware.md` is al in US1 bijgewerkt (T011).

**Independent Test**: [quickstart.md](quickstart.md) §5.

T019–T025 raken elk een ander bestand en kunnen parallel.

- [X] T019 [P] [US3] Werk `README.md` bij: (1) inleiding: "Een RGBW-led signaleert ter plaatse, ook zonder Home Assistant."; voeg bij "Ontwerp en eisen" `specs/002-zoemer-verwijderen/` toe; (2) tabel "Platform", rij Componenten: haal `ledc` weg; (3) de pinnenlijst `pin_…` zonder `pin_zoemer` en `pin_knop`; (4) tabel "Overzicht": verwijder de rijen "Piëzo-zoemer" (+ en −) en "Knop \"zoemer stil\""; de regel "Vrij:" krijgt D5 (GPIO7) erbij; (5) "Pinnen van de XIAO": D5 wordt `(vrij)` en D9 wordt `BOOT-knop (op het board, geen functie)`; (6) "Per onderdeel": verwijder de alinea's "Piëzo-zoemer (passief)" (met schema en transistor-tip) en "Knop \"zoemer stil\""; (7) "Signalering ter plaatse": vervang de alinea over de BOOT-knop door "De BOOT-knop heeft geen functie in de firmware. Houd hem **niet** ingedrukt tijdens het opstarten, want dan gaat de chip in de downloadmodus." en laat de verwijzing naar `contracts/lokale-signalering.md` staan. Controleer met `Select-String -Path README.md -Pattern 'zoemer|piëzo|piep' | Where-Object { $_.Line -notmatch '002-zoemer-verwijderen' }` dat er geen treffers over blijven
- [X] T020 [P] [US3] Werk `specs/001-pompput-monitor/contracts/lokale-signalering.md` bij: titel "Contract: Signalering ter plaatse (statusled)"; de inleiding noemt alleen `led_patroon_kiezen` en `led_knipperen` en vermeldt dat de zoemer met `specs/002-zoemer-verwijderen` is vervallen; verwijder de hoofdstukken "Zoemer (passieve piëzo…)" en "Stilzetten (FR-028)". De tabel "Statusled (RGBW)" en de twee punten eronder blijven letterlijk gelijk (002 FR-002)
- [X] T021 [P] [US3] Werk `specs/001-pompput-monitor/contracts/esphome-homeassistant.md` bij: verwijder in de tabel "Bediening" de rijen `switch` `zoemer` en `button` `zoemer_stil`, en voeg onder de tabel toe "De schakelaar Zoemer en de knop Zoemer stil zijn vervallen met `specs/002-zoemer-verwijderen`."
- [X] T022 [P] [US3] Werk `specs/001-pompput-monitor/data-model.md` bij: verwijder in §1 de rij `zoemer_aan` en in de tabel met vaste waarden de rij "Zoemertoon"; verwijder in §7 de rijen `zoemer_patroon`, `stil_voor` en `vorige_set`; zet in de rij `led_patroon` dat het patroon nooit meer wordt onderbroken; voeg bovenaan §7 een regel toe met een verwijzing naar `specs/002-zoemer-verwijderen/data-model.md`
- [X] T023 [P] [US3] Werk `specs/001-pompput-monitor/quickstart.md` bij. Beschrijf stilte steeds als "het apparaat blijft stil" en gebruik het woord "piep" niet, zodat de controle in [quickstart.md](quickstart.md) §5 geen treffers geeft: §1 "Benodigdheden": haal "piëzo" uit de onderdelenlijst; §2 stap 5 "Verwacht": alleen een witte led en het apparaat blijft stil, en zolang het apparaat niet is ingesteld knippert de led paars; §3 stap 4: het apparaat blijft stil bij herstel; §4 #4: zonder "de zoemer 3× per 10 s"; #6: zonder "korte piep"; #10: zonder "zoemer 1× per 60 s"; #13: zonder "2 piepjes"; verwijder #14 "Zoemer stil" en #15 "Zoemer uit" en vervang ze door één rij `| 14 | (vervallen) | — | De zoemer en het stilzetten zijn vervallen met specs/002-zoemer-verwijderen | — |` zodat de nummering van de overige scenario's gelijk blijft; §7: FR-028 verwijst alleen nog naar #23
- [X] T024 [P] [US3] Werk `specs/001-pompput-monitor/spec.md` bij met de stand na 002: (1) voeg onder "Clarifications" een sessie "2026-11-11 (002-zoemer-verwijderen)" toe met "De zoemer vervalt; ter plaatse signaleert alleen de statusled."; (2) laat de twee bestaande clarifications over de zoemer (sessie 2026-11-09 over led en zoemer, sessie 2026-11-11 over het herinneringsinterval en de zoemer) als geschiedenis staan en zet er "(zoemer vervallen met 002)" achter; (3) pas aan, telkens met "(gewijzigd door 002)" erachter: US2 scenario 1 (zonder "en de zoemer"), het randgeval "Home Assistant zelf is niet beschikbaar" (alleen de statusled), FR-020 (laatste zin: "De signalering ter plaatse (led) valt hier niet onder; die volgt FR-027."), FR-027 (de zin over de piëzo-zoemer vervalt), FR-028 (alleen nog: "De gebruiker MOET de helderheid van de led vanuit Home Assistant kunnen instellen."), SC-009 (alleen de statusled) en de Assumption "Home Assistant is het enige kanaal…" (alleen de led)
- [X] T025 [P] [US3] Voeg in `specs/001-pompput-monitor/plan.md` direct onder de regel **Input** één regel toe: "**Bijgewerkt**: de piëzo-zoemer, het stilzetten en de BOOT-knopfunctie zijn vervallen met [specs/002-zoemer-verwijderen](../002-zoemer-verwijderen/plan.md). De rest van dit plan beschrijft de toestand van versie 0.2.0." Wijzig verder niets in dit bestand
- [X] T026 [US3] Voer [quickstart.md](quickstart.md) §5 uit (na T011 en T019–T025) en noteer de uitkomst in `specs/002-zoemer-verwijderen/verificatie.md` onder "Documentatie". Alleen treffers die naar het vervallen van de zoemer verwijzen zijn toegestaan

**Checkpoint**: alle levende documentatie beschrijft het apparaat zonder zoemer.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: de laatste gates en het afronden.

- [X] T027 Laatste compile- en resource-gate na alle wijzigingen: herhaal T014 en zet de uitkomst in de samenvattingstabel van `specs/002-zoemer-verwijderen/verificatie.md`
- [X] T028 Review- en traceerbaarheidsgate in `specs/002-zoemer-verwijderen/verificatie.md`: loop de tabel "Constitution Check" van [plan.md](plan.md) na tegen `esphome/pompput.yaml` (enige uitgangen TRIG en statusled; pinnen in `substitutions` gelijk aan `specs/001-pompput-monitor/contracts/hardware.md`: GPIO3, 4, 5, 6, plus GPIO7 als de terugvaloptie uit T015 is toegepast) en controleer dat elke requirement uit de tabel "Traceerbaarheid" van [plan.md](plan.md) een element heeft en een scenario in [quickstart.md](quickstart.md) §6. Noteer afwijkingen onder "Afwijkingen"
- [X] T029 Verwijder het Sync Impact Report (het HTML-commentaar bovenaan) uit `.specify/memory/constitution.md` vóór de commit, zoals `/speckit-constitution` voorschrijft; de rest van het bestand blijft gelijk
- [X] T030 Zet in `specs/002-zoemer-verwijderen/spec.md` de **Status** van `Draft` op `Implemented` zodra T015, T018 en T026 zijn uitgevoerd, of op `Implemented (hardware-scenario's open)` als scenario's op het apparaat nog niet konden draaien

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: geen afhankelijkheden. T002 moet vóór T015 (bijwerken) klaar zijn, anders zijn de oude instellingen niet meer te vergelijken.
- **Foundational (Phase 2)**: leeg.
- **US1 (Phase 3)**: na Phase 1. Strikt na elkaar: T003 → T010, dan T011 (contract, samen met T010), dan T012 → T013 → T014 → T015.
- **US2 (Phase 4)**: T016 en T017 kunnen direct na Phase 1, parallel met US1. T018 vraagt T015.
- **US3 (Phase 5)**: T019–T025 kunnen direct na Phase 1, parallel met US1 en US2. T026 vraagt T011 en T019–T025.
- **Polish (Phase 6)**: na alle stories. T027 en T028 vragen T014; T030 komt als laatste.

### User Story Dependencies

- **US1 (P1)**: onafhankelijk; levert de firmware en het bijgewerkte pinnencontract.
- **US2 (P2)**: bestanden onafhankelijk; de controle op het apparaat (T018) hangt af van US1 (T015), omdat de entiteiten pas verdwijnen met firmware 0.3.0.
- **US3 (P3)**: onafhankelijk van US1 en US2, behalve dat de eindcontrole T026 ook `hardware.md` (T011) meeneemt.

### Parallel Opportunities

- T016 en T017 (US2) parallel met elkaar en met US1.
- T019–T025 (US3) allemaal parallel: elk een ander bestand.
- US1 zelf heeft geen parallelle taken: alles volgt elkaar op in `esphome/pompput.yaml`, met `hardware.md` direct na T010.

---

## Parallel Example: User Story 3

```text
Task: "T019 README.md bijwerken"
Task: "T020 specs/001-pompput-monitor/contracts/lokale-signalering.md bijwerken"
Task: "T021 specs/001-pompput-monitor/contracts/esphome-homeassistant.md bijwerken"
Task: "T022 specs/001-pompput-monitor/data-model.md bijwerken"
Task: "T023 specs/001-pompput-monitor/quickstart.md bijwerken"
Task: "T024 specs/001-pompput-monitor/spec.md bijwerken"
Task: "T025 specs/001-pompput-monitor/plan.md verwijzing toevoegen"
```

## Parallel Example: User Story 2 (naast US1)

```text
Task: "T016 homeassistant/dashboard-pompput.yaml: zoemerregels weg"
Task: "T017 homeassistant/packages/pompput.yaml: tekst Pompput onbereikbaar"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Phase 1: T001–T002 (uitgangssituatie vastleggen).
2. Phase 3: T003–T014, firmware zonder zoemer, pinnencontract bijgewerkt, compile- en resource-gate.
3. **STOP en VALIDEER**: T015 op het apparaat (geen geluid, led gelijk, instellingen behouden).
4. Het apparaat is dan al bruikbaar zonder zoemer en voldoet aan de constitutie; HA toont hooguit twee verdwenen entiteiten.

### Incremental Delivery

1. Setup → US1 (firmware en pinnencontract) → valideren.
2. US2 (HA-pakket en dashboard) → T018 valideren.
3. US3 (overige documentatie) → T026 valideren.
4. Polish: laatste gates, Sync Impact Report weg, status bijwerken.

---

## Notes

- [P] = ander bestand, geen afhankelijkheid van een open taak.
- Raak in `esphome/pompput.yaml` alleen aan wat in T003–T010 en T012 staat (en bij een falend scenario #9 de uitzondering uit T015); de secties 3, 4, 5 (behalve `status_bijwerken`) en 8 blijven ongewijzigd (FR-008).
- Behoud de id's van alle bewaarde waarden (research R3); hernoem niets.
- Scenario's die niet op het apparaat konden draaien, worden met de reden in `verificatie.md` gezet (constitutie IV, gate 4).
- Commit per fase of per logische groep.
