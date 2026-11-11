# Verificatie: Zoemer verwijderen

**Feature**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md) | **Procedure**: [quickstart.md](quickstart.md)

## Samenvatting van de gates

| Gate | Uitkomst | Datum |
|---|---|---|
| 1. Compile | ✅ geslaagd: `esphome config` geldig, `esphome compile` zonder fouten, 0 waarschuwingen | 2026-11-11 |
| 2. Resource | ✅ RAM 34,7 %, flash 47,3 %; beide lager dan 0.2.0 | 2026-11-11 |
| 3. Traceerbaarheid | ✅ elke requirement heeft een element (plan "Traceerbaarheid") en een scenario (quickstart §6) | 2026-11-11 |
| 4. Hardware | **open**: OTA-update en scenario's §3 en §4 nog niet uitgevoerd (zie hieronder) | |
| 5. Review | ✅ geen afwijkingen van de constitutie | 2026-11-11 |

## Uitgangssituatie

Vastgelegd vóór de wijziging (T002).

- **RAM en flash** van firmware 0.2.0, uit
  [001 verificatie.md](../001-pompput-monitor/verificatie.md) §"T039: laatste compile- en
  resource-gate": RAM **35,3 %** (113 398 van 321 296 bytes), flash **48,5 %** (889 218 van
  1 835 008 bytes).
- **Instellingen van het apparaat** (14 `number`-entiteiten in Home Assistant): **niet
  uitgelezen** bij de uitvoering. Reden: de uitvoering had geen toegang tot Home Assistant of
  het apparaat. De beheerder noteert ze hieronder **vóór** het bijwerken (quickstart §1),
  anders is FR-009 / SC-005 niet te controleren.

| Instelling | Waarde vóór de update | Waarde na de update |
|---|---|---|
| (in te vullen door de beheerder) | | |

## Compile- en resource-gate

T014 en T027 (de laatste build, na alle wijzigingen), ESPHome 2026.10.1:

| Controle | Uitgevoerd | Uitkomst |
|---|---|---|
| `esphome config esphome/pompput.yaml` | ja | "Configuration is valid!" |
| `esphome compile esphome/pompput.yaml` | ja | geslaagd; 0 waarschuwingen in de buildlog |
| RAM ≤ 75 % | ja | **34,7 %** (111 432 van 321 296 bytes); was 35,3 % (−1 966 bytes) |
| Flash ≤ 90 % | ja | **47,3 %** (867 140 van 1 835 008 bytes); was 48,5 % (−22 078 bytes) |
| Geen zoemerresten (quickstart §2 stap 3, T013) | ja | geen uitvoer |

## Bijwerken

(quickstart §3) **Niet uitgevoerd.** Reden: de OTA-update en de controles in Home Assistant
vragen toegang tot het apparaat en tot HA, en de instellingen moeten eerst genoteerd zijn
(zie Uitgangssituatie). Uit te voeren door de beheerder:

| Stap | Uitgevoerd | Uitkomst |
|---|---|---|
| 1. OTA-update naar 0.3.0 | nee | |
| 2. Herstart zonder piep; versie 0.3.0 in HA | nee | |
| 3. 14 instellingen ongewijzigd (FR-009, SC-005) | nee | |
| 4. Geen "Zoemer" en "Zoemer stil" in HA (FR-004, SC-004) | nee | |
| 5. Pakket en dashboard herladen; kaart "Bediening" alleen "Instellingen tonen" (FR-010) | nee | |

## Scenario's op het apparaat

(quickstart §4) **Niet uitgevoerd.** Reden: deze scenario's vragen iemand bij het apparaat
die kijkt en luistert, en de nieuwe firmware is nog niet op het apparaat gezet.

| # | Uitgevoerd | Uitkomst | Reden als niet uitgevoerd |
|---|---|---|---|
| 1 | nee | | firmware nog niet bijgewerkt; vraagt aanwezigheid bij het apparaat |
| 2 | nee | | idem |
| 3 | nee | | idem |
| 4 | nee | | idem |
| 5 | nee | | idem |
| 6 | nee | | idem |
| 7 | nee | | idem |
| 8 | nee | | idem |
| 9 | nee | | idem; faalt dit, dan geldt de terugvaloptie uit T015 |
| 10 | nee | | idem |
| 11 | nee | | idem |

## Documentatie

(quickstart §5, T026) **Geslaagd** (2026-11-11). De zoekopdracht geeft alleen:
- verwijzingen naar `specs/002-zoemer-verwijderen` of zinnen die zeggen dat de zoemer is
  vervallen (README, 001 spec sessie 002, data-model §7, quickstart #14,
  esphome-homeassistant.md, hardware.md, lokale-signalering.md, de kopcommentaar van het
  HA-pakket);
- de twee historische clarifications in `specs/001-pompput-monitor/spec.md`, die eindigen op
  "(zoemer vervallen met 002)";
- in `hardware.md` "Veilige begintoestand": een nog aangesloten passieve zoemer blijft stil
  (002 research R2). Dat beschrijft het gevolg van het vervallen, niet de zoemer als onderdeel.

De README toont D5 als vrij, in de tabel "Overzicht" en in "Pinnen van de XIAO".

## Review- en traceerbaarheidsgate

(T028) Getoetst aan constitutie 2.0.1:

| Toets | Uitkomst |
|---|---|
| Enige uitgangen | TRIG (`ultrasonic`, `trigger_pin: ${pin_trig}`) en de statusled (`esp32_rmt_led_strip`, `restore_mode: ALWAYS_OFF`). Geen `ledc`, geen `output`, geen `switch`, geen uitgang naar de pomp ✅ |
| Pinnen | `substitutions`: `pin_acs712` GPIO3, `pin_trig` GPIO4, `pin_echo` GPIO5, `pin_led` GPIO6; gelijk aan [hardware.md](../001-pompput-monitor/contracts/hardware.md) (D5 vrij, D9 zonder functie) ✅ |
| YAGNI | geen ongebruikte substitutions, globals, scripts of componenten (T013) ✅ |
| Led-logica | `led_patroon_kiezen` en `led_knipperen` letterlijk gelijk, behalve de vervallen bevestigingsvoorwaarde (FR-002) ✅ |
| Bewaarde waarden | geen id van een bewaarde `number` of `global` gewijzigd (research R3) ✅ |
| FR-commentaren | on_boot `FR-025, FR-027`; `status_bijwerken` noemt 002 FR-003; sectie 7 en `led_patroon_kiezen` noemen 002 FR-001–FR-007 en FR-002/FR-003 ✅ |
| Traceerbaarheid | FR-001–FR-011 hebben elk een element (plan.md "Traceerbaarheid") en een scenario (quickstart §6) ✅ |

## Afwijkingen

- Buiten de taken om is ook de inhoudsopgave van sectie 6 in de kopcommentaar van
  `esphome/pompput.yaml` bijgewerkt ("globals, numbers, instelling_toepassen", zonder "switch"),
  omdat de schakelaar uit die sectie is verdwenen.
- Hardware-gate open: §3 en §4 zijn nog niet uitgevoerd (zie hierboven).
