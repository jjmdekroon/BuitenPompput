# Data Model: Stroomsensor verwijderen

**Feature**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md) | **Datum**: 2026-11-18

Deze feature voegt geen gegevens toe. Dit document legt vast wat er verandert ten opzichte van
het datamodel van 001 ([data-model.md](../001-pompput-monitor/data-model.md)) en 003
([data-model.md](../003-niveau-ledbalk/data-model.md)). Alles wat hier niet staat, blijft
gelijk.

## 1. Instellingen (`Settings`, 001 §1)

| Instelling | Opslag | Was | Wordt | Spec |
|---|---|---|---|---|
| `afstand_leeg_cm` | global + regels | verplicht | ongewijzigd, verplicht | FR-008 |
| `afstand_vol_cm` | global + regels | verplicht | ongewijzigd, verplicht | FR-008 |
| `inschakelniveau_cm` | global + regels | verplicht | **vervalt** (research R3) | FR-004 |
| `alarmdrempel_pct` | global + regels | 80 % | ongewijzigd | FR-008 |
| `herstelmarge_pct` | global + regels | 5 procentpunt | ongewijzigd | FR-008 |
| `hoogwater_wachttijd_s` | number | 30 s | ongewijzigd | FR-008 |
| `sensorstoring_wachttijd_s` | number | 300 s | ongewijzigd | FR-008 |
| `led_helderheid_pct` | number | 30 % | ongewijzigd | FR-008 |
| `start_marge_cm` | number | 5 cm | **vervalt** | FR-004 |
| `start_wachttijd_s` | number | 120 s | **vervalt** | FR-004 |
| `afpomp_controletijd_s` | number | 180 s | **vervalt** | FR-004 |
| `afpomp_min_daling_cm` | number | 2 cm | **vervalt** | FR-004 |
| `max_looptijd_min` | number | 10 min | **vervalt** | FR-004 |
| `pomp_stroomdrempel_a` | number | 0,5 A | **vervalt** | FR-004 |

Van 14 naar **7** instellingen. De waarden van de vervallen instellingen blijven ongelezen in
flash staan; de sleutels van de overige veranderen niet (research R4, FR-011).

**Validatieregels** (001 §1), na de wijziging:
- `afstand_leeg_cm` ≥ `afstand_vol_cm` + 10 (ongewijzigd).
- ~~`inschakelniveau_cm` < `bereik_cm`~~ vervalt.
- `alarmdrempel_pct` − `herstelmarge_pct` > 0 (ongewijzigd).

**Afgeleide toestand**: `ingesteld` = de **twee** verplichte waarden (`afstand_leeg_cm`,
`afstand_vol_cm`) zijn gezet en voldoen aan de regels.

**Vaste constanten** (001 §1): de rijen voor de stroommeting (`ct_clamp`-venster en -interval,
ADC-interval en demping, offsetvenster, omrekenfactor ACS712), de debounce van de pomp (2 s),
de uit-hysterese (70 %), het offsetbereik (0,9–1,6 V) met zijn bevestigingstijd (10 s) en de
herstelmarge van "pomp start niet" (2 cm) vervallen.

## 2. Pompstatus en pompcyclus (`PumpState`, `PumpCycle`, 001 §4)

**Vervalt in zijn geheel**: de toestanden ONBEKEND, STIL, START_KANDIDAAT, DRAAIT en
STOP_KANDIDAAT, de cyclusgegevens (startmoment, niveau bij de start, looptijd) en de
afgeleiden "pomp draait nu" en "pomp staat stil" (FR-001).

## 3. Melding (`Alert`, 001 §5)

De levenscyclus INACTIEF → IN_AFWACHTING → ACTIEF → INACTIEF en de velden per melding blijven
ongewijzigd. Er blijven drie meldingen over (research R2):

| Index | Melding | Soort | Activeren | Herstellen | Spec |
|---|---|---|---|---|---|
| 0 | `HOOGWATER` | alarm | ongewijzigd (001 §5) | ongewijzigd | 001 FR-012, FR-021, FR-022 |
| 1 | `NIVEAUSENSOR_STORING` | storing | ongewijzigd | ongewijzigd | 001 FR-016 (niveaudeel) |
| 2 | `NIET_INGESTELD` | storing | `ingesteld` is onwaar (§1, nu twee verplichte waarden) | `ingesteld` is waar | 001 randgeval |

**Vervallen**: `POMP_START_NIET`, `POMP_POMPT_NIET_AF`, `POMP_DRAAIT_TE_LANG` (soort
waarschuwing) en `STROOMSENSOR_STORING` (soort storing). De soort **waarschuwing** komt niet
meer voor (FR-002).

**Apparaatstatus** (text sensor `status`): ALARM > NIET_INGESTELD > STORING > NORMAAL. De
waarde `waarschuwing` vervalt.

**Tekst bij een melding** (FR-009): `<melding>, niveau <x> cm` of `<melding>, niveau onbekend`.
Het deel `pomp <draait|stil|onbekend>` vervalt.

## 4. Gebeurtenis (`Event`, 001 §6)

| Type | Was | Wordt |
|---|---|---|
| `pomp_start` | bij elke bevestigde start | **vervalt** |
| `pomp_stop` | bij elke bevestigde stop, met looptijd | **vervalt** |
| `melding_actief` | met niveau en pompstatus | met niveau, zonder pompstatus (§3) |
| `melding_hersteld`, `herstart`, `instelling_afgewezen`, `niveaumeting_herstart` | — | ongewijzigd |

## 5. Signaleringstoestand (`IndicationState`, 001 §7 en 003 data-model)

Het led-patroon van led 1 kiest uit **zes** in plaats van acht toestanden (research R6, spec
FR-005, FR-007):

| Prio | Toestand | Patroon |
|---|---|---|
| 1 | Alarm | rood, 125 ms aan / 125 ms uit (ongewijzigd) |
| 2 | Storing (niveausensor storing of niet ingesteld) | paars, 500 / 1500 ms (ongewijzigd) |
| 3 | Opstarten | wit, continu (ongewijzigd) |
| 4 | Niveau onbekend | paars, 500 / 1500 ms (ongewijzigd) |
| 5 | Geen verbinding | wit, flits 100 ms per 2 s (ongewijzigd) |
| 6 | Normaal | blauw, continu (ongewijzigd) |

**Vervallen**: "waarschuwing" (oranje, was prio 2) en "pomp draait" (langzaam blauw, was
prio 7).

**Overgangen**: het patroon wordt opnieuw gekozen na elke beoordeling (elke 1 s) en bij een
verandering van verbinding of opstartfase. Een verandering van pompstatus is geen aanleiding
meer.

De niveaubalk (led 2–6, 003 data-model) verandert niet.
