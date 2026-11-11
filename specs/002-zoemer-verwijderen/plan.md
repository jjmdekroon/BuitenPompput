# Implementation Plan: Zoemer verwijderen

**Branch**: `002-zoemer-verwijderen` (spec-map `specs/002-zoemer-verwijderen`; er is geen
git-branch aangemaakt, het werk staat op `main`) | **Date**: 2026-11-11 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/002-zoemer-verwijderen/spec.md`

## Summary

De piëzo-zoemer is niet meer nodig. Het apparaat signaleert ter plaatse alleen nog met de
RGBW-statusled; meldingen op afstand blijven via Home Assistant lopen.

**Technische aanpak**:
- **Firmware** (`esphome/pompput.yaml`): alles wat alleen de zoemer, het stilzetten of de
  BOOT-knop dient, gaat eruit: de `ledc`-output, de schakelaar `zoemer`, de button
  `zoemer_stil`, de binary sensor `boot_knop`, zes scripts, zeven globals, de opstartpiep en de
  bijbehorende substitutions. `status_bijwerken` roept `led_patroon_kiezen` voortaan direct aan
  (research R1). De led-logica blijft verder letterlijk gelijk.
- **Pinnen**: GPIO7 (D5) en GPIO9 (D9) worden niet meer geconfigureerd. Een nog aangesloten
  passieve zoemer blijft daardoor stil (R2).
- **Instellingen**: geen migratie; de bewaarde waarden zijn per id opgeslagen en blijven
  behouden (R3).
- **Home Assistant**: twee entiteiten verdwijnen; het dashboard en één meldingstekst worden
  bijgewerkt (R4).
- **Documentatie**: de levende documenten van 001 en de README worden bijgewerkt; de
  historische documenten van 001 blijven staan (R5).
- **Versie**: `project_version` 0.2.0 → 0.3.0 (R6).
- Onderbouwing: [research.md](research.md).

## Technical Context

**Language/Version**: ESPHome-YAML met korte C++-lambdas, ongewijzigd ten opzichte van 001.

**Primary Dependencies**: ESPHome 2026.10.1 (vastgepind in `requirements.txt`, ongewijzigd).
Na de wijziging gebruikte componenten: `ultrasonic`, `adc`, `ct_clamp`,
`esp32_rmt_led_strip`, `template` (sensor, binary_sensor, text_sensor, number, button, event),
`globals`, `script`, `interval`, `debug`, `wifi_signal`, `uptime`, `api`, `ota`. **`ledc` en
`gpio` (binary sensor) vervallen.** Geen externe componenten.

**Storage**: alleen instellingen in flash. De schakelaar `zoemer` vervalt; de overige bewaarde
waarden blijven ongewijzigd (R3, [data-model.md](data-model.md) §1).

**Testing**: compile- en resource-gate met de ESPHome-CLI, en de scenario's op het apparaat in
[quickstart.md](quickstart.md). Geen unittests (constitutie IV).

**Target Platform**: Seeed Studio XIAO ESP32C3, `esp-idf`; Home Assistant met de
ESPHome-integratie. Ongewijzigd.

**Project Type**: ESPHome-apparaatconfiguratie plus een Home Assistant-configuratiepakket.

**Performance Goals**: led binnen 1 s na een verandering van toestand (eis ≤ 5 s, 002 FR-003).
De overige doelen van 001 blijven gelijk.

**Constraints**: die van 001 en constitutie 2.0.1 (niet-blokkerend, RAM ≤ 75 %, flash ≤ 90 %,
watchdog aan, geen uitgang naar de pomp). Bijwerken via OTA zonder verlies van instellingen
(FR-009).

**Scale/Scope**:
- 1 apparaat. HA-entiteiten: 37 → **35** (14 numbers, 0 switches, 1 button).
- Bestanden: `esphome/pompput.yaml`, `homeassistant/packages/pompput.yaml`,
  `homeassistant/dashboard-pompput.yaml`, `README.md`, en vijf levende documenten van 001
  plus één regel in `specs/001-pompput-monitor/plan.md`.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.* Getoetst aan
constitutie **2.0.1**.

| Principe | Toets | Vóór fase 0 | Na fase 1 |
|---|---|---|---|
| **I. ESPHome is het doel** | Geen wijziging in versie, board, framework, API of OTA. Er komen geen componenten bij. | ✅ PASS | ✅ PASS |
| **II. Niet-blokkerende, begrensde automatiseringen** | Er gaat alleen logica af. De led-toestandsmachine blijft in `globals` met benoemde scripts. RAM en flash kunnen alleen dalen; de resource-gate meet opnieuw (R7). | ✅ PASS | ✅ PASS |
| **III. Veilig, alleen bewaken** | De resterende uitgangen zijn TRIG en de statusled, beide in een veilige begintoestand. GPIO7 wordt niet meer aangestuurd, dus geen geluid vanaf het vroegste moment (R2). Geen uitgang naar de pomp. Watchdog blijft aan. | ✅ PASS | ✅ PASS |
| **IV. Declaratieve, traceerbare logica** | Geen dode scripts of globals (R1). FR-commentaren in sectie 7 worden bijgewerkt (R6). Elke requirement van 002 heeft een scenario ([quickstart.md](quickstart.md) §6). | ✅ PASS | ✅ PASS |
| **V. Expliciet, eenvoudig, waarneembaar** | Ongebruikte substitutions, componenten en entiteiten worden verwijderd (YAGNI). Het `substitutions`-blok en [hardware.md](../001-pompput-monitor/contracts/hardware.md) lopen gelijk. | ✅ PASS | ✅ PASS |
| **Embedded Platform Constraints** | Pinbron en YAML blijven gelijk. Lokale signalering (de statusled) werkt zonder WiFi of HA (quickstart §4 #7). `reboot_timeout` ongewijzigd. | ✅ PASS | ✅ PASS |
| **Development Workflow & Quality Gates** | Gates 1–2: quickstart §2. Gate 3: tabel hieronder en quickstart §6. Gate 4: quickstart §3–§4. Gate 5: deze tabel; geen afwijkingen. Uitkomsten in `verificatie.md` van 002. | ✅ PASS | ✅ PASS |

## Project Structure

### Documentation (this feature)

```text
specs/002-zoemer-verwijderen/
├── plan.md                                   # Dit bestand
├── research.md                               # Fase 0: beslissingen R1–R7
├── data-model.md                             # Fase 1: wat er ten opzichte van 001 verandert
├── quickstart.md                             # Fase 1: gates, bijwerken, scenario's, traceerbaarheid
├── contracts/
│   └── wijzigingen-signalering-en-ha.md      # Led, pinnen D5/D9, HA-entiteiten, pakket, dashboard
├── checklists/requirements.md
├── verificatie.md                            # Bij de uitvoering: uitkomsten van de gates
└── tasks.md                                  # Fase 2 (/speckit-tasks)
```

### Source Code (repository root)

```text
esphome/
└── pompput.yaml                    # Sectie 1 (substitutions), 2 (on_boot), 5 (status_bijwerken),
                                    # 6 (switch zoemer) en 7 (signalering) worden aangepast
homeassistant/
├── packages/pompput.yaml           # Tekst van "Pompput onbereikbaar"
└── dashboard-pompput.yaml          # Kaart "Bediening"
README.md                           # Inleiding, componenten, pinnen, schema, signalering
specs/001-pompput-monitor/
├── spec.md                         # FR-020/027/028, SC-009, US2 scen. 1, randgevallen, Assumptions
├── data-model.md                   # §1 en §7
├── quickstart.md                   # §1 onderdelen, §2 stap 5, §3, §4 #4/#6/#10/#13/#14/#15, §7
├── plan.md                         # Alleen een verwijzing naar 002 bovenaan
└── contracts/
    ├── hardware.md                 # D5 vrij, D9 zonder functie
    ├── lokale-signalering.md       # Alleen nog de statusled
    └── esphome-homeassistant.md    # Bediening: switch en button vervallen
```

**Structure Decision**: geen nieuwe bestanden of mappen buiten `specs/002-zoemer-verwijderen/`.
De opbouw van `esphome/pompput.yaml` in genummerde secties blijft; sectie 7 heet voortaan
"signalering: led, patroonscripts".

## Traceerbaarheid

Constitutie, gate 3. Requirement → wijziging. De scenario's staan in
[quickstart.md §6](quickstart.md#6-traceerbaarheid-requirement--scenario).

| Requirement | Wijziging (in `esphome/pompput.yaml`, tenzij anders vermeld) |
|---|---|
| FR-001 | geen `ledc`-output, geen zoemerscripts, geen opstartpiep in `on_boot` |
| FR-002 | `led_patroon_kiezen` en `led_knipperen` ongewijzigd, behalve de vervallen bevestigingsvoorwaarde |
| FR-003 | `status_bijwerken` roept `led_patroon_kiezen` direct aan (elke 1 s) |
| FR-004 | `switch` `zoemer` en `button` `zoemer_stil` verwijderd |
| FR-005 | `binary_sensor` `boot_knop`, `zoemer_stilzetten` en `led_bevestiging` verwijderd |
| FR-006 | `number` `led_helderheid_pct` ongewijzigd |
| FR-007 | `pin_zoemer` en de `output` op GPIO7 verwijderd; pin niet geconfigureerd |
| FR-008 | de secties 3–6 en 8 blijven ongewijzigd, behalve de aanroep in `status_bijwerken` en de verwijderde `switch` |
| FR-009 | geen wijziging in id's van bewaarde waarden (R3) |
| FR-010 | `homeassistant/packages/pompput.yaml`, `homeassistant/dashboard-pompput.yaml` |
| FR-011 | `README.md` en de levende documenten van 001 (R5) |

## Complexity Tracking

Geen schendingen van de constitutie; niets te rechtvaardigen.
