# Research: Stroomsensor verwijderen

**Feature**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md) | **Datum**: 2026-11-18

Na het invullen van de Technical Context waren er geen open punten ("NEEDS CLARIFICATION"). De
omvang is in de spec vastgelegd (Clarifications 2026-11-18: alle pompbewaking vervalt). De
beslissingen hieronder leggen vast hoe de verwijdering gebeurt en waarom dat veilig is.

## R1. Wat verdwijnt uit de firmware

- **Decision**: alles wat alleen de stroommeting, de pompstatus of de pompwaarschuwingen dient,
  gaat weg uit `esphome/pompput.yaml`.
  - **Sectie 4 (pakket `pomp`) in zijn geheel**: `adc` `acs712_spanning`, `ct_clamp`
    `pompstroom_rms`, de template-sensoren `looptijd_laatste_pompbeurt`, `pompstroom` en
    `offset_stroomsensor`, de binary sensor `pomp`, het script `pomp_bijwerken`, het
    `interval` voor de diagnose, en de globals `g_offset_v`, `g_offset_ok`,
    `g_pomp_start_ms`, `g_pomp_stop_kandidaat_ms`, `g_niveau_start_cm`, `g_looptijd_s` en
    `g_stroom_a`.
  - **Sectie 5 (meldingen)**: de binary sensors `pomp_start_niet`, `pomp_pompt_niet_af`,
    `pomp_draait_te_lang` en `stroomsensor_storing`; de scripts
    `melding_pomp_start_niet_beoordelen`, `melding_pomp_pompt_niet_af_beoordelen`,
    `melding_pomp_draait_te_lang_beoordelen` en `melding_stroomsensor_storing_beoordelen` (en
    hun aanroep in `meldingen_beoordelen`); de global `g_pomp_status`.
  - **Sectie 6 (instellingen)**: de numbers `inschakelniveau_cm`, `start_marge_cm`,
    `start_wachttijd_s`, `afpomp_controletijd_s`, `afpomp_min_daling_cm`, `max_looptijd_min`
    en `pomp_stroomdrempel_a`; de global `g_inschakelniveau_cm` (R3).
  - **Sectie 7 (signalering)**: de patronen "waarschuwing" en "pomp draait" in
    `led_patroon_kiezen` (R6).
  - **Sectie 8 (gebeurtenissen)**: de event-types `pomp_start` en `pomp_stop` (R8).
  - **Sectie 2 (`on_boot`)**: `component.update: inschakelniveau_cm`, en de vier vervallen
    meldingen in de lijst die op `false` wordt gezet.
  - **Substitutions**: `pin_acs712`; het hele blok "Pomp" (`adc_interval`, `adc_demping`,
    `stroom_venster`, `stroom_interval`, `offset_venster`, `offset_elke`, `acs712_a_per_v`,
    `pomp_debounce_ms`, `pomp_uit_factor`, `offset_min_v`, `offset_max_v`,
    `offset_storing_ms`); `start_niet_herstel_cm`; de bereiken en standaardwaarden van de zeven
    vervallen instellingen; de pompstatuscodes `pomp_*`; `idx_pomp_*`,
    `idx_stroomsensor_storing` en `bits_waarschuwing`; `led_oranje_*`,
    `led_waarschuwing_*_ms`, `led_pomp_*_ms`, `led_patroon_waarschuwing` en
    `led_patroon_pomp`; `ms_per_min` (werd alleen door "pomp draait te lang" gebruikt); en de
    gedeelde uitdrukkingen `pomp_draait_nu` en `pomp_staat_stil`.
- **Rationale**: constitutie V (YAGNI: ongebruikte componenten en entiteiten worden verwijderd)
  en IV (geen dode logica). Spec FR-001 t/m FR-005.
- **Alternatives considered**:
  - *Het pakket `pomp` laten staan en de entiteiten `internal` maken*: de ADC blijft dan elke
    100 ms meten, en zonder sensor geeft de offsetcontrole een stroomsensorstoring. Strijdig met
    FR-001, FR-012 en US1 scenario 1.
  - *Alleen de storingsmelding van de stroomsensor uitschakelen*: de pompstatus zou dan altijd
    "stil" zijn, en "pomp start niet" zou vals afgaan. Verworpen.

## R2. Meldingen: van zeven naar drie

- **Decision**: het apparaat houdt drie meldingen over, met nieuwe nummers (de index in de
  arrays en het bit in `g_actieve_set`):

  | Index | Melding | Soort | Bitgroep |
  |---|---|---|---|
  | 0 | `hoogwater` | alarm | `bits_alarm` = `0x01` |
  | 1 | `niveausensor_storing` | storing | `bits_storing` = `0x06` |
  | 2 | `niet_ingesteld` | storing | (bit 2, ook apart in `status_bijwerken`) |

  `aantal_meldingen` wordt 3. De tabellen `sensoren[]` en `namen[]` in `melding_stap` worden
  ingekort. De statustekst in `status_bijwerken` wordt ALARM > NIET_INGESTELD > STORING >
  NORMAAL; de waarde `waarschuwing` vervalt uit de text sensor `status`.
- **Rationale**: de nummers zijn alleen intern en worden niet bewaard (`restore_value: false`),
  dus hernummeren heeft geen gevolgen voor bewaarde gegevens of voor Home Assistant. Een
  aaneengesloten nummering houdt de arrays klein en de code leesbaar. De generieke
  levenscyclus in `melding_stap` blijft ongewijzigd (spec FR-006).
- **Alternatives considered**: *de oude nummers met gaten houden*: arrays van 7 met vier
  ongebruikte plaatsen, en bitmaskers die naar niet-bestaande meldingen verwijzen. Verwarrend
  voor een lezer; verworpen.

## R3. "Ingesteld" zonder inschakelniveau

- **Decision**: het inschakelniveau was een verplichte instelling. Na de wijziging zijn alleen
  `afstand_leeg_cm` en `afstand_vol_cm` verplicht. De regels worden:
  - `afstand_leeg_cm` ≥ `afstand_vol_cm` + 10 cm (ongewijzigd);
  - `alarmdrempel_pct` − `herstelmarge_pct` > 0 (ongewijzigd);
  - de regel "`inschakelniveau_cm` < meetbereik" vervalt.

  `ingesteld_bepalen` en `instelling_toepassen` verliezen de tak voor `inschakelniveau_cm`;
  `instelling_toepassen` kent nog vier sleutels. De tekst van het alert "niet ingesteld" in
  Home Assistant noemt alleen nog Afstand leeg en Afstand vol.
- **Rationale**: het inschakelniveau werd alleen door "pomp start niet" gebruikt (001
  data-model §5). Een apparaat dat vóór de update alle drie de waarden had, blijft ingesteld.
  Een apparaat dat alleen het inschakelniveau miste, wordt na de update ingesteld; dat is
  gewenst, want die waarde is niet meer nodig.
- **Alternatives considered**: *het inschakelniveau als optionele instelling houden*: een
  instelling zonder functie; strijdig met FR-004.

## R4. Bewaarde instellingen na de OTA-update

- **Decision**: geen migratie nodig. De zeven overgebleven instellingen blijven behouden (spec
  FR-011, SC-005); dit wordt op het apparaat gecontroleerd (quickstart §3).
- **Rationale**: ESPHome slaat een bewaarde global op onder een sleutel die is afgeleid van de
  id (`md5(id)`, `esphome/components/globals/__init__.py`), en een template-number onder een
  sleutel die is afgeleid van de entiteit (`make_entity_preference`). De volgorde in de YAML
  speelt geen rol. Het verwijderen van `g_inschakelniveau_cm` en de zes numbers verandert de
  sleutels van de andere waarden dus niet. De oude waarden blijven als wees in flash staan; ze
  zijn samen enkele tientallen bytes en worden nergens meer gelezen. Dezelfde redenering is bij
  002 (research R3) op het apparaat bevestigd.
- **Alternatives considered**: *flash wissen en opnieuw instellen*: onnodig werk voor de
  gebruiker en strijdig met SC-005.

## R5. De vrijgekomen pin GPIO3 (A1) en het weghalen van de sensor

- **Decision**: GPIO3 wordt niet meer geconfigureerd en krijgt geen nieuwe functie (FR-012).
  `contracts/hardware.md` van 001 markeert D1/A1 als "vrij". Er komt een paragraaf "Een
  bestaande stroomsensor weghalen" in `hardware.md` en in de README (FR-015).
- **Rationale**:
  - Een niet-geconfigureerde pin staat na een reset als hoog-ohmige ingang. Een nog
    aangesloten ACS712 geeft via de deler 10 kΩ / 10 kΩ hoogstens ongeveer 2,5 V op de pin
    (de helft van 5 V), binnen het bereik van een 3,3 V-ingang. De sensor mag dus blijven
    zitten (randgeval "de stroomsensor zit nog aangesloten").
  - De ACS712 zit in de fasedraad van de pomp: de stroom van de pomp loopt door de sensor.
    Wie de sensor weghaalt, moet de fasedraad weer doorverbinden, anders krijgt de pomp geen
    stroom meer. Dat is werk aan netspanning; de constitutie (Embedded Platform Constraints,
    "Mains") vraagt een bevoegd persoon of toezicht daarop.
- **Alternatives considered**:
  - *GPIO3 als uitgang laag zetten*: zou tegen de uitgang van een nog aangesloten ACS712 in
    sturen. Verworpen.
  - *GPIO3 met een interne pull configureren*: geen functie; YAGNI.

## R6. Led 1: twee patronen minder

- **Decision**: de patronen "waarschuwing" (oranje) en "pomp draait" (langzaam blauw) vervallen
  uit `led_patroon_kiezen`. De overgebleven patronen houden hun kleur, ritme en volgorde (spec
  FR-007). De codes worden aaneengesloten:

  | Code | Patroon | Prio (003 contracts/ledbalk.md §1) |
  |---|---|---|
  | 0 | geen (dwingt een nieuwe keuze af) | — |
  | 1 | alarm | 1 |
  | 2 | storing (ook "niveau onbekend") | 2 en 4 |
  | 3 | opstarten | 3 |
  | 4 | geen verbinding | 5 |
  | 5 | normaal | 6 |

  De niveaubalk (led 2–6) verandert niet.
- **Rationale**: zonder pompwaarschuwingen is er geen melding van de soort "waarschuwing" meer,
  en zonder pompstatus kan "pomp draait" niet meer voorkomen (spec FR-005). Een patroon dat
  nooit kan verschijnen, is dode code (constitutie IV).
- **Alternatives considered**: *de oude codes met gaten houden*: de codes zijn intern en worden
  niet bewaard; gaten maken de volgorde onduidelijk. Verworpen.

## R7. Home Assistant: entiteiten, pakket en dashboard

- **Decision**:
  - **Entiteiten van het apparaat**: 35 → **20** (de lijst staat in
    [contracts/wijzigingen-firmware-en-ha.md §1](contracts/wijzigingen-firmware-en-ha.md#1-entiteiten-van-het-apparaat)).
  - **Pakket** (`homeassistant/packages/pompput.yaml`): de `history_stats`-sensor "Pompput
    pompstarts 24 uur" en de alerts `pompput_pomp_start_niet`, `pompput_pomp_pompt_niet_af`,
    `pompput_pomp_draait_te_lang` en `pompput_stroomsensor_storing` vervallen. In de vier
    overgebleven alerts vervalt de regel `Pomp: …`. Het alert "niet ingesteld" noemt alleen
    nog Afstand leeg en Afstand vol, en "geen vulgraad" in plaats van "geen vulgraad en geen
    pompbewaking".
  - **Dashboard** (`homeassistant/dashboard-pompput.yaml`): de regels Pomp, Pompstarts 24 uur
    en Looptijd laatste pompbeurt in de hoofdkaart; Pomp in de grafiek; de sectie "Pomp" en
    Inschakelniveau in de instellingen; Offset stroomsensor en Pompstroom bij de diagnose. De
    kopcommentaar noemt SC-006 alleen nog voor de vulgraad.
  - **Verweesde entiteiten**: de ESPHome-integratie ruimt entiteiten op die het apparaat na
    een herstart niet meer meldt. De `history_stats`-sensor en de vier alerts verdwijnen als
    het pakket opnieuw wordt geladen. Blijft er toch een verweesde entiteit staan, dan
    verwijdert de gebruiker die eenmalig onder Instellingen → Entiteiten (zoals bij 002,
    research R4). De quickstart controleert dit (SC-004).
  - **Historie**: de opgeslagen pompgegevens blijven in de recorder van Home Assistant tot de
    gewone bewaartermijn ze opruimt (spec, randgeval).
- **Rationale**: spec FR-003, FR-009, FR-010, FR-013.
- **Alternatives considered**: *de pomp-alerts in het pakket laten staan*: ze zouden naar
  niet-bestaande entiteiten verwijzen. Verworpen.

## R8. Gebeurtenissen

- **Decision**: de event-types `pomp_start` en `pomp_stop` vervallen uit de entiteit
  `event.pompput_gebeurtenis`. De tekst bij `melding_actief` wordt `<melding>, niveau <x> cm`
  (of `niveau onbekend`), zonder `pomp …`.
- **Rationale**: spec FR-009 en FR-010; het script `pomp_bijwerken`, dat deze gebeurtenissen
  meldde, vervalt (R1).
- **Alternatives considered**: geen.

## R9. Welke documenten worden bijgewerkt

- **Decision**: zoals bij 002 (research R5): de **levende** documenten worden bijgewerkt, de
  **historische** blijven staan.
  - **Levend, 001**: `spec.md` (de vervallen requirements, user stories en success criteria
    krijgen "(vervallen met 004)" of "(gewijzigd door 004)"; een sessie "004" in de
    Clarifications), `data-model.md` (§1 instellingen, §4 pomp, §5 meldingen, §6
    gebeurtenissen), `quickstart.md`, `contracts/hardware.md`,
    `contracts/esphome-homeassistant.md` en `contracts/lokale-signalering.md`.
  - **Levend, 003**: `spec.md` (FR-003, FR-014, US2 scenario 3, 7 en 8, SC-004),
    `contracts/ledbalk.md` (§1 tabel en de voorbeelden met "pomp draait"), `data-model.md` en
    `quickstart.md`.
  - **Overig**: `README.md`, `homeassistant/packages/pompput.yaml`,
    `homeassistant/dashboard-pompput.yaml`.
  - **Historisch** (blijft staan): `plan.md`, `research.md`, `tasks.md` en `verificatie.md`
    van 001, 002 en 003. Bovenaan `plan.md` van 001 en 003 komt één regel dat de stroomsensor
    en de pompbewaking met 004 zijn vervallen.
  - **Uitkomsten van de gates** komen in `specs/004-stroomsensor-verwijderen/verificatie.md`.
- **Rationale**: spec FR-014 en SC-006; historische verslagen herschrijven zou de geschiedenis
  vervalsen.
- **Alternatives considered**: *alle documenten herschrijven*: veel werk zonder waarde.

## R10. Constitutie bijwerken

- **Decision**: een PATCH-wijziging (2.0.1 → **2.0.2**) via `/speckit-constitution`, met
  goedkeuring van de maintainer, als eerste taak van de uitvoering:
  - Principle II: het voorbeeld "pump status" vervalt uit de opsomming van toestandsmachines.
  - Principle IV: "pump detection" vervalt uit de opsomming van beslislogica.
  - Embedded Platform Constraints, "Mains": alleen "(the ACS712 current sensor)" vervalt; de
    regel blijft verder letterlijk gelijk ("any part on mains voltage … is measurement only …").
    Er komt geen nieuwe eis bij: het veilig weghalen staat in `hardware.md` (FR-015). Een
    toegevoegde eis zou de voorschriften uitbreiden en daarmee een MINOR-wijziging zijn.
- **Rationale**: de huidige constitutie verbiedt de verwijdering niet; ze noemt de stroomsensor
  alleen als voorbeeld. Daarom is dit een verduidelijking zonder betekenisverandering (PATCH)
  en slaagt de Constitution Check al vóór de wijziging. Zonder bijwerking zou de constitutie
  een onderdeel beschrijven dat er niet meer is.
- **Alternatives considered**: *de constitutie laten staan*: verouderde voorbeelden misleiden
  een lezer, en de spec noemt de bijwerking als afhankelijkheid. Verworpen.

## R11. Versie en opmerkingen in de YAML

- **Decision**: `project_version` gaat van `0.4.0` naar `0.5.0`. De kopcommentaar van
  `esphome/pompput.yaml` noemt geen ACS712 meer, telt drie meldingen ("hoogwater, een
  sensorstoring en niet ingesteld"), en de sectielijst nummert opnieuw: 1 substitutions,
  2 basis, 3 niveau, 4 meldingen, 5 instellingen, 6 signalering, 7 gebeurtenissen en diagnose.
  De verwijzing naar de specs noemt ook `specs/004-stroomsensor-verwijderen/`. FR-verwijzingen
  naar vervallen requirements (FR-006, FR-008, FR-009, FR-011, FR-013 t/m FR-015) verdwijnen
  met de code.
- **Rationale**: constitutie IV (traceerbaarheid) en een zichtbaar versieverschil in HA
  (apparaatinfo), zodat te controleren is dat de nieuwe firmware draait.
- **Alternatives considered**: *de sectienummers met een gat houden* (sectie 4 leeg): verwarrend
  in een bestand dat op nummers leunt. Verworpen.

## R12. Effect op RAM, flash en rekentijd

- **Decision**: geen maatregel nodig; de resource-gate wordt opnieuw gemeten en vastgelegd.
- **Rationale**: er komt niets bij. De componenten `adc` en `ct_clamp`, zes sensoren in sectie 4
  (waarvan vier zichtbaar in HA), vier meldingen, zeven numbers, vier scripts en negen globals vervallen. RAM en
  flash kunnen alleen dalen (vorige meting, 0.4.0: RAM 35,1 %, flash 47,4 %). De rekentijd
  daalt: de ADC meet niet meer elke 100 ms, `ct_clamp` niet meer elke 500 ms, en per seconde
  draaien vier beoordelingsscripts minder.
