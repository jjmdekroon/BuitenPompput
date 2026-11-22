# Verificatie: Stroomsensor verwijderen

**Feature**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md) | **Quickstart**: [quickstart.md](quickstart.md)

Uitkomsten van de gates van de constitutie (Development Workflow & Quality Gates) voor firmware
0.5.0.

## Samenvatting van de gates

| Gate | Uitkomst | Datum |
|---|---|---|
| 1. Compile | ✅ `config` geldig; `compile` geslaagd, 0 waarschuwingen (T015, T033) | 2026-11-18 |
| 2. Resource | ✅ RAM 34,5 %, flash 46,5 % (grenzen 75 % en 90 %); beide lager dan 0.4.0 | 2026-11-18 |
| 3. Traceerbaarheid | ✅ elke FR heeft een element in de YAML (of een HA-/documentbestand) en een scenario (plan.md, quickstart §6) | 2026-11-18 |
| 4. Hardware | ⏳ **open**: §3 en §4 vragen het apparaat en Home Assistant (T016, T020, T026) | — |
| 5. Review | ✅ geen afwijkingen van constitutie 2.0.2 | 2026-11-18 |

## Uitgangssituatie

- **RAM en flash** van firmware 0.4.0, uit
  [003 verificatie.md](../003-niveau-ledbalk/verificatie.md): RAM **35,1 %** (112 848 van
  321 296 bytes), flash **47,4 %** (870 384 van 1 835 008 bytes).
- **Instellingen in Home Assistant** (Afstand leeg, Afstand vol, Alarmdrempel, Herstelmarge,
  Wachttijd hoogwater, Wachttijd sensorstoring, Led-helderheid): **niet uitgelezen**. De
  uitvoering gebeurde zonder toegang tot Home Assistant of het apparaat. Noteer de zeven waarden
  vóór het bijwerken (T016), anders is quickstart §3 stap 3 niet te controleren.
- **Actieve pompwaarschuwing of stroomsensorstoring vóór de update**: niet bekend (zelfde
  reden); noteer dit vóór het bijwerken, voor quickstart §3 stap 6.

## Compile- en resource-gate

| Controle | Uitgevoerd | Uitkomst |
|---|---|---|
| Geen pomp- of stroomresten (quickstart §2 stap 3, T014) | ja | geen treffers |
| `esphome config` (T015) | ja | "Configuration is valid!" |
| `esphome compile` (T015) | ja | geslaagd; 0 compilerwaarschuwingen; RAM **34,5 %** (110 736 van 321 296 bytes), flash **46,5 %** (853 118 van 1 835 008 bytes) |
| Verschil met 0.4.0 | — | RAM −2 112 bytes, flash −17 266 bytes (research R12: alleen dalen) |

## Bijwerken

Quickstart §3.

| Stap | Uitgevoerd | Uitkomst |
|---|---|---|
| 1. Flashen via WiFi | nee | |
| 2. Opstarten, versie 0.5.0 | nee | |
| 3. Instellingen behouden | nee | |
| 4. Entiteiten opgeruimd | nee | |
| 5. Pakket en dashboard | nee | |
| 6. Geen hangende pompmelding | nee | |

## Scenario's op het apparaat

Quickstart §4.

| # | Uitgevoerd | Uitkomst | Reden als niet uitgevoerd |
|---|---|---|---|
| 1 | nee | | |
| 2 | nee | | |
| 3 | nee | | |
| 4 | nee | | |
| 5 | nee | | |
| 6 | nee | | |
| 7 | nee | | |
| 8 | nee | | |
| 9 | nee | | |
| 10 | nee | | |
| 11 | nee | | |
| 12 | nee | | |
| 13 | nee | | |

## Documentatie

Quickstart §5, uitgevoerd op 2026-11-18 (T032). Na het weglaten van regels met "(vervallen met
004)", "(gewijzigd door 004)" of `004-stroomsensor-verwijderen` blijven alleen toegestane
treffers over:

- **Weghaalparagraaf** in `README.md` en `contracts/hardware.md` (incl. de verwijzing in de
  veiligheidswaarschuwing van de README en de regel over GPIO3 in "Veilige begintoestand").
- **Verwijzingen naar 004 op een volgende regel** (de zoekterm staat op de regel ervoor):
  `data-model.md` van 001 en 003, `lokale-signalering.md`, `ledbalk.md`, `esphome-homeassistant.md`
  ("vervallen met 004"), de pintabel in `hardware.md`, 003 `spec.md` (Key Entity
  Statuspatroon) en 003 `quickstart.md` (B2 Gamma).
- **Historie** in `specs/001-pompput-monitor/spec.md`: de oorspronkelijke **Input**, de
  clarification over pompcycli van 2026-11-09, en de tekst van User Story 3 en 4, waarvan de kop
  "(vervallen met 004)" draagt.

Het aansluitschema in README en `hardware.md` toont D1/A1 als vrij. De paragraaf "Een bestaande
stroomsensor weghalen" in beide bestanden bevat de vijf punten uit contracts §4 (blijven zitten
mag; 230 V en vakbekwaam persoon; fasedraad doorverbinden, N en PE ongewijzigd; pomp op de
vlotter controleren; laagspanningskant weghalen of isoleren).

## Review- en traceerbaarheidsgate

T034, getoetst aan constitutie **2.0.2** (T003, goedgekeurd door de maintainer op 2026-11-18:
alleen de voorbeelden "pump status", "pump detection" en "(the ACS712 current sensor)" zijn
geschrapt; PATCH).

| Principe | Uitkomst |
|---|---|
| I. ESPHome | ESPHome 2026.10.1 ongewijzigd; geen nieuwe componenten; `adc` en `ct_clamp` zijn weg |
| II. Niet-blokkerend | alleen logica verwijderd; de toestandsmachines voor niveau, meldingen en led staan in `globals` met benoemde scripts; RAM en flash gedaald |
| III. Alleen bewaken | uitgangen: TRIG (`ultrasonic`) en de ledketen (`esp32_rmt_led_strip`, zes partities, `ALWAYS_OFF`); geen `output` of `switch`; GPIO3 niet geconfigureerd |
| IV. Traceerbaar | elk overgebleven script draagt zijn FR-regel; FR-verwijzingen naar vervallen eisen zijn met de code verdwenen; geen dode scripts, globals of substitutions (quickstart §2 stap 3: geen treffers) |
| V. Eenvoudig | `substitutions` bevat alleen `pin_trig` (GPIO4), `pin_echo` (GPIO5) en `pin_led` (GPIO6), gelijk aan de pintabel in `hardware.md` (D1/A1 en D5 vrij) |
| Platform | 20 HA-entiteiten: 17 `template`, plus `debug`, `wifi_signal` en `uptime`; `reboot_timeout` van WiFi en API blijft 0 s |

Traceerbaarheid: alle 15 FR's en 6 SC's van 004 staan in de tabel "Traceerbaarheid" van
[plan.md](plan.md) en in [quickstart.md §6](quickstart.md#6-traceerbaarheid-requirement--scenario).

## Afwijkingen

- **T025, scenario #20 in `specs/003-niveau-ledbalk/quickstart.md`**: de taak zei "led 1 paars
  door een storing van de niveausensor". Dat kan niet: bij een storing van de niveausensor is
  de balk uit (003 FR-007), en zonder stroomsensor bestaat er geen toestand meer met led 1 paars
  én een volle balk. #20 meet nu bij een actief alarm (led 1 rood, led 2–6 blauw); dat is sinds
  004 het zwaarste geval. De grens van ongeveer 0,6 A blijft gelden. B2 Gamma gebruikt nu paars
  (storing van de niveausensor) in plaats van oranje.
