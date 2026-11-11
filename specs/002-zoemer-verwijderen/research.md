# Research: Zoemer verwijderen

**Feature**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md) | **Datum**: 2026-11-11

Er waren na het invullen van de Technical Context geen open punten ("NEEDS CLARIFICATION").
De beslissingen hieronder leggen vast hoe de verwijdering gebeurt en waarom dat veilig is.

## R1. Wat verdwijnt uit de firmware

- **Decision**: alles wat alleen de zoemer, het stilzetten of de BOOT-knop dient, gaat weg uit
  `esphome/pompput.yaml`:
  - **Componenten**: `output` `zoemer_output` (`ledc`), `switch` `zoemer`, `button`
    `zoemer_stil`, `binary_sensor` `boot_knop`.
  - **Scripts**: `zoemer_herhalen`, `zoemer_eenmalig`, `zoemer_patroon_kiezen`,
    `zoemer_stilzetten`, `led_bevestiging`.
  - **Globals**: `g_zoemer_modus`, `g_zoemer_herstart`, `g_eenmalig_bezig`, `g_stil`,
    `g_stil_set`, `g_vorige_set`, `g_led_bevestiging_bezig`.
  - **Substitutions**: `pin_zoemer`, `pin_knop`, `knop_debounce`, alle `zoemer_*`
    (frequentie, niveau, patronen, pauze, modi) en `led_bevestiging_*`.
  - **On_boot**: stap 4, de opstartpiep.
  - **Script `signalering_bijwerken`**: dat deed alleen nog het "stil"-beheer en de eenmalige
    piepjes, en riep daarna `led_patroon_kiezen` aan. Het vervalt; `status_bijwerken` roept
    `led_patroon_kiezen` direct aan. Daarmee vervalt ook `g_vorige_set`, dat alleen daar werd
    gebruikt.
  - **`led_patroon_kiezen`**: de voorwaarde `!id(g_led_bevestiging_bezig)` vervalt; de rest van
    de keuze (prioriteiten, kleuren, minimale helderheid bij alarm) blijft letterlijk gelijk
    (spec FR-002).
- **Rationale**: constitutie V (YAGNI, ongebruikte componenten verwijderen) en constitutie IV
  (geen dode logica). Wat overblijft, wordt eenvoudiger: de led is de enige lokale signalering.
- **Alternatives considered**:
  - *Zoemer laten staan maar standaard uitschakelen*: laat ongebruikte code, een PWM-kanaal en
    een HA-schakelaar staan. Strijdig met de spec (FR-001, FR-004) en constitutie V.
  - *`signalering_bijwerken` als lege doorgeefluik laten staan*: één extra script zonder eigen
    taak; verworpen.

## R2. De vrijgekomen pinnen GPIO7 (D5) en GPIO9 (D9)

- **Decision**: beide pinnen worden niet meer geconfigureerd. `contracts/hardware.md` van 001
  markeert D5 als "vrij" en D9 als "BOOT-knop op het board, zonder functie in de firmware".
- **Rationale**:
  - Zonder `ledc`-output wordt GPIO7 door de firmware nooit als uitgang gezet en er komt geen
    PWM-signaal op. Na een reset staat de pin als ingang (hoog-ohmig, hoogstens met een zwakke
    interne pull). Een passieve piëzo maakt alleen geluid op een wisselspanning, dus een
    eventueel nog aangesloten zoemer blijft stil (spec FR-007, randgeval "er zit nog een
    zoemer aangesloten"). Dit is de veilige begintoestand die constitutie III vraagt, al vanaf
    het vroegste moment, omdat de firmware de pin niet aanraakt.
  - GPIO9 is een strapping-pin. De BOOT-knop op het board houdt zijn functie bij het opstarten
    (downloadmodus); de firmware leest de pin niet meer, dus een druk tijdens bedrijf doet
    niets (spec FR-005).
- **Verificatie**: de bewering over GPIO7 wordt op het apparaat bevestigd met quickstart §4 #9.
  Faalt die controle, dan geldt het eerste alternatief hieronder als terugvaloptie (tasks T015).
- **Alternatives considered**:
  - *GPIO7 expliciet als uitgang laag zetten*: kost een `output`-component zonder functie;
    voegt niets toe omdat een niet-aangestuurde pin bij een passieve piëzo al stil is. YAGNI.
    Blijft de terugvaloptie als quickstart §4 #9 faalt.
  - *De BOOT-knop een andere functie geven*: valt buiten de spec (Assumptions: geen nieuwe
    functionaliteit).

## R3. Bewaarde instellingen na de OTA-update

- **Decision**: geen migratie nodig. Na de update zijn alle instellingen behalve de vervallen
  zoemerschakelaar ongewijzigd (spec FR-009, SC-005); dit wordt op het apparaat gecontroleerd
  (quickstart §3).
- **Rationale**: ESPHome slaat elke bewaarde waarde op onder een sleutel die is afgeleid van de
  id van die entiteit of global, niet van de volgorde in de YAML. Het verwijderen van de
  schakelaar `zoemer` en van globals zonder `restore_value` verandert de sleutels van de andere
  waarden dus niet. De oude waarde van `zoemer` blijft als wees in flash staan; die is een paar
  bytes groot en wordt nergens meer gelezen.
- **Alternatives considered**: *flash wissen en opnieuw instellen*: onnodig werk voor de
  gebruiker en strijdig met SC-005.

## R4. Verdwenen entiteiten in Home Assistant

- **Decision**: de entiteiten `switch.pompput_zoemer` en `button.pompput_zoemer_stil` verdwijnen
  uit het apparaat. Het dashboard (`homeassistant/dashboard-pompput.yaml`) verliest de twee
  regels in de kaart "Bediening"; die kaart houdt "Instellingen tonen". De meldingstekst van
  "Pompput onbereikbaar" in `homeassistant/packages/pompput.yaml` noemt alleen nog de led.
- **Rationale**: de ESPHome-integratie ruimt entiteiten op die het apparaat na een herstart niet
  meer meldt. Blijft er toch een verweesde entiteit staan (bijvoorbeeld door een oudere
  HA-versie), dan verwijdert de gebruiker die eenmalig onder Instellingen → Entiteiten. De
  quickstart controleert dit expliciet (SC-004).
- **Alternatives considered**: *de entiteiten als `internal` laten staan*: laat dode code staan;
  verworpen (R1).

## R5. Welke documenten worden bijgewerkt

- **Decision**:
  - **Levende documenten van 001** (beschrijven het huidige apparaat) worden bijgewerkt:
    `spec.md` (FR-020, FR-027, FR-028, SC-009, US2 scenario 1, randgevallen, Assumptions,
    met een verwijzing naar 002; de clarifications over de zoemer blijven als geschiedenis
    staan met "(zoemer vervallen met 002)"), `data-model.md` (§1 instelling `zoemer_aan`, §1 constanten, §7
    signaleringstoestand), `contracts/hardware.md`, `contracts/lokale-signalering.md`,
    `contracts/esphome-homeassistant.md` en `quickstart.md` (§1 onderdelenlijst, §2 stap 5, §3,
    §4 #4, #6, #10, #13, #14, #15 en §7; stilte staat er als "het apparaat blijft stil").
    Ook `README.md`.
  - **Historische documenten van 001** blijven zoals ze zijn: `plan.md`, `research.md`,
    `tasks.md` en `verificatie.md`. Ze beschrijven wat toen is gebouwd en gecontroleerd. Bovenaan
    `plan.md` van 001 komt één regel dat de zoemer met 002 is vervallen.
  - **Uitkomsten van de gates** voor deze wijziging komen in `specs/002-zoemer-verwijderen/verificatie.md`
    (aangemaakt bij de uitvoering).
- **Rationale**: spec FR-011 en SC-006 vragen dat de bouwbeschrijving, het aansluitschema en de
  testprocedure de zoemer niet meer noemen. Historische verslagen herschrijven zou de
  geschiedenis vervalsen.
- **Alternatives considered**: *alle 001-documenten herschrijven*: veel werk zonder waarde, en
  de verificatie van toen klopt dan niet meer met wat er toen is getest.

## R6. Versie en opmerkingen in de YAML

- **Decision**: `project_version` gaat van `0.2.0` naar `0.3.0`. De kopcommentaar van
  `esphome/pompput.yaml` noemt alleen de led als lokale signalering, sectie 7 heet "signalering:
  led, patroonscripts", en de verwijzing naar de specs noemt ook `specs/002-zoemer-verwijderen/`.
  De FR-verwijzingen in sectie 7 worden `FR-027` (led) plus `002 FR-002/FR-003`; verwijzingen
  naar FR-028 blijven alleen bij `led_helderheid_pct`.
- **Rationale**: constitutie IV (traceerbaarheid) en een zichtbaar versieverschil in HA
  (apparaatinfo), zodat te controleren is dat de nieuwe firmware draait.

## R7. Effect op RAM, flash en rekentijd

- **Decision**: geen maatregel nodig; de resource-gate wordt opnieuw gemeten en vastgelegd.
- **Rationale**: er komt niets bij en er gaan een PWM-kanaal, zes scripts, zeven globals en vier
  entiteiten af. RAM en flash kunnen alleen dalen. De rekentijd per beoordeling daalt licht,
  omdat `signalering_bijwerken` en `zoemer_patroon_kiezen` niet meer elke seconde draaien.
