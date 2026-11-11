# Verificatie: Niveau-ledbalk met zes leds

**Feature**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md) | **Procedure**: [quickstart.md](quickstart.md)

## Samenvatting van de gates

| Gate | Uitkomst | Datum |
|---|---|---|
| 1. Compile | ✅ geslaagd: `esphome config` geldig, `esphome compile` zonder fouten, 0 waarschuwingen | 2026-11-11 |
| 2. Resource | ✅ RAM 35,1 %, flash 47,4 % (grenzen 75 % en 90 %) | 2026-11-11 |
| 3. Traceerbaarheid | ✅ elke requirement heeft een element (plan "Traceerbaarheid") en een scenario (quickstart §6) | 2026-11-11 |
| 4. Hardware | **open**: bedrading, OTA-update, bank-controles B1–B4 en scenario's §4 nog niet uitgevoerd (zie hieronder) | |
| 5. Review | ✅ geen afwijkingen van de constitutie | 2026-11-11 |

## Uitgangssituatie

Vastgelegd vóór de wijziging (T002).

- **RAM en flash** van firmware 0.3.0, uit
  [002 verificatie.md](../002-zoemer-verwijderen/verificatie.md): RAM **34,7 %** (111 432 van
  321 296 bytes), flash **47,3 %** (867 140 van 1 835 008 bytes).
- **Instellingen van het apparaat** (14 `number`-entiteiten in Home Assistant): **niet
  uitgelezen** bij de uitvoering. Reden: de uitvoering had geen toegang tot Home Assistant of
  het apparaat. De beheerder noteert ze hieronder **vóór** het bijwerken (quickstart §1),
  anders is FR-016 (instellingen behouden) niet te controleren.

| Instelling | Waarde vóór de update | Waarde na de update |
|---|---|---|
| (in te vullen door de beheerder) | | |

## Hardware

(T003) Te controleren tegen [contracts/ledbalk.md §4](contracts/ledbalk.md#4-aansluiting-vervangt-de-led-in-001-hardwaremd).
**Niet gecontroleerd** bij de uitvoering. Reden: de uitvoering had geen toegang tot de
hardware. In te vullen door de beheerder:

| Controle | Uitkomst |
|---|---|
| Diode is een 1N4001 (geen 1N4148) | |
| Elko 100 µF tussen VDD en GND bij led 1 | |
| DIN van led 1 via 330 Ω aan D4 | |
| DOUT → DIN door de keten van zes leds | |
| Led 1 is de eerste in de keten (het dichtst bij het apparaat), led 6 bovenaan | |

## Compile- en resource-gate

ESPHome 2026.10.1. Drie builds: na de basis (T008), na US1 (T014) en na alle wijzigingen
(T018, T022, T031).

| Controle | Uitgevoerd | Uitkomst |
|---|---|---|
| `esphome config esphome/pompput.yaml` | ja | "Configuration is valid!"; zes `partition`-lichten op `ledstrip` |
| `esphome compile` na de basis (T008) | ja | geslaagd; 0 waarschuwingen; RAM 35,1 % (112 728 bytes), flash 47,4 % (869 084 bytes). Het platform `partition` werkt op deze strip; het terugvalplan uit research R1 is niet nodig |
| `esphome compile` na US1 (T014) | ja | geslaagd; 0 waarschuwingen; RAM 35,1 % (112 848 bytes), flash 47,4 % (870 324 bytes) |
| `esphome compile` na alle wijzigingen (T018, T022, T031) | ja | geslaagd; 0 waarschuwingen; RAM **35,1 %** (112 848 van 321 296 bytes), flash **47,4 %** (870 384 van 1 835 008 bytes) |
| Verschil met 0.3.0 | — | RAM +1 416 bytes, flash +3 244 bytes (research R7: "enkele kB") |
| Geen groen en geen `led_normaal_aan_ms` (quickstart §2 stap 3, T018) | ja | geen uitvoer |

## Bijwerken

(quickstart §3)

| Stap | Uitgevoerd | Uitkomst |
|---|---|---|
| 1. OTA-update | nee | |
| 2. Opstartgedrag, versie 0.4.0 | nee | |
| 3. Instellingen behouden | nee | |
| 4. Geen nieuwe entiteiten | nee | |

**Niet uitgevoerd.** Reden: de uitvoering had geen toegang tot het apparaat of Home Assistant,
en de instellingen moeten eerst genoteerd zijn (zie Uitgangssituatie). Uit te voeren door de
beheerder.

## Bank-controles

| Controle | Uitgevoerd | Uitkomst |
|---|---|---|
| B1 Partities | nee | geen toegang tot de hardware |
| B2 Gamma (oranje is oranje) | nee | geen toegang tot de hardware |
| B3 Kleurvolgorde (normaal = blauw) | nee | geen toegang tot de hardware |
| B4 Ketenvolgorde | nee | geen toegang tot de hardware |

## Scenario's op het apparaat

| # | Uitgevoerd | Uitkomst | Reden als niet uitgevoerd |
|---|---|---|---|
| 1 | nee | | geen toegang tot het apparaat bij de uitvoering |
| 2 | nee | | geen toegang tot het apparaat bij de uitvoering |
| 3 | nee | | geen toegang tot het apparaat bij de uitvoering |
| 4 | nee | | geen toegang tot het apparaat bij de uitvoering |
| 5 | nee | | geen toegang tot het apparaat bij de uitvoering |
| 6 | nee | | geen toegang tot het apparaat bij de uitvoering |
| 7 | nee | | geen toegang tot het apparaat bij de uitvoering |
| 8 | nee | | geen toegang tot het apparaat bij de uitvoering |
| 9 | nee | | geen toegang tot het apparaat bij de uitvoering |
| 10 | nee | | geen toegang tot het apparaat bij de uitvoering |
| 11 | nee | | geen toegang tot het apparaat bij de uitvoering |
| 12 | nee | | geen toegang tot het apparaat bij de uitvoering |
| 13 | nee | | geen toegang tot het apparaat bij de uitvoering |
| 14 | nee | | geen toegang tot het apparaat bij de uitvoering |
| 15 | nee | | geen toegang tot het apparaat bij de uitvoering |
| 16 | nee | | geen toegang tot het apparaat bij de uitvoering |
| 17 | nee | | geen toegang tot het apparaat bij de uitvoering |
| 18 | nee | | geen toegang tot het apparaat bij de uitvoering |
| 19 | nee | | geen toegang tot het apparaat bij de uitvoering |
| 20 | nee | | geen toegang tot het apparaat bij de uitvoering |

## Documentatie

(quickstart §5, T029) Uitgevoerd. Treffers:

- `README.md`: "Gebruik een **1N4001**: een 1N4148 kan de stroom van zes leds niet goed aan" (zegt
  dat de 1N4148 niet meer mag).
- `contracts/hardware.md`: "De 1N4148 is niet geschikt voor zes leds" (idem).
- `contracts/lokale-signalering.md`: "was prio 7, groen, flits per 5 s" (zegt wat vervallen is).
- `quickstart.md`: "bij 'normaal' is led 1 blauw en niet groen of wit" (kleurcontrole).

Geen treffer beschrijft de led nog als groen bij "normaal", als enkele led of met een 1N4148.
Bijgewerkt: `README.md`, en van 001 `spec.md`, `plan.md`, `data-model.md`, `quickstart.md`,
`contracts/hardware.md` en `contracts/lokale-signalering.md`.

## Review- en traceerbaarheidsgate

(T033)

- **Requirement-commentaar**: `003 FR-001` t/m `FR-014` en `FR-017` staan bij de elementen uit de
  tabel "Traceerbaarheid" in [plan.md](plan.md). FR-016 ("de rest blijft gelijk") is
  gecontroleerd met `git diff`: alleen sectie 1 (substitutions), sectie 2 (commentaar in
  `on_boot`), sectie 5 (`status_bijwerken`) en sectie 7 (signalering) zijn gewijzigd; de
  secties 3, 4, 6 en 8 niet. FR-015 is documentatie (zie hierboven).
- **Constitutie I**: geen externe componenten; `partition` is een standaardplatform van ESPHome
  2026.10.1.
- **Constitutie II**: `niveaubalk_bijwerken` heeft geen `delay`, geen `new`/`malloc` en geen
  `std::string`; de lijst met lichten is een vaste lokale array. Twee korte lambdas: berekenen
  (in de `if`-voorwaarde) en uitvoeren (in de `then`).
- **Constitutie III**: alle zeven lichten hebben `restore_mode: ALWAYS_OFF`. Bij een niet
  bruikbaar niveau is de balk uit en toont led 1 het storingspatroon (FR-007, FR-017).
- **Constitutie IV**: led 1 en de balk gebruiken dezelfde `${niveau_bruikbaar}`; de helderheid
  één gedeelde uitdrukking `${led_helderheid_nu}`, ook in `led_patroon_kiezen`.
- **Constitutie V**: alle nieuwe getallen zijn substitutions (`led_index_*`,
  `niveaubalk_stappen`, `niveaubalk_herstelmarge_pct`, `led_pomp_aan_ms`). Ongebruikt en dus
  verwijderd: `led_groen_*` en `led_normaal_aan_ms`. Geen nieuwe HA-entiteiten: de strip en de
  partities zijn `internal: true`.
- **Pinnen**: `pin_led` (GPIO6), `led_aantal` (6) en `led_index_1` … `led_index_6` komen overeen
  met `specs/001-pompput-monitor/contracts/hardware.md`.

## Afwijkingen

- Geen afwijkingen van de constitutie.
- **Hardware-gate open**: de controle van de bedrading (T003), de OTA-update, de
  bank-controles B1–B4 en de scenario's #1–#20 zijn niet uitgevoerd, omdat de uitvoering geen
  toegang had tot het apparaat of Home Assistant. De beheerder voert ze uit met
  [quickstart.md](quickstart.md) §3–§4 en vult de tabellen hierboven in. Let vooral op B1
  (partities zonder ingeschakelde moederstrip) en B2 (gamma): die zijn niet uit de
  documentatie vast te stellen (research, "Te controleren op de bank").
