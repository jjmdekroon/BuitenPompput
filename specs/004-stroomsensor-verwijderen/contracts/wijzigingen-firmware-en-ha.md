# Contract: Wijzigingen in hardware, signalering en Home Assistant

**Status**: ontwerp. Dit contract beschrijft wat er voor de gebruiker, de bouwer en Home
Assistant verandert door het verwijderen van de stroomsensor (spec FR-001 t/m FR-015). De
volledige contracten van 001 en 003 worden bij de uitvoering op deze punten bijgewerkt
(research R9):
- [hardware.md](../../001-pompput-monitor/contracts/hardware.md)
- [esphome-homeassistant.md](../../001-pompput-monitor/contracts/esphome-homeassistant.md)
- [lokale-signalering.md](../../001-pompput-monitor/contracts/lokale-signalering.md)
- [ledbalk.md](../../003-niveau-ledbalk/contracts/ledbalk.md)

## 1. Entiteiten van het apparaat

Van 35 naar **20** entiteiten. Alle overgebleven entity-id's blijven gelijk (FR-006).

| Domein | Entity-id | Wordt |
|---|---|---|
| `binary_sensor` | `binary_sensor.pompput_pomp` | **vervalt** (FR-001, FR-003) |
| `sensor` | `sensor.pompput_looptijd_laatste_pompbeurt` | **vervalt** |
| `sensor` | `sensor.pompput_pompstroom` | **vervalt** |
| `sensor` | `sensor.pompput_offset_stroomsensor` | **vervalt** |
| `binary_sensor` | `binary_sensor.pompput_pomp_start_niet` | **vervalt** (FR-002) |
| `binary_sensor` | `binary_sensor.pompput_pomp_pompt_niet_af` | **vervalt** |
| `binary_sensor` | `binary_sensor.pompput_pomp_draait_te_lang` | **vervalt** |
| `binary_sensor` | `binary_sensor.pompput_stroomsensor_storing` | **vervalt** |
| `number` | `number.pompput_inschakelniveau` | **vervalt** (FR-004) |
| `number` | `number.pompput_marge_pomp_start_niet` | **vervalt** |
| `number` | `number.pompput_wachttijd_pomp_start_niet` | **vervalt** |
| `number` | `number.pompput_controletijd_afpompen` | **vervalt** |
| `number` | `number.pompput_minimale_daling_afpompen` | **vervalt** |
| `number` | `number.pompput_maximale_looptijd_pomp` | **vervalt** |
| `number` | `number.pompput_stroomdrempel_pomp` | **vervalt** |

**Blijven** (20): `sensor.pompput_waterhoogte`, `sensor.pompput_vulgraad`,
`sensor.pompput_afstand_ruw`, `sensor.pompput_wifi_signaal`, `sensor.pompput_uptime`,
`binary_sensor.pompput_hoogwater`, `binary_sensor.pompput_niveausensor_storing`,
`binary_sensor.pompput_niet_ingesteld`, `sensor.pompput_status`,
`sensor.pompput_laatste_gebeurtenis`, `sensor.pompput_herstartreden`,
`event.pompput_gebeurtenis`, `button.pompput_niveaumeting_opnieuw_starten`, en de numbers
`afstand_leeg`, `afstand_vol`, `alarmdrempel`, `herstelmarge`, `wachttijd_hoogwater`,
`wachttijd_sensorstoring` en `led_helderheid` (FR-008).

**Veranderde waarden**:
- `sensor.pompput_status`: `alarm`, `niet_ingesteld`, `storing` of `normaal`. De waarde
  `waarschuwing` vervalt (data-model §3).
- `event.pompput_gebeurtenis`: de `event_types` `pomp_start` en `pomp_stop` vervallen
  (data-model §4).
- `sensor.pompput_laatste_gebeurtenis` bij `melding_actief`: zonder `pomp …` (FR-009).
- De apparaatversie in HA wordt `0.5.0` (research R11).

## 2. Home Assistant-pakket (`homeassistant/packages/pompput.yaml`)

| Onderdeel | Wordt |
|---|---|
| `sensor` `history_stats` "Pompput pompstarts 24 uur" | **vervalt** (FR-010) |
| alert `pompput_pomp_start_niet` | **vervalt** (FR-002) |
| alert `pompput_pomp_pompt_niet_af` | **vervalt** |
| alert `pompput_pomp_draait_te_lang` | **vervalt** |
| alert `pompput_stroomsensor_storing` | **vervalt** |
| alerts `pompput_hoogwater`, `pompput_onbereikbaar`, `pompput_niet_ingesteld`, `pompput_niveausensor_storing` | blijven; de regel `Pomp: …` vervalt uit de tekst (FR-009) |
| tekst van `pompput_niet_ingesteld` | "vul Afstand leeg en Afstand vol in … Zonder deze waarden is er geen vulgraad." (research R3) |
| `input_boolean` `pompput_toon_instellingen`, template `Pompput onbereikbaar` | ongewijzigd |

Elk overgebleven alert noemt nog het soort melding, de vermoedelijke oorzaak, het niveau en
"sinds" (FR-009). Herinneringen (`repeat`), herstelmeldingen (`done_message`) en bevestigen
(`can_acknowledge`) blijven gelijk.

## 3. Dashboard (`homeassistant/dashboard-pompput.yaml`)

| Kaart | Vervalt | Blijft |
|---|---|---|
| Pompput (entities) | Pomp, Pompstarts 24 uur, Looptijd laatste pompbeurt | Vulgraad, Waterhoogte, Status, Laatste gebeurtenis |
| Laatste 24 uur (history-graph) | Pomp | Waterhoogte |
| Gebeurtenissen (logbook) | — (de titelregel noemt geen pompbeurten meer) | `event.pompput_gebeurtenis` |
| Bediening | — | Instellingen tonen |
| Instellingen | Inschakelniveau; de hele sectie "Pomp"; Offset stroomsensor en Pompstroom onder Diagnose | Put (Afstand leeg, Afstand vol), Hoogwater, Apparaat, Diagnose (Afstand ruw, WiFi-signaal, Uptime, Herstartreden) |

Na het herladen staan er geen regels "entiteit niet gevonden" op het dashboard (FR-013,
SC-004).

**Verweesde entiteiten**: blijft na de update en het herladen een vervallen entiteit als
"niet beschikbaar" of "niet meer geleverd" staan, dan verwijdert de gebruiker die eenmalig
onder Instellingen → Apparaten en diensten → Entiteiten (research R7).

## 4. Hardware

| XIAO-pin | GPIO | Was | Wordt |
|---|---|---|---|
| D1 / A1 | 3 | analoog in, ACS712 OUT via deler 10k/10k | **vrij**; niet geconfigureerd door de firmware (FR-012) |
| 5V | — | ook ACS712 VCC | HC-SR04 VCC en de leds |

- De substitution `pin_acs712` verdwijnt uit `esphome/pompput.yaml`; de pintabel in
  `hardware.md` en het blok blijven gelijk lopen (Embedded Platform Constraints).
- De paragraaf "ACS712, 30 A-variant (pompstroom), netspanning" in `hardware.md` wordt
  vervangen door **"Een bestaande stroomsensor weghalen"** (FR-015):
  1. Een nog aangesloten sensor mag blijven zitten: de firmware leest A1 niet meer, en de pin
     krijgt via de deler hoogstens ongeveer 2,5 V (research R5).
  2. Weghalen is werk aan netspanning en gebeurt door of onder toezicht van een vakbekwaam
     persoon, met de pompgroep spanningsloos.
  3. De fasedraad liep door de sensor (IP+ → IP−). Verbind de fase naar de pomp weer door,
     met een geschikte, geïsoleerde verbinding in een afgesloten behuizing. N en PE blijven
     ongewijzigd.
  4. Controleer daarna dat de pomp op zijn eigen vlotter aanslaat en afslaat (001 SC-007).
  5. Haal de deler en de draden naar A1, 5V en GND van de sensor weg, of laat ze los en
     geïsoleerd achter.
- De onderdelenlijst, het aansluitschema en de voeding in `README.md` noemen de ACS712 niet
  meer; A1 staat als vrij aangegeven (FR-014).

## 5. Signalering ter plaatse (led 1)

De tabel in [ledbalk.md §1](../../003-niveau-ledbalk/contracts/ledbalk.md#1-led-1-statuslampje)
wordt:

| Prio | Toestand | Kleur (R,G,B,W) | Patroon | Wijziging |
|---|---|---|---|---|
| 1 | Alarm (HOOGWATER actief) | rood (255,0,0,0) | 125 ms aan / 125 ms uit | — |
| 2 | Storing (niveausensor storing of niet ingesteld) | paars (160,0,255,0) | 500 ms aan / 1500 ms uit | was prio 3 |
| 3 | Opstarten | wit (0,0,0,255) | continu | was prio 4 |
| 4 | Niveau onbekend | paars (160,0,255,0) | 500 ms aan / 1500 ms uit | was prio 5 |
| 5 | Geen verbinding met Home Assistant | wit (0,0,0,255) | flits 100 ms, elke 2 s | was prio 6 |
| 6 | Normaal | blauw (0,0,255,0) | continu | was prio 8 |

- **Vervallen**: "Waarschuwing" (oranje, was prio 2) en "Pomp draait" (blauw 1 s / 1 s, was
  prio 7) (FR-005).
- De onderlinge volgorde van de overgebleven toestanden is ongewijzigd (FR-007). De minimale
  helderheid van 10 % bij een alarm blijft.
- De niveaubalk (led 2–6, ledbalk.md §2) is ongewijzigd. De voorbeelden in ledbalk.md met "pomp
  draait" vervallen.
