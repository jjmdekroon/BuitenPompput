# Implementation Plan: Pompput-monitor

**Branch**: `001-pompput-monitor` (spec-map `specs/001-pompput-monitor`) | **Date**: 2026-11-11 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/001-pompput-monitor/spec.md`

**Bijgewerkt**: de piëzo-zoemer, het stilzetten en de BOOT-knopfunctie zijn vervallen met [specs/002-zoemer-verwijderen](../002-zoemer-verwijderen/plan.md). De rest van dit plan beschrijft de toestand van versie 0.2.0.

**Herziening**: dit plan vervangt dat van 2026-11-09 (Arduino/PlatformIO met MQTT).
- **Aanleiding**: het besluit van de beheerder op 2026-11-11 en constitutie 2.0.0. Het
  project gaat over op ESPHome, met alle logica in YAML, net als het zusterproject Buitenlamp.
- **Ongewijzigd**: de spec en de hardware.

## Summary

Een bewakingsapparaat bij de pompput.
- **Meten**: het waterniveau met een HC-SR04; of de pomp draait, via een ACS712.
- **Meldingen**: het apparaat bepaalt er zelf zeven: hoogwater, drie pompwaarschuwingen,
  twee sensorstoringen en "niet ingesteld".
- **Ter plaatse**: een RGBW-led en een PWM-gestuurde piëzo-zoemer, die ook werken als
  Home Assistant niet bereikbaar is.
- **Home Assistant**: alles gaat via de versleutelde native API van ESPHome. Daar komen de
  historie, de meldingen op afstand (`alert`) en de instellingen (`number`).

**Technische aanpak**:
- De firmware is één ESPHome-configuratie (`esphome/pompput.yaml`), met ESPHome 2026.10.1 en
  het esp-idf-framework op de XIAO ESP32C3.
- Alle beslislogica staat in benoemde scripts en template-entiteiten, met korte lambdas. De
  rekenregels zijn overgenomen uit de vorige versie (zie [data-model.md](data-model.md)).
- De meting is niet-blokkerend: `ultrasonic` werkt met een interrupt, en `ct_clamp` meet in de
  lus. Led- en zoemerpatronen zijn generieke scripts met niet-blokkerende `delay`s.
- Instellingen met onderlinge regels bewaren hun geaccepteerde waarde zelf in `globals`. Zo
  komt een afgewezen waarde na een herstart niet terug (research R11).
- De eerste keer wordt via USB geflasht, daarna via WiFi (OTA).
- Controle gebeurt op het apparaat, volgens [quickstart.md](quickstart.md); er zijn geen
  unittests.
- Onderbouwing: [research.md](research.md).

## Technical Context

**Language/Version**: ESPHome-YAML met korte C++-lambdas (C++ zoals de ESP-IDF-toolchain van
ESPHome 2026.10.1 levert; geen exceptions, geen RTTI)

**Primary Dependencies**:
- **ESPHome 2026.10.1**, exact vastgepind in `requirements.txt`, in een eigen `.venv`.
  Bouwen met de eigen ESP-IDF-toolchain van ESPHome, zonder PlatformIO (R2).
- Gebruikte componenten:
  - `ultrasonic`, `adc`, `ct_clamp`;
  - `esp32_rmt_led_strip`, `ledc`;
  - `template` (sensor, binary_sensor, text_sensor, number, switch, button, event);
  - `globals`, `script`, `interval`;
  - `debug`, `wifi_signal`, `uptime`, `api`, `ota`.
- Geen externe componenten.

**Storage**: Alleen de instellingen, in flash (ESPHome-preferences):
- vijf via `globals` met `restore_value`;
- negen via template-`number` met `restore_value`;
- de schakelaar via `restore_mode`.

Er worden geen meetgegevens of gebeurtenissen bewaard (FR-010). De historie staat in de
recorder van Home Assistant.

**Testing**:
- `esphome config` en `esphome compile`: de compile- en de resource-gate.
- Handmatige scenario's op het apparaat volgens [quickstart.md](quickstart.md) §3–§4: de
  hardware-gate.
- Geen unittests op de pc (constitutie IV).

**Target Platform**:
- Seeed Studio XIAO ESP32C3 (ESP32-C3, RISC-V 160 MHz, 400 KB SRAM, 4 MB flash, WiFi 2,4 GHz,
  externe antenne), `esp32: variant: esp32c3`, `framework: esp-idf`.
- Home Assistant met de ESPHome-integratie.

**Project Type**: ESPHome-apparaatconfiguratie plus een Home Assistant-configuratiepakket
(YAML).

**Performance Goals**:
- Pompstatus in HA ≤ 5 s na het werkelijke moment (FR-009). De stroom wordt elke 500 ms
  gemeten, met een debounce van 2 s.
- Waterhoogte in HA elke 10 s, en direct bij een statuswijziging. Een ongeldige of verouderde
  meting wordt "onbekend" (FR-003).
- Meldingen worden elke 1 s beoordeeld.
- Led en zoemer reageren binnen 1 s op een nieuwe melding (SC-009: 5 s).
- Een gewijzigde instelling geldt binnen 1 s (FR-023: binnen 1 min).

**Constraints**:
- Geen blokkerende lambdas; elke lambda duurt maximaal 10 ms.
- Overloopveilige `millis()`; geen eigen interrupts; geen heap-allocatie in lambdas die steeds
  opnieuw draaien.
- RAM ≤ 75 % en flash ≤ 90 % (constitutie II).
- De taak-watchdog blijft aan. `wifi`/`api` `reboot_timeout: 0s`.
- De GPIO's zijn niet 5 V-tolerant: niveau-aanpassing met weerstandsdelers (eis van de
  gebruiker). Alleen ADC1.
- Geen uitgang die de pomp beïnvloedt.
- Netspanning (230 V) door de ACS712: zie [hardware.md](contracts/hardware.md).

**Scale/Scope**:
- 1 apparaat, 1 put, 1 pomp.
- 37 HA-entiteiten:
  - 8 sensoren en 3 tekstsensoren;
  - 8 binary sensors;
  - 14 numbers, 1 switch, 2 buttons;
  - 1 event.
- 15 instellingen.

**Bevestigde hardware**: een adresseerbare RGBW-led (SK6812-compatibel) en de ACS712
30 A-variant (66 mV/A). De bankverificaties die nog openstaan, staan in research
("Open punten").

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.* Getoetst aan
constitutie **2.0.0**.

| Principe | Toets | Vóór fase 0 | Na fase 1 |
|---|---|---|---|
| **I. ESPHome is het doel** | ESPHome 2026.10.1, vastgepind (R2). XIAO ESP32C3 met `esp-idf`. Native API met versleuteling, OTA met dezelfde sleutel, geen MQTT (R3). Geen externe componenten. Platformgedrag (ADC1 en `12db`, strapping-pin GPIO9, 3,3 V) staat in [hardware.md](contracts/hardware.md) en research. | ✅ PASS | ✅ PASS |
| **II. Niet-blokkerende, begrensde automatiseringen** | Interrupt-gestuurde `ultrasonic`; `ct_clamp` meet in de lus; patronen als scripts met `delay`; verwerking in `interval: 1s` (R12). De toestandsmachines voor niveau, pomp en meldingen staan in `globals`, met benoemde toestanden (codes met een naam in `substitutions`) en de overgangen in benoemde scripts. Overloopveilige `millis()`; ringbuffers als vaste `std::array`; geen eigen interrupts. RAM en flash worden gemeten in de resource-gate. De rekentijd per script wordt apart gemeten (T039), omdat ESPHome pas boven 50 ms waarschuwt. | ✅ PASS | ✅ PASS |
| **III. Veilig, alleen bewaken** | Geen uitgang naar de pomp (FR-007). Led `ALWAYS_OFF`, zoemer uit, TRIG via de component ([hardware.md](contracts/hardware.md)). Ongeldig of verouderd → `NAN` en een storing. Hoogwater wordt vastgehouden (FR-022). Filters en hysterese: mediaan, snelheidscontrole, debounce en de herstelmarges. De watchdog blijft aan. Storingen zijn zichtbaar in HA en het log (R13). | ✅ PASS | ✅ PASS |
| **IV. Declaratieve, traceerbare logica** | Alle logica in benoemde scripts en template-entiteiten. Eén levenscyclus-script voor alle meldingen, zodat er geen dubbele logica is (R5). Lambdas doen één berekening. Elk script of elke entiteit krijgt een commentaar met de FR-nummers. Elke beslisregel heeft een scenario in [quickstart.md](quickstart.md) §4 (zie §7 en de tabel hieronder). | ✅ PASS | ✅ PASS |
| **V. Expliciet, eenvoudig, waarneembaar** | Pinnen, constanten, grenzen, standaardwaarden en de codes van toestanden en meldingen staan in één `substitutions:`-blok; lambdas bevatten geen losse getallen. Geheimen staan in `secrets.yaml`, met `secrets.example.yaml` in de repository. Geen `web_server` of `captive_portal`. Gebeurtenissen, Laatste gebeurtenis en diagnostische sensoren; `logger` op `INFO`. | ✅ PASS | ✅ PASS |
| **Embedded Platform Constraints** | Gezaghebbende pinbron: [hardware.md](contracts/hardware.md) ↔ `substitutions`. Bouwen met de ESPHome-CLI uit `.venv`. `.esphome/` en `secrets.yaml` staan in `.gitignore`. Geen map met tests op de pc. Werkt zonder WiFi of HA. Netspanning alleen voor de meting. | ✅ PASS | ✅ PASS |
| **Development Workflow & Quality Gates** | De gates 1–5 staan in [quickstart.md](quickstart.md) §2 (compile, resource), §7 en hieronder (traceerbaarheid), en §4 (hardware). De review-gate is deze tabel; er zijn geen afwijkingen. | ✅ PASS | ✅ PASS |

**Aandachtspunt (geen schending)**: de ACS712 staat in serie met de pomp. Een defecte module kan
de pomp onderbreken, en dat raakt de bedoeling van FR-007 en SC-007. Dit is een keuze van de
gebruiker en benoemd als risico met maatregelen (research R7); een stroomtang (CT) is het
alternatief.

## Project Structure

### Documentation (this feature)

```text
specs/001-pompput-monitor/
├── plan.md                         # Dit bestand
├── research.md                     # Fase 0: beslissingen R1–R16 + open punten
├── data-model.md                   # Fase 1: instellingen, toestanden, meldingen, gebeurtenissen
├── quickstart.md                   # Fase 1: bouwen, flashen, HA koppelen, scenario's, traceerbaarheid
├── contracts/
│   ├── hardware.md                 # Pinnen, schakelingen, voeding, veiligheid (hardware ongewijzigd)
│   ├── esphome-homeassistant.md    # Entiteiten, entity-id's, gedrag, HA-pakket (vervangt mqtt-homeassistant.md)
│   └── lokale-signalering.md       # Led-kleuren/patronen, zoemerpatronen, stilzetten
├── checklists/requirements.md
├── verificatie.md                  # Uitkomsten van de gates en de controles op het apparaat (nieuwe opzet: T006)
└── tasks.md                        # Fase 2 (/speckit-tasks), bij dit plan
```

### Source Code (repository root)

```text
requirements.txt                    # esphome==2026.10.1
esphome/
├── pompput.yaml                    # Het hele apparaat, in genummerde secties:
│                                   #  1. substitutions: pinnen, constanten, grenzen en standaardwaarden
│                                   #  2. basis: esphome (on_boot), esp32, preferences, logger, wifi, api, ota, debug
│                                   #  3. niveau: ultrasonic, waterhoogte/vulgraad/afstand, niveau_verwerken
│                                   #  4. pomp: adc, ct_clamp, pompstroom/offset, pomp_bijwerken
│                                   #  5. meldingen: globals, melding_stap, beoordelingsscripts, binary sensors, status
│                                   #  6. instellingen: globals, numbers, switch, instelling_toepassen
│                                   #  7. signalering: led, ledc, led/zoemer-scripts, knoppen
│                                   #  8. gebeurtenissen en diagnose: event, Laatste gebeurtenis, intervals
├── secrets.example.yaml            # wifi_ssid, wifi_password, api_encryption_key (voorbeeldwaarden)
└── secrets.yaml                    # Eigen waarden; in .gitignore
homeassistant/
├── packages/pompput.yaml           # history_stats, onbereikbaar-sensor, 8 alerts, dashboard-helper
└── dashboard-pompput.yaml          # Voorbeelddashboard (SC-006)
README.md                           # Herschreven voor ESPHome; het hoofdstuk "Aansluiten" blijft
.gitignore                          # .venv/, esphome/.esphome/, esphome/secrets.yaml
```

Wat verdwijnt (research R16): `platformio.ini`, `include/`, `lib/`, `src/` en `test/`, de
Arduino/MQTT-implementatie.

**Structure Decision**:
- **Eén apparaat-YAML met genummerde secties**, net als de Buitenlamp. Constitutie 2.0.0
  noemt één YAML per apparaat.
- **De Home Assistant-configuratie** blijft als package in `homeassistant/`, zodat ze samen
  met de firmware wordt beheerd.
- **Geen aparte ESPHome-packagebestanden of externe componenten**: die zijn hier niet nodig
  (constitutie V).
- **Secties 3–8 als pakketten in hetzelfde bestand** (bijgesteld bij de implementatie,
  2026-11-11). ESPHome staat een sleutel als `sensor:` of `script:` maar één keer toe. Sectie 1
  en 2 staan daarom bovenaan, en elke volgende sectie is een pakket binnen `packages:` in
  `esphome/pompput.yaml`; ESPHome voegt hun lijsten samen. Zo blijft elke sectie per onderwerp
  bij elkaar. Een id komt maar in één sectie voor.

## Traceerbaarheid

Constitutie, gate 3. Het overzicht requirement → YAML-element. De scenario's per requirement
staan in [quickstart.md §7](quickstart.md#7-traceerbaarheid-requirement--scenario). De
YAML-elementen krijgen een commentaar met dezelfde FR-nummers.

| Requirement | YAML-element(en) in `esphome/pompput.yaml` (of het HA-pakket) |
|---|---|
| FR-001 | sensor `niveau_ping` (`ultrasonic`) |
| FR-002 | sensors `waterhoogte`, `vulgraad`; berekening in `niveau_verwerken` |
| FR-003 | `niveau_verwerken` (VEROUDERD/STORING → `NAN`); `interval: 10s` en publicatie bij een statuswijziging |
| FR-004 | `niveau_verwerken`: mediaan van 5, minimaal 3 geldig |
| FR-005 | `niveau_verwerken`: bereik, snelheidscontrole, `NAN` bij geen echo |
| FR-006 | `adc` `acs712_spanning`, `ct_clamp` `pompstroom_rms`, script `pomp_bijwerken` |
| FR-007 | geen uitgang naar de pomp (review van de YAML) |
| FR-008 | binary sensor `pomp` |
| FR-009 | `pomp_bijwerken` (debounce, `pomp_start`/`pomp_stop`), sensor `looptijd_laatste_pompbeurt`; HA `history_stats` |
| FR-010 | geen opslag van metingen of gebeurtenissen; `gebeurtenis_melden` alleen live |
| FR-011 | `pomp_bijwerken`: 2 s debounce |
| FR-012 | `melding_hoogwater_beoordelen` + `melding_stap`; binary sensor `hoogwater` |
| FR-013 | `melding_pomp_start_niet_beoordelen`; binary sensor `pomp_start_niet` |
| FR-014 | `melding_pomp_pompt_niet_af_beoordelen`; binary sensor `pomp_pompt_niet_af` |
| FR-015 | `melding_pomp_draait_te_lang_beoordelen`; binary sensor `pomp_draait_te_lang` |
| FR-016 | `melding_niveausensor_storing_beoordelen`, `melding_stroomsensor_storing_beoordelen`; sensor `offset_stroomsensor`; `niveau_verwerken` (STORING en opnieuw inleren, R6) |
| FR-017 | HA-pakket: template-binary-sensor `pompput_onbereikbaar` + alert |
| FR-018 | binary sensors van de meldingen, text sensor `status`; HA-alerts |
| FR-019, FR-020, FR-021 (aflevering) | HA-pakket: `message`, `repeat: 60`, `done_message` |
| FR-021 (hysterese) | de herstelvoorwaarden in de beoordelingsscripts |
| FR-022 | `melding_hoogwater_beoordelen`: niet beoordelen bij een niveau ≠ GELDIG |
| FR-023 | numbers (sectie 6), script `instelling_toepassen` |
| FR-024 | `globals` met `restore_value`, `number` met `restore_value`, `switch` met `restore_mode`; `preferences` |
| FR-025 | `on_boot`; `reboot_timeout: 0s`; meten zonder verbinding |
| FR-026 | event `gebeurtenis`, text sensor `laatste_gebeurtenis`, script `gebeurtenis_melden`; diagnostische sensoren |
| FR-027 | light `statusled`, output `zoemer_output`, scripts `led_patroon_kiezen`, `led_knipperen`, `zoemer_patroon_kiezen`, `zoemer_herhalen` en `zoemer_eenmalig` |
| FR-028 | script `zoemer_stilzetten`, binary sensor `boot_knop`, button `zoemer_stil`, switch `zoemer`, number `led_helderheid_pct` |
| Randgeval "niet ingesteld" | `melding_niet_ingesteld_beoordelen`; globals met `NAN` |
| Randgeval "ongeldige instellingen" | `instelling_toepassen` (weigeren, oude state opnieuw publiceren, gebeurtenis) |

## Complexity Tracking

Geen schendingen van constitutie 2.0.0. Er is geen rechtvaardiging nodig.

Uitleg bij principe V (review-gate, 2026-11-11): de rekenkundige 1 en 2 (ophogen, een bit
maken, de mediaan) gelden niet als magisch getal; alle afstelwaarden, codes en omrekeningen
hebben een naam in `substitutions`. De afwijkingen van plan en taken staan in
[verificatie.md](verificatie.md#afwijkingen).
