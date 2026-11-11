# Research: Niveau-ledbalk met zes leds

**Feature**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md) | **Datum**: 2026-11-11

Na het invullen van de Technical Context waren er drie open punten: hoe de firmware zes leds
apart aanstuurt (R1), of de voeding en de diode de extra leds aankunnen (R4), en hoe de
kleuren correct op de leds komen als de strip in stukken is verdeeld (R1, gamma). De
beslissingen hieronder lossen ze op. Alles wat niet genoemd wordt, blijft zoals in 001 en 002.

## R1. Zes leds apart aansturen: één strip, zes partities

- **Decision**:
  - De bestaande `esp32_rmt_led_strip` blijft de enige driver, nu met `num_leds: 6`. Hij heet
    voortaan `ledstrip`, blijft `internal: true` en `restore_mode: ALWAYS_OFF`, en wordt door
    geen enkel script direct aangestuurd.
  - Daarboven komen **zes lichten van het platform `partition`**, elk met één segment van één
    led:
    - `statusled` = led 1 (index 0). Dezelfde id als nu, dus `led_knipperen` en
      `led_patroon_kiezen` blijven deze id aansturen.
    - `niveauled_2` t/m `niveauled_6` = led 2 t/m 6 (index 1 t/m 5).
  - Elke partitie: `internal: true`, `restore_mode: ALWAYS_OFF`, `gamma_correct: ${led_gamma}`
    en `default_transition_length: 0s`, net als de huidige statusled.
  - De led-indexen staan als substitutions (`led_index_1` … `led_index_6`), niet als losse
    getallen in de partities (constitutie V).
- **Rationale**:
  - Het knipperscript `led_knipperen` (001 R9) is beproefd en blijft letterlijk gelijk: het
    stuurt een gewoon licht aan, en dat is nu de partitie van led 1.
  - De niveaubalk knippert nooit. Vijf gewone lichten die alleen aan of uit gaan, passen in
    een kort script zonder eigen timing.
  - Een defecte led of een knipperende status op led 1 heeft geen invloed op de andere
    partities: elk licht heeft een eigen toestand.
  - Geen nieuwe externe componenten: `partition` is een standaardplatform van ESPHome
    2026.10.1 (constitutie I).
- **Alternatives considered**:
  - *Eén `addressable_lambda`-effect dat alle zes leds per frame tekent*: dan verhuist de
    knippertiming van het script naar een lambda die elke lusronde draait. Dat is minder
    declaratief (constitutie IV) en vervangt een werkend script. Verworpen; wel het
    terugvalplan als de partities op de bank niet goed werken (zie "Te controleren op de
    bank").
  - *Eén partitie voor led 1 en één partitie voor led 2–6 met een effect voor de balk*: spaart
    vier lichten, maar vraagt toch een effect-lambda per frame voor iets dat alleen bij een
    grensovergang verandert. Verworpen.
  - *Een tweede datapin voor de balk*: kost een pin en een tweede RMT-kanaal, en de gebruiker
    heeft de leds al als één keten aangesloten. Verworpen.

## R2. De niveaubalk berekenen

- **Decision**: een nieuw script **`niveaubalk_bijwerken`** bepaalt elke seconde het aantal
  brandende balkleds *N* (0–5) en zet alleen de leds die veranderen.
  - **Grenzen** (FR-004): grens *k* = *k* / 5 × alarmdrempel, voor *k* = 1 t/m 5. Led
    *k* + 1 hoort bij grens *k*. Het aantal stappen (5) is de substitution
    `niveaubalk_stappen`.
  - **Herstelmarge** (FR-008): `niveaubalk_herstelmarge_pct` = 2,0 procentpunt, vast in de
    firmware.
  - **Rekenregel**, met *v* = vulgraad en *N*vorig = het vorige aantal:
    - *N*aan = het aantal grenzen met *v* ≥ grens;
    - *N*houd = het aantal grenzen met *v* ≥ grens − marge;
    - *N* = max(*N*aan, min(*N*vorig, *N*houd)).
    - Een led gaat dus aan zodra de grens bereikt is, en pas uit als *v* de marge onder de
      grens zakt. De balk is altijd aaneengesloten (FR-006), omdat de grenzen oplopen.
  - **Ongeldig niveau** (FR-007): als `${niveau_bruikbaar}` onwaar is of `g_vulgraad_pct` NAN,
    dan is *N* = 0 en wordt *N*vorig ook 0. Dat dekt opstarten, verouderd, storing, niet
    ingesteld en een meting die uitvalt tijdens een alarm.
  - **Helderheid** (FR-011): de balk gebruikt dezelfde `led_helderheid_pct` als led 1. Bij 0 %
    zijn alle balkleds uit. De minimale alarmhelderheid geldt niet voor de balk (spec,
    randgeval "Helderheid op 0 %").
  - **Alleen bij een verandering**: het script onthoudt *N* en de helderheid in
    `g_niveaubalk_aantal` en `g_niveaubalk_helderheid`. Zijn beide gelijk, dan doet het niets.
    Anders loopt het één keer over de vijf partities: `turn_on` met blauw en de helderheid voor
    index < *N*, `turn_off` voor de rest.
  - **Aanroep**: `status_bijwerken` (elke seconde, na het niveau en de meldingen) roept naast
    `led_patroon_kiezen` ook `niveaubalk_bijwerken` aan. Een nieuwe alarmdrempel werkt dus in
    de volgende seconde door (FR-010), en een grensovergang is binnen ongeveer 2 s zichtbaar
    (FR-009: ≤ 5 s).
- **Rationale**:
  - De vulgraad en `niveau_bruikbaar` bestaan al (001). De balk voegt geen eigen meting of
    filter toe en kan het alarm dus niet tegenspreken.
  - Eén regel voor de hysterese voorkomt knipperen rond een grens (SC-003) zonder extra
    timers.
  - Schrijven alleen bij een verandering houdt de lambda ruim onder 10 ms en zet de strip niet
    elke seconde opnieuw (constitutie II).
- **Alternatives considered**:
  - *Een `delayed_on`/`delayed_off`-filter per led*: vijf extra template-sensoren en een
    vertraging die de spec niet vraagt. Verworpen.
  - *Herstelmarge gelijk aan de herstelmarge van het alarm (standaard 5 procentpunt)*: bij de
    kleinste alarmdrempel (10 %) is een stap maar 2 procentpunt; een marge van 5 maakt de balk
    daar traag en onduidelijk. Verworpen; de balk krijgt een eigen, kleinere marge.

## R3. Het statuspatroon op led 1

- **Decision**: `led_patroon_kiezen` behoudt de volgorde en de patroonnummers. Alleen twee
  patronen veranderen:

  | Patroon | Was | Wordt | Spec |
  |---|---|---|---|
  | `led_patroon_pomp` | blauw, continu | **blauw, 1000 ms aan / 1000 ms uit** | FR-014 |
  | `led_patroon_normaal` | groen, flits 100 ms per 5 s | **blauw, continu** | FR-002, FR-013 |

  - Substitutions: `led_pomp_aan_ms` "1000" en `led_pomp_uit_ms` "1000" (was "0");
    `led_normaal_uit_ms` wordt "0" en `led_normaal_aan_ms` vervalt (continu heeft geen
    aan-tijd nodig, net als `led_opstarten_uit_ms`).
  - De kleur `led_groen_*` wordt nergens meer gebruikt en vervalt (constitutie V).
  - Alarm, waarschuwing, storing, opstarten en geen verbinding blijven gelijk, inclusief de
    minimale helderheid bij een alarm (FR-003, FR-011).
- **Niveau onbekend** (FR-017, uit `/speckit-analyze` U1): in de `if`-keten komt direct na
  `g_opstarten` de regel `else if (!${niveau_bruikbaar}) patroon = ${led_patroon_storing};`.
  - Geen nieuw patroonnummer: hetzelfde nummer als storing, dus de overgang van "verouderd" naar
    de echte storingsmelding herstart `led_knipperen` niet.
  - `${niveau_bruikbaar}` is dezelfde uitdrukking die de niveaubalk gebruikt (constitutie IV);
    led 1 en de balk zijn het dus altijd eens over "geldig".
  - Alarm (ook vastgehouden zonder meting), waarschuwing en storing staan hoger en blijven
    winnen; tijdens de opstartfase blijft led 1 wit.
- **Rationale**: de kleinste wijziging die de spec waarmaakt. De prioriteitsregel ("de bovenste
  regel die geldt, wint") blijft het enige beslispunt voor led 1. Zonder de regel voor "niveau
  onbekend" ziet een weggevallen meting er tot de storingsmelding (standaard 5 min) uit als
  een lege put: led 1 blauw, balk uit.
- **Alternatives considered**: *pomp als blauw flitsen (100 ms per 2 s)*: lijkt te veel op "geen
  verbinding" (wit, hetzelfde ritme). De gebruiker koos 1 s aan / 1 s uit (spec, Clarifications).

## R4. Voeding, diode en aansluiting van zes leds

- **Decision**:
  - **Keten**: DIN van led 1 aan D4 (via 330 Ω, zoals nu); DOUT van led *n* aan DIN van led
    *n* + 1. Led 1 zit het dichtst bij het apparaat.
  - **Voeding van de leds**: één gezamenlijke VDD-lijn via de diode. De diode wordt een
    **1N4001** (1 A); de 1N4148 is niet meer toegestaan (zie berekening).
  - **Ontkoppeling**: 100 nF per led (op de meeste SK6812-strips al aanwezig) plus één
    elko van 100 µF tussen VDD en GND bij led 1.
  - **Adapter**: de bestaande 5 V-adapter van minimaal 1 A blijft voldoende.
- **Berekening** (SK6812 RGBW: ongeveer 20 mA per kanaal op volle sterkte, ongeveer 1 mA in
  rust per led):
  - Slechtste geval dat de firmware kan maken: led 1 paars (rood + blauw, 40 mA) en vijf
    blauwe balkleds (5 × 20 mA), op 100 % helderheid: **≈ 146 mA**.
  - Theoretisch maximum van zes leds op volledig wit (4 kanalen): ≈ 480 mA. De firmware
    gebruikt nooit meer dan één kanaal op de balk en nooit RGB + W samen, dus dit treedt niet
    op.
  - Totaal met XIAO (pieken ≈ 350 mA), HC-SR04 (≈ 15 mA) en ACS712 (≈ 10 mA): **≈ 520 mA
    piek**, ruim onder 1 A.
  - De 1N4148 is gespecificeerd voor ongeveer 200 mA gemiddeld en heeft bij 150 mA al ruim 1 V
    spanningsval; VDD zou dan te laag worden voor een betrouwbare 3,3 V-datasignaal. De 1N4001
    houdt VDD rond 4,2–4,3 V.
- **Rationale**: met VDD ≈ 4,3 V ligt de ingangsdrempel van led 1 rond 0,7 × 4,3 ≈ 3,0 V, dus
  3,3 V-data van de XIAO blijft betrouwbaar (001 hardware.md). De volgende leds krijgen een
  opgefrist signaal van hun voorganger.
- **Alternatives considered**: *de leds direct op 5 V zonder diode*: dan is 3,3 V-data
  marginaal (drempel ≈ 3,5 V) en is een niveauverschuiver nodig. Verworpen; de diode werkt al.

## R5. Wat Home Assistant ziet

- **Decision**: niets nieuws. De strip en de zes partities zijn `internal: true`. De enige
  bediening blijft `number` `led_helderheid_pct`. Het pakket en het dashboard veranderen niet.
- **Rationale**: spec, Assumptions ("Home Assistant krijgt geen nieuwe entiteiten"); constitutie
  V (YAGNI).
- **Alternatives considered**: *de balk als lamp in HA tonen*: de vulgraad staat al in HA, en
  een bedienbare lamp zou de signalering kunnen overschrijven. Verworpen.

## R6. Versie, documentatie en verificatie

- **Decision**:
  - `project_version` 0.3.0 → **0.4.0**.
  - De levende documenten van 001 worden bijgewerkt:
    - `contracts/lokale-signalering.md` krijgt de inhoud van [contracts/ledbalk.md](contracts/ledbalk.md);
    - `contracts/hardware.md`: keten van zes leds, diode 1N4001, elko, voeding;
    - `data-model.md` §7: niveaubalk;
    - `quickstart.md` §1 en §4: zes leds, kleurcontrole, scenario's;
    - `spec.md`: FR-027, FR-028 en SC-009 krijgen "(gewijzigd door 003)".
  - `README.md`: componenten (`partition`), aansluittabel, schema, ledtabel en voeding.
  - Uitkomsten van de gates komen in `specs/003-niveau-ledbalk/verificatie.md`.
- **Rationale**: dezelfde werkwijze als 002 (R5 daar): levende documenten volgen de hardware,
  historische beslissingen blijven staan.

## R7. Geheugen

- **Decision**: de resource-gate meet opnieuw. Verwacht: RAM stijgt met hooguit enkele kB (zes
  `LightState`-objecten en een buffer van 6 × 4 bytes), flash met enkele kB voor het
  partition-platform. Beide blijven ver onder de grenzen (laatste meting 0.3.0: RAM 34,7 %,
  flash 47,3 %).
- **Rationale**: constitutie II, resource-gate.

## Te controleren op de bank

Deze punten zijn niet uit documentatie vast te stellen en gaan als controle in
[quickstart.md](quickstart.md) §4:

1. **Partities zonder ingeschakelde moederstrip**: de partities sturen hun leds aan terwijl
   `ledstrip` zelf uit staat. Lukt dat niet, dan valt de uitvoering terug op het
   `addressable_lambda`-alternatief uit R1, en wordt dit plan bijgewerkt.
2. **Gamma**: oranje (255,80,0) is oranje en niet rood op led 1. Is het rood, dan geldt de
   gamma van de moederstrip niet en moet `gamma_correct: 1.0` ook op `ledstrip` staan (dat is
   het plan al) of in de partitie anders worden ingesteld.
3. **Kleurvolgorde**: led 1 is bij "normaal" blauw en niet groen of wit.
4. **Volgorde van de keten**: bij een stijgend niveau gaat de led naast led 1 als eerste aan.
