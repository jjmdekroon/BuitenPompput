# Contract: Wijzigingen in signalering, hardware en Home Assistant

**Status**: ontwerp. Dit contract beschrijft wat er voor de gebruiker en voor Home Assistant
verandert door het verwijderen van de zoemer (spec FR-001 t/m FR-011). De volledige
contracten van 001 worden bij de uitvoering op deze punten bijgewerkt (research R5):
- [lokale-signalering.md](../../001-pompput-monitor/contracts/lokale-signalering.md)
- [hardware.md](../../001-pompput-monitor/contracts/hardware.md)
- [esphome-homeassistant.md](../../001-pompput-monitor/contracts/esphome-homeassistant.md)

## 1. Signalering ter plaatse

- **Statusled**: de tabel "Statusled (RGBW)" van 001 blijft **letterlijk** gelijk: zeven
  toestanden, kleuren, knipperpatronen, prioriteiten, de helderheid uit
  `led_helderheid_pct` en minimaal 10 % bij alarm (FR-002).
- **Reactietijd**: het gekozen patroon is binnen 1 s na een verandering van toestand zichtbaar,
  ook zonder verbinding met Home Assistant (FR-003; eis ≤ 5 s).
- **Zoemer**: het hoofdstuk "Zoemer" vervalt. Het apparaat maakt op geen enkel moment geluid,
  ook niet bij opstarten (FR-001).
- **Stilzetten**: het hoofdstuk "Stilzetten (FR-028)" vervalt, inclusief de bevestiging
  (piep en 2× witte flits). De led toont altijd de werkelijke toestand.

## 2. Hardware

| XIAO-pin | GPIO | Was | Wordt |
|---|---|---|---|
| D5 | 7 | uit (PWM), piëzo-zoemer via 100 Ω | **vrij**; niet geconfigureerd door de firmware (FR-007) |
| D9 | 9 | in, BOOT-knop = "zoemer stil" | BOOT-knop op het board, **geen functie in de firmware**; houdt zijn functie bij het opstarten (downloadmodus) (FR-005) |

- Een nog aangesloten passieve zoemer blijft stil en mag blijven zitten (research R2).
- De `substitutions` `pin_zoemer` en `pin_knop` verdwijnen uit `esphome/pompput.yaml`; de
  pintabel in `hardware.md` en het blok blijven gelijk lopen (Embedded Platform Constraints).
- De onderdelenlijst en het aansluitschema in `README.md` noemen de zoemer en de knop
  "zoemer stil" niet meer; D5 staat als vrij aangegeven (FR-011).

## 3. Home Assistant

### Entiteiten van het apparaat

| Domein | Entity-id | Wordt |
|---|---|---|
| `switch` | `switch.pompput_zoemer` | **vervalt** (FR-004) |
| `button` | `button.pompput_zoemer_stil` | **vervalt** (FR-004) |
| `number` | `number.pompput_led_helderheid` | ongewijzigd (FR-006) |
| `button` | `button.pompput_niveaumeting_opnieuw_starten` | ongewijzigd |

Alle overige entiteiten, entity-id's, `event_types` en de opstartvolgorde blijven gelijk
(FR-008). De apparaatversie in HA wordt `0.3.0` (research R6).

### Pakket en dashboard

- **`homeassistant/packages/pompput.yaml`**: de melding "Pompput onbereikbaar" eindigt met
  "Bij de put signaleert de led nog wel." De overige meldingen zijn ongewijzigd (FR-010).
- **`homeassistant/dashboard-pompput.yaml`**: in de kaart "Bediening" vervallen
  `button.pompput_zoemer_stil` en `switch.pompput_zoemer`; "Instellingen tonen" blijft
  (FR-010).
- **Bevestigen van meldingen** (`can_acknowledge`) is een functie van Home Assistant en blijft
  ongewijzigd.

### Verweesde entiteiten

Na de update mogen er geen onbeschikbare zoemer-entiteiten meer in HA staan (SC-004). Zie
research R4 en [quickstart.md](../quickstart.md) §3.
