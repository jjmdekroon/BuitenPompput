---

description: "Takenlijst voor de niveau-ledbalk met zes leds"
---

# Tasks: Niveau-ledbalk met zes leds

**Input**: Design documents from `specs/003-niveau-ledbalk/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md),
[data-model.md](data-model.md), [contracts/ledbalk.md](contracts/ledbalk.md),
[quickstart.md](quickstart.md)

**Tests**: geen geautomatiseerde tests (constitutie IV). Controle gebeurt met de compile- en
resource-gate en de scenario's op het apparaat in [quickstart.md](quickstart.md).

**Organization**: taken per user story uit [spec.md](spec.md).
- **Foundational**: de strip met zes leds en de zes partities. Alle stories hangen daarvan af,
  en het pinnencontract `hardware.md` verandert daarmee tegelijk (constitutie: het
  `substitutions`-blok en `hardware.md` lopen gelijk).
- **US1**: de niveaubalk op led 2–6.
- **US2**: het statuspatroon op led 1.
- **US3**: de balk uit bij een ongeldige meting.
- **US4**: de overige documentatie.

Bijna alle firmwaretaken raken `esphome/pompput.yaml` en lopen dus na elkaar, ook tussen
stories.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: kan parallel (ander bestand, geen afhankelijkheid van een open taak)
- **[Story]**: US1 t/m US4 uit [spec.md](spec.md)

## Path Conventions

- Firmware: `esphome/pompput.yaml` (genummerde secties 1–8; sectie 1 = `substitutions:`,
  sectie 5 = pakket `meldingen:`, sectie 7 = pakket `signalering:`)
- Documentatie: `README.md`, `specs/001-pompput-monitor/`, `specs/003-niveau-ledbalk/`
- Commando's draaien in PowerShell vanuit de root van de repository met `.venv\Scripts\esphome`
- Requirement-commentaar in de YAML voor deze feature krijgt het voorvoegsel `003`, bijvoorbeeld
  `# 003 FR-004`, om het te onderscheiden van de FR-nummers van 001 (zoals 002 dat deed)

---

## Phase 1: Setup

**Purpose**: de uitgangssituatie vastleggen en de hardware klaarzetten.

- [X] T001 Maak `specs/003-niveau-ledbalk/verificatie.md` met de koppen: "Samenvatting van de gates" (tabel gate 1–5 met kolommen Gate, Uitkomst, Datum), "Uitgangssituatie", "Hardware", "Compile- en resource-gate", "Bijwerken" (quickstart §3), "Bank-controles" (B1–B4 uit quickstart §4), "Scenario's op het apparaat" (tabel #1–#20 uit quickstart §4 met kolommen #, Uitgevoerd ja/nee, Uitkomst, Reden als niet uitgevoerd), "Documentatie" (quickstart §5), "Review- en traceerbaarheidsgate" en "Afwijkingen"
- [X] T002 Noteer onder "Uitgangssituatie" in `specs/003-niveau-ledbalk/verificatie.md`: de RAM- en flashwaarden van firmware 0.3.0 uit `specs/002-zoemer-verwijderen/verificatie.md` (RAM 34,7 %, 111 432 bytes; flash 47,3 %, 867 140 bytes), en de actuele waarden van de 14 `number`-instellingen in Home Assistant. Kan het apparaat niet worden uitgelezen, schrijf dat op met de reden (constitutie IV)
- [X] T003 Leg onder "Hardware" in `specs/003-niveau-ledbalk/verificatie.md` vast hoe de zes leds zijn aangesloten, en controleer dat het klopt met [contracts/ledbalk.md §4](contracts/ledbalk.md#4-aansluiting-vervangt-de-led-in-001-hardwaremd): diode **1N4001** (geen 1N4148; vervangen als die er nog zit), elko 100 µF tussen VDD en GND bij led 1, DIN van led 1 via 330 Ω aan D4, DOUT → DIN door de keten, led 1 het dichtst bij het apparaat. Dit is handwerk door de beheerder; kan het niet worden gecontroleerd, noteer dat met de reden

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: zes leds aanstuurbaar maken, met led 1 onder de oude naam `statusled`, zodat de
bestaande scripts blijven werken. Na deze fase compileert de firmware en doet led 1 precies wat
de enkele led deed; led 2–6 blijven uit.

**⚠️ CRITICAL**: geen user story kan beginnen voordat deze fase klaar is.

- [X] T004 Werk in sectie 1 (`substitutions:`, blok "Signalering") van `esphome/pompput.yaml` bij: `led_aantal: "6"` (was "1") en voeg direct daaronder toe `led_index_1: "0"` t/m `led_index_6: "5"`, met de commentaarregel "# Positie in de keten; led 1 zit het dichtst bij het apparaat (003 FR-001, contracts/ledbalk.md §4)". Zet in het blok "Apparaat" `project_version: "0.4.0"`. Werk ook het commentaar bij `pin_led` bij naar "# D4; DIN van led 1 van 6 (SK6812-keten) via 330 ohm"
- [X] T005 Vervang in sectie 7 (`signalering:` → `light:`) van `esphome/pompput.yaml` het ene licht door zeven lichten (research R1): (1) het bestaande `esp32_rmt_led_strip`-licht krijgt `id: ledstrip`, `name: "Ledstrip"`, behoudt `internal: true`, `pin`, `num_leds: ${led_aantal}`, `chipset`, `channel_colors`, `use_psram: false`, `restore_mode: ALWAYS_OFF`, `gamma_correct: ${led_gamma}`, `default_transition_length: 0s`, met het commentaar "# De driver van de keten. Geen script stuurt hem direct; alleen de partities hieronder (003 research R1)"; (2) een `platform: partition`-licht `id: statusled`, `name: "Statusled"`, `internal: true`, `restore_mode: ALWAYS_OFF`, `gamma_correct: ${led_gamma}`, `default_transition_length: 0s`, `segments: [{id: ledstrip, from: ${led_index_1}, to: ${led_index_1}}]`, commentaar "# Led 1: statuslampje (003 FR-001, FR-002, FR-003)"; (3) vijf gelijke `partition`-lichten `niveauled_2` … `niveauled_6` (`name: "Niveauled 2"` enz.) met `from`/`to` = `${led_index_2}` … `${led_index_6}`, commentaar "# Led 2–6: niveaubalk (003 FR-001, FR-004)". De scripts `led_knipperen` en `led_patroon_kiezen` blijven `statusled` aansturen en hoeven hier niet te veranderen
- [X] T006 Werk in sectie 2 (`esphome:` → `on_boot`) van `esphome/pompput.yaml` alleen het kopcommentaar bij: "De leds staan al uit (restore_mode ALWAYS_OFF op de strip en de zes partities) en TRIG is laag" en voeg `003 FR-012` toe aan de requirement-regel. De stappen zelf blijven gelijk
- [X] T007 Werk `specs/001-pompput-monitor/contracts/hardware.md` bij, tegelijk met T004 (constitutie, Embedded Platform Constraints): (1) de pintabel: D4 = "DIN van led 1 van een keten van 6 SK6812-RGBW-leds"; (2) vervang de sectie "Adresseerbare RGBW-led" door "Ledketen (6 × SK6812 RGBW)" met het schema en de punten uit [contracts/ledbalk.md §4](contracts/ledbalk.md#4-aansluiting-vervangt-de-led-in-001-hardwaremd) (1N4001, elko 100 µF, 100 nF per led, volgorde); (3) "Voeding": leds samen hoogstens ≈ 146 mA (led 1 paars en vijf blauwe balkleds op 100 %), totaal ≈ 520 mA piek, adapter ≥ 1 A blijft; (4) "Veilige begintoestand": strip en zes partities `restore_mode: ALWAYS_OFF`; (5) een regel bovenaan "Gewijzigd door [003](../../003-niveau-ledbalk/spec.md): zes leds in plaats van één"
- [X] T008 Draai de compile-gate: `.venv\Scripts\esphome config esphome/pompput.yaml` en `.venv\Scripts\esphome compile esphome/pompput.yaml`. Los fouten op tot er geen fouten en geen waarschuwingen uit eigen lambdas zijn. Noteer de uitkomst onder "Compile- en resource-gate" in `specs/003-niveau-ledbalk/verificatie.md`. Weigert ESPHome het `partition`-platform op deze strip, stop dan en werk eerst [research.md](research.md) R1 bij naar het `addressable_lambda`-alternatief

**Checkpoint**: firmware 0.4.0 compileert; led 1 werkt als de oude statusled; led 2–6 zijn uit.

---

## Phase 3: User Story 1 - Waterniveau aflezen op de ledbalk (Priority: P1) 🎯 MVP

**Goal**: led 2–6 tonen het niveau als blauwe balk in vijf gelijke stappen tot de
alarmdrempel, met een herstelmarge van 2 procentpunt; led 6 brandt vanaf de alarmdrempel
(FR-004, FR-005, FR-006, FR-008, FR-009, FR-010, FR-011).

**Independent Test**: [quickstart.md](quickstart.md) §4 B1, B4 en #2, #3, #4, #15, #16, #17
(balkdeel) en #18.

- [X] T009 [US1] Voeg in sectie 1 (`substitutions:`, blok "Signalering") van `esphome/pompput.yaml` toe: `niveaubalk_stappen: "5"` met commentaar "# Led 2–6: vijf gelijke stappen tot de alarmdrempel (003 FR-004)" en `niveaubalk_herstelmarge_pct: "2.0"` met commentaar "# Een balkled gaat pas uit 2 procentpunt onder zijn grens (003 FR-008)"
- [X] T010 [US1] Voeg in sectie 1, blok "Gedeelde uitdrukkingen" van `esphome/pompput.yaml` de uitdrukking `led_helderheid_nu` toe met commentaar "# De ingestelde led-helderheid als 0–1; de standaardwaarde zolang de instelling nog onbekend is (003 FR-011)": `"((std::isnan(id(led_helderheid_pct).state) ? ${led_helderheid_pct_std} : id(led_helderheid_pct).state) / ${pct_schaal})"`. Vervang in script `led_patroon_kiezen` (sectie 7) de drie regels `float pct = …; if (std::isnan(pct)) …; float helderheid = pct / ${pct_schaal};` door `float helderheid = ${led_helderheid_nu};`, zodat er één definitie is (constitutie IV)
- [X] T011 [US1] Voeg in sectie 7, `globals:` van `esphome/pompput.yaml` toe (data-model §2): `g_niveaubalk_aantal` (`type: uint8_t`, `restore_value: false`, `initial_value: '0'`, commentaar "# Aantal brandende balkleds, 0–5") en `g_niveaubalk_helderheid` (`type: float`, `restore_value: false`, `initial_value: 'NAN'`, commentaar "# Helderheid waarmee de balk het laatst is gezet; NAN dwingt de eerste keer af")
- [X] T012 [US1] Voeg in sectie 7, `script:` van `esphome/pompput.yaml`, na `led_patroon_kiezen`, het script `niveaubalk_bijwerken` toe met commentaar "# 003 FR-004, FR-005, FR-006, FR-008, FR-010, FR-011; data-model §3, contracts/ledbalk.md §2" en **twee korte stappen** (constitutie IV: één berekening, één uitvoering; plan Constitution Check IV). **Stap 1, een `if` met lambda-voorwaarde (de berekening)**, volgens [data-model.md §3](data-model.md#3-niveaubalk-levelbar-regels): `v = id(g_vulgraad_pct)`, `D = id(g_alarmdrempel_pct)`, `m = ${niveaubalk_herstelmarge_pct}`; tel in een lus over *k* = 1 … `${niveaubalk_stappen}` met grens = *k* × D / `${niveaubalk_stappen}`: `n_aan` (v ≥ grens) en `n_houd` (v ≥ grens − m); `n = max(n_aan, min(g_niveaubalk_aantal, n_houd))`; `h = ${led_helderheid_nu}`. Als `n == g_niveaubalk_aantal && h == g_niveaubalk_helderheid` → `return false` (niets te doen). Anders beide globals bijwerken, bij een verandering van *n* één `ESP_LOGD("${log_tag}", "niveaubalk %u", n)` loggen en `return true`. **Stap 2, de `then` van die `if` (de uitvoering)**: één lambda die over een lokale, vaste lijst `light::LightState *const leds[] = {id(niveauled_2), …, id(niveauled_6)}` loopt met `n = id(g_niveaubalk_aantal)` en `h = id(g_niveaubalk_helderheid)`: index < n én h > 0 → `turn_on()` met `set_rgbw(${led_blauw_r}, ${led_blauw_g}, ${led_blauw_b}, ${led_blauw_w})`, `set_brightness(h)`, `set_transition_length(0)`, `perform()`; anders `turn_off()` met `set_transition_length(0)`, `perform()`. Geen heap, geen `std::string`, geen wachten (constitutie II)
- [X] T013 [US1] Voeg in sectie 5, script `status_bijwerken` van `esphome/pompput.yaml`, na `- script.execute: led_patroon_kiezen` toe: `- script.execute: niveaubalk_bijwerken`, en werk het commentaar erboven bij naar "# Led 1 en de niveaubalk volgen de nieuwe toestand direct (002 FR-003; 003 FR-009)"
- [X] T014 [US1] Draai de compile- en resource-gate ([quickstart.md](quickstart.md) §2 stap 1–2). Los fouten en waarschuwingen uit eigen lambdas op. Noteer RAM en flash onder "Compile- en resource-gate" in `specs/003-niveau-ledbalk/verificatie.md` (verwacht: RAM ≤ 75 %, flash ≤ 90 %, research R7)
- [ ] T015 [US1] Werk het apparaat bij via WiFi (`.venv\Scripts\esphome run esphome/pompput.yaml --device pompput.local`, na T002) en voer [quickstart.md](quickstart.md) §3 stap 1–4, §4 B1, B4 en #2, #3, #4, #15, #16, #17 (balkdeel) en #18 uit. Noteer elke uitkomst in `specs/003-niveau-ledbalk/verificatie.md`; scenario's die niet konden draaien met de reden. Faalt B1, stop dan en werk eerst [research.md](research.md) R1 bij

**Checkpoint**: de balk toont het niveau; led 1 werkt nog met het oude groen/blauw. Let op: lever
US1 niet zonder US3 af als eindtoestand, want pas US3 maakt de regel voor een ongeldig niveau
expliciet (al geeft een NAN-vulgraad in T012 vanzelf *n* = 0).

---

## Phase 4: User Story 2 - Toestand van het apparaat op led 1 (Priority: P1)

**Goal**: led 1 is normaal continu blauw, knippert blauw 1 s / 1 s als de pomp draait, en toont
anders het bestaande statuspatroon (FR-002, FR-003, FR-013, FR-014).

**Independent Test**: [quickstart.md](quickstart.md) §2 stap 3, §4 B2, B3 en #1, #5, #7–#10,
#17.

- [X] T016 [US2] Werk in sectie 1 (`substitutions:`, blok "Signalering") van `esphome/pompput.yaml` bij (research R3): `led_pomp_aan_ms: "1000"` (nieuw) en `led_pomp_uit_ms: "1000"` (was "0"), met commentaar "# Pomp draait: blauw 1 s aan / 1 s uit (003 FR-014)"; `led_normaal_uit_ms: "0"` (was "4900") met commentaar "# Normaal: continu blauw (003 FR-002, FR-013)". Verwijder `led_normaal_aan_ms` en de vier regels `led_groen_r`, `led_groen_g`, `led_groen_b`, `led_groen_w`
- [X] T017 [US2] Werk in sectie 7, script `led_patroon_kiezen` van `esphome/pompput.yaml` de `switch` bij: in `case ${led_patroon_pomp}` de tijden `0, ${led_pomp_uit_ms}` → `${led_pomp_aan_ms}, ${led_pomp_uit_ms}`; in `default` (normaal) de kleur `led_groen_*` → `led_blauw_*` en de tijden → `0, ${led_normaal_uit_ms}`. Laat de volgorde van de `if`-keten, de patroonnummers en de minimale alarmhelderheid ongewijzigd (003 FR-003); T021 voegt later de regel voor "niveau onbekend" aan de keten toe. Werk het commentaar boven het script bij naar "# FR-027; 002 FR-002, FR-003; 003 FR-002, FR-003, FR-013, FR-014; contracts/ledbalk.md §1"
- [X] T018 [US2] Controleer dat er geen resten zijn ([quickstart.md](quickstart.md) §2 stap 3): `Select-String -Path esphome/pompput.yaml -Pattern 'led_groen|led_normaal_aan_ms'` geeft geen uitvoer. Draai daarna de compile-gate (§2 stap 1) en noteer beide in `specs/003-niveau-ledbalk/verificatie.md`
- [ ] T019 [US2] Werk het apparaat bij via WiFi en voer [quickstart.md](quickstart.md) §4 B2, B3 en #1, #5, #7, #8, #9, #10 en #17 uit. Noteer de uitkomsten in `specs/003-niveau-ledbalk/verificatie.md`

**Checkpoint**: led 1 en de balk werken samen zoals in [contracts/ledbalk.md §3](contracts/ledbalk.md#3-samen-gezien-voorbeelden-alarmdrempel-80-).

---

## Phase 5: User Story 3 - Geen misleidende niveauweergave bij een ongeldige meting (Priority: P2)

**Goal**: bij opstarten, een verouderde meting, een sensorstoring, "niet ingesteld" of een
meting die uitvalt tijdens een alarm is de balk uit, en led 1 is nooit continu blauw: na de
opstartfase toont hij het storingspatroon ("niveau onbekend"), ook vóór de storingsmelding
(FR-007, FR-012, FR-017).

**Independent Test**: [quickstart.md](quickstart.md) §4 #6, #11, #12, #13 en #14.

- [X] T020 [US3] Voeg in sectie 7, script `niveaubalk_bijwerken` van `esphome/pompput.yaml`, bovenaan de lambda van stap 1 (de berekening) een expliciete bewaking toe: `const bool bruikbaar = ${niveau_bruikbaar} && !std::isnan(id(g_vulgraad_pct));` en bij `!bruikbaar` geldt `n = 0` (de hysterese vervalt, het vorige aantal wordt dus ook 0), met commentaar "# Geen geldig niveau: balk uit, ook tijdens een vastgehouden alarm (003 FR-007; constitutie III)". Voeg `FR-007` toe aan het requirement-commentaar van het script
- [X] T021 [US3] Voeg in sectie 7, script `led_patroon_kiezen` van `esphome/pompput.yaml`, in de `if`-keten direct na `else if (id(g_opstarten)) patroon = ${led_patroon_opstarten};` de regel `else if (!${niveau_bruikbaar}) patroon = ${led_patroon_storing};` toe, met erachter het commentaar "// niveau onbekend: paars, ook vóór de storingsmelding (003 FR-017)". Gebruik hetzelfde patroonnummer als storing (geen nieuw nummer, geen nieuwe kleur), zodat de overgang naar de echte storingsmelding het knipperen niet herstart (research R3). Voeg `FR-017` toe aan het requirement-commentaar boven het script, en werk in sectie 1 het commentaar boven de `led_patroon_*`-codes bij: "Ledpatroon, in volgorde van prioriteit (contracts/ledbalk.md §1; 'niveau onbekend' gebruikt led_patroon_storing)"
- [ ] T022 [US3] Draai de compile-gate ([quickstart.md](quickstart.md) §2 stap 1), werk het apparaat bij via WiFi en voer [quickstart.md](quickstart.md) §4 #6, #11, #12, #13 en #14 uit (#14 alleen bij een apparaat zonder bewaarde put-maten; anders overslaan met de reden). Noteer de uitkomsten in `specs/003-niveau-ledbalk/verificatie.md`

**Checkpoint**: alle firmware is af; de balk liegt nooit over het niveau (SC-005).

---

## Phase 6: User Story 4 - Bouwbeschrijving en testprocedure voor zes leds (Priority: P3)

**Goal**: README, signaleringscontract, datamodel, testprocedure en spec van 001 beschrijven zes
leds (FR-015). `hardware.md` is al bijgewerkt in T007. Historische documenten van 001 en 002
(`research.md`, `tasks.md`, `verificatie.md`) blijven ongewijzigd.

**Independent Test**: [quickstart.md](quickstart.md) §5.

T023–T028 raken elk een ander bestand en kunnen parallel, ook naast US1–US3.

- [X] T023 [P] [US4] Vervang de inhoud van `specs/001-pompput-monitor/contracts/lokale-signalering.md` door die van [contracts/ledbalk.md](contracts/ledbalk.md) §1, §2, §3 en §5: titel "Contract: Signalering ter plaatse (ledbalk)", een inleiding die zegt dat de enkele statusled sinds [003](../../003-niveau-ledbalk/spec.md) een keten van zes leds is, met `led_patroon_kiezen` (led 1) en `niveaubalk_bijwerken` (led 2–6) als uitvoering. Voor de aansluiting verwijst het naar `hardware.md`
- [X] T024 [P] [US4] Werk `README.md` bij: (1) inleiding: "Zes RGBW-leds signaleren ter plaatse: led 1 de toestand, led 2–6 het waterniveau"; voeg bij "Ontwerp en eisen" `specs/003-niveau-ledbalk/` toe; (2) tabel Componenten: `partition` toevoegen; (3) aansluittabel en schema: zes SK6812-leds in een keten, diode **1N4001** (niet meer "1N4001 of 1N4148"), elko 100 µF bij led 1, led 1 het dichtst bij het apparaat; (4) voeding: het verbruik uit research R4; (5) vervang de tabel "Signalering ter plaatse" door een tabel voor led 1 (zoals [contracts/ledbalk.md §1](contracts/ledbalk.md#1-led-1-statuslampje), met "blauw, continu = normaal" en "blauw, 1 s aan / 1 s uit = pomp draait"; "groen" komt niet meer voor) en een korte uitleg van de balk met de grenzen bij 80 % (16/32/48/64/80 %)
- [X] T025 [P] [US4] Werk `specs/001-pompput-monitor/data-model.md` §7 bij: voeg de rijen `niveaubalk_aantal` en `niveaubalk_helderheid` toe zoals in [data-model.md §2](data-model.md#2-signaleringstoestand-indicationstate-001-7), zet bij `led_patroon` "op led 1 (partitie `statusled`)", en voeg een regel toe "Niveaubalk en fysieke leds: zie [003 data-model](../003-niveau-ledbalk/data-model.md) §3–§4". Werk in §1 de vaste waarden bij als daar de led-tijden of `led_aantal` staan
- [X] T026 [P] [US4] Werk `specs/001-pompput-monitor/quickstart.md` bij: §1 "RGBW-led" → "zes RGBW-leds (SK6812) en een diode 1N4001"; §2 stap 5: "led 1 wit, de balk uit"; §3: "binnen 1 minuut is de status `normaal` en brandt led 1 continu blauw"; §4 bank-controle "Kleurvolgorde": "bij normaal is led 1 blauw en niet groen of wit"; #6 "groene led" → "led 1 continu blauw"; #8 "de led blauw tijdens het pompen" → "led 1 knippert blauw 1 s / 1 s tijdens het pompen"; #4, #10, #16, #23: "de led" → "led 1"; #27 (geometrie wijzigen): voeg aan de verwachting toe "led 1 knippert kort paars en de balk is uit zolang het niveau onbekend is; daarna weer blauw (003 FR-017)". Verwijs bovenaan §4 naar [003 quickstart](../003-niveau-ledbalk/quickstart.md) §4 voor de scenario's van de balk
- [X] T027 [P] [US4] Werk `specs/001-pompput-monitor/spec.md` bij: voeg aan FR-027, FR-028 en SC-009 "(gewijzigd door 003)" toe en zet in FR-027 "statuslampje (RGBW-led)" → "statuslampje (led 1 van de ledbalk)"; voeg onder "Clarifications" een sessie "2026-11-11 (003-niveau-ledbalk)" toe met "De enkele led is vervangen door zes leds: led 1 toont de toestand, led 2–6 het niveau. Zie [specs/003-niveau-ledbalk](../003-niveau-ledbalk/spec.md)."
- [X] T028 [P] [US4] Voeg in `specs/001-pompput-monitor/plan.md`, direct onder de bestaande regel **Bijgewerkt** (van 002), één regel toe: "**Bijgewerkt**: de enkele statusled is vervangen door een ledbalk van zes leds; zie [specs/003-niveau-ledbalk](../003-niveau-ledbalk/plan.md)."
- [X] T029 [US4] Voer [quickstart.md](quickstart.md) §5 uit (na T007 en T023–T028) en noteer de uitkomst in `specs/003-niveau-ledbalk/verificatie.md` onder "Documentatie". Alleen treffers die zeggen dat iets vervallen of gewijzigd is, zijn toegestaan

**Checkpoint**: alle levende documentatie beschrijft de ledbalk met zes leds (FR-015).

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: de laatste gates en het afronden.

- [X] T030 Werk de kopcommentaar van `esphome/pompput.yaml` bij: "Ter plaatse signaleren zes RGBW-leds: led 1 de toestand, led 2–6 het waterniveau"; sectie 7 in de inhoudsopgave → "signalering: ledbalk, patroonscripts"; voeg `specs/003-niveau-ledbalk/` toe aan de regel "Spec, plan, contracten en validatie". Pas ook de sectiekop van sectie 7 aan
- [X] T031 Laatste compile- en resource-gate na alle wijzigingen: herhaal [quickstart.md](quickstart.md) §2 stap 1–3 en zet de uitkomst (RAM, flash, waarschuwingen) in de samenvattingstabel van `specs/003-niveau-ledbalk/verificatie.md`
- [ ] T032 Voer [quickstart.md](quickstart.md) §4 #19 en #20 uit (overige bewaking en stroomverbruik) en noteer ze in `specs/003-niveau-ledbalk/verificatie.md`; niet uitgevoerd → met de reden
- [X] T033 Review- en traceerbaarheidsgate in `specs/003-niveau-ledbalk/verificatie.md`: (1) controleer met `Select-String -Path esphome/pompput.yaml -Pattern '003 FR-0'` dat elke requirement FR-001 t/m FR-014, FR-016 en FR-017 bij het element uit de tabel "Traceerbaarheid" in [plan.md](plan.md) staat (FR-015 is documentatie); (2) loop de tabel "Constitution Check" van [plan.md](plan.md) na tegen de YAML: alle zeven lichten `restore_mode: ALWAYS_OFF`, geen `delay` of heap in `niveaubalk_bijwerken`, geen losse getallen (alleen 0, 1, 2 en substitutions), geen nieuwe externe componenten; (3) controleer dat de `substitutions` (pin, `led_aantal`, `led_index_*`) overeenkomen met `specs/001-pompput-monitor/contracts/hardware.md`. Noteer afwijkingen onder "Afwijkingen"
- [X] T034 Zet in `specs/003-niveau-ledbalk/spec.md` de **Status** van `Draft` op `Implemented` zodra T015, T019, T022 en T029 zijn uitgevoerd, of op `Implemented (hardware-scenario's open)` als scenario's op het apparaat nog niet konden draaien

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: geen afhankelijkheden. T002 moet klaar zijn vóór de eerste update van het apparaat (T015); T003 vóór elke controle op het apparaat.
- **Foundational (Phase 2)**: na Phase 1. T004 → T005 → T006 na elkaar (zelfde bestand); T007 tegelijk met T004 (ander bestand, maar inhoudelijk gekoppeld); T008 na T004–T006. Blokkeert alle stories.
- **US1 (Phase 3)**: na Phase 2. T009 → T010 → T011 → T012 → T013 → T014 → T015.
- **US2 (Phase 4)**: na Phase 2. Bestandswijzigingen T016–T017 na T013 (zelfde bestand, geen inhoudelijke afhankelijkheid van US1). T019 vraagt een geflashte firmware.
- **US3 (Phase 5)**: T020 na T012 (het script moet bestaan); T021 na T017 als US2 al gedaan is (zelfde script `led_patroon_kiezen`; inhoudelijk onafhankelijk). T022 vraagt een geflashte firmware.
- **US4 (Phase 6)**: T023–T028 direct na Phase 1, parallel met alles. T029 na T007 en T023–T028.
- **Polish (Phase 7)**: na alle stories. T030 vóór T031; T034 als laatste.

### User Story Dependencies

- **US1 (P1)**: hangt af van Foundational.
- **US2 (P1)**: hangt af van Foundational; inhoudelijk onafhankelijk van US1 (eigen script, eigen substitutions). Alleen omdat het hetzelfde bestand is, komt het na US1.
- **US3 (P2)**: breidt het script van US1 uit, dus na T012.
- **US4 (P3)**: onafhankelijk; alleen de eindcontrole T029 vraagt T007.

### Parallel Opportunities

- T007 (`hardware.md`) parallel met T004–T006 (`pompput.yaml`).
- T023–T028 (US4) allemaal parallel: elk een ander bestand, en parallel met de firmwaretaken.
- Binnen US1, US2 en US3 geen parallelle taken: alles zit in `esphome/pompput.yaml`.
- De scenario's op het apparaat (T015, T019, T022) kunnen in één sessie worden uitgevoerd als US1–US3 samen zijn geflasht.

---

## Parallel Example: User Story 4 (naast de firmware)

```text
Task: "T023 specs/001-pompput-monitor/contracts/lokale-signalering.md vervangen door ledbalk-contract"
Task: "T024 README.md: zes leds, 1N4001, ledtabel"
Task: "T025 specs/001-pompput-monitor/data-model.md §7 bijwerken"
Task: "T026 specs/001-pompput-monitor/quickstart.md bijwerken"
Task: "T027 specs/001-pompput-monitor/spec.md: (gewijzigd door 003)"
Task: "T028 specs/001-pompput-monitor/plan.md: verwijzing toevoegen"
```

## Parallel Example: Foundational

```text
Task: "T004–T006 esphome/pompput.yaml: led_aantal 6, led_index_*, strip + 6 partities"
Task: "T007 specs/001-pompput-monitor/contracts/hardware.md: keten van 6 leds, 1N4001"
```

---

## Implementation Strategy

### MVP First

1. Phase 1: T001–T003 (uitgangssituatie, hardware).
2. Phase 2: T004–T008 (zes leds aanstuurbaar, compileert).
3. Phase 3: T009–T015 (niveaubalk) **en** Phase 5: T020–T022 (balk uit en led 1 paars bij een ongeldige meting).
   Samen het kleinste geheel dat veilig is (constitutie III): de balk toont het niveau en liegt
   nooit.
4. **STOP en VALIDEER** op het apparaat.

### Incremental Delivery

1. Setup + Foundational → led 1 werkt als vroeger; balk uit.
2. US1 + US3 → niveaubalk, veilig bij een ongeldige meting → valideren.
3. US2 → led 1 blauw / pomp knipperend → valideren.
4. US4 → documentatie → T029.
5. Polish: kopcommentaar, laatste gates, status.

---

## Notes

- [P] = ander bestand, geen afhankelijkheid van een open taak.
- Raak in `esphome/pompput.yaml` alleen aan wat in T004–T006, T009–T013, T016–T017, T020–T021 en T030 staat. De secties 3, 4, 6 en 8 blijven ongewijzigd, en in sectie 5 alleen `status_bijwerken` (003 FR-016).
- Hernoem geen id's van bewaarde waarden; `statusled` blijft als id bestaan (nu de partitie van led 1).
- Scenario's die niet op het apparaat konden draaien, worden met de reden in `verificatie.md` gezet (constitutie IV, gate 4).
- Commit per fase of per logische groep.
