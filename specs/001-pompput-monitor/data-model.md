# Data Model: Pompput-monitor

**Feature**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md) | **Research**: [research.md](research.md)

(Herzien op 2026-11-11 voor ESPHome. De grootheden, toestanden en rekenregels zijn
ongewijzigd ten opzichte van de Arduino-versie. Alleen de uitvoering verandert:
`substitutions`, `globals`, scripts en template-entiteiten.)

Alle grootheden hebben een expliciete eenheid en geldigheidsgrenzen (constitutie,
principe III).
- **Tijd** op het apparaat is `millis()` (ms sinds de start, `uint32_t`), en wordt
  overloopveilig vergeleken met `now - t >= d`.
- **Klok**: het apparaat heeft er geen nodig. Tijdstippen voor de gebruiker komen uit
  Home Assistant (`last_changed`).
- **Namen**: codenamen staan tussen haakjes. `id`'s in de YAML zijn Nederlandse snake_case.
  De getalcodes van toestanden en meldingen hebben in de YAML een naam in het
  `substitutions:`-blok (bijvoorbeeld `${niveau_geldig}`); scripts gebruiken nooit het getal
  zelf (constitutie II en V).

---

## 1. Instellingen (`Settings`)

Worden beheerd in Home Assistant, als `number` en `switch`. Het apparaat bewaart de laatst
geldige waarden in flash (ESPHome-preferences). Dit zijn de **enige** gegevens die het
apparaat blijvend bewaart (FR-024).

| Sleutel | Eenheid | Standaard | Bereik | Stap | Opslag | Spec |
|---|---|---|---|---|---|---|
| `afstand_leeg_cm` | cm (sensor → water bij leeg) | — (verplicht) | 10–400 | 0,1 | global + regels | FR-002 |
| `afstand_vol_cm` | cm (sensor → water bij vol) | — (verplicht) | 3–390 | 0,1 | global + regels | FR-002 |
| `alarmdrempel_pct` | % vulgraad | 80 | 10–120 | 1 | global + regels | FR-012 |
| `hoogwater_wachttijd_s` | s | 30 | 0–600 | 1 | number | FR-012 |
| `sensorstoring_wachttijd_s` | s | 300 | 30–3600 | 1 | number | FR-016 |
| `herstelmarge_pct` | procentpunt | 5 | 1–50 | 1 | global + regels | FR-021 |
| `led_helderheid_pct` | % | 30 | 0–100 | 1 | number | FR-028 |

**De twee soorten opslag** (research R11):
- **global + regels**: de geaccepteerde waarde staat in een `globals` met
  `restore_value: true` (bijvoorbeeld `g_afstand_leeg_cm`; verplichte waarden beginnen op
  `NAN`). Het template-`number` laat die waarde zien. Een nieuwe waarde gaat via het script
  `instelling_toepassen`.
- **number**: een template-`number` met `optimistic: true` en `restore_value: true`. Daarvan
  wordt `id(...).state` gelezen.
- **switch**: een template-`switch` met `optimistic: true` en
  `restore_mode: RESTORE_DEFAULT_ON`.

Het bereik controleert ESPHome zelf: een waarde buiten min/max wordt met een logmelding
geweigerd. Alle grenzen en standaardwaarden staan in het `substitutions:`-blok.

**Validatieregels** (een ongeldige wijziging wordt geweigerd en de oude waarde blijft staan):
- `afstand_leeg_cm` ≥ `afstand_vol_cm` + 10. Het verschil is het meetbereik `bereik_cm`.
- `alarmdrempel_pct` − `herstelmarge_pct` > 0.
- Een regel wordt alleen beoordeeld als alle waarden die erin voorkomen, gezet zijn.
- Een wijziging van de geometrie (`afstand_leeg_cm` of `afstand_vol_cm`) zet de niveaumeting
  terug naar ONBEKEND (zie 3).

**Afgeleide toestand**: `ingesteld` = de twee verplichte waarden (`afstand_leeg_cm`,
`afstand_vol_cm`) zijn gezet en voldoen aan de regels (004: het inschakelniveau is vervallen). Is dat niet zo, dan is de melding **NIET_INGESTELD** actief.

**Vaste constanten**, in het `substitutions:`-blok:

| Constante | Waarde | Spec of research |
|---|---|---|
| Interval tussen metingen | 250 ms | R6 |
| Mediaanvenster | 5 metingen, minimaal 3 geldig | R6, FR-004 |
| Minimale sensorafstand | 3 cm | R6, FR-005 |
| Bereik zonder instelling | 420 cm (400 + 20) | R6 |
| Maximale niveausnelheid | 2 cm/s, Δt maximaal 5 s, minimale tolerantie 3 cm | R6, FR-005 |
| Inleren | 5 medianen binnen 2 cm | R6 |
| Niveau "verouderd" na | 30 s zonder geaccepteerde waarde | FR-003 |
| Verwerking en meldingen | elke 1 s | R5 |
| Publicatie waterhoogte en vulgraad | elke 10 s, en direct bij een statuswijziging | FR-003 |
| Opstarten (led 1 wit) | maximaal 10 s | lokale signalering |
| Kleurvolgorde led | GRBW (controleren op de bank) | R9 |
| Aantal leds | 6 in één keten: led 1 statuslampje, led 2–6 niveaubalk | 003 R1, R4 |
| Niveaubalk | 5 gelijke stappen tot de alarmdrempel; herstelmarge 2 procentpunt | 003 FR-004, FR-008 |

---

## 2. Niveaumeting (`LevelSample`)

| Veld | Type / eenheid | Betekenis | In ESPHome |
|---|---|---|---|
| ruwe meting (`Ping`) | float, cm of NAN | elke 250 ms | `ultrasonic`, `on_raw_value` (meters of NAN) × `${m_naar_cm}` → ringbuffer `g_pings` (`std::array<float,5>`) |
| `afstand_cm` | float, cm | mediaan van de geldige metingen | `g_mediaan_cm` → sensor "Afstand (ruw)" |
| `hoogte_cm` | float, cm | `afstand_leeg_cm − afstand_cm` (kan iets negatief zijn) | sensor "Waterhoogte" |
| `vulgraad_pct` | float, % | `hoogte_cm / bereik_cm × 100` (niet afgekapt: > 100 betekent boven vol) | sensor "Vulgraad" |
| `t_ms` | uint32 | moment waarop de waarde geaccepteerd is | `g_geaccepteerd_ms` |

Een **ruwe meting** is geldig als er binnen de timeout een echo is en
3 cm ≤ afstand ≤ `afstand_leeg_cm + 20` cm.

## 3. Niveaustatus (`LevelState`): toestanden

Opgeslagen in `g_niveau_status` (0 = ONBEKEND, 1 = GELDIG, 2 = VEROUDERD, 3 = STORING).
De overgangen staan allemaal in het script `niveau_verwerken`.

```text
            eerste 5 medianen binnen 2 cm
ONBEKEND ─────────────────────────────────▶ GELDIG ◀───────────────┐
   ▲  │                                       │ geen geaccepteerde  │ geaccepteerde
   │  │ geen referentie                       │ mediaan > 30 s      │ mediaan
   │  │ > sensorstoring_wachttijd_s           ▼                     │
   │  └──────────────────────────┐        VEROUDERD ───────────────┤
   │ herstart / geometrie         │           │ > sensorstoring_    │
   │ gewijzigd / HA-knop          ▼           ▼ wachttijd_s         │
   └──────────────────────────  STORING ◀─────┘                     │
                                   └────────────────────────────────┘
```

- **GELDIG**: waterhoogte en vulgraad krijgen een waarde.
- **VEROUDERD**, **STORING** en **ONBEKEND**: waterhoogte en vulgraad worden `NAN`, en Home
  Assistant toont "onbekend" (FR-003).
- **STORING**: de melding NIVEAUSENSOR_STORING wordt actief (FR-016). Dat geldt ook als er
  sinds de start of een reset nog nooit een geldige waarde is geweest.
- **Zonder referentie** (na een start of reset) wordt er ingeleerd. Mét referentie geldt de
  snelheidscontrole. Een mediaan die de controle niet haalt, wordt verworpen; de toestand
  verandert dan pas via de tijdsgrenzen, of via opnieuw inleren in STORING.
- **Opnieuw inleren in STORING** (research R6): in STORING loopt naast de snelheidscontrole
  ook een inleerreeks. Zijn er 5 opeenvolgende medianen binnen 2 cm, dan wordt de laatste de
  nieuwe referentie (STORING → GELDIG), met de gebeurtenis `niveaumeting_herstart` (oorzaak
  `automatisch`). Zo herstelt de meting ook als het niveau tijdens de storing meer is
  veranderd dan de snelheidscontrole toelaat. Prijs: een hindernis die langer dan
  `sensorstoring_wachttijd_s` stil blijft hangen, wordt daarna als niveau gezien (spec,
  randgeval).

## 4. Pompstatus (`PumpState`) en pompcyclus (`PumpCycle`)

Vervallen met `specs/004-stroomsensor-verwijderen`.

## 5. Melding (`Alert`)

| Type | Ernst | Wordt actief als | Wordt hersteld als | Spec |
|---|---|---|---|---|
| `HOOGWATER` | ALARM | `vulgraad_pct > alarmdrempel_pct` gedurende `hoogwater_wachttijd_s` | `vulgraad_pct < alarmdrempel_pct − herstelmarge_pct`. **Niet** beoordelen bij een niveaustatus ≠ GELDIG (vasthouden) | FR-012, FR-021, FR-022 |
| `NIVEAUSENSOR_STORING` | STORING | niveaustatus = STORING | niveaustatus = GELDIG | FR-016 |
| `NIET_INGESTELD` | STORING | geen geldige verplichte instellingen | instellingen geldig | randgeval |

**Levenscyclus per melding**: één script met parameters, `melding_stap`. Elk type heeft een
eigen beoordelingsscript dat de voorwaarden bepaalt. De toestand staat per melding in
`globals`-arrays (`g_melding_fase`, `g_melding_sinds_ms`, `g_melding_herstel_sinds_ms`),
geïndexeerd met de volgorde van de tabel (0 = HOOGWATER, 1 = NIVEAUSENSOR_STORING,
2 = NIET_INGESTELD; 004 data-model §3).

```text
INACTIEF ──voorwaarde waar──▶ IN_AFWACHTING ──wachttijd verstreken──▶ ACTIEF ──herstel (+ hersteltijd)──▶ INACTIEF
    ▲                              │ voorwaarde onwaar of                          (melding_hersteld)
    └──────────────────────────────┘ niet te beoordelen
```

- Bij ACTIEF volgt de gebeurtenis `melding_actief`. Meldingen zonder wachttijd gaan direct naar
  ACTIEF.
- **"Niet beoordelen"** (vasthouden): een actieve melding blijft staan, en een lopende
  wachttijd stopt.
- **"Onderdrukken"**: de melding gaat naar INACTIEF. Was hij ACTIEF, dan volgt
  `melding_hersteld`.
- **Afhankelijkheden**:
  - Is de niveaustatus ≠ GELDIG, dan worden niveau-afhankelijke meldingen niet nieuw
    geactiveerd. HOOGWATER blijft vastgehouden (FR-022).
  - Bij NIET_INGESTELD wordt HOOGWATER onderdrukt, en wordt NIVEAUSENSOR_STORING niet nieuw
    geactiveerd.
- **Na een herstart** beginnen alle meldingen bij INACTIEF (de globals worden niet bewaard)
  en worden ze opnieuw beoordeeld.

**Apparaatstatus** (`DeviceStatus`, text sensor "Status", en voor de led): volgorde
ALARM > NIET_INGESTELD > STORING > NORMAAL. `niet_ingesteld` is een aparte waarde voor de
duidelijkheid. De publicatiewaarden zijn `normaal`, `alarm`, `storing` en `niet_ingesteld`.
De soort WAARSCHUWING en de vier pomp- en stroommeldingen zijn vervallen met 004.

## 6. Gebeurtenis (`Event`)

Een ESPHome-`event` draagt alleen een `event_type` (research R13). De details gaan tegelijk
als tekst naar de text sensor "Laatste gebeurtenis".

| `event_type` | Details in "Laatste gebeurtenis" | Moment |
|---|---|---|
| `melding_actief` | melding, niveau (cm) | melding → ACTIEF |
| `melding_hersteld` | melding | melding → INACTIEF |
| `herstart` | herstartreden | eenmalig, 3 s na de eerste API-verbinding na het opstarten |
| `instelling_afgewezen` | instelling, waarde, reden | ongeldige combinatie |
| `niveaumeting_herstart` | oorzaak (`knop`, `instelling`, `automatisch`) | niveaustatus teruggezet, of in STORING opnieuw ingeleerd |

Het apparaat bewaart geen gebeurtenissen. Is er geen verbinding, dan gaan ze verloren
(FR-010).

## 7. Signaleringstoestand (`IndicationState`)

Vervallen met [002](../002-zoemer-verwijderen/data-model.md): de zoemer, "stil" en de
led-bevestiging. Uitgebreid met [003](../003-niveau-ledbalk/data-model.md): zes leds, met de
niveaubalk op led 2–6. Met [004](../004-stroomsensor-verwijderen/data-model.md) zijn de patronen
"waarschuwing" en "pomp draait" vervallen; led 1 kiest uit zes patronen.

| Veld | Betekenis | In ESPHome |
|---|---|---|
| `led_patroon` | patroon van led 1; volgt uit de apparaatstatus, de niveaustatus ("niveau onbekend", 003 FR-017), de verbinding en "opstarten"; wordt nooit onderbroken | script `led_patroon_kiezen` → script `led_knipperen` op partitie `statusled` (led 1); huidige keuze in `g_led_patroon` |
| `verbonden` | is de API verbonden (voor het led-patroon "geen verbinding") | `g_verbonden`, gezet door `api: on_client_connected` en `on_client_disconnected` |
| `opstarten` | waar tot de eerste GELDIGE niveaumeting, maximaal 10 s | `g_opstarten` |
| `niveaubalk_aantal` | aantal brandende balkleds (led 2–6), 0–5; 0 bij een ongeldig niveau | script `niveaubalk_bijwerken`; `g_niveaubalk_aantal` |
| `niveaubalk_helderheid` | helderheid waarmee de balk het laatst is gezet | `g_niveaubalk_helderheid` |

Niveaubalk en fysieke leds: zie [003 data-model](../003-niveau-ledbalk/data-model.md) §3–§4.

Kleuren, ritmes en prioriteiten: [contracts/lokale-signalering.md](contracts/lokale-signalering.md).
