---

description: "Takenlijst voor het verwijderen van de stroomsensor en de pompbewaking"
---

# Tasks: Stroomsensor verwijderen

**Input**: Design documents from `specs/004-stroomsensor-verwijderen/`

**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md),
[data-model.md](data-model.md), [contracts/wijzigingen-firmware-en-ha.md](contracts/wijzigingen-firmware-en-ha.md),
[quickstart.md](quickstart.md)

**Tests**: geen geautomatiseerde tests (constitutie IV). Controle gebeurt met de compile- en
resource-gate en de scenario's op het apparaat in [quickstart.md](quickstart.md).

**Organization**: taken per user story. Alle firmwarewerk zit in US1: het pakket `pomp`, de
pompmeldingen, de pompinstellingen en de led-patronen verwijzen naar elkaar (`g_pomp_status`,
`pomp_draait_nu`, `bits_waarschuwing`), dus de firmware compileert pas weer als alles weg is.
Daarom zitten ook de led-wijzigingen van US3 (FR-005, FR-007) in de firmware van US1; US3 doet
de documentatie en de controle van de led. Het pinnencontract `hardware.md` zit in US1, omdat de
constitutie eist dat het `substitutions`-blok en `hardware.md` gelijk lopen. US2 doet de Home
Assistant-kant, US4 de overige documentatie.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: kan parallel (ander bestand, geen afhankelijkheid van een open taak)
- **[Story]**: US1 t/m US4 uit [spec.md](spec.md)

## Path Conventions

- Firmware: `esphome/pompput.yaml` (nu genummerde secties 1–8; na deze feature 1–7)
- Home Assistant: `homeassistant/packages/pompput.yaml`, `homeassistant/dashboard-pompput.yaml`
- Documentatie: `README.md`, `specs/001-pompput-monitor/`, `specs/003-niveau-ledbalk/`,
  `specs/004-stroomsensor-verwijderen/`
- Commando's draaien in PowerShell vanuit de root van de repository met `.venv\Scripts\esphome`

---

## Phase 1: Setup

**Purpose**: vastleggen hoe het apparaat er vóór de wijziging voor staat, zodat FR-011, SC-005
en de resource-gate te controleren zijn.

- [X] T001 Maak `specs/004-stroomsensor-verwijderen/verificatie.md` met de koppen: "Samenvatting van de gates" (tabel gate 1–5 met kolommen Gate, Uitkomst, Datum), "Uitgangssituatie", "Compile- en resource-gate", "Bijwerken" (quickstart §3 stap 1–6), "Scenario's op het apparaat" (tabel #1–#13 uit quickstart §4 met kolommen #, Uitgevoerd ja/nee, Uitkomst, Reden als niet uitgevoerd), "Documentatie" (quickstart §5), "Review- en traceerbaarheidsgate" en "Afwijkingen"
- [X] T002 Noteer onder "Uitgangssituatie" in `specs/004-stroomsensor-verwijderen/verificatie.md`: de laatste RAM- en flashwaarden van firmware 0.4.0 uit `specs/003-niveau-ledbalk/verificatie.md` (RAM 35,1 %, 112 848 van 321 296 bytes; flash 47,4 %, 870 384 van 1 835 008 bytes), en de actuele waarden in Home Assistant van de zeven instellingen die blijven: Afstand leeg, Afstand vol, Alarmdrempel, Herstelmarge, Wachttijd hoogwater, Wachttijd sensorstoring en Led-helderheid. Noteer ook of er op dat moment een pompwaarschuwing of stroomsensorstoring actief is (voor quickstart §3 stap 6). Kan het apparaat niet worden uitgelezen, schrijf dat op met de reden (constitutie IV)

---

## Phase 2: Foundational

**Purpose**: de constitutie in lijn brengen (research R10). Dit blokkeert de code niet, maar de
spec noemt het als afhankelijkheid en de review-gate (T034) toetst aan de bijgewerkte versie.

- [X] T003 Werk de constitutie bij met `/speckit-constitution` naar versie **2.0.2** (PATCH), met de reden "stroomsensor vervallen met 004-stroomsensor-verwijderen" en een Sync Impact Report. Wijzigingen in `.specify/memory/constitution.md`: (1) Principle II, derde punt: "(level status, pump status, alert lifecycle)" wordt "(level status, alert lifecycle)"; (2) Principle IV, eerste punt: "(level validity, pump detection, alerts, local indication, settings validation)" wordt "(level validity, alerts, local indication, settings validation)"; (3) Embedded Platform Constraints, "Mains": schrap alleen "(the ACS712 current sensor)", zodat de regel luidt "any part on mains voltage is measurement only, MUST be documented with its safety requirements in the hardware contract, and is connected by or under supervision of a qualified person." Voeg geen nieuwe eis toe; dan blijft het een PATCH (Governance, Versioning). Het veilig weghalen staat in `hardware.md` (T012, FR-015) **De maintainer moet de wijziging goedkeuren** (Governance); zonder goedkeuring blijft 2.0.1 staan en wordt dat onder "Afwijkingen" in `specs/004-stroomsensor-verwijderen/verificatie.md` genoteerd

**Checkpoint**: constitutie 2.0.2 goedgekeurd (of de afwijking genoteerd).

---

## Phase 3: User Story 1 - De bewaking werkt zonder stroomsensor (Priority: P1) 🎯 MVP

**Goal**: firmware 0.5.0 zonder stroommeting, pompstatus, pompwaarschuwingen, pompinstellingen
en de led-patronen "waarschuwing" en "pomp draait"; de niveaubewaking werkt ongewijzigd; het
pinnencontract klopt met de firmware (FR-001 t/m FR-008, FR-011, FR-012, FR-015).

**Independent Test**: compile- en resource-gate ([quickstart.md](quickstart.md) §2), bijwerken
via WiFi (§3 stap 1–3) en de scenario's §4 #1–#8, #11 en #12: geen storing zonder sensor, hoogwater
en niveausensor storing zoals voorheen, instellingen behouden.

De taken T004–T013 wijzigen `esphome/pompput.yaml` en lopen dus na elkaar. T012 wijzigt
`hardware.md` en hoort direct na T011, zodat pinlijst en contract samen veranderen.

- [X] T004 [US1] Verwijder in `esphome/pompput.yaml` het hele pakket `pomp:` (sectie 4, van de bannerregel "---- 4. Pomp ----" tot vóór de banner "---- 5. Meldingen ----"), inclusief de commentaarregels "FR-006 t/m FR-009 …" en "Alleen meten: de ACS712 …". Daarmee verdwijnen de globals `g_offset_v`, `g_offset_ok`, `g_pomp_start_ms`, `g_pomp_stop_kandidaat_ms`, `g_niveau_start_cm`, `g_looptijd_s`, `g_stroom_a`, de sensoren `acs712_spanning` (`adc`), `pompstroom_rms` (`ct_clamp`), `looptijd_laatste_pompbeurt`, `pompstroom`, `offset_stroomsensor`, de binary sensor `pomp`, het script `pomp_bijwerken` en het `interval` voor de diagnose (research R1; FR-001, FR-003, FR-012)
- [X] T005 [US1] Werk het pakket `meldingen:` (sectie 5) in `esphome/pompput.yaml` bij (research R2; FR-002, FR-009): (1) verwijder de binary sensors `pomp_start_niet`, `pomp_pompt_niet_af`, `pomp_draait_te_lang` en `stroomsensor_storing`; de FR-regel erboven wordt `# FR-012, FR-016, FR-018, randgeval "niet ingesteld"`; (2) verwijder de scripts `melding_stroomsensor_storing_beoordelen`, `melding_pomp_start_niet_beoordelen`, `melding_pomp_pompt_niet_af_beoordelen` en `melding_pomp_draait_te_lang_beoordelen` met hun commentaar, ook het blok "De drie pompwaarschuwingen (FR-013 t/m FR-015) samen"; (3) `meldingen_beoordelen` roept alleen nog `melding_niet_ingesteld_beoordelen`, `melding_hoogwater_beoordelen`, `melding_niveausensor_storing_beoordelen` en `status_bijwerken` aan; de FR-regel wordt `# FR-012, FR-016, FR-018`; (4) verwijder de global `g_pomp_status`; het commentaar boven `g_hoogte_cm` wordt "Waterhoogte, hier al gedeclareerd omdat melding_stap hem in de tekst van een melding gebruikt; sectie 3 vult hem."; (5) in `melding_stap`: `sensoren[]` wordt `{id(hoogwater), id(niveausensor_storing), id(niet_ingesteld)}` en `namen[]` wordt `{"hoogwater", "niveausensor_storing", "niet_ingesteld"}` (deze volgorde = de nieuwe nummers 0, 1, 2); verwijder de variabele `pomp` en zet de tekst op `str_sprintf("%s, niveau onbekend", namen[index])` en `str_sprintf("%s, niveau %.1f cm", namen[index], hoogte)`; het commentaar wordt "De tekst bij een melding noemt de melding en het niveau (FR-019; 004 FR-009)" en "Eén script voor alle drie"; (6) in `status_bijwerken` vervalt de tak `(actief & ${bits_waarschuwing}) ? "waarschuwing"`; het commentaar wordt "ALARM > NIET_INGESTELD > STORING > NORMAAL"
- [X] T006 [US1] Werk `on_boot` (sectie 2) in `esphome/pompput.yaml` bij: verwijder `- component.update: inschakelniveau_cm`; het commentaar van stap 2 wordt "De vier instellingen met regels publiceren hun waarde, of "onbekend" (FR-023)."; de lijst in de lambda van stap 3 wordt `{id(hoogwater), id(niveausensor_storing), id(niet_ingesteld)}` (FR-002, FR-004)
- [X] T007 [US1] Werk het pakket `instellingen:` (sectie 6) in `esphome/pompput.yaml` bij (research R3; FR-004, FR-008, FR-011): (1) verwijder de global `g_inschakelniveau_cm`; het commentaar wordt "De vier instellingen met onderlinge regels …"; (2) verwijder de numbers `inschakelniveau_cm`, `start_marge_cm`, `start_wachttijd_s`, `afpomp_controletijd_s`, `afpomp_min_daling_cm`, `max_looptijd_min` en `pomp_stroomdrempel_a` met hun FR-regels; (3) `ingesteld_bepalen`: alleen `leeg` en `vol` verplicht: `id(g_ingesteld) = !std::isnan(leeg) && !std::isnan(vol) && leeg >= vol + ${geometrie_min_verschil_cm} && id(g_alarmdrempel_pct) - id(g_herstelmarge_pct) > 0;`; (4) `instelling_toepassen`: verwijder de variabele `inschakel`, de tak `sleutel == "inschakelniveau_cm"` en de regel "inschakelniveau_cm moet kleiner zijn dan het meetbereik"; het commentaar zegt "de vier waarden met de nieuwe waarde". Raak de id's `g_afstand_leeg_cm`, `g_afstand_vol_cm`, `g_alarmdrempel_pct`, `g_herstelmarge_pct`, `hoogwater_wachttijd_s`, `sensorstoring_wachttijd_s` en `led_helderheid_pct` **niet** aan (research R4)
- [X] T008 [US1] Werk `led_patroon_kiezen` in het pakket `signalering:` (sectie 7) van `esphome/pompput.yaml` bij (research R6; FR-005, FR-007): verwijder de regels `else if (actief & ${bits_waarschuwing}) patroon = ${led_patroon_waarschuwing};` en `else if (${pomp_draait_nu}) patroon = ${led_patroon_pomp};` en de `case`-blokken `${led_patroon_waarschuwing}` en `${led_patroon_pomp}`. De overige regels, kleuren, ritmes en `led_min_helderheid_alarm` blijven letterlijk gelijk. De FR-regel boven het script wordt `# FR-027; 002 FR-002, FR-003; 003 FR-002, FR-003, FR-013, FR-017; 004 FR-005, FR-007; contracts/ledbalk.md §1`
- [X] T009 [US1] Verwijder in het pakket `gebeurtenissen:` (sectie 8) van `esphome/pompput.yaml` de `event_types` `pomp_start` en `pomp_stop` (research R8; FR-010)
- [X] T010 [US1] Controleer dat geen lambda meer verwijst naar `g_pomp_status`, `pomp_draait_nu`, `pomp_staat_stil`, `pomp_stroomdrempel_a`, `start_marge_cm`, `g_inschakelniveau_cm` of `idx_pomp_` (`Select-String -Path esphome/pompput.yaml -Pattern 'g_pomp_status|pomp_draait_nu|pomp_staat_stil|pomp_stroomdrempel|start_marge|g_inschakel|idx_pomp_|idx_stroom'`); de enige treffers mogen nog in het `substitutions:`-blok staan (dat volgt in T011)
- [X] T011 [US1] Werk het `substitutions:`-blok (sectie 1) van `esphome/pompput.yaml` bij (research R1, R2, R6, R11): (1) verwijder `pin_acs712` en zet bij de pinnen het commentaar "D1/A1 (GPIO3) is vrij sinds 004 en wordt niet geconfigureerd; een nog aangesloten ACS712 doet geen kwaad (004 research R5)." naast de bestaande regel over D5 en D9; (2) verwijder het hele blok "Pomp (data-model §4, research R7)" (`adc_interval` t/m `offset_storing_ms`); (3) verwijder `start_niet_herstel_cm`; (4) verwijder de bereiken en standaardwaarden `inschakelniveau_cm_*`, `start_marge_cm_*`, `start_wachttijd_s_*`, `afpomp_controletijd_s_*`, `afpomp_min_daling_cm_*`, `max_looptijd_min_*` en `pomp_stroomdrempel_a_*`; (5) verwijder `led_oranje_r/g/b/w`, `led_waarschuwing_aan_ms`, `led_waarschuwing_uit_ms`, `led_pomp_aan_ms`, `led_pomp_uit_ms` en het commentaar "Pomp draait: blauw 1 s aan / 1 s uit"; (6) verwijder het blok "Pompstatus (data-model §4)" (`pomp_onbekend` t/m `pomp_stop_kandidaat`); (7) meldingsnummers worden `idx_hoogwater: "0"`, `idx_niveausensor_storing: "1"`, `idx_niet_ingesteld: "2"`, `aantal_meldingen: "3"`; verwijder `idx_pomp_start_niet`, `idx_pomp_pompt_niet_af`, `idx_pomp_draait_te_lang` en `idx_stroomsensor_storing`; het commentaar verwijst naar "004 data-model §3"; (8) bitgroepen: `bits_alarm: "0x01"  # bit 0`, `bits_storing: "0x06"  # bits 1-2`; verwijder `bits_waarschuwing`; (9) ledpatronen worden `led_patroon_geen: "0"`, `led_patroon_alarm: "1"`, `led_patroon_storing: "2"`, `led_patroon_opstarten: "3"`, `led_patroon_geen_verbinding: "4"`, `led_patroon_normaal: "5"`; verwijder `led_patroon_waarschuwing` en `led_patroon_pomp`; (10) verwijder `ms_per_min`; (11) verwijder de gedeelde uitdrukkingen `pomp_draait_nu` en `pomp_staat_stil` met hun commentaar; (12) zet `project_version` op `"0.5.0"`
- [X] T012 [US1] Werk `specs/001-pompput-monitor/contracts/hardware.md` bij, tegelijk met T011 (constitutie, Embedded Platform Constraints): (1) de inleiding noemt `pin_acs712` niet meer en meldt "D1/A1 is vrij sinds 004-stroomsensor-verwijderen"; (2) in de pintabel wordt de rij D1/A1 `| D1 / A1 | 3 | — | vrij (was ACS712 OUT, vervallen met 004) | — |` en de rij 5V noemt geen ACS712 VCC meer; (3) vervang de paragraaf "ACS712, 30 A-variant (pompstroom), netspanning" door "Een bestaande stroomsensor weghalen" met de vijf punten uit [contracts §4](contracts/wijzigingen-firmware-en-ha.md#4-hardware) (blijven zitten mag; netspanning, vakbekwaam persoon, pompgroep spanningsloos; fasedraad doorverbinden in een afgesloten behuizing, N en PE ongewijzigd; daarna controleren dat de pomp op zijn vlotter aan- en afslaat; deler en draden weghalen of geïsoleerd laten); (4) in "Voeding" vervalt "ACS712 ongeveer 10 mA"; (5) in "Veilige begintoestand" komt "GPIO3 wordt niet geconfigureerd; een nog aangesloten ACS712 geeft via de deler hoogstens ongeveer 2,5 V (004 research R5)." (FR-012, FR-015)
- [X] T013 [US1] Werk de kopcommentaar en de sectienummers van `esphome/pompput.yaml` bij (research R11): (1) eerste punt: "Een HC-SR04 meet van bovenaf het waterniveau." (zonder ACS712); (2) tweede punt: "Het apparaat bepaalt zelf drie meldingen: hoogwater, een storing van de niveausensor en "niet ingesteld"."; (3) de inhoudsopgave wordt: 1 substitutions, 2 basis, 3 niveau, 4 meldingen, 5 instellingen, 6 signalering, 7 gebeurtenissen en diagnose (met dezelfde onderdelen per regel als nu, zonder `pomp`); "de secties 3-8" wordt "de secties 3-7"; (4) de regel "Spec, plan, contracten en validatie" noemt ook `specs/004-stroomsensor-verwijderen/`; (5) hernummer de bannerregels: "5. Meldingen" → "4. Meldingen", "6. Instellingen" → "5. Instellingen", "7. Signalering" → "6. Signalering", "8. Gebeurtenissen en diagnose" → "7. Gebeurtenissen en diagnose", en pas verwijzingen in commentaar als "sectie 8" aan (bijvoorbeeld bij `debug:` "Diagnose: herstartreden (sectie 7)")
- [X] T014 [US1] Controleer dat er geen resten zijn ([quickstart.md](quickstart.md) §2 stap 3): `Select-String -Path esphome/pompput.yaml -Pattern 'pomp_|_pomp|pompstroom|looptijd|stroom|acs712|ct_clamp|adc|offset|inschakel|afpomp|start_marge|start_wachttijd|waarschuwing|oranje|ms_per_min' -CaseSensitive:$false | Where-Object { $_.Line -notmatch '004-stroomsensor-verwijderen|stroomonderbreking' }` geeft geen uitvoer. Verwijder wat er nog staat; een treffer in een commentaar dat beschrijft dat iets vervallen is, mag blijven als die naar 004 verwijst
- [X] T015 [US1] Draai de compile-gate: `.venv\Scripts\esphome config esphome/pompput.yaml` en `.venv\Scripts\esphome compile esphome/pompput.yaml`. Los fouten op tot er geen fouten en geen waarschuwingen uit eigen lambdas zijn. Noteer RAM en flash in `specs/004-stroomsensor-verwijderen/verificatie.md` onder "Compile- en resource-gate" en vergelijk met de uitgangssituatie (verwacht: RAM ≤ 75 %, flash ≤ 90 %, beide lager dan 0.4.0; research R12)
- [ ] T016 [US1] Werk het apparaat bij via WiFi (`.venv\Scripts\esphome run esphome/pompput.yaml --device pompput.local`) en voer [quickstart.md](quickstart.md) §3 stap 1–3 en §4 #1–#8, #11 en #12 uit. #12 (veldcontrole 24 uur) loopt door na de andere scenario's; noteer de uitkomst na 24 uur. Noteer per scenario de uitkomst in `specs/004-stroomsensor-verwijderen/verificatie.md`; is het apparaat of een onderdeel (bijvoorbeeld een opstelling zonder ACS712 voor #8) niet beschikbaar, noteer "niet uitgevoerd" met de reden

**Checkpoint**: firmware 0.5.0 meldt zonder stroomsensor geen storing, de niveaubewaking werkt
als voorheen, de zeven instellingen zijn behouden en het pinnencontract klopt. US1 is als MVP
bruikbaar.

---

## Phase 4: User Story 2 - Geen pompgegevens en pompbediening meer in Home Assistant (Priority: P2)

**Goal**: geen pomp- of stroomentiteiten, pomp-alerts, `history_stats`, dashboardregels of
meldingsteksten over de pomp meer in Home Assistant; de overgebleven instellingen werken (FR-003,
FR-008, FR-009, FR-010, FR-013).

**Independent Test**: [quickstart.md](quickstart.md) §3 stap 4–6 en §4 #10 en #13. De
bestandswijzigingen (T017–T019) kunnen los van US1 gebeuren; de controle (T020) vraagt firmware
0.5.0 uit US1.

- [X] T017 [P] [US2] Werk `homeassistant/packages/pompput.yaml` bij volgens [contracts §2](contracts/wijzigingen-firmware-en-ha.md#2-home-assistant-pakket-homeassistantpackagespompputyaml): (1) verwijder het blok `sensor:` met de `history_stats` "Pompput pompstarts 24 uur" (en de kop `sensor:` zelf); (2) verwijder de alerts `pompput_pomp_start_niet`, `pompput_pomp_pompt_niet_af`, `pompput_pomp_draait_te_lang` en `pompput_stroomsensor_storing`; (3) verwijder in `pompput_hoogwater`, `pompput_onbereikbaar`, `pompput_niet_ingesteld` en `pompput_niveausensor_storing` de regel `Pomp: {{ 'draait' if … }}.`; (4) de tekst van `pompput_niet_ingesteld` wordt "De pompput-monitor is nog niet ingesteld: vul Afstand leeg en Afstand vol in (Instellingen > Apparaten > Pompput > Configuratie). Zonder deze waarden is er geen vulgraad."; (5) de kopcommentaar noemt bij "Contract" ook `specs/004-stroomsensor-verwijderen/contracts/wijzigingen-firmware-en-ha.md`. `input_boolean` en de template "Pompput onbereikbaar" blijven gelijk (FR-009, FR-010, FR-013)
- [X] T018 [P] [US2] Werk `homeassistant/dashboard-pompput.yaml` bij volgens [contracts §3](contracts/wijzigingen-firmware-en-ha.md#3-dashboard-homeassistantdashboard-pompputyaml): (1) kopcommentaar: "(SC-006: de vulgraad binnen 10 s af te lezen)"; (2) kaart "Pompput": verwijder de regels Pomp (`binary_sensor.pompput_pomp`), Pompstarts 24 uur en Looptijd laatste pompbeurt; (3) kaart "Laatste 24 uur": verwijder Pomp; (4) kaart "Gebeurtenissen": het commentaar wordt "(meldingen, herstart, afgewezen instellingen, herstart van de niveaumeting)"; (5) kaart "Instellingen": verwijder `number.pompput_inschakelniveau`, de hele sectie "Pomp" (label en zes numbers), en onder "Diagnose" `sensor.pompput_offset_stroomsensor` en `sensor.pompput_pompstroom` (FR-013)
- [X] T019 [P] [US2] Werk `specs/001-pompput-monitor/contracts/esphome-homeassistant.md` bij volgens [contracts §1–§2](contracts/wijzigingen-firmware-en-ha.md): (1) versie in de tabel "Algemeen" wordt `0.5.0`; (2) verwijder de rijen `looptijd_laatste_pompbeurt`, `pompstroom` en `offset_stroomsensor` (Sensoren), `pomp`, `pomp_start_niet`, `pomp_pompt_niet_af`, `pomp_draait_te_lang` en `stroomsensor_storing` (Binaire sensoren) en de zeven vervallen numbers (Instellingen); (3) `status` heeft de waarden `normaal`, `alarm`, `storing`, `niet_ingesteld`; het voorbeeld bij `laatste_gebeurtenis` wordt `melding_actief: hoogwater, niveau 52,0 cm`; (4) `event_types` zonder `pomp_start` en `pomp_stop`; (5) in "Home Assistant-pakket": punt 1 (`history_stats`) vervalt, het alert-punt spreekt van "elk van de drie probleem-binary-sensors" en noemt de pomp niet meer in de tekst; (6) voeg onder de inleiding één regel toe: "De pompbewaking en de stroomsensor zijn vervallen met `specs/004-stroomsensor-verwijderen`."
- [ ] T020 [US2] Laad het pakket en het dashboard opnieuw in Home Assistant en voer [quickstart.md](quickstart.md) §3 stap 4–6 en §4 #10 en #13 uit (na T016). Zet bij #13 (b) `binary_sensor.pompput_niet_ingesteld` na de controle weer op `off`. Verwijder achtergebleven entiteiten uit de lijst van [contracts §1](contracts/wijzigingen-firmware-en-ha.md#1-entiteiten-van-het-apparaat) en een achtergebleven `sensor.pompput_pompstarts_24_uur` onder Instellingen → Entiteiten (research R7). Noteer de uitkomsten in `specs/004-stroomsensor-verwijderen/verificatie.md`

**Checkpoint**: Home Assistant toont 20 entiteiten van het apparaat, zonder pompgegevens en
zonder onbeschikbare entiteiten (SC-004).

---

## Phase 5: User Story 3 - Led 1 zonder pomp- en waarschuwingspatroon (Priority: P2)

**Goal**: de beschrijving van de ledbalk klopt met de firmware: geen "waarschuwing" (oranje) en
geen "pomp draait" meer; de overige patronen ongewijzigd (FR-005, FR-007, FR-014). De
firmwarewijziging zelf zit in US1 (T008, T011).

**Independent Test**: [quickstart.md](quickstart.md) §4 #9, en de ledbeschrijving in §5.

T021–T025 raken elk een ander bestand en kunnen parallel.

- [X] T021 [P] [US3] Werk `specs/003-niveau-ledbalk/contracts/ledbalk.md` bij: (1) vervang de tabel in §1 door de tabel uit [contracts §5](contracts/wijzigingen-firmware-en-ha.md#5-signalering-ter-plaatse-led-1) (zes toestanden, kolom "Wijziging" met de verschuivingen); (2) voeg onder de tabel toe: "De toestanden Waarschuwing (oranje) en Pomp draait zijn vervallen met `specs/004-stroomsensor-verwijderen`."; (3) de uitleg bij prio 5 (nu prio 4, "niveau onbekend") verwijst naar de nieuwe nummers (prio 2 en prio 4 delen het patroon); (4) in §3 "Samen gezien": "Leeg, pomp stil, verbonden" wordt "Leeg, verbonden", en de rij "50 %, pomp draait" wordt `| 50 %, verbonden | ● ● ● ● ○ ○ |`
- [X] T022 [P] [US3] Werk `specs/001-pompput-monitor/contracts/lokale-signalering.md` bij op dezelfde punten als T021 (de tabel in §1, de regel over de vervallen toestanden, de uitleg bij "niveau onbekend" en de voorbeelden in §3), en vermeld in de inleiding ook 004 FR-005 en FR-007 naast de bestaande verwijzingen
- [X] T023 [P] [US3] Werk `specs/003-niveau-ledbalk/data-model.md` bij: verwijder in §1 de rij `led_pomp_aan_ms` / `led_pomp_uit_ms`; zet in §2, rij `led_patroon`, dat er na 004 zes patronen zijn (zonder waarschuwing en pomp draait) met een verwijzing naar `specs/004-stroomsensor-verwijderen/data-model.md` §5
- [X] T024 [P] [US3] Werk `specs/003-niveau-ledbalk/spec.md` bij, telkens met "(gewijzigd door 004)" of "(vervallen met 004)" erachter: (1) voeg onder "Clarifications" een sessie "2026-11-18 (004-stroomsensor-verwijderen)" toe met "De stroomsensor en de pompbewaking vervallen; led 1 toont geen "pomp draait" en geen "waarschuwing" meer. Zie [specs/004-stroomsensor-verwijderen](../004-stroomsensor-verwijderen/spec.md)."; zet achter de eerste clarification (over "pomp draait") "(vervallen met 004)"; (2) "Begrippen", Statuspatroon: zonder waarschuwing (oranje) en zonder "pomp draait"; (3) US2: scenario 3 wordt "Gegeven er is tegelijk een alarm en een storing actief, wanneer ik naar led 1 kijk, dan toont die het alarm (rood), volgens de bestaande prioriteit."; scenario 7 en 8 krijgen "(vervallen met 004)"; de Independent Test van US2 noemt geen pompwaarschuwing meer; (4) het randgeval "Pomp draait" krijgt "(vervallen met 004)"; (5) FR-003: de opsomming zonder waarschuwing en pomp draait, en de uitzonderingen "behalve FR-013 en FR-017"; FR-014 krijgt "(vervallen met 004)"; FR-016: "pompdetectie" vervalt uit de opsomming; (6) SC-004: "Bij het nabootsen van opstarten, alarm, sensorstoring en geen verbinding …"
- [X] T025 [P] [US3] Werk `specs/003-niveau-ledbalk/quickstart.md` bij: (1) §1: verwijder "Een belasting op de stroomsensor (zoals 001 §4 #8) voor "pomp draait"."; (2) §4, B2 Gamma: gebruik paars (een storing van de niveausensor) in plaats van oranje; (3) #7, #8 en #9 worden `| # | (vervallen) | — | Waarschuwing en pomp draait zijn vervallen met specs/004-stroomsensor-verwijderen | — |` zodat de nummering gelijk blijft; (4) #19: "Voer 001 §4 #4 uit … Meldingen en herstelmeldingen in HA zoals voorheen"; (5) #20: led 1 paars door een storing van de **niveausensor** (afstandsmeter afdekken) in plaats van de stroomsensor; (6) werk de traceerbaarheidstabel in §6 bij voor de vervallen scenario's
- [ ] T026 [US3] Voer [quickstart.md](quickstart.md) §4 #9 uit (na T016) en noteer de uitkomst in `specs/004-stroomsensor-verwijderen/verificatie.md`

**Checkpoint**: led 1 toont zes toestanden, en de documentatie van de ledbalk beschrijft precies
die zes.

---

## Phase 6: User Story 4 - Documentatie en aansluitschema zonder stroomsensor (Priority: P3)

**Goal**: bouwbeschrijving, aansluitschema, voeding, testprocedure en de levende specs noemen de
stroomsensor en de pompbewaking niet meer als onderdeel; A1 staat als vrij; het weghalen van een
bestaande sensor is beschreven (FR-014, FR-015, SC-006). Historische documenten van 001–003
(`research.md`, `tasks.md`, `verificatie.md`) blijven ongewijzigd (research R9). Het
pinnencontract `hardware.md` is al in US1 bijgewerkt (T012).

**Independent Test**: [quickstart.md](quickstart.md) §5.

T027–T031 raken elk een ander bestand (T031 twee plan-bestanden) en kunnen parallel.

- [X] T027 [P] [US4] Werk `README.md` bij: (1) inleiding: "Bewaking van een buitenpompput. Een HC-SR04 meet het waterniveau. Daarmee bepaalt het apparaat zelf de meldingen (hoogwater, sensorstoring, niet ingesteld) …"; voeg bij "Ontwerp en eisen" `specs/004-stroomsensor-verwijderen/` toe; (2) vervang de netspanningswaarschuwing over de ACS712 (het blok met ">" bovenaan) door een korte verwijzing: "Heb je nog een ACS712 in de fasedraad van de pomp? Lees dan eerst 'Een bestaande stroomsensor weghalen' hieronder."; (3) "Platform": de sectie-opsomming zonder "pomp" en met zeven secties; de pinnenlijst zonder `pin_acs712`; (4) tabel "Overzicht": verwijder de vier rijen "ACS712 30 A" en zet D1/A1 bij "Vrij:"; (5) "Pinnen van de XIAO": D1 wordt `(vrij)` en de 5V-regel noemt geen ACS712; (6) "Per onderdeel": vervang de alinea "ACS712, 30 A-variant …" (met schema) door de paragraaf "Een bestaande stroomsensor weghalen" met de vijf punten uit [contracts §4](contracts/wijzigingen-firmware-en-ha.md#4-hardware); (7) "Voeding": zonder ACS712; (8) "Controle voor het eerste gebruik": verwijder punt 1 over 1,25 V op A1 en hernummer; (9) "Home Assistant": stap met de instellingen vraagt alleen Afstand leeg en Afstand vol; (10) "Signalering ter plaatse": verwijder de rijen "oranje, 1 Hz | waarschuwing (pomp)" en "blauw, 1 s aan / 1 s uit | pomp draait". Controleer met `Select-String -Path README.md -Pattern 'acs712|stroomsensor|pompstroom|looptijd|pomp draait|pompwaarschuwing|oranje|inschakelniveau' -CaseSensitive:$false` dat alleen de weghaalparagraaf en verwijzingen naar 004 overblijven
- [X] T028 [P] [US4] Werk `specs/001-pompput-monitor/spec.md` bij met de stand na 004: (1) voeg onder "Clarifications" een sessie "2026-11-18 (004-stroomsensor-verwijderen)" toe met "De stroomsensor en alle pompbewaking vervallen; het apparaat bewaakt alleen het niveau. Zie [specs/004-stroomsensor-verwijderen](../004-stroomsensor-verwijderen/spec.md)."; de oorspronkelijke **Input** blijft als geschiedenis staan; (2) zet "(vervallen met 004)" achter de titels van User Story 3 en User Story 4, achter FR-006, FR-008, FR-009, FR-011, FR-013, FR-014, FR-015 en SC-002, en achter de randgevallen "Pomp krijgt geen stroom", "Pomp loopt droog", "Pomp slaat vaak kort aan en af" en de Assumptions over de stroommeting en het pendelen; (3) pas aan, telkens met "(gewijzigd door 004)": FR-010 (zonder pompstarts en -stops), FR-016 (alleen de niveaumeting), FR-019 (zonder pompstatus), FR-023 (zonder inschakelniveau en zonder de wachttijden en marges van FR-013 t/m FR-015 en de stroomsensor), FR-026 (zonder pomp start/stop), SC-003 (alleen niveau boven alarmdrempel, niveausensor storing en apparaat losgekoppeld), SC-006 (alleen de vulgraad), het randgeval "Hevige regen" (alleen het hoogwater-alarm), het randgeval "Apparaat niet bereikbaar" (zonder pompstatus) en de Key Entities (Pompput-configuratie zonder inschakelniveau en looptijd; "Pompstatus / pompcyclus" vervallen met 004); (4) pas ook aan, telkens met "(gewijzigd door 004)": FR-027 ("… tonen: opstarten, normaal, geen verbinding met Home Assistant, storing en alarm."), de Key Entity Melding ("soort (alarm of storing)") en de Assumption "Alle standaardwaarden…" (zonder de marge van "pomp start niet", de controle van "pomp pompt niet af" en de maximale looptijd)
- [X] T029 [P] [US4] Werk `specs/001-pompput-monitor/data-model.md` bij volgens [data-model.md](data-model.md): (1) §1: verwijder de zeven vervallen rijen, de regel "`inschakelniveau_cm` < `bereik_cm`", de rijen voor de stroommeting, de omrekening ACS712, de pomp-debounce, de pomp-uit-hysterese, de offset en de herstelmarge "pomp start niet"; "ingesteld" = twee verplichte waarden; het voorbeeld `${pomp_draait}` in de inleiding wordt `${niveau_geldig}`; (2) §4: vervang de inhoud door "Vervallen met `specs/004-stroomsensor-verwijderen`." en laat de kop staan zodat de nummering gelijk blijft; (3) §5: alleen HOOGWATER, NIVEAUSENSOR_STORING en NIET_INGESTELD, met de nieuwe nummers 0–2; de regels over de pompstatus en STROOMSENSOR_STORING vervallen; de statusvolgorde wordt ALARM > NIET_INGESTELD > STORING > NORMAAL; (4) §6: zonder `pomp_start` en `pomp_stop`, `melding_actief` zonder pomp; (5) §7: `led_patroon` volgt niet meer uit de pompstatus; zes patronen
- [X] T030 [P] [US4] Werk `specs/001-pompput-monitor/quickstart.md` bij: (1) §1: haal de ACS712 uit de onderdelenlijst en verwijder het punt "Voor de bankproef van de stroommeting"; (2) §3: de verwachte entiteiten verwijzen naar de 20 van [contracts §1](contracts/wijzigingen-firmware-en-ha.md#1-entiteiten-van-het-apparaat); stap 4 vraagt alleen Afstand leeg en Afstand vol; (3) §4: verwijder het punt "ACS712 in rust"; de scenario's #8, #9, #10, #11, #12, #13, #24, #25, #26 en #31 worden elk `| # | (vervallen) | — | Pompbewaking en stroomsensor zijn vervallen met specs/004-stroomsensor-verwijderen | — |` zodat de nummering gelijk blijft; #19 wordt "XIAO stroomloos → de pomp slaat gewoon aan en af op zijn eigen vlotter"; #20 zonder "pompbeurten"; #30 zonder de stap met Inschakelniveau; controleer de overige rijen op "pomp" en "oranje"; (4) §5: verwijder stap 4 (Inschakelniveau bepalen) en stap 5 (Pompstroom en Stroomdrempel) en hernummer; (5) §6: SC-002 vervallen; SC-006 alleen de vulgraad; "een pompwaarschuwing terwijl de pomp…" vervalt uit het voorbeeld bij SC-004; (6) §7: de rijen van de vervallen requirements krijgen "vervallen met 004"
- [X] T031 [P] [US4] Voeg in `specs/001-pompput-monitor/plan.md` en in `specs/003-niveau-ledbalk/plan.md` direct onder de regel **Input** één regel toe: "**Bijgewerkt**: de stroomsensor en de pompbewaking zijn vervallen met [specs/004-stroomsensor-verwijderen](../004-stroomsensor-verwijderen/plan.md)." In `specs/001-pompput-monitor/plan.md` staan al twee regels **Bijgewerkt** (002 en 003); zet de nieuwe regel daaronder, met een lege regel ertussen, zoals bij die twee. Wijzig verder niets in deze bestanden
- [X] T032 [US4] Voer [quickstart.md](quickstart.md) §5 uit (na T012, T017–T019, T021–T025 en T027–T031), lees de weghaalparagraaf in `hardware.md` en `README.md` na op de vijf punten, en noteer de uitkomst in `specs/004-stroomsensor-verwijderen/verificatie.md` onder "Documentatie". Alleen treffers die naar het vervallen van de stroomsensor of de pompbewaking verwijzen, of die in de weghaalparagraaf staan, zijn toegestaan

**Checkpoint**: alle levende documentatie beschrijft het apparaat zonder stroomsensor.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: de laatste gates en het afronden.

- [X] T033 Laatste compile- en resource-gate na alle wijzigingen: herhaal T015 en zet de uitkomst in de samenvattingstabel van `specs/004-stroomsensor-verwijderen/verificatie.md`
- [X] T034 Review- en traceerbaarheidsgate in `specs/004-stroomsensor-verwijderen/verificatie.md`: loop de tabel "Constitution Check" van [plan.md](plan.md) na tegen `esphome/pompput.yaml` en de constitutie (2.0.2, of 2.0.1 als T003 niet is goedgekeurd): enige uitgangen TRIG en de ledketen; pinnen in `substitutions` (GPIO4, 5, 6) gelijk aan `specs/001-pompput-monitor/contracts/hardware.md`; geen `adc` of `ct_clamp`. Controleer dat elke requirement uit de tabel "Traceerbaarheid" van [plan.md](plan.md) een element heeft en een scenario in [quickstart.md](quickstart.md) §6. Noteer afwijkingen onder "Afwijkingen"
- [X] T035 Verwijder het Sync Impact Report (het HTML-commentaar bovenaan) uit `.specify/memory/constitution.md` vóór de commit, zoals bij 002 (T029); de rest van het bestand blijft gelijk
- [X] T036 Zet in `specs/004-stroomsensor-verwijderen/spec.md` de **Status** van `Draft` op `Implemented` zodra T016, T020, T026 en T032 zijn uitgevoerd, of op `Implemented (hardware-scenario's open)` als scenario's op het apparaat nog niet konden draaien

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: geen afhankelijkheden. T002 moet vóór T016 (bijwerken) klaar zijn, anders zijn de oude instellingen niet meer te vergelijken.
- **Foundational (Phase 2)**: T003 kan direct; het wacht op de goedkeuring van de maintainer, maar houdt de andere fasen niet tegen. Alleen T034 en T035 vragen T003.
- **US1 (Phase 3)**: na Phase 1. Strikt na elkaar: T004 → T010, dan T011 en T012 (contract, samen met T011), dan T013 → T014 → T015 → T016.
- **US2 (Phase 4)**: T017–T019 kunnen direct na Phase 1, parallel met US1. T020 vraagt T016.
- **US3 (Phase 5)**: T021–T025 kunnen direct na Phase 1, parallel met US1 en US2. T026 vraagt T016.
- **US4 (Phase 6)**: T027–T031 kunnen direct na Phase 1, parallel met de andere stories. T032 vraagt T012, T017–T019, T021–T025 en T027–T031.
- **Polish (Phase 7)**: na alle stories. T033 en T034 vragen T015; T035 vraagt T003; T036 komt als laatste.

### User Story Dependencies

- **US1 (P1)**: onafhankelijk; levert de firmware (ook de led-wijziging van US3) en het bijgewerkte pinnencontract.
- **US2 (P2)**: bestanden onafhankelijk; de controle op het apparaat (T020) hangt af van US1 (T016), omdat de entiteiten pas verdwijnen met firmware 0.5.0.
- **US3 (P2)**: de documentatie is onafhankelijk; het gedrag zit in de firmware van US1, dus de controle (T026) hangt af van T016.
- **US4 (P3)**: onafhankelijk van de andere stories, behalve dat de eindcontrole T032 ook `hardware.md` (T012) en de bestanden van US2 en US3 meeneemt.

### Parallel Opportunities

- T017, T018 en T019 (US2) parallel met elkaar en met US1.
- T021–T025 (US3) allemaal parallel: elk een ander bestand.
- T027–T031 (US4) allemaal parallel: elk een ander bestand.
- US1 zelf heeft geen parallelle taken: alles volgt elkaar op in `esphome/pompput.yaml`, met `hardware.md` direct na T011.
- T003 (constitutie) kan op elk moment, parallel met alles.

---

## Parallel Example: User Story 2 (naast US1)

```text
Task: "T017 homeassistant/packages/pompput.yaml: history_stats en vier alerts weg, 'Pomp:' uit de teksten"
Task: "T018 homeassistant/dashboard-pompput.yaml: pomponderdelen weg"
Task: "T019 specs/001-pompput-monitor/contracts/esphome-homeassistant.md bijwerken"
```

## Parallel Example: User Story 3

```text
Task: "T021 specs/003-niveau-ledbalk/contracts/ledbalk.md bijwerken"
Task: "T022 specs/001-pompput-monitor/contracts/lokale-signalering.md bijwerken"
Task: "T023 specs/003-niveau-ledbalk/data-model.md bijwerken"
Task: "T024 specs/003-niveau-ledbalk/spec.md bijwerken"
Task: "T025 specs/003-niveau-ledbalk/quickstart.md bijwerken"
```

## Parallel Example: User Story 4

```text
Task: "T027 README.md bijwerken"
Task: "T028 specs/001-pompput-monitor/spec.md bijwerken"
Task: "T029 specs/001-pompput-monitor/data-model.md bijwerken"
Task: "T030 specs/001-pompput-monitor/quickstart.md bijwerken"
Task: "T031 verwijzing in plan.md van 001 en 003"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Phase 1: T001–T002 (uitgangssituatie vastleggen).
2. Phase 3: T004–T015: firmware zonder stroomsensor en pompbewaking, pinnencontract bijgewerkt,
   compile- en resource-gate.
3. **STOP en VALIDEER**: T016 op het apparaat (geen storing zonder sensor, niveaubewaking gelijk,
   instellingen behouden).
4. Het apparaat is dan al bruikbaar zonder stroomsensor. HA toont hooguit vervallen entiteiten
   als onbeschikbaar, en de pomp-alerts in het oude pakket gaan niet meer af.

### Incremental Delivery

1. Setup → US1 (firmware en pinnencontract) → valideren.
2. US2 (HA-pakket, dashboard en HA-contract) → T020 valideren.
3. US3 (ledbalk-documentatie) → T026 valideren.
4. US4 (overige documentatie) → T032 valideren.
5. Foundational T003 zodra de maintainer akkoord is; Polish: laatste gates, Sync Impact Report
   weg, status bijwerken.

---

## Notes

- [P] = ander bestand, geen afhankelijkheid van een open taak.
- Raak in `esphome/pompput.yaml` alleen aan wat in T004–T013 staat; de logica van sectie 3
  (niveau), `melding_stap` (behalve de tabellen en de tekst), de hoogwater-, niveausensor- en
  niet-ingesteld-beoordeling, `led_knipperen` en `niveaubalk_bijwerken` blijven ongewijzigd
  (FR-006).
- Behoud de id's van alle bewaarde waarden die blijven (research R4); hernoem niets.
- Scenario's die niet op het apparaat konden draaien, worden met de reden in `verificatie.md`
  gezet (constitutie IV, gate 4).
- Commit per fase of per logische groep.
