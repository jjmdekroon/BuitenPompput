# Research: Pompput-monitor

**Feature**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md) | **Datum**: 2026-11-11
(herzien voor ESPHome; vervangt de versie van 2026-11-09 voor Arduino en MQTT)

Elke beslissing staat in het format Beslissing / Onderbouwing / Overwogen alternatieven.

- **Hardware van de gebruiker**: Seeed Studio XIAO ESP32C3, HC-SR04 (TRIG/ECHO),
  ACS712 30 A, een passieve piëzo-zoemer (PWM) en een adresseerbare RGBW-led (SK6812).
- **Eis van de gebruiker**: niveau-aanpassing voor sensoren gebeurt alleen met weerstanden.
- **De hardware is ongewijzigd** ten opzichte van de vorige versie van dit onderzoek. De
  hardwarebeslissingen R8 en R15 en de onderdeelkeuzes in R6, R7, R9 en R10 gelden dus nog.
  Alleen de firmware, de koppeling met Home Assistant en het testen veranderen.

**Bron voor ESPHome-details**: de broncode van ESPHome 2026.10.1, zoals geïnstalleerd in het
zusterproject Buitenlamp (`.venv/Lib/site-packages/esphome`). Dat is exact de versie die hier
wordt vastgepind.

---

## R1. Firmware-aanpak: ESPHome, alle logica in YAML

**Beslissing**:
- De firmware is één **ESPHome**-configuratie.
- Alle beslislogica staat in benoemde scripts, automatiseringen en template-entiteiten, met
  korte lambdas. Dat geldt voor niveaufilter, pompdetectie, meldingen, signalering en
  instellingenvalidatie.
- Er is geen eigen C++-component en er zijn geen unittests op de pc.
- Controle gebeurt op het apparaat, volgens [quickstart.md](quickstart.md).

**Onderbouwing**:
- Dit is het besluit van de beheerder (2026-11-11), vastgelegd in constitutie 2.0.0
  (principes I en IV).
- Het is dezelfde werkwijze als bij de Buitenlamp: één gereedschap en één manier van koppelen
  voor alle apparaten in huis.
- Updates kunnen via WiFi, en de koppeling met Home Assistant werkt zonder broker.

**Overwogen alternatieven**:
- **Eigen Arduino-firmware met MQTT** (de vorige versie van dit plan): werkend en getest op
  de pc, maar met een eigen gereedschap en een extra broker. Vervangen door het besluit van de
  beheerder; de code staat in de geschiedenis van deze sessie, niet in de repository.
- **ESPHome met een eigen C++-component** (de bestaande, geteste bibliotheek
  `lib/pompput_core`): afgewezen door de beheerder ten gunste van alles in YAML.
- **Alle logica in Home Assistant**: afgewezen, want de led en de zoemer moeten ook zonder
  Home Assistant werken (FR-027).

## R2. Gereedschap en build

**Beslissing**:
- **ESPHome 2026.10.1**, exact vastgepind in een `requirements.txt` in de root
  (`esphome==2026.10.1`), en geïnstalleerd in een eigen `.venv` van dit project.
- Configuratie:
  - `esp32: variant: esp32c3`, `framework: type: esp-idf`;
  - `esphome: min_version: 2026.10.1`.
- Commando's: `esphome config esphome/pompput.yaml` (validatie),
  `esphome compile esphome/pompput.yaml` (build en geheugenrapport) en
  `esphome run esphome/pompput.yaml` (flashen: de eerste keer via USB, daarna via WiFi).

**Onderbouwing**:
- Dezelfde versie als de Buitenlamp; die bouwt aantoonbaar op deze pc (build van
  2026-11-02).
- **ESPHome 2026.10 bouwt voor de ESP32 standaard met de eigen ESP-IDF-toolchain, zonder
  PlatformIO.** De build-map van de Buitenlamp bevat `CMakeLists.txt` en `sdkconfig`, en geen
  `platformio.ini`. In de broncode is de standaard voor de ESP32 `Toolchain.ESP_IDF`.
- Daardoor heeft de beschadigde PlatformIO-omgeving (`~/.platformio/penv`, zie de vorige
  verificatie) geen invloed op deze route.
- Een eigen `.venv` per project houdt de versie vast en los van de Buitenlamp.

**Overwogen alternatieven**:
- **De `.venv` van de Buitenlamp delen**: dan zitten de projecten aan elkaars versie vast.
- **Het ESPHome-dashboard (de add-on in Home Assistant)**: kan, maar dan zit de configuratie
  niet in deze repository en is de versie niet vastgepind.

## R3. Koppeling met Home Assistant

**Beslissing**:
- Home Assistant koppelt via de **native API van ESPHome**, met `encryption.key` uit
  `secrets.yaml`.
- Updates via **ESPHome OTA** (`ota: - platform: esphome` met `encryption:`). OTA gebruikt
  dezelfde sleutel; er is geen apart wachtwoord, net als bij de Buitenlamp.
- `api: reboot_timeout: 0s` en `wifi: reboot_timeout: 0s`.
- Er is geen MQTT, geen `web_server` en geen `captive_portal`.

**Onderbouwing**:
- De ESPHome-integratie in Home Assistant herkent het apparaat vanzelf. Entiteiten,
  beschikbaarheid en bediening komen zonder extra configuratie.
- Is de API-verbinding weg, dan zet Home Assistant alle entiteiten op "unavailable". Daar
  bouwt "bewaking niet bereikbaar" (FR-017) op.
- Met de herstarttimeouts op 0 s herstart het apparaat niet bij netwerkverlies. Een actief
  alarm en de lokale signalering blijven dus staan (constitutie, Independent operation).

**Overwogen alternatieven**: MQTT (R1); de REST-API van Home Assistant (dan moet er een token
in het apparaat, moet het pollen, en is er geen beschikbaarheid).

## R4. Entiteiten en entity-id's

**Beslissing**:
- Het apparaat heet `name: pompput`, `friendly_name: Pompput`.
- Home Assistant maakt de entity-id's uit de apparaatnaam plus de entiteitsnaam:
  `<domein>.pompput_<naam in kleine letters, met underscores>`.
- De namen worden zo gekozen dat de id's stabiel en voorspelbaar zijn. Het overzicht staat in
  [contracts/esphome-homeassistant.md](contracts/esphome-homeassistant.md).
- Het HA-pakket en het dashboard gebruiken precies die id's.

**Onderbouwing**:
- In ESPHome 2026.10 bepaalt de naam de entity-id; een losse `object_id` is er niet meer.
- De meldingen houden hun oude id's: bijvoorbeeld "Niveausensor storing" wordt
  `binary_sensor.pompput_niveausensor_storing`. Zo blijven de `alert`-regels in het pakket
  gelijk.

## R5. Waar de meldingslogica draait

**Beslissing**:
- Ongewijzigd: het **apparaat** bepaalt alle zeven meldingen, en publiceert elke melding als
  template-`binary_sensor` (`device_class: problem`).
- Home Assistant zorgt voor de aflevering op afstand:
  - `alert` voor herinneringen (FR-020) en herstelmeldingen (FR-021);
  - een template-binary-sensor met `delay_on` voor "niet bereikbaar" (FR-017).
- **Uitvoering in ESPHome**:
  - Elke melding heeft een eigen beoordelingsscript dat de activatie- en herstelvoorwaarde
    bepaalt.
  - Eén gemeenschappelijk script met parameters doet de levenscyclus
    `INACTIEF → IN_AFWACHTING → ACTIEF → INACTIEF`. Dat script heeft een wachttijd voor
    activeren, een optionele wachttijd voor herstellen, "niet beoordelen" (vasthouden) en
    "onderdrukken".
  - De toestanden staan in `globals`.
  - Het geheel draait in `interval: 1s`.

**Onderbouwing**:
- De led en de zoemer hebben de meldingen lokaal nodig.
- Eén levenscyclus-script voorkomt dat dezelfde logica zeven keer in de YAML staat
  (constitutie IV).
- `millis()` met verschil-rekenen is overloopveilig (constitutie II).

**Overwogen alternatieven**: zeven losse kopieën van de levenscyclus (dubbele logica);
ESPHome-filters `delayed_on`/`delayed_off` op de template-binary-sensors (kunnen geen
vasthouden en geen voorwaarde "niet beoordelen" uitdrukken).

## R6. Waterniveau met de HC-SR04

**Beslissing**:
- **Meten**: de ESPHome-component **`ultrasonic`**.
  - `update_interval: 250ms`, `timeout: 4.2m`, intern (niet zichtbaar in HA).
  - Geen echo levert `NAN`.
- **Verwerken**: elke ruwe meting (meters of `NAN`) gaat via `on_raw_value`, omgerekend naar cm
  met `${m_naar_cm}`, in een ringbuffer van 5 (`globals`, `std::array<float, 5>`). Elke seconde
  doet het script `niveau_verwerken` het volgende:
  - het **bereik** controleren: geldig van 3 cm tot `afstand_leeg_cm + 20` cm, en zonder
    instelling tot 420 cm;
  - de **mediaan van de laatste 5**, alleen als er minstens 3 geldig zijn;
  - de **snelheidscontrole**: |Δ| ≤ max(3 cm, 2 cm/s × Δt), met Δt maximaal 5 s;
  - het **inleren**: 5 opeenvolgende medianen binnen 2 cm van elkaar;
  - de **status**: VEROUDERD na 30 s, STORING na `sensorstoring_wachttijd_s`. Ook zonder
    eerste geldige waarde volgt STORING na die tijd (FR-016);
  - het **opnieuw inleren in STORING** (nieuw, 2026-11-11): naast de snelheidscontrole loopt
    in STORING ook een inleerreeks. Zijn er 5 opeenvolgende medianen binnen 2 cm, dan wordt de
    laatste de nieuwe referentie (GELDIG), met de gebeurtenis `niveaumeting_herstart` (oorzaak
    `automatisch`).
- De overige rekenregels zijn ongewijzigd ten opzichte van de vorige versie; alleen de
  uitvoering is nu YAML.

**Waarom opnieuw inleren in STORING** (uit `/speckit-analyze`, bevinding C1):
- De snelheidscontrole rekent met hoogstens 5 s, dus een nieuwe mediaan mag hoogstens 10 cm
  van de referentie afwijken. Is het niveau tijdens een blinde periode (overstroomd,
  afgedekt) meer veranderd, dan zou elke nieuwe meting worden afgewezen. Het apparaat bleef
  dan in storing tot iemand op de knop drukte of het herstartte, en herkende zolang geen
  hoogwater; een vastgehouden hoogwater-alarm bleef voorgoed staan.
- **Prijs**: een hindernis die langer dan `sensorstoring_wachttijd_s` stil blijft hangen, wordt
  daarna als niveau gezien en kan een onterecht hoogwater-alarm geven. Voor een bewaking tegen
  overlopen is een onterecht alarm beter dan een gemist alarm (spec, randgeval "Vervuilde of
  beslagen afstandsmeter"). Hetzelfde geldt al bij het inleren na een herstart of reset.
- **Afgewezen alternatief**: in storing blijven tot een handmatige reset, met een aanwijzing in
  de HA-melding. Dat laat de put onbewaakt tot iemand ingrijpt.

**Onderbouwing**:
- **De `ultrasonic`-component is in deze versie interrupt-gestuurd en niet-blokkerend.** De
  bron start de puls en meet de echo met een GPIO-interrupt; er is geen `pulseIn`. Een timeout
  van 4,2 m kost dus geen lustijd. Het bereik wordt in de verwerking per instelling begrensd.
- Het ingebouwde `median`-filter kent geen "minimaal 3 geldig" en geen snelheidscontrole. FR-004
  en FR-005 vragen dat wel, dus is een eigen, korte berekening nodig.

**Risico's** (ongewijzigd): de HC-SR04 is niet waterdicht (een JSN-SR04T past op dezelfde
pinnen); de bundel is ongeveer 15°; het vol-niveau ligt minstens 20 cm onder de sensor.

## R7. Pompstroom met de ACS712

**Beslissing**:
- De hardware is ongewijzigd: 30 A-variant, in serie met de fase, en een deler van
  10 kΩ / 10 kΩ naar A1 (GPIO3, ADC1).
- **Eén `adc`-sensor** op GPIO3 met `attenuation: 12db`, intern. Die heeft twee rollen:
  - **bron** voor een **`ct_clamp`**-sensor (`sample_duration: 100ms`,
    `update_interval: 500ms`), die de RMS van het AC-deel berekent. Een filter
    `multiply: 30.303` (= 1 / (0,066 V/A × 0,5)) zet dat om in ampère;
  - een eigen **gepubliceerde spanning** (`update_interval: 100ms`, filter
    `sliding_window_moving_average` over 10 waarden, elke 5 publiceren). Dat is de
    **DC-offset** voor de storingscontrole (0,9–1,6 V).
- **Pompstatus**: het script `pomp_bijwerken` draait bij elke stroomwaarde. Het voert de
  toestandsmachine STIL / START_KANDIDAAT / DRAAIT / STOP_KANDIDAAT / ONBEKEND uit, met een
  debounce van 2 s en hysterese (uit onder 70 % van de drempel). De regels zijn ongewijzigd.

**Onderbouwing**:
- **`ct_clamp` meet niet-blokkerend.** `update()` start alleen de meetperiode. In `loop()`
  komt er per doorloop één sample bij, met een verhoogde lusfrequentie; na de meetperiode
  volgt de berekening RMS² = gem(x²) − gem(x)². Er is dus geen begrensd blokkerend venster
  meer, zoals in de Arduino-versie.
- `ct_clamp` geeft de offset niet door. Een aparte, gemiddelde aflezing van **dezelfde**
  `adc`-sensor geeft die wel, zonder de pin te delen.
  - Het gemiddelde over 10 aflezingen op willekeurige fase middelt de 50 Hz-golf
    grotendeels weg.
  - Bij 3 A is de amplitude op de pin ±0,14 V. De fout van het gemiddelde is dan ongeveer
    ±0,05 V, ruim binnen de marge van 1,25 V ± 0,35 V.
- `11db` is in ESPHome verouderd; `12db` is de juiste instelling (0 tot ongeveer 2,5 V op de
  ESP32-C3).
- **"Onmogelijke waarde" van de stroommeting (FR-016)** is een offset buiten 0,9–1,6 V. Een
  bovengrens voor de stroom voegt niets toe: de ACS712-30A geraakt rond 30 A piek (ongeveer
  21 A RMS) in verzadiging, en na de deler blijft het signaal binnen het bereik van de ADC.
  Een veel te hoge RMS-waarde kan dus niet ontstaan; een defecte of losse sensor verschuift
  wel de offset.

**Overwogen alternatieven**:
- Twee `adc`-sensoren op dezelfde pin: dan moet de pin gedeeld worden en moet ADC1 twee keer
  worden opgezet. Onnodig.
- Piekdetectie: gevoelig voor ruis.

**Veiligheid en het risico voor FR-007**: ongewijzigd. De ACS712 staat op netspanning en in
serie met de pomp. Een stroomtang (CT) blijft het aanbevolen alternatief, zie
[contracts/hardware.md](contracts/hardware.md).

## R8. Niveau-aanpassing alleen met weerstanden

**Beslissing**: ongewijzigd.
- ECHO via 1 kΩ / 2 kΩ.
- ACS712 OUT via 10 kΩ / 10 kΩ.
- TRIG rechtstreeks op 3,3 V (een bankverificatie).
- De led-data via 330 Ω, met de voeding van de led via een diode.

Zie [contracts/hardware.md](contracts/hardware.md).

## R9. Statusled (RGBW)

**Beslissing**:
- **`esp32_rmt_led_strip`**:
  - `chipset: SK6812`, `num_leds: 1`;
  - `channel_colors` voor de kleurvolgorde (GRBW, op de bank controleren);
  - `use_psram: false`, `restore_mode: ALWAYS_OFF`;
  - `internal: true`, zodat de led niet als lamp in Home Assistant verschijnt.
- **Eén generiek script `led_knipperen`** (`mode: restart`) speelt elk led-patroon af.
  - Parameters: `rood`, `groen`, `blauw`, `wit` (0–1), `aan_ms`, `uit_ms` en `helderheid`
    (0–1).
  - Met `uit_ms` = 0 gaat de led continu aan (opstarten, pomp draait).
  - Anders loopt een `while`-lus: `light.turn_on` met kleur en helderheid, `delay: aan_ms`,
    `light.turn_off`, `delay: uit_ms`.
  - Kleuren en tijden komen uit [contracts/lokale-signalering.md](contracts/lokale-signalering.md).
- Het script `led_patroon_kiezen` kiest het patroon volgens de prioriteit, en start
  `led_knipperen` opnieuw als het patroon of de helderheid verandert.
  - Helderheid: 0 % geeft uit, behalve bij een alarm (dan 10 %).
  - Het draait bij elke statuswijziging, bij elke wijziging van de verbinding of de pomp,
    bij een gewijzigde Led-helderheid, en in de seconde-cyclus.
- Het script `led_bevestiging` doet de 2 witte flitsen bij "stil", en roept daarna
  `led_patroon_kiezen` aan.

**Onderbouwing**:
- Een `while`-lus met `delay` blokkeert de lus niet, en volgt de helderheid die de
  gebruiker instelt.
- `restore_mode: ALWAYS_OFF` wordt expliciet gezet, omdat de standaard in 2026.10.0 is
  veranderd (zie de Buitenlamp).

**Overwogen alternatieven**:
- **`strobe`-effecten** (een eerdere keuze, bijgesteld op 2026-11-11). Elke stap van het
  effect zet zijn eigen, vaste helderheid via `from_light_color_values` (broncode
  `base_light_effects.h`), dus de instelling Led-helderheid zou niet werken (FR-028).
- Een `interval: 20ms` met een lambda die de kleur berekent: een eigen timer en meer code.
- `status_led`: kan maar één kleur.

## R10. Zoemer: passieve piëzo via PWM

**Beslissing**:
- Een **`ledc`**-output op GPIO7 met `frequency: 2700Hz`. Toon aan is `output.set_level: 50%`,
  toon uit is `output.turn_off`.
- **De patronen zijn twee generieke scripts** met de niet-blokkerende acties `delay`,
  `repeat` en `while`:
  - **`zoemer_herhalen`** (`mode: restart`; parameters `aantal`, `aan_ms`, `uit_ms`,
    `periode_ms`):
    - alarm: 3 keer 150 ms aan / 150 ms uit, elke 10 000 ms;
    - waarschuwing: 1 keer 300 ms, elke 60 000 ms.
  - **`zoemer_eenmalig`** (`mode: queued`; parameters `aantal`, `aan_ms`, `uit_ms`):
    - opstarten: 1 keer 100 ms;
    - storing: 2 keer 100 ms, met 100 ms ertussen;
    - hersteld: 1 keer 50 ms;
    - bevestiging: 1 keer 50 ms.
  - `zoemer_eenmalig` stopt eerst `zoemer_herhalen` en roept na afloop
    `zoemer_patroon_kiezen` aan, zodat een herhalend patroon daarna gewoon doorgaat.
- Het script `zoemer_patroon_kiezen` start of stopt het herhalende patroon op basis van de
  hoogste ernst, de stilzet-set (FR-028) en de schakelaar "Zoemer".
- De BOOT-knop is een `binary_sensor` van het type `gpio` op GPIO9: `inverted`, `pullup`,
  filter `delayed_on: 50ms`, en `ignore_strapping_warning: true`, omdat GPIO9 een
  strapping-pin is.

**Onderbouwing**:
- LEDC maakt de toon in hardware.
- Scripts met `delay` blokkeren de lus niet. In de Arduino-versie werd de zoemer tijdens het
  stroomvenster niet bijgewerkt; dat probleem verdwijnt hier.

**Overwogen alternatieven**: `rtttl`, dat voor melodieën bedoeld is (de patronen zijn
eenvoudige piepjes, scripts zijn duidelijker); `output: gpio` (geeft geen toon).

## R11. Instellingen: Home Assistant, met een kopie in flash

**Beslissing**: twee soorten instellingen.
- **Instellingen met onderlinge regels.** Dat zijn `afstand_leeg_cm`, `afstand_vol_cm`,
  `inschakelniveau_cm`, `alarmdrempel_pct` en `herstelmarge_pct`.
  - De geaccepteerde waarde staat in een `globals` met `restore_value: true`. Het
    startpunt van de drie verplichte waarden is `NAN`, dus "niet gezet".
  - Het template-`number` toont die waarde via een `lambda`. Een nieuwe waarde gaat via
    `set_action` naar het script `instelling_toepassen`.
  - Is de waarde geldig, dan wordt de global bijgewerkt en de nieuwe state gepubliceerd. Gaat
    het om de geometrie, dan start de niveaumeting opnieuw.
  - Is de waarde ongeldig, dan wordt de oude state opnieuw gepubliceerd. Daarbij komt de
  gebeurtenis `instelling_afgewezen`, met de reden in "Laatste gebeurtenis".
- **Zelfstandige instellingen.** Dat zijn de overige negen getallen.
  - Een template-`number` met `optimistic: true` en `restore_value: true`.
  - De schakelaar `zoemer` is een template-`switch` met `optimistic: true` en
    `restore_mode: RESTORE_DEFAULT_ON`.
- `preferences: flash_write_interval` is kort (500 ms, net als bij de Buitenlamp). ESPHome
  schrijft alleen gewijzigde waarden.

**Onderbouwing** (gecontroleerd in de broncode van 2026.10.1):
- Het bereik (min/max) controleert ESPHome zelf. `NumberCall` weigert een waarde buiten het
  bereik met een logmelding, en roept `control()` dan niet aan.
- **Maar `TemplateNumber::control()` slaat met `restore_value` de gevráágde waarde op, ook
  als de `set_action` hem weigert.** Een afgewezen combinatie zou dan na een herstart
  terugkomen, en dat is in strijd met het randgeval "ongeldige instellingen".
- **Zonder opgeslagen waarde en zonder `initial_value` begint `restore_value` op `min_value`,
  niet op "niet gezet".** Dan kan NIET_INGESTELD niet werken.
- Daarom bewaren de vijf instellingen met regels hun geaccepteerde waarde zelf in een
  `globals`. Een `NAN` uit de lambda geeft in Home Assistant "onbekend" (niet gezet).

**Overwogen alternatieven**:
- Alle 14 instellingen via `globals`: meer YAML voor geen winst.
- Een schildwachtwaarde (bijvoorbeeld 0 = niet gezet): verwarrend in Home Assistant.

## R12. Tijdsgedrag en robuustheid

**Beslissing**:
- De componenten zijn niet-blokkerend:
  - `ultrasonic` werkt met een interrupt;
  - `ct_clamp` meet in `loop()`;
  - de led- en zoemerpatronen zijn scripts met `delay`;
  - `interval: 1s` doet de verwerking (niveau, meldingen en status).
- Lambdas zijn kort, en de lus doet geen heap-allocatie. De ringbuffers zijn vaste
  `std::array`-globals.
- De taak-watchdog van ESP-IDF blijft aan, zoals standaard in ESPHome.
- **Veilige start**:
  - de led heeft `restore_mode: ALWAYS_OFF`;
  - de `ledc`-output is standaard uit;
  - TRIG wordt door de `ultrasonic`-component als uitgang gezet;
  - `on_boot` start de opstartpiep en de witte led.
- Na een herstart beginnen alle meldingen bij INACTIEF; stil is opgeheven.

**Onderbouwing**: constitutie II en III. De ESPHome-lus doet geen blokkerende netwerkaanroep;
de API en WiFi draaien in de taken van ESP-IDF.

## R13. Diagnose en waarneembaarheid (FR-026)

**Beslissing**:
- Een **template-`event`** "Gebeurtenis" met de types `pomp_start`, `pomp_stop`,
  `melding_actief`, `melding_hersteld`, `herstart`, `instelling_afgewezen` en
  `niveaumeting_herstart`.
- Een **template-`text_sensor` "Laatste gebeurtenis"** met de details in gewone tekst,
  bijvoorbeeld `melding_actief: hoogwater, niveau 68,0 cm, pomp stil` of
  `instelling_afgewezen: afstand_vol_cm = 115: leeg moet minstens 10 cm groter zijn dan vol`.
- De binary sensors van de meldingen en de pomp staan zelf ook in het logboek van Home
  Assistant.
- **Diagnostische sensoren**:
  - Afstand (ruw) en Pompstroom;
  - Offset stroomsensor (nieuw: maakt de ACS712-storing en de bankverificatie zichtbaar);
  - WiFi-signaal (`wifi_signal`), Uptime (`uptime`), en Herstartreden (`debug`,
    `reset_reason`).
- `logger: level: INFO` in de opgeleverde firmware.

**Onderbouwing**:
- **De ESPHome-`event` draagt alleen een `event_type`, geen extra velden** (het schema van
  `event.trigger` kent alleen `event_type`). De details uit data-model §6 komen daarom in de
  tekstsensor.
- Er wordt alleen bij een gebeurtenis een tekst opgebouwd, niet per lusdoorloop of per meting
  (constitutie II).
- Zonder verbinding gaat een gebeurtenis verloren (FR-010).

## R14. Home Assistant-pakket

**Beslissing**: blijft `homeassistant/packages/pompput.yaml`, met:
- `history_stats` (pompstarts in 24 uur);
- de template-binary-sensor "Pompput onbereikbaar" (5 minuten `delay_on` op
  `sensor.pompput_status` met `unavailable` of `unknown`);
- 8 `alert`-regels (de 7 meldingen plus "onbereikbaar");
- de helper voor het dashboard.

Alleen de entity-id's die anders worden (zie R4) worden bijgewerkt. Het voorbeelddashboard
gaat mee.

## R15. Voeding, netwerk, behuizing en updates

**Beslissing**:
- Voeding en behuizing zijn ongewijzigd: een 5 V USB-C-adapter van minimaal 1 A, op een groep
  die niet uitvalt met de pomp, en een IP65-behuizing.
- De WiFi- en API-gegevens staan in `esphome/secrets.yaml`, dat in `.gitignore` staat;
  `secrets.example.yaml` staat in de repository.
- **Updates via WiFi (OTA) vallen nu binnen deze versie.** Bij de Arduino-versie vielen ze
  erbuiten.

## R16. De vorige Arduino/MQTT-implementatie

**Beslissing**:
- `platformio.ini`, `include/`, `lib/`, `src/` en `test/` horen niet bij de nieuwe opzet.
  Ze worden in de nieuwe takenlijst verwijderd.
- `README.md` wordt herschreven, maar het hoofdstuk "Aansluiten" blijft, want de hardware is
  ongewijzigd.
- `verificatie.md` beschrijft de vorige implementatie. Het wordt bij de nieuwe implementatie
  vervangen.

**Onderbouwing**: constitutie 2.0.0, principe I. De rekenregels uit de vorige versie (filter,
pompdetectie en meldingen) zijn overgenomen in [data-model.md](data-model.md); alleen de
uitvoering verandert.

---

## Open punten

1. **Bankverificatie**, vóór de definitieve installatie. Dit staat nog open:
   - TRIG op 3,3 V (R8);
   - de ADC van de ESP32-C3 met `12db`, en het verschil tussen de ACS712 in rust en bij een
     draaiende pomp (R7);
   - de led-data op 3,3 V met de diodevoeding (R8);
   - de kleurvolgorde van de led (R9).
2. **Rekentijd van de lambdas** (constitutie II: maximaal 10 ms per lambda). De logmelding
   "took a long time" van ESPHome 2026.10.1 komt pas boven 50 ms
   (`WARN_IF_BLOCKING_OVER_CS = 5` in `core/component.h`), en is dus niet genoeg. De rekentijd
   wordt apart gemeten met `micros()` rond de scripts (tasks T039). De mediaan van 5 waarden
   en de meldingen van 7 slots zijn klein; een probleem wordt niet verwacht.
