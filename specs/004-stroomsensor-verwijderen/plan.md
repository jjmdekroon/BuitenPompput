# Implementation Plan: Stroomsensor verwijderen

**Branch**: `004-stroomsensor-verwijderen` (afgesplitst van `003-niveau-ledbalk`, dat nog niet in
`main` zit) | **Date**: 2026-11-18 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/004-stroomsensor-verwijderen/spec.md`

## Summary

De stroomsensor (ACS712) wordt niet gebruikt. Daarmee vervalt alle pompbewaking: de pompstatus,
de looptijd, de pompstarts, de drie pompwaarschuwingen, de storing van de stroomsensor, de
patronen "waarschuwing" en "pomp draait" op led 1, en de zeven instellingen die alleen daarvoor
bestonden. De niveaubewaking (niveau, hoogwater-alarm, niveausensor storing, niet ingesteld,
ledbalk) blijft ongewijzigd.

**Technische aanpak**:
- **Firmware** (`esphome/pompput.yaml`): sectie 4 (pakket `pomp`) vervalt in zijn geheel,
  inclusief de componenten `adc` en `ct_clamp`. Uit de meldingen gaan vier binary sensors en
  vier beoordelingsscripts; de overige drie meldingen worden 0–2 genummerd (research R1, R2).
  Uit de instellingen gaan zeven numbers en `g_inschakelniveau_cm`; "ingesteld" vraagt alleen
  nog Afstand leeg en Afstand vol (R3). `led_patroon_kiezen` verliest twee patronen (R6).
  De event-types `pomp_start` en `pomp_stop` vervallen (R8). De secties worden opnieuw
  genummerd 1–7 (R11).
- **Pin**: GPIO3 (A1) wordt niet meer geconfigureerd. Een nog aangesloten ACS712 mag blijven
  zitten (R5).
- **Instellingen**: geen migratie; de bewaarde waarden zijn per id opgeslagen (R4).
- **Home Assistant**: 35 → 20 entiteiten; het pakket verliest de `history_stats`-sensor en vier
  alerts, de overige alerts verliezen de regel "Pomp:"; het dashboard verliest alle
  pomponderdelen (R7).
- **Documentatie**: de levende documenten van 001 en 003 en de README worden bijgewerkt, met
  een paragraaf over het veilig weghalen van de sensor (R5, R9).
- **Constitutie**: PATCH 2.0.1 → 2.0.2 via `/speckit-constitution`, alleen het weghalen van de
  verouderde voorbeelden (R10).
- **Versie**: `project_version` 0.4.0 → 0.5.0 (R11).
- Onderbouwing: [research.md](research.md).

## Technical Context

**Language/Version**: ESPHome-YAML met korte C++-lambdas, ongewijzigd ten opzichte van 003.

**Primary Dependencies**: ESPHome 2026.10.1 (vastgepind in `requirements.txt`, ongewijzigd).
Na de wijziging gebruikte componenten: `ultrasonic`, `esp32_rmt_led_strip`, `partition`,
`template` (sensor, binary_sensor, text_sensor, number, button, event), `globals`, `script`,
`interval`, `debug`, `wifi_signal`, `uptime`, `api`, `ota`. **`adc` en `ct_clamp` vervallen.**
Geen externe componenten.

**Storage**: alleen instellingen in flash. Zeven blijven, zeven vervallen; de sleutels van de
overgebleven waarden veranderen niet (R4, [data-model.md](data-model.md) §1).

**Testing**: compile- en resource-gate met de ESPHome-CLI, en de scenario's op het apparaat in
[quickstart.md](quickstart.md). Geen unittests (constitutie IV).

**Target Platform**: Seeed Studio XIAO ESP32C3, `esp-idf`; Home Assistant met de
ESPHome-integratie. Ongewijzigd.

**Project Type**: ESPHome-apparaatconfiguratie plus een Home Assistant-configuratiepakket.

**Performance Goals**: ongewijzigd ten opzichte van 003: led 1 en de niveaubalk binnen 5 s na
een verandering (gemeten binnen 1 s), instellingen binnen 1 minuut van kracht.

**Constraints**: die van 001–003 en constitutie 2.0.1 (niet-blokkerend, RAM ≤ 75 %, flash
≤ 90 %, watchdog aan, geen uitgang naar de pomp). Bijwerken via OTA zonder verlies van de
overgebleven instellingen (FR-011).

**Scale/Scope**:
- 1 apparaat. HA-entiteiten: 35 → **20** (7 numbers, 3 meldingen, 1 button).
- Bestanden: `esphome/pompput.yaml`, `homeassistant/packages/pompput.yaml`,
  `homeassistant/dashboard-pompput.yaml`, `README.md`, `.specify/memory/constitution.md`, zes
  levende documenten van 001, vier van 003, en één regel bovenaan `plan.md` van 001 en 003.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.* Getoetst aan
constitutie **2.0.1**.

| Principe | Toets | Vóór fase 0 | Na fase 1 |
|---|---|---|---|
| **I. ESPHome is het doel** | Geen wijziging in versie, board, framework, API of OTA. Er komen geen componenten bij; `adc` en `ct_clamp` gaan eraf. | ✅ PASS | ✅ PASS |
| **II. Niet-blokkerende, begrensde automatiseringen** | Er gaat alleen logica af. De toestandsmachines voor niveau, meldingen en led blijven in `globals` met benoemde scripts; de pomp-toestandsmachine vervalt in zijn geheel, er blijft geen halve over. RAM en flash kunnen alleen dalen; de resource-gate meet opnieuw (R12). | ✅ PASS | ✅ PASS |
| **III. Veilig, alleen bewaken** | Het apparaat had al geen uitgang naar de pomp; nu ook geen meting meer aan de pompkring. GPIO3 wordt niet geconfigureerd en stuurt dus nooit tegen een nog aangesloten sensor in (R5). Het hoogwater-alarm blijft vastgehouden bij een uitgevallen meting. Watchdog blijft aan. Het weghalen van de sensor wordt gedocumenteerd zodat de pomp op zijn vlotter blijft werken (FR-015). | ✅ PASS | ✅ PASS |
| **IV. Declaratieve, traceerbare logica** | Geen dode scripts, globals, patronen of substitutions (R1, R6). FR-verwijzingen naar vervallen requirements gaan met de code mee (R11). Elke requirement van 004 heeft een scenario ([quickstart.md](quickstart.md) §6). | ✅ PASS | ✅ PASS |
| **V. Expliciet, eenvoudig, waarneembaar** | Ongebruikte substitutions, componenten en entiteiten worden verwijderd (YAGNI). Het `substitutions`-blok en [hardware.md](../001-pompput-monitor/contracts/hardware.md) lopen gelijk (`pin_acs712` vervalt, A1 vrij). Elke overgebleven storing is waarneembaar in HA en het log. | ✅ PASS | ✅ PASS |
| **Embedded Platform Constraints** | Pinbron en YAML blijven gelijk. Lokale signalering werkt zonder WiFi of HA (quickstart §4 #5). `reboot_timeout` ongewijzigd. "Mains": het weghalen van de ACS712 gebeurt door of onder toezicht van een vakbekwaam persoon (contracts §4). | ✅ PASS | ✅ PASS |
| **Development Workflow & Quality Gates** | Gates 1–2: quickstart §2. Gate 3: tabel hieronder en quickstart §6. Gate 4: quickstart §3–§4. Gate 5: deze tabel; geen afwijkingen. Uitkomsten in `verificatie.md` van 004. | ✅ PASS | ✅ PASS |

**Opmerking**: de constitutie noemt de ACS712 en de pompstatus alleen als voorbeeld. De
verwijdering is daarmee niet in strijd, maar de tekst raakt verouderd. De bijwerking naar 2.0.2
(R10) is een PATCH en een taak van de uitvoering; ze verandert niets aan de toetsing hierboven.

## Project Structure

### Documentation (this feature)

```text
specs/004-stroomsensor-verwijderen/
├── plan.md                                   # Dit bestand
├── research.md                               # Fase 0: beslissingen R1–R12
├── data-model.md                             # Fase 1: wat er ten opzichte van 001 en 003 verandert
├── quickstart.md                             # Fase 1: gates, bijwerken, scenario's, traceerbaarheid
├── contracts/
│   └── wijzigingen-firmware-en-ha.md         # HA-entiteiten, pakket, dashboard, pin A1, led 1
├── checklists/requirements.md
├── verificatie.md                            # Bij de uitvoering: uitkomsten van de gates
└── tasks.md                                  # Fase 2 (/speckit-tasks)
```

### Source Code (repository root)

```text
esphome/
└── pompput.yaml                    # Kop; sectie 1 (substitutions), 2 (on_boot), 4 (pomp, vervalt),
                                    # 5 (meldingen), 6 (instellingen), 7 (signalering), 8 (events);
                                    # daarna hernummerd 1-7
homeassistant/
├── packages/pompput.yaml           # history_stats en vier alerts weg; "Pomp:" uit vier alerts
└── dashboard-pompput.yaml          # Hoofdkaart, grafiek, instellingen, diagnose
README.md                           # Inleiding, onderdelen, pinnen, schema, voeding, signalering,
                                    # Home Assistant, nieuw: stroomsensor weghalen
.specify/memory/constitution.md     # 2.0.2 (R10), via /speckit-constitution
specs/001-pompput-monitor/
├── spec.md                         # US3, US4, FR-006/008/009/010/011/013-016/019/023/026,
│                                   # SC-002/003/006, randgevallen, Assumptions, Clarifications
├── data-model.md                   # §1, §4, §5, §6, §7
├── quickstart.md                   # §1 onderdelen, §2–§4 pompstappen, §5 drempel, §6, §7
├── plan.md                         # Alleen een verwijzing naar 004 bovenaan
└── contracts/
    ├── hardware.md                 # A1 vrij, ACS712-paragraaf → "stroomsensor weghalen"
    ├── esphome-homeassistant.md    # Entiteiten, pakket, events
    └── lokale-signalering.md       # Tabel led 1 en voorbeelden, gelijk aan ledbalk.md
specs/003-niveau-ledbalk/
├── spec.md                         # FR-003, FR-014, US2 scen. 3/7/8, SC-004, randgeval "pomp draait"
├── data-model.md                   # Statuspatroon
├── quickstart.md                   # Scenario's met waarschuwing of pomp draait
├── plan.md                         # Alleen een verwijzing naar 004 bovenaan
└── contracts/ledbalk.md            # §1 tabel, voorbeelden met "pomp draait"
```

**Structure Decision**: geen nieuwe bestanden of mappen buiten
`specs/004-stroomsensor-verwijderen/`. De opbouw van `esphome/pompput.yaml` in genummerde
secties blijft, met zeven in plaats van acht secties (R11).

## Traceerbaarheid

Constitutie, gate 3. Requirement → wijziging. De scenario's staan in
[quickstart.md §6](quickstart.md#6-traceerbaarheid-requirement--scenario).

| Requirement | Wijziging (in `esphome/pompput.yaml`, tenzij anders vermeld) |
|---|---|
| FR-001 | pakket `pomp` verwijderd (`adc`, `ct_clamp`, `pomp_bijwerken`, `pomp`, `looptijd_laatste_pompbeurt`); `g_pomp_status` en `pomp_draait_nu`/`pomp_staat_stil` verwijderd |
| FR-002 | binary sensors en beoordelingsscripts van de drie pompwaarschuwingen en `stroomsensor_storing` verwijderd; `meldingen_beoordelen` roept drie scripts aan |
| FR-003 | entiteiten uit FR-001 en FR-002, plus `pompstroom` en `offset_stroomsensor`, verwijderd |
| FR-004 | zeven numbers en `g_inschakelniveau_cm` verwijderd; `ingesteld_bepalen` en `instelling_toepassen` zonder inschakelniveau |
| FR-005 | `led_patroon_kiezen` zonder de takken "waarschuwing" en "pomp draait"; substitutions `led_oranje_*`, `led_waarschuwing_*`, `led_pomp_*` verwijderd |
| FR-006 | secties niveau, meldingen (hoogwater, niveausensor, niet ingesteld), signalering en gebeurtenissen verder ongewijzigd |
| FR-007 | `led_patroon_kiezen`: overgebleven takken in dezelfde volgorde, kleuren en ritmes |
| FR-008 | numbers `afstand_leeg_cm`, `afstand_vol_cm`, `alarmdrempel_pct`, `herstelmarge_pct`, `hoogwater_wachttijd_s`, `sensorstoring_wachttijd_s`, `led_helderheid_pct` ongewijzigd |
| FR-009 | `melding_stap` zonder pompdeel in de tekst; `homeassistant/packages/pompput.yaml` zonder "Pomp:" |
| FR-010 | event-types `pomp_start`/`pomp_stop` verwijderd; `history_stats` uit het pakket verwijderd |
| FR-011 | geen wijziging in id's van bewaarde waarden (R4) |
| FR-012 | `pin_acs712` en de `adc` op GPIO3 verwijderd; pin niet geconfigureerd |
| FR-013 | `homeassistant/packages/pompput.yaml`, `homeassistant/dashboard-pompput.yaml` |
| FR-014 | `README.md` en de levende documenten van 001 en 003 (R9) |
| FR-015 | `specs/001-pompput-monitor/contracts/hardware.md` en `README.md`, paragraaf "Een bestaande stroomsensor weghalen" |

## Complexity Tracking

Geen schendingen van de constitutie; niets te rechtvaardigen.
