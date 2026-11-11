# Verificatie: Pompput-monitor (ESPHome)

**Feature**: [spec.md](spec.md) | **Taken**: [tasks.md](tasks.md) | **Quickstart**: [quickstart.md](quickstart.md)

Dit document legt vast wat geverifieerd is, hoe, en wat (nog) **niet** geverifieerd is en
waarom (constitutie 2.0.0, gate 4). Het vervangt de verificatie van de vorige
Arduino/MQTT-implementatie; die staat in git (commit `75ddd7e`).

## Samenvatting van de gates

| Gate | Wat | Status | Waar |
|---|---|---|---|
| 1. Compile | `esphome config` en `esphome compile`, zonder waarschuwingen uit de eigen lambdas | ✅ geslaagd (2026-11-11), 0 waarschuwingen | §T039 |
| 2. Resource | RAM ≤ 75 %, flash ≤ 90 %, rekentijd ≤ 10 ms per script | deels: RAM 35,3 % en flash 48,5 % ✅; rekentijd open (apparaat) | §T039 |
| 3. Traceerbaarheid | elke FR heeft een YAML-element en een scenario | ✅ | §T040 |
| 4. Hardware | alle scenario's uit quickstart §4 op de testbank | open: nog niets op hardware uitgevoerd | §T019–§T043 |
| 5. Review | de YAML tegen de constitutie | ✅, geen schendingen; afwijkingen van plan en taken staan onder Afwijkingen | §T040 |

## Controletaken

Per taak: welk scenario, of het is uitgevoerd, het resultaat, en zo niet, waarom niet.

### T019: basis (quickstart §2, §3 stappen 1–3, 5, 6)

| Scenario | Uitgevoerd | Resultaat | Reden als niet uitgevoerd |
|---|---|---|---|
| Compile-gate en RAM/flash | ja | geslaagd; RAM 35,3 %, flash 48,5 % (zie §T039) | |
| Eerste flash via USB | ja (2026-11-11, door de beheerder) | **Crashlus gevonden**: `Store access fault` in `setup()`, 6 keer, daarna veilige modus (alleen OTA, geen API: HA kon niet verbinden). Oorzaak: de `on_value` van Led-helderheid ging af bij het herstellen van de waarde en zette de led aan voordat de led-strip was opgezet (addr2line: `TemplateNumber::setup` → `led_patroon_kiezen` → `light.turn_on` → `ESPColorView::set_red`). Opgelost: geen `on_value` meer; `led_patroon_kiezen` draait elke seconde. Nieuwe build geslaagd; opnieuw flashen en controleren | |
| Opstartpiep, witte led, daarna paars met 2 piepjes | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| Status `niet_ingesteld` in HA | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| Afgewezen combinatie springt terug en komt na een herstart niet terug | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |

### T025: US1 niveau (§4 #1–#3, #27–#30; §5 stap 2)

| Scenario | Uitgevoerd | Resultaat | Reden als niet uitgevoerd |
|---|---|---|---|
| Compile-gate | ja | geslaagd, de definitieve build (zie §T039) | |
| #1 Niveau aflezen | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| #2 Uitschieter | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| #3 Onmogelijke sprong | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| #27 Geometrie wijzigen | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| #28 Herstel na blinde periode | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| #29 Hindernis bij inleren | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| #30 Buiten het bereik | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| §5 stap 2: Afstand (ruw) zonder instellingen | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| Historie Waterhoogte in HA (US1 scen. 5) | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |

### T027: US2 hoogwater (§4 #4–#7)

| Scenario | Uitgevoerd | Resultaat | Reden als niet uitgevoerd |
|---|---|---|---|
| Compile-gate | ja | geslaagd, de definitieve build (zie §T039) | |
| #4 Hoogwater (incl. SC-009, herinnering, inhoud HA-melding) | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| #5 Golf | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| #6 Herstel en hysterese | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| #7 Vasthouden (alarm gaat voor op storing) | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |

### T032: US3 pomp (§4 #8, #9, #13, #19, #24, #31; §5 stap 5)

| Scenario | Uitgevoerd | Resultaat | Reden als niet uitgevoerd |
|---|---|---|---|
| Compile-gate | ja | geslaagd, de definitieve build (zie §T039) | |
| #8 Pomp aan en uit | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| #9 Stroompiek | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| #13 Stroomsensor los | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| #19 Pomp onafhankelijk | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| #24 Stroomdip tijdens pompen | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| #31 Uitschakelgrens pomp | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| §5 stap 5: Stroomdrempel afstellen | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |

### T036: US4 pompwaarschuwingen (§4 #10–#12, #14, #15, #23, #25, #26)

| Scenario | Uitgevoerd | Resultaat | Reden als niet uitgevoerd |
|---|---|---|---|
| Compile-gate | ja | geslaagd, de definitieve build (zie §T039) | |
| #10 Pomp start niet | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| #11 Pompt niet af | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| #12 Draait te lang | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| #14 Zoemer stil | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| #15 Zoemer uit | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| #23 Led-helderheid | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| #25 Stroomsensor los tijdens waarschuwing | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| #26 Herstel zonder pomp | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |

### T039: laatste compile- en resource-gate

| Controle | Uitgevoerd | Resultaat | Reden als niet uitgevoerd |
|---|---|---|---|
| `esphome config` en `esphome compile` | ja (2026-11-11, ESPHome 2026.10.1, esp-idf 5.5.6) | geslaagd; **0 waarschuwingen** in de hele build. Een eerdere build gaf één vals alarm van GCC (`-Warray-bounds` in `std::sort` op het mediaanvenster); `std::sort` is vervangen door gesorteerd invoegen | |
| RAM ≤ 75 %, flash ≤ 90 % | ja | RAM **35,3 %** (113 398 van 321 296 bytes), flash **48,5 %** (889 218 van 1 835 008 bytes) | |
| Een uur zonder "took a long time" | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| Rekentijd per script ≤ 10 ms (meting met `micros()`) | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |

### T040: review- en traceerbaarheidsgate

Uitgevoerd op 2026-11-11 op `esphome/pompput.yaml`, met de build hierboven.

| Controle (constitutie 2.0.0) | Resultaat |
|---|---|
| I: vastgepinde ESPHome, esp-idf, versleutelde API, OTA met dezelfde sleutel, geen MQTT, geen externe componenten | ✅ `requirements.txt` en `min_version: 2026.10.1`; `api: encryption`, `ota: encryption:` |
| II: geen blokkerende lambdas (`delay()`, busy-wait), `millis()` met aftrekken zonder teken, geen `new`/`malloc` | ✅ gezocht in alle lambdas: niets gevonden; elke tijdvergelijking is `(uint32_t)(now - t)` |
| II: tekst alleen bij gebeurtenissen | ✅ `str_sprintf` en `std::string` alleen bij een melding, pompstart/-stop, instelling of statuswijziging |
| II: toestandsmachines met benoemde toestanden in `globals`, overgangen in benoemde scripts | ✅ niveau (`niveau_verwerken`), pomp (`pomp_bijwerken`), meldingen (`melding_stap`) |
| II: RAM en flash | ✅ zie §T039; de rekentijd is nog open |
| III: geen uitgang naar de pomp (FR-007) | ✅ de enige uitgangen zijn TRIG, de led en de zoemer |
| III: veilige begintoestand | ✅ led `restore_mode: ALWAYS_OFF`; `ledc` begint uit; TRIG via `ultrasonic` |
| III: ongeldig of verouderd → onbekend plus storing; hoogwater vasthouden (FR-022) | ✅ `NAN` buiten GELDIG; `melding_hoogwater_beoordelen` beoordeelt niet bij een niveau anders dan GELDIG |
| III: filters en hysterese | ✅ mediaan, snelheidscontrole, inleren, debounce van 2 s, uit-hysterese 70 %, herstelmarges, `delayed_on` op de knop |
| III: watchdog niet uitgezet | ✅ geen `watchdog_timeout` en geen lambda die hem uitzet |
| IV: logica in benoemde scripts en template-entiteiten, met FR-commentaar | ✅ elk script en elke entiteit heeft een FR-commentaar; FR-017 en FR-020 staan (volgens plan) in het HA-pakket |
| IV: elke beslisregel een scenario | ✅ quickstart §4 #1–#31 en §7 |
| V: alle pinnen, tijden, grenzen en codes in `substitutions` | ✅ geen GPIO buiten het blok; een scan van alle lambdas vond geen getallen behalve 0, 1 en 2 |
| V: geheimen alleen in `secrets.yaml` | ✅ alleen `!secret`; `esphome/secrets.yaml` staat in `.gitignore` |
| V: geen `web_server`, `captive_portal` of `mqtt`; `logger` op INFO | ✅ |
| Pinnen gelijk aan `contracts/hardware.md` | ✅ GPIO3 (ACS712), 4 (TRIG), 5 (ECHO), 6 (led), 7 (zoemer), 9 (BOOT-knop) |
| Entiteiten gelijk aan `contracts/esphome-homeassistant.md` | ✅ alle 37: YAML-id en naam gecontroleerd met een script |

**Traceerbaarheid (gate 3)**: elke FR-001 t/m FR-028 heeft een YAML-element (zie de tabel in
[plan.md](plan.md#traceerbaarheid); FR-017 en FR-020 in het HA-pakket) en minstens één
scenario in [quickstart.md §7](quickstart.md#7-traceerbaarheid-requirement--scenario).

**Uitkomst**: geen schendingen van de constitutie. De afwijkingen van plan en taken staan
hieronder.

### T041: overige scenario's (§4 #16–#18, #20–#22)

| Scenario | Uitgevoerd | Resultaat | Reden als niet uitgevoerd |
|---|---|---|---|
| #16 Geen verbinding ("Pompput onbereikbaar" na 5 min) | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| #17 Herstel verbinding | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| #18 Herstart zonder HA | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| #20 Gebeurtenissen | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| #21 Langdurig netwerkverlies | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |
| #22 Update via WiFi | nee | | Vraagt het apparaat op de testbank; niet beschikbaar in de implementatiesessie van 2026-11-11 |

### T042: installatie en kalibratie (§5, SC-001)

| Controle | Uitgevoerd | Resultaat | Reden als niet uitgevoerd |
|---|---|---|---|
| Kalibratie §5 stappen 1–6 | nee | | Vraagt de installatie in de put, na de testbank |
| 10 controlemetingen (SC-001, ≤ 2 cm) | nee | | Vraagt de installatie in de put, na de testbank |

### T043: veldvalidatie (§6)

| Controle | Uitgevoerd | Resultaat | Reden als niet uitgevoerd |
|---|---|---|---|
| SC-002: 7 dagen pompbeurten | nee | | Vraagt 7 dagen gebruik in de put |
| SC-004: 30 dagen, ≤ 1 onterechte melding | nee | | Vraagt 30 dagen gebruik in de put |
| SC-006: dashboard | nee | | Vraagt het apparaat en HA; het dashboard zelf is bijgewerkt (T037) |

## Afwijkingen

Van plan en taken, bij de implementatie op 2026-11-11. Geen ervan is een afwijking van de
constitutie.

| # | Afwijking | Waarom | Bijgewerkt |
|---|---|---|---|
| 1 | Secties 3–8 zijn pakketten binnen `packages:` in hetzelfde bestand | ESPHome staat een sleutel als `sensor:` maar één keer toe | plan.md (Structure Decision), tasks.md (conventies) |
| 2 | YAML-id `uptime_s` in plaats van `uptime` | ESPHome weigert een id met de naam van een gebruikte integratie; de entity-id `sensor.pompput_uptime` blijft | contract |
| 3 | De rekenkundige 1 en 2 mogen letterlijk (ophogen, een bit maken, de mediaan) | Zonder die getallen kan de mediaan niet berekend worden; het zijn geen afstelwaarden | tasks.md (conventies, T040) |
| 4 | Extra substitutions: `log_tag`, `adc_demping`, `knop_debounce`, `led_aantal`, `led_gamma` | Geen losse waarden in de configuratie. `led_gamma: 1.0`: met de standaard-gamma 2,8 zou oranje (255,80,0) vrijwel rood worden | — |
| 5 | Gedeelde uitdrukkingen in substitutions: `pomp_draait_nu`, `pomp_staat_stil`, `niveau_bruikbaar` | Eén definitie voor de drie pompwaarschuwingen, de led en de tekst van een melding (constitutie IV) | — |
| 6 | `on_boot` met `priority: -100`, en publiceert de zeven meldingen als "uit" | Alle componenten (ook de bewaarde instellingen en de led) zijn dan opgezet; HA toont de meldingen dan als "uit" in plaats van "onbekend" | — |
| 7 | Global `g_eenmalig_bezig` | Voorkomt dat een herhaald zoemerpatroon over een eenmalig patroon heen speelt | — |
| 8 | `herstart_melden` start niet opnieuw zolang het al loopt | Twee clients kort na elkaar (bijvoorbeeld HA en `esphome logs`) zouden anders een waarschuwing in het log geven | — |
| 9 | Het voorbeeldwachtwoord in `secrets.example.yaml` is langer gemaakt | ESPHome eist voor WPA minstens 8 tekens, ook bij `esphome config` | — |
| 10 | Mediaan met gesorteerd invoegen in plaats van `std::sort` | `std::sort` gaf een vals alarm van GCC (`-Warray-bounds`), en gate 1 eist nul waarschuwingen | — |
| 11 | Geen `on_value` op Led-helderheid (T015 vroeg dat wel) | Die trigger gaat al af tijdens `setup()` en zette de led aan voordat de strip was opgezet: een crashlus op het apparaat. `led_patroon_kiezen` neemt een nieuwe helderheid binnen 1 s over | esphome/pompput.yaml |

## Open bankpunten

Uit research, "Open punten" 1. Vast te leggen bij de eerste controles op de testbank (T019,
T025, T032).

| Punt | Hoe te controleren | Uitkomst |
|---|---|---|
| TRIG op 3,3 V (R8) | Afstand (ruw) geeft kloppende waarden | ✅ 2026-11-11: stabiele echo's van 0,614–0,632 m, vier per seconde (log via de API) |
| ADC met `12db` en de ACS712 in rust (R7) | Offset stroomsensor ≈ 1,25 V; Pompstroom met en zonder belasting | open. 2026-11-11: 2,54 V, maar de ACS712 was nog **niet aangesloten** (zwevende pin); Stroomsensor storing is dan het verwachte gedrag (scenario #13). Meten zodra de ACS712 met de deler is aangesloten |
| Led-data op 3,3 V met de diodevoeding (R8) | De led reageert betrouwbaar | |
| Kleurvolgorde van de led (R9) | "Pomp draait" is blauw, niet groen; anders `led_kleurvolgorde` aanpassen | |
