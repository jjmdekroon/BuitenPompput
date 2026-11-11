---

description: "Takenlijst voor de implementatie van de pompput-monitor (ESPHome)"
---

# Tasks: Pompput-monitor

**Input**: Ontwerpdocumenten uit `specs/001-pompput-monitor/` (herzien op 2026-11-11 voor ESPHome).

**Prerequisites**:
- [plan.md](plan.md) en [spec.md](spec.md);
- [research.md](research.md) en [data-model.md](data-model.md);
- [contracts/hardware.md](contracts/hardware.md),
  [contracts/esphome-homeassistant.md](contracts/esphome-homeassistant.md) en
  [contracts/lokale-signalering.md](contracts/lokale-signalering.md);
- [quickstart.md](quickstart.md);
- constitutie 2.0.0 (`.specify/memory/constitution.md`).

**Tests**: Er zijn **geen** geautomatiseerde tests (constitutie 2.0.0, principe IV). Elke fase
eindigt met:
- de **compile-gate**: `.venv\Scripts\esphome config esphome/pompput.yaml` en
  `.venv\Scripts\esphome compile esphome/pompput.yaml`, zonder fouten en zonder
  waarschuwingen uit de eigen lambdas;
- een **controletaak op het apparaat**, volgens [quickstart.md](quickstart.md). De uitkomst
  wordt vastgelegd in `specs/001-pompput-monitor/verificatie.md`. Een scenario dat niet op
  hardware is uitgevoerd, wordt daar met de reden vastgelegd (gate 4).

**Organization**: De taken zijn per user story gegroepeerd, zodat elke story los te bouwen en
te controleren is.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Kan parallel lopen (ander bestand, geen afhankelijkheid van een onafgemaakte taak)
- **[Story]**: De user story waar de taak bij hoort (US1–US4)
- Elke beschrijving noemt het exacte bestand. "Sectie N" verwijst naar de genummerde secties
  van `esphome/pompput.yaml` uit [plan.md](plan.md) → Project Structure.

## Conventies voor `esphome/pompput.yaml`

Deze gelden voor elke taak.
- **Eén bestand**, in 8 secties. Elke sectie begint met een kopcommentaar
  `# ---- N. <naam> ----`.
- **Geen magische getallen** in scripts of lambdas. Elke pin, tijd, grens en constante komt uit
  het `substitutions:`-blok (sectie 1, constitutie V). Ook omrekeningen hebben een naam
  (`ms_per_s`, `ms_per_min`, `pct_schaal`). Alleen 0 (geen wachttijd, beginwaarde),
  `true`/`false`, en de rekenkundige 1 en 2 (ophogen, een bit maken, de mediaan en het midden
  van twee waarden) mogen letterlijk.
- **Indeling**: ESPHome staat een sleutel als `sensor:` maar één keer toe. Sectie 1 en 2 staan
  daarom bovenaan, en de secties 3–8 zijn elk een pakket in hetzelfde bestand
  (`packages: niveau: …`); ESPHome voegt hun lijsten samen. Een id komt maar in één sectie
  voor.
- **Elk script, elke automatisering en elke template-entiteit** die een requirement uitvoert,
  krijgt een commentaar met de FR-nummers, bijvoorbeeld `# FR-012, FR-022` (constitutie IV).
- **Lambdas** doen één beslissing of één berekening, en duren maximaal 10 ms.
  - Geen `delay()`; wachten gaat met de acties `delay` en `while`.
  - Tijd: `millis()` met `(uint32_t)(now - t) >= d`.
  - Geen `new`/`malloc`. Geen `std::string`-opbouw in code die per lusdoorloop of per
    meting draait; alleen bij gebeurtenissen.
- **Codes in `globals`** hebben een naam in sectie 1 (T008; constitutie II en V). Scripts en
  lambdas gebruiken alleen die namen, nooit het getal. Bijvoorbeeld
  `id(g_pomp_status) == ${pomp_draait}` en `std::array<uint8_t, ${aantal_meldingen}>`. Waar een
  taak een toestand in hoofdletters noemt (GELDIG, DRAAIT, ACTIEF), is de naam uit dit
  overzicht bedoeld:
  - niveaustatus: `niveau_onbekend` 0, `niveau_geldig` 1, `niveau_verouderd` 2, `niveau_storing` 3;
  - pompstatus: `pomp_onbekend` 0, `pomp_stil` 1, `pomp_start_kandidaat` 2, `pomp_draait` 3,
    `pomp_stop_kandidaat` 4;
  - meldingfase: `fase_inactief` 0, `fase_in_afwachting` 1, `fase_actief` 2.
- **Meldingsnummer** (de volgorde van [data-model.md §5](data-model.md#5-melding-alert), ook het
  bitnummer in `g_actieve_set`). In de YAML altijd via de naam. De bitgroepen zijn
  `bits_alarm` (bit 0), `bits_waarschuwing` (bits 1–3) en `bits_storing` (bits 4–6).

  | Nummer | Naam | Melding | Ernst |
  |---|---|---|---|
  | 0 | `idx_hoogwater` | `hoogwater` | ALARM |
  | 1 | `idx_pomp_start_niet` | `pomp_start_niet` | WAARSCHUWING |
  | 2 | `idx_pomp_pompt_niet_af` | `pomp_pompt_niet_af` | WAARSCHUWING |
  | 3 | `idx_pomp_draait_te_lang` | `pomp_draait_te_lang` | WAARSCHUWING |
  | 4 | `idx_niveausensor_storing` | `niveausensor_storing` | STORING |
  | 5 | `idx_stroomsensor_storing` | `stroomsensor_storing` | STORING |
  | 6 | `idx_niet_ingesteld` | `niet_ingesteld` | STORING |

- **Namen en YAML-id's van entiteiten**: precies volgens
  [contracts/esphome-homeassistant.md](contracts/esphome-homeassistant.md), zodat de entity-id's
  in Home Assistant kloppen.
- **Referentie**: het zusterproject `C:\Git\ArduinoProjects\Buitenlamp\esphome\buitenlamp.yaml`
  (ESPHome 2026.10.1, esp-idf) laat de werkende syntax zien, bijvoorbeeld voor `ota`,
  `esp32_rmt_led_strip` en `api`.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Het nieuwe project neerzetten en de vorige implementatie opruimen

- [X] T001 [P] Maak `requirements.txt` in de root met precies één regel `esphome==2026.10.1`, plus een commentaarregel dat de versie bewust is vastgepind (constitutie I, research R2).
- [X] T002 [P] Herschrijf `.gitignore` in de root.
  - Weg: de PlatformIO-regels (`.pio/`, `.vscode/.browse*`, `.vscode/ipch`, `.vscode/c_cpp_properties.json`, `.vscode/launch.json`) en `include/secrets.h`.
  - Erbij: `.venv/`, `esphome/.esphome/` en `esphome/secrets.yaml`.
  - Houd: `.DS_Store`, `Thumbs.db`, `*.tmp`, `*.swp` en `*.log`.
- [X] T003 [P] Maak `esphome/secrets.example.yaml` met `wifi_ssid: "mijn-wifi"`, `wifi_password: "geheim"` en `api_encryption_key: "genereer-met-het-commando-hieronder"`.
  - Commentaar in het bestand: kopieer naar `esphome/secrets.yaml` en commit dat nooit; WiFi alleen op 2,4 GHz.
  - Commentaar: de sleutel beveiligt zowel de API als de updates via WiFi.
  - Commentaar met het commando: `python -c "import secrets,base64;print(base64.b64encode(secrets.token_bytes(32)).decode())"`.
- [X] T004 Maak `esphome/secrets.yaml` als kopie van `esphome/secrets.example.yaml` (hangt af van T003).
  - Vul `api_encryption_key` met een nieuw gegenereerde sleutel volgens het commando uit T003. Toon de sleutel **niet** in de uitvoer.
  - Laat `wifi_ssid` en `wifi_password` als voorbeeldwaarden staan, en meld de beheerder dat hij die zelf invult.
  - Controleer met `git check-ignore esphome/secrets.yaml` dat het bestand genegeerd wordt.
- [X] T005 Maak de virtuele omgeving `.venv/` en installeer ESPHome uit `requirements.txt` (hangt af van T001).
  - `python -m venv .venv`, daarna `.venv\Scripts\python -m pip install -r requirements.txt`.
  - Controleer dat `.venv\Scripts\esphome version` `Version: 2026.10.1` meldt.
- [X] T006 [P] Vervang `specs/001-pompput-monitor/verificatie.md` door een nieuwe opzet voor de ESPHome-implementatie.
  - Een samenvattingstabel van de gates 1–5 uit de constitutie, met de status "open".
  - Een sectie per controletaak (T019, T025, T027, T032, T036, T039–T043), met een tabel: scenario, uitgevoerd ja/nee, resultaat, en de reden als het niet is uitgevoerd.
  - Een sectie "Afwijkingen", en een sectie "Open bankpunten". Die laatste komt uit research ("Open punten" 1): TRIG op 3,3 V, de ADC met 12 dB en de ACS712 in rust, de led-data met de diode, en de kleurvolgorde.
- [X] T007 Verwijder de vorige Arduino/MQTT-implementatie: `platformio.ini`, `include/`, `lib/`, `src/`, `test/`, en `.pio/` als die bestaat (research R16).
  - **Vraag de beheerder eerst** of de huidige stand gecommit moet worden. Die staat nog niet in git, en de kopie in de scratchpad van de vorige sessie is tijdelijk.
  - Voer de verwijdering pas uit na diens antwoord.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: De basis die alle stories nodig hebben. Dat zijn:
- de configuratie, het netwerk en OTA;
- de instellingen en hun validatie;
- het meldingsframework met NIET_INGESTELD en de apparaatstatus;
- de gebeurtenissen;
- de led en de zoemer;
- de opstartvolgorde.

**⚠️ CRITICAL**: Werk aan user stories pas als deze fase af is

- [X] T008 Maak `esphome/pompput.yaml` met een kopcommentaar en **sectie 1**, het `substitutions:`-blok.
  - **Kopcommentaar**: doel van het apparaat, een verwijzing naar `specs/001-pompput-monitor/`, de 8 secties, en "pinnen volgen `contracts/hardware.md`".
  - **Apparaat**:
    - `device_name: pompput`, `friendly_name: Pompput`, `project_version: "0.2.0"`;
    - `log_level: INFO`, `flash_write_interval: 500ms`.
  - **Pinnen** (letterlijk uit `contracts/hardware.md`):
    - `pin_acs712: GPIO3`, `pin_trig: GPIO4`, `pin_echo: GPIO5`;
    - `pin_led: GPIO6`, `pin_zoemer: GPIO7`, `pin_knop: GPIO9`.
  - **Niveau** (data-model §1, "Vaste constanten"):
    - meten: `ping_interval: 250ms`, `ultrasoon_timeout: 4.2m`, `m_naar_cm: "100.0"`;
    - mediaan: `mediaan_venster: "5"`, `mediaan_min_geldig: "3"`;
    - bereik: `min_afstand_cm: "3.0"`, `bereik_extra_cm: "20.0"`;
    - snelheid: `max_niveausnelheid_cm_s: "2.0"`, `snelheid_dt_max_s: "5.0"`, `snelheid_min_tolerantie_cm: "3.0"`;
    - inleren: `inleer_aantal: "5"`, `inleer_spreiding_cm: "2.0"`;
    - tijden: `niveau_verouderd_ms: "30000"`, `publicatie_interval: 10s`.
  - **Pomp**:
    - meten: `adc_interval: 100ms`, `stroom_venster: 100ms`, `stroom_interval: 500ms`;
    - offset-gemiddelde: `offset_venster: "10"`, `offset_elke: "5"`;
    - omrekening en drempels: `acs712_a_per_v: "30.303"`, `pomp_debounce_ms: "2000"`, `pomp_uit_factor: "0.7"`;
    - offset: `offset_min_v: "0.9"`, `offset_max_v: "1.6"`, `offset_storing_ms: "10000"`.
  - **Meldingen**:
    - `verwerking_interval: 1s`, `start_niet_herstel_cm: "2.0"`, `geometrie_min_verschil_cm: "10.0"`;
    - `herstart_melden_na: 3s`, `opstarten_max: 10s`, `diagnose_interval: 60s`.
  - **Instellingen**: per sleutel uit data-model §1 `<sleutel>_min`, `<sleutel>_max` en `<sleutel>_stap`, en voor de niet-verplichte ook `<sleutel>_std`. Letterlijk:
    - `afstand_leeg_cm` 10–400, stap 0,1;
    - `afstand_vol_cm` 3–390, stap 0,1;
    - `inschakelniveau_cm` 0–400, stap 0,1;
    - `alarmdrempel_pct` 10–120, stap 1, standaard 80;
    - `hoogwater_wachttijd_s` 0–600, stap 1, standaard 30;
    - `start_marge_cm` 0–50, stap 1, standaard 5;
    - `start_wachttijd_s` 10–3600, stap 1, standaard 120;
    - `afpomp_controletijd_s` 30–3600, stap 1, standaard 180;
    - `afpomp_min_daling_cm` 0,5–50, stap 0,1, standaard 2;
    - `max_looptijd_min` 1–240, stap 1, standaard 10;
    - `sensorstoring_wachttijd_s` 30–3600, stap 1, standaard 300;
    - `herstelmarge_pct` 1–50, stap 1, standaard 5;
    - `pomp_stroomdrempel_a` 0,05–20, stap 0,05, standaard 0,5;
    - `led_helderheid_pct` 0–100, stap 1, standaard 30.
  - **Signalering** (letterlijk uit `contracts/lokale-signalering.md`):
    - `led_kleurvolgorde` (GRBW, in de vorm die `channel_colors` verwacht; zie de Buitenlamp);
    - `led_min_helderheid_alarm: "0.10"`.
    - Kleuren als fracties, `led_<kleur>_r/_g/_b/_w`:
      - rood 1,0/0/0/0;
      - oranje 1,0/0,314/0/0 (80/255);
      - paars 0,627/0/1,0/0 (160/255);
      - wit 0/0/0/1,0;
      - blauw 0/0/1,0/0;
      - groen 0/1,0/0/0.
    - Tijden in ms:
      - alarm 125 aan / 125 uit;
      - waarschuwing 500/500;
      - storing 500/1500;
      - geen verbinding 100/1900;
      - normaal 100/4900;
      - bevestiging 100/100 (2 keer).
    - Zoemer:
      - `zoemer_frequentie: 2700Hz`, `zoemer_niveau: 50%`;
      - alarm 3 keer 150/150, elke 10 000 ms;
      - waarschuwing 1 keer 300, elke 60 000 ms;
      - storing 2 keer 100/100;
      - hersteld 1 keer 50;
      - bevestiging 1 keer 50;
      - opstarten 1 keer 100;
      - pauze tussen eenmalige patronen 150 ms.
    - Namen van de tijden: `led_<patroon>_aan_ms` en `led_<patroon>_uit_ms` (bij bevestiging ook `led_bevestiging_aantal`); `zoemer_<patroon>_aantal`, `_aan_ms`, `_uit_ms` en bij herhalen `_periode_ms`; `zoemer_pauze_ms`. De patronen heten `alarm`, `waarschuwing`, `storing`, `opstarten`, `geen_verbinding`, `pomp`, `normaal`, `bevestiging` en `hersteld`. Een continu patroon heeft `_uit_ms: "0"`.
  - **Codes** (benoemde toestanden; constitutie II en V; zie de conventies bovenaan):
    - niveau: `niveau_onbekend: "0"`, `niveau_geldig: "1"`, `niveau_verouderd: "2"`, `niveau_storing: "3"`;
    - pomp: `pomp_onbekend: "0"`, `pomp_stil: "1"`, `pomp_start_kandidaat: "2"`, `pomp_draait: "3"`, `pomp_stop_kandidaat: "4"`;
    - meldingfase: `fase_inactief: "0"`, `fase_in_afwachting: "1"`, `fase_actief: "2"`;
    - meldingsnummer: `idx_hoogwater: "0"`, `idx_pomp_start_niet: "1"`, `idx_pomp_pompt_niet_af: "2"`, `idx_pomp_draait_te_lang: "3"`, `idx_niveausensor_storing: "4"`, `idx_stroomsensor_storing: "5"`, `idx_niet_ingesteld: "6"`, en `aantal_meldingen: "7"` (de grootte van de arrays);
    - bitgroepen in `g_actieve_set`: `bits_alarm: "0x01"`, `bits_waarschuwing: "0x0E"`, `bits_storing: "0x70"`;
    - zoemermodus: `zoemer_modus_stil: "0"`, `zoemer_modus_alarm: "1"`, `zoemer_modus_waarschuwing: "2"`;
    - ledpatroon, in volgorde van prioriteit: `led_patroon_geen: "0"` (dwingt een nieuwe keuze af), `led_patroon_alarm: "1"`, `led_patroon_waarschuwing: "2"`, `led_patroon_storing: "3"`, `led_patroon_opstarten: "4"`, `led_patroon_geen_verbinding: "5"`, `led_patroon_pomp: "6"`, `led_patroon_normaal: "7"`.
  - **Omrekening**: `ms_per_s: "1000"`, `ms_per_min: "60000"`, `pct_schaal: "100.0"`.
- [X] T009 Voeg **sectie 2 (basis)** toe aan `esphome/pompput.yaml`.
  - **`esphome:`**:
    - `name: ${device_name}`, `friendly_name: ${friendly_name}`, `min_version: 2026.10.1`;
    - `project: {name: diy.pompput, version: ${project_version}}`;
    - `on_boot` komt in T017.
  - **`esp32:`**: `variant: esp32c3`, `framework: type: esp-idf`.
  - **`preferences:`**: `flash_write_interval: ${flash_write_interval}`.
  - **`logger:`**: `level: ${log_level}`.
  - **`wifi:`**: `ssid` en `password` uit `!secret`, en `reboot_timeout: 0s` (constitutie, Independent operation).
  - **`api:`**: `encryption: key: !secret api_encryption_key`, en `reboot_timeout: 0s`.
  - **`ota:`**: `- platform: esphome` met `encryption:`, zonder eigen sleutel, zoals in de Buitenlamp (research R3).
  - **`debug:`**.
  - **Niet toevoegen**: `web_server`, `captive_portal`, `mqtt`.
  - Draai `.venv\Scripts\esphome config esphome/pompput.yaml` en los fouten op.
- [X] T010 Voeg in `esphome/pompput.yaml` **sectie 8 (gebeurtenissen en diagnose)** toe.
  - **`event:`** `- platform: template`, `id: gebeurtenis`, `name: "Gebeurtenis"`, met `event_types`: `pomp_start`, `pomp_stop`, `melding_actief`, `melding_hersteld`, `herstart`, `instelling_afgewezen` en `niveaumeting_herstart`.
  - **`text_sensor:`**:
    - template `laatste_gebeurtenis` ("Laatste gebeurtenis");
    - `- platform: debug` met `reset_reason:` als `herstartreden` ("Herstartreden", `entity_category: diagnostic`).
  - **`sensor:`**:
    - `wifi_signal` als `wifi_signaal` ("WiFi-signaal", `update_interval: ${diagnose_interval}`, diagnostic);
    - `uptime` als `uptime` ("Uptime", `${diagnose_interval}`, diagnostic).
  - **Script `gebeurtenis_melden`** (parameters `type: string`, `detail: string`; FR-026, FR-010):
    - `event.trigger` met `event_type: !lambda 'return type;'`;
    - `laatste_gebeurtenis` publiceren als `"<type>: <detail>"`;
    - `logger.log` op INFO.
  - **Script `herstart_melden`** (`mode: single`; FR-026):
    - `delay: ${herstart_melden_na}`;
    - `gebeurtenis_melden` met `herstart` en de waarde van `herstartreden`;
    - global `g_herstart_gemeld` (bool, false, niet bewaard) op true.
  - **`api: on_client_connected`**: als `g_herstart_gemeld` false is, `herstart_melden` uitvoeren.
- [X] T011 Voeg in `esphome/pompput.yaml` **sectie 6 (instellingen)** toe: de globals en de entiteiten (research R11, data-model §1).
  - **`globals`** met `restore_value: true`:
    - `g_afstand_leeg_cm`, `g_afstand_vol_cm`, `g_inschakelniveau_cm` (float, `initial_value: 'NAN'`);
    - `g_alarmdrempel_pct` (float, `${alarmdrempel_pct_std}`);
    - `g_herstelmarge_pct` (float, `${herstelmarge_pct_std}`).
  - **Global** `g_ingesteld` (bool, false, niet bewaard).
  - **Vijf template-`number`'s met regels**: `afstand_leeg_cm`, `afstand_vol_cm`, `inschakelniveau_cm`, `alarmdrempel_pct` en `herstelmarge_pct`.
    - `lambda: 'return id(g_<sleutel>);'` en `update_interval: never`.
    - `set_action`: `script.execute` van `instelling_toepassen`, met `sleutel: "<sleutel>"` en `waarde: !lambda 'return x;'`.
  - **Negen template-`number`'s** met `optimistic: true`, `restore_value: true` en `initial_value: ${<sleutel>_std}`: de overige instellingen uit data-model §1.
  - **Voor alle 14 numbers**:
    - `min_value`, `max_value` en `step` uit de substitutions;
    - `mode: box`, `entity_category: config`;
    - de eenheid uit data-model §1;
    - naam en YAML-id precies volgens `contracts/esphome-homeassistant.md`, bijvoorbeeld id `afstand_leeg_cm` met naam "Afstand leeg".
  - **Template-`switch`** `zoemer` ("Zoemer"): `optimistic: true`, `restore_mode: RESTORE_DEFAULT_ON` (FR-028).
  - Commentaar: FR-023, FR-024, FR-028.
- [X] T012 Voeg in `esphome/pompput.yaml` (sectie 6) de scripts **`ingesteld_bepalen`** en **`instelling_toepassen`** toe (FR-023, FR-024, randgevallen "niet ingesteld" en "ongeldige instellingen").
  - **`ingesteld_bepalen`** zet `g_ingesteld` op waar als alles hieronder geldt:
    - de drie verplichte waarden zijn niet `NAN`;
    - "`afstand_leeg_cm` ≥ `afstand_vol_cm` + 10";
    - "`inschakelniveau_cm` < `bereik_cm`" (bereik = leeg − vol);
    - "`alarmdrempel_pct` − `herstelmarge_pct` > 0".
  - **`instelling_toepassen`** (parameters `sleutel: string`, `waarde: float`):
    - **Kandidaatset**: neem de vijf globals over, met de nieuwe waarde voor `sleutel`.
    - **Gelijk aan de huidige waarde**: alleen de state opnieuw publiceren.
    - **Regels**: controleer alleen de regels waarvan alle waarden gezet zijn, in deze volgorde. Bij de eerste die faalt, is de reden:
      1. "`afstand_leeg_cm` ≥ `afstand_vol_cm` + 10" → reden `afstand_leeg_cm moet minstens 10 cm groter zijn dan afstand_vol_cm`;
      2. "`inschakelniveau_cm` < `bereik_cm`" → reden `inschakelniveau_cm moet kleiner zijn dan het meetbereik`;
      3. "`alarmdrempel_pct` − `herstelmarge_pct` > 0" → reden `alarmdrempel_pct min herstelmarge_pct moet groter zijn dan 0`.
    - **Geaccepteerd**:
      - de global zetten (ESPHome bewaart hem);
      - `id(<number>).publish_state(...)`;
      - `ingesteld_bepalen` uitvoeren;
      - op INFO loggen.
    - **Afgewezen**:
      - de oude waarde opnieuw publiceren met `publish_state`, zodat HA terugspringt;
      - `gebeurtenis_melden` met `instelling_afgewezen` en `"<sleutel> = <waarde>: <reden>"`;
      - op INFO loggen;
      - de global **niet** wijzigen.
    - Laat een duidelijk gemarkeerd haakje open voor "geometrie gewijzigd" (bij `afstand_leeg_cm` of `afstand_vol_cm`). Dat wordt ingevuld in T023.
- [X] T013 Voeg in `esphome/pompput.yaml` **sectie 5 (meldingen)** toe: het framework (research R5, data-model §5).
  - **`globals`** (niet bewaard):
    - `g_melding_fase` (`std::array<uint8_t, ${aantal_meldingen}>`, alles `${fase_inactief}`);
    - `g_melding_sinds_ms` (`std::array<uint32_t, ${aantal_meldingen}>`);
    - `g_melding_herstel_bezig` (`std::array<bool, ${aantal_meldingen}>`);
    - `g_melding_herstel_sinds_ms` (`std::array<uint32_t, ${aantal_meldingen}>`);
    - `g_actieve_set` (`uint8_t`, 0).
  - **Zeven template-`binary_sensor`'s** met `device_class: problem`, met de id's en namen uit het contract en het meldingsnummer (`${idx_...}`) uit de tabel bovenaan: `hoogwater`, `pomp_start_niet`, `pomp_pompt_niet_af`, `pomp_draait_te_lang`, `niveausensor_storing`, `stroomsensor_storing` en `niet_ingesteld`.
  - **Script `melding_stap`**. Parameters: `index: int`, `beoordelen: bool`, `activeren: bool`, `herstellen: bool`, `wachttijd_ms: int`, `hersteltijd_ms: int` en `onderdrukken: bool`. **Niet genoemde parameters** bij een aanroep zijn `onderdrukken = false` en `hersteltijd_ms = 0`; geef bij elke aanroep toch alle zeven parameters mee. Eén lambda met de levenscyclus uit data-model §5:
    - **Onderdrukken**: naar INACTIEF. Was de fase ACTIEF, dan is dat een herstel.
    - **INACTIEF**: als `beoordelen && activeren`, dan `sinds = now`, en daarna:
      - bij `wachttijd_ms == 0` direct naar ACTIEF (een activering);
      - anders naar IN_AFWACHTING.
    - **IN_AFWACHTING**:
      - als `!beoordelen || !activeren`: naar INACTIEF, **zonder** gebeurtenis;
      - als `now - sinds >= wachttijd_ms`: naar ACTIEF (een activering).
    - **ACTIEF**:
      - als `!beoordelen || !herstellen`: `herstel_bezig = false` (vasthouden);
      - anders: start de hersteltijd als die nog niet loopt, en als `now - herstel_sinds >= hersteltijd_ms`, naar INACTIEF (een herstel).
    - **Bij een activering**: bit `index` in `g_actieve_set` zetten, de bijbehorende binary sensor `publish_state(true)`, en `gebeurtenis_melden` met `melding_actief` en `"<melding>, niveau <hoogte> cm, pomp <draait|stil|onbekend>"`.
      - Hoogte en pomp komen uit `g_hoogte_cm` en `g_pomp_status`. Declareer die hier alvast als globals (`NAN` en `${pomp_onbekend}`), zodat ze vóór US1 en US3 bestaan.
    - **Bij een herstel**: het bit wissen, `publish_state(false)`, en `gebeurtenis_melden` met `melding_hersteld` en `"<melding>"`.
- [X] T014 Voeg in `esphome/pompput.yaml` (sectie 5) de apparaatstatus en de seconde-cyclus toe.
  - **Template-`text_sensor` `status`** ("Status"; FR-018).
  - **Script `status_bijwerken`**:
    - bepaal de status uit `g_actieve_set`, in de volgorde "ALARM > WAARSCHUWING > NIET_INGESTELD > STORING > NORMAAL" (een bit uit `${bits_alarm}` → `alarm`; uit `${bits_waarschuwing}` → `waarschuwing`; bit `${idx_niet_ingesteld}` → `niet_ingesteld`; een ander bit uit `${bits_storing}` → `storing`; anders `normaal`);
    - publiceer bij een verandering;
    - roep daarna `signalering_bijwerken` aan (T016).
  - **Script `melding_niet_ingesteld_beoordelen`**: `melding_stap` met `${idx_niet_ingesteld}`, `beoordelen = true`, `activeren = !g_ingesteld`, `herstellen = g_ingesteld` en wachttijd 0.
  - **Script `meldingen_beoordelen`**: roept de beoordelingsscripts aan die er zijn (nu alleen NIET_INGESTELD; de stories voegen er regels aan toe), en daarna `status_bijwerken`.
  - **`interval:`** `- interval: ${verwerking_interval}`, dat `meldingen_beoordelen` uitvoert.
  - Commentaar: FR-018, randgeval "niet ingesteld".
- [X] T015 Voeg **sectie 7 (signalering): de led** toe aan `esphome/pompput.yaml` (FR-027, FR-028; `contracts/lokale-signalering.md`, research R9).
  - **`light:`** `- platform: esp32_rmt_led_strip`, `id: statusled`:
    - `pin: ${pin_led}`, `num_leds: 1`, `chipset: SK6812`;
    - `channel_colors: ${led_kleurvolgorde}`, `use_psram: false`;
    - `restore_mode: ALWAYS_OFF`, `internal: true`, `default_transition_length: 0s`.
  - **`globals`**: `g_led_patroon` (int, `${led_patroon_geen}`), `g_led_helderheid` (float, `NAN`: nog niet gekozen; NAN is ongelijk aan elke waarde, dus de eerste keuze wordt altijd doorgevoerd), `g_led_bevestiging_bezig` (bool), `g_opstarten` (bool, true) en `g_verbonden` (bool, false).
  - **Script `led_knipperen`** (`mode: restart`; parameters `rood`, `groen`, `blauw`, `wit`, `helderheid` als float, en `aan_ms`, `uit_ms` als int):
    - bij `helderheid <= 0`: `light.turn_off`;
    - bij `uit_ms == 0`: `light.turn_on` met kleur en helderheid (continu);
    - anders een `while` (altijd waar): `light.turn_on` → `delay aan_ms` → `light.turn_off` → `delay uit_ms`.
  - **Script `led_patroon_kiezen`**. Stopt direct als `g_led_bevestiging_bezig` waar is. Anders:
    - **Patroon** (de bovenste regel die geldt, wint):
      1. alarm: `g_actieve_set & ${bits_alarm}`;
      2. waarschuwing: `g_actieve_set & ${bits_waarschuwing}`;
      3. storing: `g_actieve_set & ${bits_storing}`;
      4. opstarten: `g_opstarten`;
      5. geen verbinding: `!g_verbonden`;
      6. pomp draait: `g_pomp_status` is `${pomp_draait}` of `${pomp_stop_kandidaat}`;
      7. normaal.
    - Het gekozen patroon wordt `${led_patroon_<naam>}`, met de kleur en de tijden uit `led_<patroon>_...`.
    - **Helderheid**: `led_helderheid_pct` / `${pct_schaal}`. Bij een alarm minstens `${led_min_helderheid_alarm}`; bij 0 % en geen alarm uit.
    - Start `led_knipperen` met de kleur en de tijden van het patroon, maar alleen als het patroon of de helderheid veranderd is.
  - **Script `led_bevestiging`**:
    - `g_led_bevestiging_bezig = true`;
    - `${led_bevestiging_aantal}` keer wit, met `${led_bevestiging_aan_ms}` aan en `${led_bevestiging_uit_ms}` uit (2 × 100/100);
    - `g_led_bevestiging_bezig = false`, `g_led_patroon = ${led_patroon_geen}` (forceren), en `led_patroon_kiezen`.
  - **`api: on_client_connected`**: `g_verbonden = true`.
  - **`api: on_client_disconnected`**: als `not api.connected` (met `state_subscription_only: true`), dan `g_verbonden = false`.
  - In beide gevallen daarna `led_patroon_kiezen`.
  - **Number `led_helderheid_pct`**: voeg `on_value: led_patroon_kiezen` toe.
- [X] T016 Voeg **sectie 7 (signalering): de zoemer en de knoppen** toe aan `esphome/pompput.yaml` (FR-027, FR-028, SC-009; research R10).
  - **`output:`** `- platform: ledc`, `id: zoemer_output`, `pin: ${pin_zoemer}`, `frequency: ${zoemer_frequentie}`.
  - **`globals`**: `g_zoemer_modus` (int, `${zoemer_modus_stil}`; verder `${zoemer_modus_alarm}` en `${zoemer_modus_waarschuwing}`), `g_zoemer_herstart` (bool), `g_stil` (bool), `g_stil_set` (uint8_t) en `g_vorige_set` (uint8_t). Geen enkele wordt bewaard.
  - **Script `zoemer_herhalen`** (`mode: restart`; parameters `aantal`, `aan_ms`, `uit_ms`, `periode_ms`): een `while` (altijd waar) met:
    - `repeat aantal`: `output.set_level ${zoemer_niveau}` → `delay aan_ms` → `output.turn_off` → `delay uit_ms`;
    - daarna `delay (periode_ms − aantal × (aan_ms + uit_ms))`.
  - **Script `zoemer_eenmalig`** (`mode: queued`; parameters `aantal`, `aan_ms`, `uit_ms`, `altijd: bool`):
    - stop `zoemer_herhalen`, en `output.turn_off`;
    - speel alleen als `altijd` of de schakelaar `zoemer` aan staat;
    - wacht de pauze tussen eenmalige patronen;
    - `g_zoemer_herstart = true`, en `zoemer_patroon_kiezen`.
  - **Script `zoemer_patroon_kiezen`**:
    - **gewenste modus**:
      - `${zoemer_modus_stil}` als de schakelaar `zoemer` uit staat, of als `g_stil` waar is;
      - anders `${zoemer_modus_alarm}` bij een bit uit `${bits_alarm}`, `${zoemer_modus_waarschuwing}` bij een bit uit `${bits_waarschuwing}`, en `${zoemer_modus_stil}` zonder melding;
    - **bij een andere modus, of als `g_zoemer_herstart` waar is**:
      - start of stop `zoemer_herhalen`: alarm met `zoemer_alarm_...` (3×150/150 per 10 000 ms), waarschuwing met `zoemer_waarschuwing_...` (1×300 per 60 000 ms), en bij `${zoemer_modus_stil}` stoppen en `output.turn_off`;
      - zet `g_zoemer_herstart = false`.
  - **Script `signalering_bijwerken`** (aangeroepen door `status_bijwerken`), met `nieuw = g_actieve_set & ~g_vorige_set`:
    1. als `g_stil` en `(g_actieve_set & ~g_stil_set) != 0`: `g_stil = false` en `g_zoemer_herstart = true` (FR-028);
    2. als `!g_stil` en `nieuw & ${bits_storing}`: `zoemer_eenmalig` met `zoemer_storing_...` (2 × 100/100) en `altijd = false`;
    3. als `!g_stil` en `nieuw & (${bits_alarm} | ${bits_waarschuwing})`: `g_zoemer_herstart = true` (SC-009: het eerste patroon direct);
    4. als `g_vorige_set != 0` en `g_actieve_set == 0`: `zoemer_eenmalig` met `zoemer_hersteld_...` (1 × 50) en `altijd = false`;
    5. `g_vorige_set = g_actieve_set`;
    6. `led_patroon_kiezen` en `zoemer_patroon_kiezen`.
  - **Script `zoemer_stilzetten`**:
    - `g_stil = true` en `g_stil_set = g_actieve_set`;
    - `zoemer_eenmalig` met `zoemer_bevestiging_...` (1 × 50) en `altijd = false`;
    - `led_bevestiging`.
  - **`binary_sensor:`** `- platform: gpio`, `id: boot_knop`:
    - `pin: number ${pin_knop}`, `inverted: true`, `mode: input: true, pullup: true`;
    - `ignore_strapping_warning: true`, `internal: true`;
    - `filters: - delayed_on: 50ms`;
    - `on_press: zoemer_stilzetten`.
  - **`button:`** `- platform: template`, `id: zoemer_stil`, naam "Zoemer stil", `on_press: zoemer_stilzetten`.
  - **Switch `zoemer`** (uit T011): `on_turn_on` en `on_turn_off` → `zoemer_patroon_kiezen`.
  - Alle tijden en aantallen komen uit de substitutions van T008.
- [X] T017 Vul in `esphome/pompput.yaml` (sectie 2) `esphome: on_boot` met de opstartvolgorde (constitutie III, FR-025, `contracts/hardware.md` "Veilige begintoestand"). De led staat al uit door `ALWAYS_OFF`, en de zoemer-output begint uit.
  1. `ingesteld_bepalen`.
  2. `component.update` voor de vijf numbers met een lambda, zodat ze hun waarde of "onbekend" publiceren.
  3. `status` publiceren met `normaal`, als startwaarde.
  4. `zoemer_eenmalig` met `zoemer_opstarten_...` (1 × 100) en `altijd = true` (de opstartpiep, ook als de zoemer uit staat).
  5. `led_patroon_kiezen` (wit, opstarten).
  6. `delay: ${opstarten_max}`, daarna `g_opstarten = false` en `led_patroon_kiezen`.

  Commentaar met de redenen.
- [X] T018 [P] Werk de kop van `homeassistant/packages/pompput.yaml` bij.
  - Contract: `specs/001-pompput-monitor/contracts/esphome-homeassistant.md`, sectie "Home Assistant-pakket".
  - "De entiteiten van het apparaat komen via de ESPHome-integratie (native API)" in plaats van MQTT Discovery.
  - Controleer dat elke entity-id in het pakket voorkomt in het contract of door het pakket zelf wordt aangemaakt. De verwachting is dat de rest ongewijzigd kan blijven: "Pompput onbereikbaar" met `delay_on` 5 min (FR-017), de alerts met `repeat: 60` (FR-020), en de berichtteksten en `done_message` (FR-019, FR-021).
- [ ] T019 Compile-gate, de eerste keer flashen en controle van de basis volgens `specs/001-pompput-monitor/quickstart.md` §2 en §3 stappen 1–3, 5 en 6.
  - **Vereist**: de beheerder heeft de echte WiFi-gegevens in `esphome/secrets.yaml` gezet (T004), en de XIAO met led, zoemer en sensoren is aangesloten volgens `contracts/hardware.md`.
  - `esphome config` en `esphome compile`: geen fouten, en geen waarschuwingen uit de eigen lambdas.
  - Noteer RAM en flash.
  - Flash via USB.
  - Controleer:
    - de opstartpiep, de witte led, en daarna de paarse led met 2 piepjes;
    - de status `niet_ingesteld` in HA;
    - een afgewezen combinatie springt terug, en komt na een herstart niet terug.
  - Leg alles vast in `specs/001-pompput-monitor/verificatie.md`.

**Checkpoint**: Het apparaat staat in HA. Instellingen werken, worden gevalideerd en bewaard. "Niet ingesteld", de led en de zoemer werken. De user stories kunnen beginnen.

---

## Phase 3: User Story 1 - Vulgraad van de put aflezen in Home Assistant (Priority: P1) 🎯 MVP

**Goal**: Waterhoogte (cm) en vulgraad (%) zijn betrouwbaar af te lezen in HA, met historie. Een ongeldige meting is zichtbaar als "onbekend", en een langdurige storing als melding.

**Independent Test**: Houd een plank of een gevulde bak op bekende afstanden onder de sensor. HA toont een waarde die ≤ 10 s oud is en ≤ 2 cm afwijkt. Een hand door de bundel geeft geen sprong. Een object vlak voor de sensor geeft "onbekend" en daarna een storing (quickstart §4, scenario's 1–3).

- [X] T020 [US1] Voeg **sectie 3 (niveau)** toe aan `esphome/pompput.yaml`: de meting (FR-001; research R6).
  - **`sensor:`** `- platform: ultrasonic`, `id: niveau_ping`:
    - `trigger_pin: ${pin_trig}`, `echo_pin: ${pin_echo}`;
    - `update_interval: ${ping_interval}`, `timeout: ${ultrasoon_timeout}`, `internal: true`.
  - **`globals`** (niet bewaard): `g_pings` (`std::array<float, ${mediaan_venster}>`, alles `NAN`), `g_ping_index` (uint8_t), `g_ping_aantal` (uint8_t) en `g_nieuwe_pings` (uint8_t).
  - **`on_raw_value`** (meters of `NAN`): zet `x * ${m_naar_cm}` in de ringbuffer, verhoog `g_ping_aantal` tot maximaal de venstergrootte, en verhoog `g_nieuwe_pings`.
- [X] T021 [US1] Voeg in `esphome/pompput.yaml` (sectie 3) de niveaustatus en het script **`niveau_verwerken`** toe (FR-003, FR-004, FR-005, FR-016; data-model §2–§3).
  - **`globals`** (niet bewaard):
    - status: `g_niveau_status` (0), `g_heeft_referentie` (false);
    - geaccepteerde waarde: `g_geaccepteerd_cm` (`NAN`), `g_geaccepteerd_ms`;
    - klok: `g_klok_gestart` (false), `g_onbekend_sinds_ms`;
    - inleren: `g_inleer` (`std::array<float, ${inleer_aantal}>`), `g_inleer_aantal`, `g_inleer_index`;
    - mediaan: `g_mediaan_cm` (`NAN`).
  - **Het script**, als drie korte lambdas plus acties:
    1. **Mediaan**. Start de klok bij de eerste aanroep (`g_onbekend_sinds_ms = now`). Is `g_nieuwe_pings == 0`, dan is de mediaan `NAN`. Anders:
       - een meting is geldig als "3 cm ≤ afstand ≤ `afstand_leeg_cm + 20` cm". Zonder instelling geldt `${afstand_leeg_cm_max} + ${bereik_extra_cm}`;
       - de mediaan van de geldige waarden, "alleen als er minimaal 3 geldig" zijn. Bij een even aantal het gemiddelde van de middelste twee;
       - zet daarna `g_nieuwe_pings = 0` en `g_mediaan_cm`.
    2. **Accepteren**.
       - **Zonder referentie**: een `NAN`-mediaan breekt de inleerreeks af (de reeks gaat terug naar 0). Anders de mediaan toevoegen, en accepteren als er "5 opeenvolgende medianen binnen 2 cm van elkaar" zijn (max − min ≤ `${inleer_spreiding_cm}`).
       - **Met referentie**: accepteren als "|Δ| ≤ max(3 cm, 2 cm/s × Δt), waarbij Δt maximaal 5 s telt" (Δt = de tijd sinds `g_geaccepteerd_ms`).
       - **Met referentie, in STORING** (FR-016, FR-022; research R6, "opnieuw inleren"): houd naast de snelheidscontrole ook de inleerreeks bij, op dezelfde manier als zonder referentie. Zijn er "5 opeenvolgende medianen binnen 2 cm", accepteer dan de laatste als nieuwe referentie, en voer `gebeurtenis_melden` uit met `niveaumeting_herstart` en `"oorzaak automatisch"`. Zo herstelt de meting zich ook als het niveau tijdens de storing meer is veranderd dan de snelheidscontrole toelaat.
       - **Bij acceptatie**: `g_heeft_referentie = true`, de inleerreeks leeg, `g_geaccepteerd_cm` en `_ms` zetten, en de status GELDIG.
    3. **Tijdsgrenzen**, als er niet geaccepteerd is.
       - **Met referentie**:
         - "STORING na `sensorstoring_wachttijd_s`" sinds `g_geaccepteerd_ms`;
         - anders "VEROUDERD na 30 s zonder geaccepteerde mediaan".
       - **Zonder referentie**: STORING als `now − g_onbekend_sinds_ms ≥ sensorstoring_wachttijd_s`.
  - **Afgeleide waarden** in globals: `g_hoogte_cm` = "`afstand_leeg_cm − afstand_cm`" en `g_vulgraad_pct` = "`hoogte_cm / bereik_cm × 100` (niet afgekapt)".
    - Gebruik daarvoor `g_geaccepteerd_cm`.
    - Alleen bij GELDIG en `g_ingesteld`; anders `NAN`.
  - **Bij een statuswijziging**: op INFO loggen ("niveau <oud> → <nieuw>"), en de sensoren uit T022 direct bijwerken.
- [X] T022 [US1] Voeg in `esphome/pompput.yaml` (sectie 3) de sensoren toe (FR-002, FR-003; het contract, "Sensoren").
  - **Template-`sensor` `waterhoogte`**: "Waterhoogte", cm, `device_class: distance`, `state_class: measurement`, `accuracy_decimals: 1`, `lambda: return id(g_hoogte_cm);`, `update_interval: never`.
  - **`vulgraad`**: "Vulgraad", %, `measurement`, 0 decimalen, uit `g_vulgraad_pct`.
  - **`afstand_ruw`**: "Afstand (ruw)", cm, `distance`, diagnostic, 1 decimaal, uit `g_mediaan_cm`.
  - **`interval: ${publicatie_interval}`**: `component.update` van alle drie.
  - **In `niveau_verwerken`**:
    - bij een wijziging van de niveaustatus direct `component.update` van `waterhoogte` en `vulgraad`;
    - bij de eerste GELDIG: `g_opstarten = false` en `led_patroon_kiezen`.
  - **Seconde-interval** (T014): laat het eerst `niveau_verwerken` uitvoeren en daarna `meldingen_beoordelen`.
- [X] T023 [US1] Voeg in `esphome/pompput.yaml` het opnieuw starten van de niveaumeting toe (R6; FR-026).
  - **Script `niveaumeting_herstarten`** (parameter `oorzaak: string`):
    - wis de ringbuffer, de tellers en de inleerreeks;
    - `g_niveau_status = 0`, `g_heeft_referentie = false`, `g_geaccepteerd_cm = NAN`, `g_mediaan_cm = NAN`, `g_klok_gestart = false`;
    - `g_hoogte_cm` en `g_vulgraad_pct` op `NAN`, en de sensoren bijwerken;
    - `gebeurtenis_melden` met `niveaumeting_herstart` en `"oorzaak <oorzaak>"`.
  - **Template-`button` `niveaumeting_herstart`**: naam "Niveaumeting opnieuw starten", `entity_category: config`, `on_press` met `oorzaak "knop"`.
  - **Het haakje uit T012** in `instelling_toepassen`: bij een geaccepteerde wijziging van `afstand_leeg_cm` of `afstand_vol_cm` `niveaumeting_herstarten` met `"instelling"`.
- [X] T024 [US1] Voeg in `esphome/pompput.yaml` (sectie 5) het script **`melding_niveausensor_storing_beoordelen`** toe, en roep het aan vanuit `meldingen_beoordelen` (FR-016).
  - `melding_stap` met `${idx_niveausensor_storing}`: `beoordelen = g_ingesteld`, `activeren` = "niveaustatus = STORING", `herstellen` = "niveaustatus = GELDIG", wachttijd 0.
  - Gevolg: bij NIET_INGESTELD wordt deze melding niet nieuw geactiveerd.
- [ ] T025 [US1] Compile-gate, en controle volgens `specs/001-pompput-monitor/quickstart.md`:
  - §4 scenario's 1–3 en 27–30 (geometrie wijzigen, herstel na een blinde periode, hindernis bij inleren, buiten het bereik);
  - §5 stap 2: Afstand (ruw) werkt ook zonder instellingen;
  - de historiegrafiek van Waterhoogte in HA (US1 scenario 5);
  - de eerste bankpunten: TRIG op 3,3 V, en de kleurvolgorde van de led.

  Leg alles vast in `specs/001-pompput-monitor/verificatie.md`.

**Checkpoint**: US1 werkt zelfstandig: dit is de MVP.

---

## Phase 4: User Story 2 - Alarm bij te hoog waterniveau (Priority: P2)

**Goal**: Een hoogwater-alarm via HA en ter plaatse (rode led, zoemer), met een herinnering, een herstelmelding en vasthouden bij een uitgevallen meting.

**Independent Test**: Breng het niveau > 30 s boven de alarmdrempel. Er komt een alarm in HA, de led knippert snel rood, en de zoemer piept 3× per 10 s. Een golf van minder dan 30 s geeft geen alarm. Als het niveau 5 procentpunt onder de drempel zakt, volgt een herstelmelding. Valt de sensor uit tijdens een alarm, dan blijft het alarm staan (quickstart §4, scenario's 4–7).

- [X] T026 [US2] Voeg in `esphome/pompput.yaml` (sectie 5) het script **`melding_hoogwater_beoordelen`** toe, en roep het aan vanuit `meldingen_beoordelen` (FR-012, FR-021, FR-022).
  - `melding_stap` met `${idx_hoogwater}`:
    - `onderdrukken = !g_ingesteld`;
    - `beoordelen` = "niveaustatus = GELDIG" en `g_vulgraad_pct` niet `NAN`;
    - `activeren` = "`vulgraad_pct > alarmdrempel_pct`";
    - `herstellen` = "`vulgraad_pct < alarmdrempel_pct − herstelmarge_pct`";
    - `wachttijd_ms` = `hoogwater_wachttijd_s` × `${ms_per_s}`.
  - Commentaar: "**niet** beoordelen bij een niveaustatus ≠ GELDIG (vasthouden, FR-022)".
- [ ] T027 [US2] Compile-gate, en controle volgens `specs/001-pompput-monitor/quickstart.md` §4 scenario's 4–7, inclusief:
  - SC-009: led en zoemer binnen 5 s na het actief worden;
  - de herinnering, met tijdelijk `repeat: 2` in het pakket (FR-020);
  - de inhoud van de HA-melding (FR-019);
  - dat het alarm voorgaat op een storing.

  Leg alles vast in `specs/001-pompput-monitor/verificatie.md`.

**Checkpoint**: US1 en US2 werken allebei zelfstandig.

---

## Phase 5: User Story 3 - Pompactiviteit volgen (Priority: P3)

**Goal**: De pompstatus, elke start en stop, de looptijd en het aantal starts per 24 uur in HA. Een storing van de stroomsensor wordt herkend.

**Independent Test**: Zet een belasting 10 s aan. "Pomp" is binnen 5 s aan en weer uit, er komen de gebeurtenissen `pomp_start` en `pomp_stop`, de looptijd is ≈ 10 s, en de teller gaat met 1 omhoog. Een piek korter dan 2 s telt niet. Zonder voeding naar de ACS712 volgt een storing (quickstart §4, scenario's 8, 9, 13 en 19).

- [X] T028 [US3] Voeg **sectie 4 (pomp)** toe aan `esphome/pompput.yaml`: de meting (FR-006; research R7).
  - **`sensor:`** `- platform: adc`, `id: acs712_spanning`:
    - `pin: ${pin_acs712}`, `attenuation: 12db`, `update_interval: ${adc_interval}`, `internal: true`;
    - `filters: - sliding_window_moving_average: {window_size: ${offset_venster}, send_every: ${offset_elke}}`.
  - **`on_value`**: `g_offset_v = x`, en `g_offset_ok` = "offset binnen 0,9–1,6 V" (`${offset_min_v}` ≤ x ≤ `${offset_max_v}`).
  - **`sensor:`** `- platform: ct_clamp`, `id: pompstroom_rms`:
    - `sensor: acs712_spanning`, `sample_duration: ${stroom_venster}`, `update_interval: ${stroom_interval}`;
    - `filters: - multiply: ${acs712_a_per_v}`, `internal: true`;
    - `on_value`: `pomp_bijwerken` uitvoeren.
  - **Globals**: `g_offset_v` (`NAN`) en `g_offset_ok` (true).
- [X] T029 [US3] Voeg in `esphome/pompput.yaml` (sectie 4) het script **`pomp_bijwerken`** toe (FR-006, FR-009, FR-011; data-model §4).
  - **Globals** (niet bewaard): `g_pomp_status` (uit T013), `g_pomp_start_ms`, `g_pomp_stop_kandidaat_ms`, `g_niveau_start_cm` (`NAN`), `g_looptijd_s` (float, `NAN`; wordt alleen bij een stop gepubliceerd) en `g_stroom_a` (`NAN`).
  - **Begin**: `I` = de waarde uit `pompstroom_rms`. Is `I` `NAN`, dan gebeurt er niets.
  - **Offset fout** (`!g_offset_ok`): status ONBEKEND, `g_stroom_a = NAN`, en geen gebeurtenis.
  - **Anders** `g_stroom_a = I`, en de toestandsmachine:
    - `aan` = `pomp_stroomdrempel_a`, `uit` = `aan × ${pomp_uit_factor}`;
    - **ONBEKEND of STIL**:
      - bij I ≥ aan: START_KANDIDAAT, met `g_pomp_start_ms = now` ("het eerste venster boven de drempel") en `g_niveau_start_cm = g_hoogte_cm`;
      - anders STIL.
    - **START_KANDIDAAT**:
      - bij I < uit: STIL ("korte piek: geen start", FR-011);
      - anders, na `${pomp_debounce_ms}` sinds de start: DRAAIT, en `gebeurtenis_melden` met `pomp_start` en `"niveau <g_niveau_start_cm> cm"` (of "niveau onbekend").
    - **DRAAIT**: bij I < uit naar STOP_KANDIDAAT, met `g_pomp_stop_kandidaat_ms = now`.
    - **STOP_KANDIDAAT**:
      - bij I ≥ uit: terug naar DRAAIT;
      - anders, na `${pomp_debounce_ms}`: STIL. Daarbij `g_looptijd_s` = afgerond (stop_kandidaat_ms − start_ms) / `${ms_per_s}`, en `gebeurtenis_melden` met `pomp_stop` en `"looptijd <s> s, niveau <hoogte> cm"`.
- [X] T030 [US3] Voeg in `esphome/pompput.yaml` (sectie 4) de entiteiten van de pomp toe (FR-008, FR-009; het contract).
  - **Template-`binary_sensor` `pomp`**: "Pomp", `device_class: running`.
  - **Template-`sensor`'s**:
    - `looptijd_laatste_pompbeurt`: "Looptijd laatste pompbeurt", s, `duration`. Alleen bij een stop gepubliceerd, uit `g_looptijd_s`;
    - `pompstroom`: "Pompstroom", A, `current`, `measurement`, diagnostic, 2 decimalen, uit `g_stroom_a`;
    - `offset_stroomsensor`: "Offset stroomsensor", V, `voltage`, diagnostic, 2 decimalen, uit `g_offset_v`.
  - **Publicatie**:
    - `pompstroom` en `offset_stroomsensor` met `update_interval: never`, en bijgewerkt door het interval `${publicatie_interval}` (T022);
    - na elke `pomp_bijwerken`: verandert de zichtbare toestand (draait = `${pomp_draait}` of `${pomp_stop_kandidaat}`, stil = `${pomp_stil}` of `${pomp_start_kandidaat}`, onbekend = `${pomp_onbekend}`), dan direct `pomp` publiceren (aan/uit, of `invalidate_state()` bij onbekend), `pompstroom` bijwerken en `led_patroon_kiezen` uitvoeren;
    - bij de overgang naar STIL vanuit draaien: `looptijd_laatste_pompbeurt` publiceren.
- [X] T031 [US3] Voeg in `esphome/pompput.yaml` (sectie 5) het script **`melding_stroomsensor_storing_beoordelen`** toe, en roep het aan vanuit `meldingen_beoordelen` (FR-016).
  - `melding_stap` met `${idx_stroomsensor_storing}`:
    - `beoordelen = true`, `activeren = !g_offset_ok`, `herstellen = g_offset_ok`;
    - `wachttijd_ms` en `hersteltijd_ms` = `${offset_storing_ms}` ("offset buiten 0,9–1,6 V gedurende 10 s"; "offset 10 s binnen het bereik").
  - Wordt ook beoordeeld zonder instellingen.
- [ ] T032 [US3] Compile-gate, en controle volgens `specs/001-pompput-monitor/quickstart.md`:
  - §4 scenario's 8, 9, 13, 19, 24 (stroomdip tijdens pompen) en 31 (uitschakelgrens);
  - §5 stap 5: de Stroomdrempel pomp afstellen;
  - het bankpunt "ACS712 in rust ≈ 1,25 V" (Offset stroomsensor);
  - de blauwe led bij een draaiende pomp;
  - de teller "Pompput pompstarts 24 uur".

  Leg alles vast in `specs/001-pompput-monitor/verificatie.md`.

**Checkpoint**: US1, US2 en US3 werken zelfstandig.

---

## Phase 6: User Story 4 - Waarschuwing als de pomp niet goed afpompt (Priority: P4)

**Goal**: Vroege waarschuwingen "pomp start niet", "pomp pompt niet af" en "pomp draait te lang", via HA en ter plaatse (oranje led, zoemer 1× per 60 s).

**Independent Test**: Boots elke storing na (quickstart §4, scenario's 10–12). Bij elke situatie komt de juiste waarschuwing, en volgt een herstelmelding als de pomp weer normaal werkt.

Gemeenschappelijk voor T033–T035:
- `onderdrukken = !g_ingesteld`;
- `beoordelen` = "pompstatus ≠ ONBEKEND". Is de pomp onbekend, dan blijft een actieve waarschuwing staan;
- "draait" = `g_pomp_status` is `${pomp_draait}` of `${pomp_stop_kandidaat}`; "staat stil" = `${pomp_stil}` of `${pomp_start_kandidaat}`;
- "niveau geldig" = de niveaustatus is GELDIG en `g_hoogte_cm` is niet `NAN`.

- [X] T033 [US4] Voeg in `esphome/pompput.yaml` (sectie 5) het script **`melding_pomp_start_niet_beoordelen`** toe, en roep het aan vanuit `meldingen_beoordelen` (FR-013).
  - `melding_stap` met `${idx_pomp_start_niet}`:
    - `activeren` = niveau geldig, en "`hoogte_cm > inschakelniveau_cm + start_marge_cm` en de pomp staat stil";
    - `herstellen` = "de pomp draait, of het niveau zakt onder `inschakelniveau_cm + start_marge_cm − 2 cm`" (met `${start_niet_herstel_cm}`; de niveauvoorwaarde alleen bij een geldig niveau);
    - `wachttijd_ms` = `start_wachttijd_s` × `${ms_per_s}`.
- [X] T034 [US4] Voeg in `esphome/pompput.yaml` (sectie 5) het script **`melding_pomp_pompt_niet_af_beoordelen`** toe, en roep het aan vanuit `meldingen_beoordelen` (FR-014).
  - `melding_stap` met `${idx_pomp_pompt_niet_af}`, met daling = `g_niveau_start_cm − g_hoogte_cm`:
    - `activeren` = de pomp draait, het niveau is geldig, `now − g_pomp_start_ms ≥ afpomp_controletijd_s × ${ms_per_s}`, en "`niveau_start_cm − hoogte_cm < afpomp_min_daling_cm`";
    - `herstellen` = de pomp draait, het niveau is geldig, en "binnen een cyclus de daling ≥ `afpomp_min_daling_cm`";
    - wachttijd 0.
  - Een `NAN` in een van beide maakt de vergelijking onwaar.
- [X] T035 [US4] Voeg in `esphome/pompput.yaml` (sectie 5) het script **`melding_pomp_draait_te_lang_beoordelen`** toe, en roep het aan vanuit `meldingen_beoordelen` (FR-015).
  - `melding_stap` met `${idx_pomp_draait_te_lang}`:
    - `activeren` = "de pomp draait > `max_looptijd_min`" (`now − g_pomp_start_ms > max_looptijd_min × ${ms_per_min}`); daarvoor is geen niveau nodig;
    - `herstellen` = "de pomp staat stil";
    - wachttijd 0.
- [ ] T036 [US4] Compile-gate, en controle volgens `specs/001-pompput-monitor/quickstart.md` §4 scenario's 10–12, 14, 15, 23, 25 (stroomsensor los tijdens een waarschuwing) en 26 (herstel zonder pomp). Leg alles vast in `specs/001-pompput-monitor/verificatie.md`.

**Checkpoint**: Alle user stories werken zelfstandig.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Dashboard, documentatie, de laatste gates en de veldvalidatie

- [X] T037 [P] Werk `homeassistant/dashboard-pompput.yaml` bij naar de entity-id's van `specs/001-pompput-monitor/contracts/esphome-homeassistant.md`.
  - `number.pompput_<sleutel>` wordt de id uit het contract, bijvoorbeeld `number.pompput_afstand_leeg`.
  - `sensor.pompput_afstand` wordt `sensor.pompput_afstand_ruw`.
  - `sensor.pompput_looptijd_laatste_cyclus` wordt `sensor.pompput_looptijd_laatste_pompbeurt`.
  - `button.pompput_niveaumeting_herstart` wordt `button.pompput_niveaumeting_opnieuw_starten`.
  - Voeg `sensor.pompput_laatste_gebeurtenis` toe aan de hoofdkaart, en `sensor.pompput_offset_stroomsensor` aan "Diagnose".
  - Controleer dat elke id bestaat (in het contract of in het pakket), en dat de YAML geldig is.
- [X] T038 [P] Herschrijf `README.md` voor ESPHome.
  - **Behouden**: de inleiding, de veiligheidswaarschuwing voor 230 V, het hele hoofdstuk "Aansluiten" (de hardware is ongewijzigd), "Signalering ter plaatse" en de stappen voor Home Assistant.
  - **Platformtabel**: ESPHome 2026.10.1 (vastgepind in `requirements.txt`), esp-idf, de XIAO ESP32C3, de gebruikte componenten, en geen externe componenten.
  - **"Pinnen"**: de pinnen staan in het `substitutions:`-blok van `esphome/pompput.yaml`, en `contracts/hardware.md` is gezaghebbend. Haal alle verwijzingen naar `include/pins.h` weg.
  - **"Bouwen en flashen"**: de `.venv`, `secrets.yaml` met de sleutel, `esphome config`/`compile`/`run`, de eerste keer via USB (met B+R voor de downloadmodus) en daarna via WiFi, en `esphome logs`.
  - **"Opbouw"**: `esphome/`, `homeassistant/`, `specs/` en `requirements.txt`. Geen `lib/`, `src/` of `test/`.
  - **HA koppelen**: via de ESPHome-integratie met de sleutel, in plaats van Mosquitto.
- [ ] T039 Laatste compile- en resource-gate voor `esphome/pompput.yaml`.
  - `esphome config` en `esphome compile` zonder fouten, en zonder waarschuwingen uit de eigen lambdas.
  - RAM ≤ 75 % en flash ≤ 90 % (constitutie II).
  - Tijdens een uur draaien geen logmelding "took a long time" voor de eigen scripts.
  - **Rekentijd** (constitutie II: hoogstens 10 ms). ESPHome 2026.10.1 meldt een trage component pas boven 50 ms, dus meet apart (research, Open punten 2):
    - bouw tijdelijk een versie waarin het seconde-interval en `pompstroom_rms.on_value` hun scripts aanroepen vanuit een lambda, met `micros()` ervoor en erna. Het gaat om `niveau_verwerken`, `meldingen_beoordelen` (met de beoordelingsscripts en de signalering) en `pomp_bijwerken`. Deze scripts hebben geen `delay`, dus `execute()` loopt in één keer door;
    - log elke minuut het maximum per script, en laat de versie 15 min draaien terwijl je scenario 4, 8 en 10 uitvoert;
    - leg de maxima vast in `specs/001-pompput-monitor/verificatie.md`. Elk maximum moet hoogstens 10 ms zijn;
    - haal daarna de meetcode weg, en controleer met `esphome config` dat die weg is.

  Leg alles vast in `specs/001-pompput-monitor/verificatie.md`.
- [X] T040 Review-gate van `esphome/pompput.yaml` tegen constitutie 2.0.0 (gate 5) en traceerbaarheid (gate 3).
  - **Constitutie**:
    - elke pin, tijd en grens komt uit `substitutions`;
    - geen losse getallen in lambdas: toestanden, meldingsnummers, bitgroepen, patronen, omrekeningen en beginwaarden gebruiken hun naam uit `substitutions` of `NAN`; alleen 0, `true`/`false` en de rekenkundige 1 en 2 staan letterlijk (constitutie II en V);
    - elk script en elke template-entiteit heeft een FR-commentaar dat klopt met de tabel "Traceerbaarheid" in `plan.md`;
    - de led heeft `ALWAYS_OFF`;
    - `reboot_timeout: 0s` voor `wifi` en `api`;
    - geen uitgang naar de pomp (FR-007);
    - geen `web_server`, `captive_portal` of `mqtt`;
    - de pinnen komen overeen met `contracts/hardware.md`;
    - de geheimen staan alleen in `secrets.yaml`.
  - Leg de uitkomst en eventuele afwijkingen vast in `specs/001-pompput-monitor/verificatie.md`.
- [ ] T041 Voer de rest van `specs/001-pompput-monitor/quickstart.md` §4 uit: scenario's 16–18 en 20–22 (geen verbinding, met "Pompput onbereikbaar" na 5 min, FR-017; herstel zonder nazenden; herstart zonder HA; gebeurtenissen; langdurig netwerkverlies zonder herstart; update via WiFi). Leg in `specs/001-pompput-monitor/verificatie.md` vast wat níet op hardware geverifieerd is, en waarom.
- [ ] T042 Installeer het apparaat in de put, en kalibreer volgens `specs/001-pompput-monitor/quickstart.md` §5, stappen 1–6. Doe 10 controlemetingen voor SC-001, en leg ze vast in `specs/001-pompput-monitor/verificatie.md`.
- [ ] T043 Doe de veldvalidatie volgens `specs/001-pompput-monitor/quickstart.md` §6: 7 dagen steekproeven van pompbeurten (SC-002), 30 dagen onterechte meldingen tellen (SC-004, ≤ 1; wat "onterecht" is, staat in quickstart §6), en het dashboard (SC-006). Leg de resultaten vast in `specs/001-pompput-monitor/verificatie.md`.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (fase 1)**: geen afhankelijkheden.
  - T004 hangt af van T003, en T005 van T001.
  - T007 wacht op het antwoord van de beheerder.
- **Foundational (fase 2)**: hangt af van T001–T005, en **blokkeert alle user stories**.
  - T008–T017 bewerken hetzelfde bestand en lopen dus na elkaar.
  - T018 kan parallel lopen.
  - T019 wacht daarnaast op de beheerder: de echte WiFi-gegevens in `esphome/secrets.yaml`, en de aangesloten hardware.
- **US1 (fase 3)**: hangt af van fase 2.
- **US2 (fase 4)**: hangt af van fase 2, en voor een zinvolle controle van US1 (`g_vulgraad_pct`, niveaustatus).
- **US3 (fase 5)**: hangt af van fase 2. Is los van US1 en US2 te bouwen en te controleren; zonder US1 is `g_niveau_start_cm` gewoon `NAN`.
- **US4 (fase 6)**: hangt af van US1 (niveau) en US3 (pompstatus).
- **Polish (fase 7)**: hangt af van alle gewenste stories. T037 en T038 kunnen eerder, omdat het andere bestanden zijn.

### User Story Dependencies

```text
Setup ─▶ Foundational ─┬─▶ US1 (P1, MVP) ─┬─▶ US2 (P2)
                       │                  └─┐
                       └─▶ US3 (P3) ────────┴─▶ US4 (P4) ─▶ Polish
```

### Within Each User Story

- Eerst de metingen en de globals, dan de verwerkingsscripts, dan de entiteiten en de
  publicatie, dan de beoordeling van de melding, en dan de compile-gate en de controle op het
  apparaat.
- Alle firmwaretaken bewerken `esphome/pompput.yaml`, en zijn daarom niet [P] ten opzichte van
  elkaar.

### Parallel Opportunities

- **Setup**: T001, T002, T003 en T006 parallel.
- **Foundational**: T018 (het HA-pakket) parallel met T008–T017.
- **Polish**: T037 (dashboard) en T038 (README) parallel met elkaar, en met de firmwaretaken.
- US1 en US3 kunnen na fase 2 in willekeurige volgorde worden opgepakt, maar niet tegelijk in
  hetzelfde bestand.

---

## Parallel Example: Setup

```bash
Task: "requirements.txt met esphome==2026.10.1 (T001)"
Task: ".gitignore herschrijven (T002)"
Task: "esphome/secrets.example.yaml (T003)"
Task: "specs/001-pompput-monitor/verificatie.md, nieuwe opzet (T006)"
```

## Parallel Example: Polish

```bash
Task: "homeassistant/dashboard-pompput.yaml naar de nieuwe entity-id's (T037)"
Task: "README.md herschrijven voor ESPHome (T038)"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Fase 1: Setup. ESPHome is geïnstalleerd en de oude code is opgeruimd.
2. Fase 2: Foundational. Het apparaat staat in HA, met instellingen, led en zoemer (T019).
3. Fase 3: US1. Het waterniveau is betrouwbaar in HA.
4. **Stop en valideer**: quickstart §4, scenario's 1–3 (T025).
5. Eventueel nu al in de put hangen, om alleen het niveau te volgen. Latere stories komen er
   via WiFi bij.

### Incremental Delivery

1. Setup + Foundational: de basis.
2. US1: het niveau in HA (MVP).
3. US2: het hoogwater-alarm. Vanaf hier beschermt het apparaat tegen overlopen.
4. US3: pompactiviteit en historie.
5. US4: vroege pompwaarschuwingen.
6. Polish: dashboard, README, de laatste gates en de veldvalidatie.

---

## Notes

- [P] = ander bestand, geen afhankelijkheden.
- Het [Story]-label koppelt een taak aan een user story uit [spec.md](spec.md).
- Draai na elke firmwaretaak minstens `esphome config esphome/pompput.yaml`, en aan het einde
  van elke fase ook `esphome compile`.
- Controletaken (T019, T025, T027, T032, T036, T041–T043) vragen het apparaat, Home Assistant
  of de put. Wat niet uitgevoerd kan worden, wordt met de reden vastgelegd in
  `verificatie.md`, en de taak blijft dan open.
- Commit na elke taak of logische groep, op aanwijzing van de beheerder.
- Netspanningswerk (ACS712) alleen door of onder toezicht van een vakbekwaam persoon.
