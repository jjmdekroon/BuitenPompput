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
  `substitutions:`-blok (bijvoorbeeld `${pomp_draait}`); scripts gebruiken nooit het getal
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
| `inschakelniveau_cm` | cm boven leeg-niveau | — (verplicht) | 0–400 | 0,1 | global + regels | FR-013 |
| `alarmdrempel_pct` | % vulgraad | 80 | 10–120 | 1 | global + regels | FR-012 |
| `hoogwater_wachttijd_s` | s | 30 | 0–600 | 1 | number | FR-012 |
| `start_marge_cm` | cm | 5 | 0–50 | 1 | number | FR-013 |
| `start_wachttijd_s` | s | 120 | 10–3600 | 1 | number | FR-013 |
| `afpomp_controletijd_s` | s | 180 | 30–3600 | 1 | number | FR-014 |
| `afpomp_min_daling_cm` | cm | 2 | 0,5–50 | 0,1 | number | FR-014 |
| `max_looptijd_min` | min | 10 | 1–240 | 1 | number | FR-015 |
| `sensorstoring_wachttijd_s` | s | 300 | 30–3600 | 1 | number | FR-016 |
| `herstelmarge_pct` | procentpunt | 5 | 1–50 | 1 | global + regels | FR-021 |
| `pomp_stroomdrempel_a` | A (RMS) | 0,5 | 0,05–20 | 0,05 | number | FR-006 |
| `led_helderheid_pct` | % | 30 | 0–100 | 1 | number | FR-028 |
| `zoemer_aan` | aan/uit | aan | — | — | switch | FR-028 |

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
- `inschakelniveau_cm` < `bereik_cm`.
- `alarmdrempel_pct` − `herstelmarge_pct` > 0.
- Een regel wordt alleen beoordeeld als alle waarden die erin voorkomen, gezet zijn.
- Een wijziging van de geometrie (`afstand_leeg_cm` of `afstand_vol_cm`) zet de niveaumeting
  terug naar ONBEKEND (zie 3).

**Afgeleide toestand**: `ingesteld` = de drie verplichte waarden zijn gezet en voldoen aan de
regels. Is dat niet zo, dan is de melding **NIET_INGESTELD** actief.

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
| Stroommeting | `ct_clamp`: venster 100 ms, elke 500 ms | R7 |
| Omrekening ACS712 | 30,303 A per V op de pin (= 1 / (0,066 × 0,5)) | R7 |
| Pomp-debounce | 2 s (aan en uit) | FR-011 |
| Pomp-uit-hysterese | 70 % van `pomp_stroomdrempel_a` | R7 |
| Offset stroomsensor geldig | 0,9–1,6 V (na de deler), storing na 10 s, herstel na 10 s | R7, FR-016 |
| Herstelmarge "pomp start niet" | 2 cm | §5 |
| Verwerking en meldingen | elke 1 s | R5 |
| Publicatie waterhoogte en vulgraad | elke 10 s, en direct bij een statuswijziging | FR-003 |
| Opstarten (witte led) | maximaal 10 s | lokale signalering |
| Zoemertoon | 2700 Hz, 50 % | R10 |
| Kleurvolgorde led | GRBW (controleren op de bank) | R9 |

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

Opgeslagen in `g_pomp_status` (0 = ONBEKEND, 1 = STIL, 2 = START_KANDIDAAT, 3 = DRAAIT,
4 = STOP_KANDIDAAT). De overgangen staan allemaal in het script `pomp_bijwerken`, dat bij
elke waarde van de stroommeting draait (elke 500 ms).

```text
          I ≥ drempel                     2 s aaneengesloten (I ≥ 70%)
STIL ───────────────▶ START_KANDIDAAT ─────────────────────▶ DRAAIT
 ▲  ◀─────────────── (I < 70% drempel)                         │ I < 70% drempel
 │                                                             ▼
 └──────────── 2 s aaneengesloten (I < 70%) ──────────── STOP_KANDIDAAT
                                                      (I ≥ 70% → DRAAIT)
ONBEKEND: offset buiten 0,9–1,6 V, of nog geen meting. Bij de eerstvolgende geldige meting
          naar STIL, of bij I ≥ drempel naar START_KANDIDAAT.
```

| Pompcyclus-veld | Type / eenheid | Betekenis | In ESPHome |
|---|---|---|---|
| `start_ms` | uint32 | eerste meting boven de drempel | `g_pomp_start_ms` |
| `looptijd_s` | float, s (`NAN` tot de eerste stop) | bij de stop: (eerste meting onder 70 %) − `start_ms`, afgerond | sensor "Looptijd laatste pompbeurt" |
| `niveau_start_cm` | float, cm | `hoogte_cm` bij de start (referentie voor FR-014); NAN als het niveau toen onbekend was | `g_niveau_start_cm` |

- "Draait" omvat DRAAIT en STOP_KANDIDAAT. "Staat stil" omvat STIL en START_KANDIDAAT.
- Binary sensor "Pomp": aan = draait, uit = stil, onbekend bij ONBEKEND.
- Gebeurtenissen: `pomp_start` bij START_KANDIDAAT → DRAAIT, en `pomp_stop` bij
  STOP_KANDIDAAT → STIL. Bij ONBEKEND is er geen gebeurtenis.
- Korte pieken van minder dan 2 s leveren geen start op (FR-011).

## 5. Melding (`Alert`)

| Type | Ernst | Wordt actief als | Wordt hersteld als | Spec |
|---|---|---|---|---|
| `HOOGWATER` | ALARM | `vulgraad_pct > alarmdrempel_pct` gedurende `hoogwater_wachttijd_s` | `vulgraad_pct < alarmdrempel_pct − herstelmarge_pct`. **Niet** beoordelen bij een niveaustatus ≠ GELDIG (vasthouden) | FR-012, FR-021, FR-022 |
| `POMP_START_NIET` | WAARSCHUWING | `hoogte_cm > inschakelniveau_cm + start_marge_cm` en de pomp staat stil, gedurende `start_wachttijd_s` | de pomp draait, of het niveau zakt onder `inschakelniveau_cm + start_marge_cm − 2 cm` | FR-013 |
| `POMP_POMPT_NIET_AF` | WAARSCHUWING | de pomp draait ≥ `afpomp_controletijd_s` en `niveau_start_cm − hoogte_cm < afpomp_min_daling_cm` | binnen een cyclus is de daling ≥ `afpomp_min_daling_cm` | FR-014 |
| `POMP_DRAAIT_TE_LANG` | WAARSCHUWING | de pomp draait > `max_looptijd_min` | de pomp staat stil | FR-015 |
| `NIVEAUSENSOR_STORING` | STORING | niveaustatus = STORING | niveaustatus = GELDIG | FR-016 |
| `STROOMSENSOR_STORING` | STORING | offset buiten 0,9–1,6 V gedurende 10 s | offset 10 s binnen het bereik | FR-016 |
| `NIET_INGESTELD` | STORING | geen geldige verplichte instellingen | instellingen geldig | randgeval |

**Levenscyclus per melding**: één script met parameters, `melding_stap`. Elk type heeft een
eigen beoordelingsscript dat de voorwaarden bepaalt. De toestand staat per melding in
`globals`-arrays (`g_melding_fase`, `g_melding_sinds_ms`, `g_melding_herstel_sinds_ms`),
geïndexeerd met de volgorde van de tabel.

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
  - Is de pompstatus ONBEKEND, dan worden de drie pompwaarschuwingen niet beoordeeld. Een
    actieve waarschuwing blijft staan.
  - Bij NIET_INGESTELD worden HOOGWATER en de pompwaarschuwingen onderdrukt, en wordt
    NIVEAUSENSOR_STORING niet nieuw geactiveerd. STROOMSENSOR_STORING wordt altijd beoordeeld.
- **Na een herstart** beginnen alle meldingen bij INACTIEF (de globals worden niet bewaard)
  en worden ze opnieuw beoordeeld.

**Apparaatstatus** (`DeviceStatus`, text sensor "Status", en voor de led): volgorde
ALARM > WAARSCHUWING > NIET_INGESTELD > STORING > NORMAAL. `niet_ingesteld` is een aparte
waarde voor de duidelijkheid. De publicatiewaarden zijn `normaal`, `waarschuwing`, `alarm`,
`storing` en `niet_ingesteld`.

## 6. Gebeurtenis (`Event`)

Een ESPHome-`event` draagt alleen een `event_type` (research R13). De details gaan tegelijk
als tekst naar de text sensor "Laatste gebeurtenis".

| `event_type` | Details in "Laatste gebeurtenis" | Moment |
|---|---|---|
| `pomp_start` | niveau bij de start (cm) | START_KANDIDAAT → DRAAIT |
| `pomp_stop` | looptijd (s), niveau (cm) | STOP_KANDIDAAT → STIL |
| `melding_actief` | melding, niveau (cm), pomp | melding → ACTIEF |
| `melding_hersteld` | melding | melding → INACTIEF |
| `herstart` | herstartreden | eenmalig, 3 s na de eerste API-verbinding na het opstarten |
| `instelling_afgewezen` | instelling, waarde, reden | ongeldige combinatie |
| `niveaumeting_herstart` | oorzaak (`knop`, `instelling`, `automatisch`) | niveaustatus teruggezet, of in STORING opnieuw ingeleerd |

Het apparaat bewaart geen gebeurtenissen. Is er geen verbinding, dan gaan ze verloren
(FR-010).

## 7. Signaleringstoestand (`IndicationState`)

| Veld | Betekenis | In ESPHome |
|---|---|---|
| `led_patroon` | volgt uit de apparaatstatus, de pompstatus, de verbinding en "opstarten" | script `led_patroon_kiezen` → script `led_knipperen`; huidige keuze in `g_led_patroon` |
| `zoemer_patroon` | volgt uit de hoogste ernst (ALARM of WAARSCHUWING); storing, hersteld, bevestiging en opstarten zijn eenmalig | script `zoemer_patroon_kiezen` → `zoemer_herhalen`; eenmalige piepjes via `zoemer_eenmalig` |
| `stil_voor` | de set actieve meldingen op het moment van "stil". Zolang de actuele set daar een deelverzameling van is, blijft de zoemer stil (FR-028) | `g_stil`, `g_stil_set` (bitmasker) |
| `vorige_set` | de actieve meldingen bij de vorige beoordeling (voor "nieuwe melding" en "alles hersteld") | `g_vorige_set` |
| `verbonden` | is de API verbonden (voor het led-patroon "geen verbinding") | `g_verbonden`, gezet door `api: on_client_connected` en `on_client_disconnected` |
| `opstarten` | waar tot de eerste GELDIGE niveaumeting, maximaal 10 s | `g_opstarten` |

Kleuren, ritmes en prioriteiten: [contracts/lokale-signalering.md](contracts/lokale-signalering.md).
