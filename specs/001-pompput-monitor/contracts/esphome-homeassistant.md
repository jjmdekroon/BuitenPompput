# Contract: ESPHome en Home Assistant

**Status**: ontwerp (2026-11-11). Vervangt `mqtt-homeassistant.md`. Dit beschrijft de externe
interface van het apparaat: de entiteiten die Home Assistant via de native API van ESPHome
ziet, hun gedrag, en het HA-pakket dat in deze repository staat. Gecontroleerd wordt dit met
[quickstart.md](../quickstart.md) §3 en §4; er zijn geen unittests (constitutie IV).

## Algemeen

| Onderwerp | Waarde |
|---|---|
| Koppeling | ESPHome native API, `encryption.key` uit `secrets.yaml`, alleen binnen het thuisnetwerk |
| Apparaat | `name: pompput`, `friendly_name: Pompput`, `project: diy.pompput`, versie `0.2.0` |
| Updates | ESPHome OTA, beveiligd met dezelfde sleutel; de eerste keer via USB |
| Beschikbaarheid | Is de API-verbinding weg, dan zet HA alle entiteiten op `unavailable` |
| Herstarten bij netwerkverlies | nee: `wifi: reboot_timeout: 0s` en `api: reboot_timeout: 0s` |
| Na (her)verbinden | HA krijgt de actuele state van alle entiteiten. Gebeurtenissen en metingen uit de periode zonder verbinding worden **niet** nagezonden (FR-010) |
| Onbekende waarde | een sensor met `NAN` wordt in HA "onbekend" |
| Entity-id's | `<domein>.pompput_<naam>`, door HA afgeleid van de apparaat- en de entiteitsnaam (research R4). De tabellen hieronder geven de verwachte id's |

## Sensoren (`sensor`)

| YAML-id | Naam | Entity-id | Eenheid / class | Publicatie | Spec |
|---|---|---|---|---|---|
| `waterhoogte` | Waterhoogte | `sensor.pompput_waterhoogte` | cm, `distance`, `measurement`, 1 decimaal | elke 10 s en direct bij een wijziging van de niveaustatus; `NAN` als de status ≠ GELDIG | FR-002, FR-003 |
| `vulgraad` | Vulgraad | `sensor.pompput_vulgraad` | %, `measurement`, 0 decimalen | zoals waterhoogte | FR-002, FR-003 |
| `looptijd_laatste_pompbeurt` | Looptijd laatste pompbeurt | `sensor.pompput_looptijd_laatste_pompbeurt` | s, `duration` | bij elke stop | FR-009 |
| `afstand_ruw` | Afstand (ruw) | `sensor.pompput_afstand_ruw` | cm, `distance`, diagnostic, 1 decimaal | elke 10 s; de laatste mediaan, of `NAN` | FR-026, kalibratie |
| `pompstroom` | Pompstroom | `sensor.pompput_pompstroom` | A, `current`, `measurement`, diagnostic, 2 decimalen | elke 10 s en bij een wijziging van de pompstatus | R7, afstellen drempel |
| `offset_stroomsensor` | Offset stroomsensor | `sensor.pompput_offset_stroomsensor` | V, `voltage`, diagnostic, 2 decimalen | elke 10 s | FR-016, bankverificatie |
| `wifi_signaal` | WiFi-signaal | `sensor.pompput_wifi_signaal` | dBm, `signal_strength`, diagnostic | elke 60 s | FR-026 |
| `uptime_s` | Uptime | `sensor.pompput_uptime` | s, `duration`, diagnostic | elke 60 s | FR-026 |

## Tekstsensoren (`text_sensor`; in HA in het domein `sensor`)

| YAML-id | Naam | Entity-id | Waarden | Publicatie | Spec |
|---|---|---|---|---|---|
| `status` | Status | `sensor.pompput_status` | `normaal`, `waarschuwing`, `alarm`, `storing`, `niet_ingesteld` | bij verandering, en bij het opstarten | FR-018 |
| `laatste_gebeurtenis` | Laatste gebeurtenis | `sensor.pompput_laatste_gebeurtenis` | tekst, bijvoorbeeld `pomp_stop: looptijd 95 s, niveau 31,5 cm` | bij elke gebeurtenis | FR-026 |
| `herstartreden` | Herstartreden | `sensor.pompput_herstartreden` | `reset_reason` van de component `debug` | bij het opstarten (diagnostic) | FR-026 |

## Binaire sensoren (`binary_sensor`)

| YAML-id | Naam | Entity-id | Device class | Spec |
|---|---|---|---|---|
| `pomp` | Pomp | `binary_sensor.pompput_pomp` | `running`; onbekend bij de pompstatus ONBEKEND | FR-008 |
| `hoogwater` | Hoogwater | `binary_sensor.pompput_hoogwater` | `problem` | FR-012 |
| `pomp_start_niet` | Pomp start niet | `binary_sensor.pompput_pomp_start_niet` | `problem` | FR-013 |
| `pomp_pompt_niet_af` | Pomp pompt niet af | `binary_sensor.pompput_pomp_pompt_niet_af` | `problem` | FR-014 |
| `pomp_draait_te_lang` | Pomp draait te lang | `binary_sensor.pompput_pomp_draait_te_lang` | `problem` | FR-015 |
| `niveausensor_storing` | Niveausensor storing | `binary_sensor.pompput_niveausensor_storing` | `problem` | FR-016 |
| `stroomsensor_storing` | Stroomsensor storing | `binary_sensor.pompput_stroomsensor_storing` | `problem` | FR-016 |
| `niet_ingesteld` | Niet ingesteld | `binary_sensor.pompput_niet_ingesteld` | `problem` | randgeval |

De meldingen worden bij elke verandering gepubliceerd, binnen de seconde-cyclus.

## Gebeurtenis (`event`)

- **Entiteit**: YAML-id `gebeurtenis`, naam "Gebeurtenis", entity-id `event.pompput_gebeurtenis`.
- **`event_types`**: `pomp_start`, `pomp_stop`, `melding_actief`, `melding_hersteld`, `herstart`,
  `instelling_afgewezen`, `niveaumeting_herstart`.
- **Details**: een ESPHome-event heeft geen extra velden. Tegelijk met elke gebeurtenis wordt
  "Laatste gebeurtenis" bijgewerkt (zie [data-model.md §6](../data-model.md#6-gebeurtenis-event)).
- **Oorzaken bij `niveaumeting_herstart`**: `knop`, `instelling` en `automatisch`. Die laatste
  betekent: opnieuw ingeleerd na een storing (research R6).
- Spec: FR-026.

## Instellingen (`number`; `entity_category: config`, `mode: box`)

| YAML-id | Naam | Entity-id | Eenheid | Bereik / stap |
|---|---|---|---|---|
| `afstand_leeg_cm` | Afstand leeg | `number.pompput_afstand_leeg` | cm | 10–400 / 0,1 |
| `afstand_vol_cm` | Afstand vol | `number.pompput_afstand_vol` | cm | 3–390 / 0,1 |
| `inschakelniveau_cm` | Inschakelniveau | `number.pompput_inschakelniveau` | cm | 0–400 / 0,1 |
| `alarmdrempel_pct` | Alarmdrempel | `number.pompput_alarmdrempel` | % | 10–120 / 1 |
| `hoogwater_wachttijd_s` | Wachttijd hoogwater | `number.pompput_wachttijd_hoogwater` | s | 0–600 / 1 |
| `start_marge_cm` | Marge pomp start niet | `number.pompput_marge_pomp_start_niet` | cm | 0–50 / 1 |
| `start_wachttijd_s` | Wachttijd pomp start niet | `number.pompput_wachttijd_pomp_start_niet` | s | 10–3600 / 1 |
| `afpomp_controletijd_s` | Controletijd afpompen | `number.pompput_controletijd_afpompen` | s | 30–3600 / 1 |
| `afpomp_min_daling_cm` | Minimale daling afpompen | `number.pompput_minimale_daling_afpompen` | cm | 0,5–50 / 0,1 |
| `max_looptijd_min` | Maximale looptijd pomp | `number.pompput_maximale_looptijd_pomp` | min | 1–240 / 1 |
| `sensorstoring_wachttijd_s` | Wachttijd sensorstoring | `number.pompput_wachttijd_sensorstoring` | s | 30–3600 / 1 |
| `herstelmarge_pct` | Herstelmarge | `number.pompput_herstelmarge` | % | 1–50 / 1 |
| `pomp_stroomdrempel_a` | Stroomdrempel pomp | `number.pompput_stroomdrempel_pomp` | A | 0,05–20 / 0,05 |
| `led_helderheid_pct` | Led-helderheid | `number.pompput_led_helderheid` | % | 0–100 / 1 |

**Gedrag** (FR-023, FR-024, het randgeval "ongeldige instellingen"):
- **Buiten het bereik**: ESPHome weigert de waarde zelf (logmelding), en de HA-interface
  laat zo'n waarde ook niet toe.
- **Geldig** (binnen het bereik en voldoet aan de regels uit data-model §1):
  - de waarde wordt overgenomen en de nieuwe state wordt gepubliceerd;
  - de waarde wordt bewaard in flash; ESPHome schrijft alleen gewijzigde waarden;
  - de waarde geldt binnen 1 s, ruim binnen FR-023 (1 minuut);
  - een wijziging van de geometrie start de niveaumeting opnieuw, met de gebeurtenis
    `niveaumeting_herstart` (oorzaak `instelling`).
- **Ongeldige combinatie**:
  - de oude waarde blijft staan en wordt opnieuw gepubliceerd, zodat HA terugspringt;
  - er volgt de gebeurtenis `instelling_afgewezen`, met de reden in "Laatste gebeurtenis";
  - de afgewezen waarde wordt **niet** bewaard: na een herstart staat de oude waarde er nog.
- **Niet gezet**: de drie verplichte instellingen tonen "onbekend" zolang ze niet gezet zijn.

## Bediening

| Domein | YAML-id | Naam | Entity-id | Gedrag | Spec |
|---|---|---|---|---|---|
| `button` | `niveaumeting_herstart` | Niveaumeting opnieuw starten | `button.pompput_niveaumeting_opnieuw_starten` | de niveaustatus gaat naar ONBEKEND en daarna wordt opnieuw ingeleerd; gebeurtenis `niveaumeting_herstart` (oorzaak `knop`); `entity_category: config` | R6 |

De schakelaar Zoemer en de knop Zoemer stil zijn vervallen met `specs/002-zoemer-verwijderen`.

## Home Assistant-pakket (`homeassistant/packages/pompput.yaml`)

Dit bestand staat in de repository en wordt in HA geladen via `packages`. Het bevat:

1. **`history_stats`** "Pompput pompstarts 24 uur": `binary_sensor.pompput_pomp`, state `on`,
   `type: count`, 24 uur terug (FR-009).
2. **Template-binary-sensor** "Pompput onbereikbaar" (`problem`): aan als
   `sensor.pompput_status` `unavailable` of `unknown` is, met `delay_on: 00:05:00` (FR-017).
3. **`alert`** voor elk van de zeven probleem-binary-sensors en voor "Pompput onbereikbaar":
   - `state: "on"`, `repeat: 60` minuten (FR-020), `can_acknowledge: true`, `skip_first: false`;
   - `title`: het soort melding, bijvoorbeeld "Alarm: hoogwater in pompput";
   - `message` (FR-019): de oorzaak in gewone taal, plus het niveau
     (`sensor.pompput_vulgraad` en `sensor.pompput_waterhoogte`), de pomp
     (`binary_sensor.pompput_pomp`) en "Sinds" (`relative_time` van `last_changed`);
   - `done_message` (FR-021): "Hersteld: <melding>";
   - `notifiers`: de notify-service van de gebruiker, in te vullen bij de installatie.
4. **`input_boolean.pompput_toon_instellingen`**: alleen voor het voorbeelddashboard.

De herinneringstermijn (`repeat`) en de wachttijd voor "onbereikbaar" (`delay_on`) stel je
in dit pakket in. Alle andere drempels zijn `number`-entiteiten van het apparaat.

## Opstartvolgorde (gezien vanuit HA)

1. Het apparaat start, meet en bewaakt direct, ook zonder WiFi (FR-025, FR-027).
2. WiFi verbindt, en HA maakt verbinding met de API. De entiteiten krijgen hun actuele state.
3. Drie seconden na de eerste verbinding na het opstarten volgt eenmalig de gebeurtenis
   `herstart`, met de reden in "Laatste gebeurtenis".
