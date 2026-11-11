# Implementation Plan: Niveau-ledbalk met zes leds

**Branch**: `003-niveau-ledbalk` | **Date**: 2026-11-11 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/003-niveau-ledbalk/spec.md`

## Summary

De enkele RGBW-statusled is vervangen door een keten van zes RGBW-leds. Led 1 blijft het
statuslampje: normaal continu blauw, en anders het bestaande statuspatroon (alarm, waarschuwing,
storing, opstarten, geen verbinding), met "pomp draait" voortaan als langzaam blauw knipperen.
Led 2 t/m 6 vormen een blauwe niveaubalk in vijf gelijke stappen tot de alarmdrempel; led 6
brandt zodra de vulgraad de alarmdrempel bereikt.

**Technische aanpak**:
- **Driver**: de bestaande `esp32_rmt_led_strip` krijgt `num_leds: 6` en heet `ledstrip`.
  Daarboven komen zes `partition`-lichten van één led: `statusled` (led 1) en
  `niveauled_2` … `niveauled_6` (research R1).
- **Led 1**: `led_patroon_kiezen` en `led_knipperen` blijven; alleen de patronen "normaal"
  (continu blauw) en "pomp draait" (blauw 1 s / 1 s) veranderen, en er komt één regel bij:
  na het opstarten zonder bruikbaar niveau toont led 1 het storingspatroon ("niveau
  onbekend"). De groene kleur vervalt (R3).
- **Niveaubalk**: een nieuw script `niveaubalk_bijwerken` berekent elke seconde het aantal
  brandende leds uit de vulgraad, de alarmdrempel en een vaste herstelmarge van 2 procentpunt,
  en zet alleen de leds die veranderen. Geen geldig niveau → balk uit (R2).
- **Hardware**: diode 1N4001 (niet meer 1N4148) en een elko van 100 µF bij led 1; de
  5 V-adapter van 1 A blijft voldoende (R4).
- **Home Assistant**: geen wijziging (R5).
- **Versie**: `project_version` 0.3.0 → 0.4.0; levende documenten van 001 en de README worden
  bijgewerkt (R6).
- Onderbouwing: [research.md](research.md).

## Technical Context

**Language/Version**: ESPHome-YAML met korte C++-lambdas, ongewijzigd ten opzichte van 001/002.

**Primary Dependencies**: ESPHome 2026.10.1 (vastgepind in `requirements.txt`, ongewijzigd).
Componenten: die van 002, plus het light-platform **`partition`** (standaard in ESPHome, geen
externe component).

**Storage**: geen nieuwe bewaarde waarden; twee nieuwe niet-bewaarde globals
(`g_niveaubalk_aantal`, `g_niveaubalk_helderheid`) ([data-model.md](data-model.md) §2).

**Testing**: compile- en resource-gate met de ESPHome-CLI, en de scenario's op het apparaat in
[quickstart.md](quickstart.md). Geen unittests (constitutie IV).

**Target Platform**: Seeed Studio XIAO ESP32C3, `esp-idf`; Home Assistant met de
ESPHome-integratie. Ongewijzigd.

**Project Type**: ESPHome-apparaatconfiguratie plus een Home Assistant-configuratiepakket
(het pakket verandert niet).

**Performance Goals**: led 1 binnen 1 s na een verandering van toestand; de niveaubalk binnen
ongeveer 2 s na een grensovergang (eis ≤ 5 s, FR-009); een gewijzigde alarmdrempel binnen 1 s
op de balk (eis ≤ 1 min, FR-010).

**Constraints**: constitutie 2.0.1 (niet-blokkerend, lambda < 10 ms, geen heap in herhaalde
paden, RAM ≤ 75 %, flash ≤ 90 %, watchdog aan, veilige begintoestand van elke uitgang). Bijwerken
via OTA zonder verlies van instellingen (FR-016). Ledstroom ≤ ≈ 150 mA bij 100 % helderheid
(R4).

**Scale/Scope**:
- 1 apparaat, 6 leds op één datapin (GPIO6). HA-entiteiten: ongewijzigd (35).
- Bestanden: `esphome/pompput.yaml` (secties 1, 5 en 7), `README.md`, en de levende documenten
  van 001 (`spec.md`, `data-model.md`, `quickstart.md`, `contracts/hardware.md`,
  `contracts/lokale-signalering.md`).

Er bleven na het invullen geen "NEEDS CLARIFICATION" over; de drie technische vragen (aansturing
per led, voeding, gamma bij partities) zijn beslist in research R1 en R4, met een controle op de
bank voor wat niet uit documentatie vast te stellen is.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.* Getoetst aan
constitutie **2.0.1**.

| Principe | Toets | Vóór fase 0 | Na fase 1 |
|---|---|---|---|
| **I. ESPHome is het doel** | Versie, board, framework, API en OTA ongewijzigd. `partition` is een standaardplatform van de vastgepinde ESPHome-versie; geen externe component (R1). | ✅ PASS | ✅ PASS |
| **II. Niet-blokkerende, begrensde automatiseringen** | Knipperen blijft via `delay` in `led_knipperen`. `niveaubalk_bijwerken` is één korte lambda zonder wachten, die alleen bij een verandering vijf lichten zet; geen heap (vaste lijst van pointers). De balk is een afgeleide waarde met één stap geheugen, geen nieuwe toestandsmachine; led 1 houdt zijn bestaande toestandsmachine (`g_led_patroon`). Resource-gate meet opnieuw (R7). | ✅ PASS | ✅ PASS |
| **III. Veilig, alleen bewaken** | Strip en zes partities `restore_mode: ALWAYS_OFF`. De balk toont niets bij een ongeldig, verouderd of onbekend niveau (FR-007); het alarm blijft vastgehouden en zichtbaar op led 1. De balk gebruikt de bestaande gefilterde vulgraad plus een herstelmarge, dus geen knipperen rond een grens. Geen uitgang naar de pomp. | ✅ PASS | ✅ PASS |
| **IV. Declaratieve, traceerbare logica** | Eén nieuw benoemd script met FR-commentaar; geen dubbele logica (led 1 en de balk gebruiken dezelfde `${niveau_bruikbaar}`; de helderheid één gedeelde uitdrukking). Het script heeft twee korte lambdas: één berekening (het aantal leds) en één uitvoering (de leds zetten). Elke requirement heeft een scenario ([quickstart.md](quickstart.md) §6). | ✅ PASS | ✅ PASS |
| **V. Expliciet, eenvoudig, waarneembaar** | Led-indexen, aantal stappen, herstelmarge en knippertijden zijn substitutions. Ongebruikte kleur `led_groen_*` en `led_normaal_aan_ms` worden verwijderd. Geen nieuwe HA-entiteiten (YAGNI). `substitutions` en [hardware.md](../001-pompput-monitor/contracts/hardware.md) lopen gelijk. | ✅ PASS | ✅ PASS |
| **Embedded Platform Constraints** | Zelfde pin (GPIO6), 3,3 V-data met VDD ≈ 4,3 V via diode. Ledbalk werkt zonder WiFi of HA (quickstart §4 #10). Mains ongewijzigd. | ✅ PASS | ✅ PASS |
| **Development Workflow & Quality Gates** | Gates 1–2: quickstart §2. Gate 3: tabel hieronder en quickstart §6. Gate 4: quickstart §3–§4, inclusief de bank-controles B1–B4. Gate 5: deze tabel; geen afwijkingen. Uitkomsten in `verificatie.md`. | ✅ PASS | ✅ PASS |

## Project Structure

### Documentation (this feature)

```text
specs/003-niveau-ledbalk/
├── plan.md                 # Dit bestand
├── research.md             # Fase 0: beslissingen R1–R7 en bank-controles
├── data-model.md           # Fase 1: niveaubalk-regels, nieuwe constanten en globals
├── quickstart.md           # Fase 1: gates, bijwerken, scenario's, traceerbaarheid
├── contracts/
│   └── ledbalk.md          # Led 1, niveaubalk, voorbeelden, aansluiting, HA
├── checklists/requirements.md
├── verificatie.md          # Bij de uitvoering: uitkomsten van de gates
└── tasks.md                # Fase 2 (/speckit-tasks)
```

### Source Code (repository root)

```text
esphome/
└── pompput.yaml                    # Sectie 1: led_aantal, led_index_*, niveaubalk_*, pomp- en
                                    #   normaal-tijden, groen weg, project_version 0.4.0
                                    # Sectie 5: status_bijwerken roept ook niveaubalk_bijwerken aan
                                    # Sectie 7: ledstrip + 6 partities, normaal/pomp in
                                    #   led_patroon_kiezen, nieuw script niveaubalk_bijwerken,
                                    #   twee globals
README.md                           # Componenten, aansluittabel, schema, voeding, ledtabel
specs/001-pompput-monitor/
├── spec.md                         # FR-027, FR-028, SC-009: "(gewijzigd door 003)"
├── data-model.md                   # §7: niveaubalk
├── quickstart.md                   # §1 onderdelen, §4 kleurcontrole en led-verwachtingen
└── contracts/
    ├── hardware.md                 # Keten van 6 leds, 1N4001, elko, voeding
    └── lokale-signalering.md       # Inhoud van contracts/ledbalk.md
```

**Structure Decision**: geen nieuwe bestanden of mappen buiten `specs/003-niveau-ledbalk/`. De
opbouw van `esphome/pompput.yaml` in genummerde secties blijft; sectie 7 heet voortaan
"signalering: ledbalk, patroonscripts".

## Traceerbaarheid

Constitutie, gate 3. Requirement → element (in `esphome/pompput.yaml`, tenzij anders vermeld).
De scenario's staan in [quickstart.md §6](quickstart.md#6-traceerbaarheid-requirement--scenario).

| Requirement | Element |
|---|---|
| FR-001 | `light` `ledstrip` (`num_leds: ${led_aantal}`), partities `statusled`, `niveauled_2` … `niveauled_6`; substitutions `led_index_*` |
| FR-002 | `led_patroon_kiezen`: patroon normaal = blauw, `led_normaal_uit_ms` 0 |
| FR-003 | `led_patroon_kiezen`: volgorde en patronen alarm … geen verbinding ongewijzigd |
| FR-004 | `niveaubalk_bijwerken`: grenzen *k* × alarmdrempel / `niveaubalk_stappen` |
| FR-005 | `niveaubalk_bijwerken`: grens 5 = alarmdrempel, los van `hoogwater` |
| FR-006 | `niveaubalk_bijwerken`: leds 2 … *N*+1 aan, rest uit |
| FR-007 | `niveaubalk_bijwerken`: `${niveau_bruikbaar}` onwaar of NAN → *N* = 0 |
| FR-008 | `niveaubalk_bijwerken` met `niveaubalk_herstelmarge_pct`, `g_niveaubalk_aantal` |
| FR-009 | `status_bijwerken` roept elke seconde `led_patroon_kiezen` en `niveaubalk_bijwerken` aan |
| FR-010 | `niveaubalk_bijwerken` leest `g_alarmdrempel_pct` elke ronde |
| FR-011 | `led_helderheid_pct` in `led_patroon_kiezen` en `niveaubalk_bijwerken`; `g_niveaubalk_helderheid` |
| FR-012 | `restore_mode: ALWAYS_OFF` op `ledstrip` en de partities; `on_boot` stap 4 ongewijzigd |
| FR-013 | substitutions: `led_groen_*` en `led_normaal_aan_ms` verwijderd |
| FR-014 | `led_patroon_kiezen`: patroon pomp met `led_pomp_aan_ms` / `led_pomp_uit_ms` |
| FR-015 | `README.md` en de levende documenten van 001 (R6) |
| FR-016 | secties 3, 4, 6 en 8 ongewijzigd; geen nieuwe bewaarde waarden |
| FR-017 | `led_patroon_kiezen`: regel `!${niveau_bruikbaar}` → `led_patroon_storing`, direct na `g_opstarten` |

## Complexity Tracking

Geen schendingen van de constitutie; niets te rechtvaardigen.
