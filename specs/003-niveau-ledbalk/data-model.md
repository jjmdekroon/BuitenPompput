# Data Model: Niveau-ledbalk met zes leds

**Feature**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md) | **Datum**: 2026-11-11

Deze feature voegt geen bewaarde gegevens toe. Dit document legt vast wat er verandert ten
opzichte van het datamodel van 001 ([data-model.md](../001-pompput-monitor/data-model.md)),
zoals gewijzigd door [002](../002-zoemer-verwijderen/data-model.md). Alles wat hier niet staat,
blijft gelijk.

## 1. Bewaarde instellingen (`Settings`, 001 §1)

Geen wijziging. De niveaubalk heeft geen eigen instelling.

| Instelling | Rol voor deze feature | Spec |
|---|---|---|
| `alarmdrempel_pct` | bepaalt de grenzen van de niveaubalk (§3) | FR-004, FR-010 |
| `afstand_leeg_cm`, `afstand_vol_cm` | bepalen de vulgraad, dus indirect de balk | FR-010 |
| `led_helderheid_pct` | geldt voor alle zes leds | FR-011 |

**Nieuwe constanten** (vast in de firmware, substitutions):

| Constante | Waarde | Spec |
|---|---|---|
| `led_aantal` | 6 (was 1) | FR-001 |
| `led_index_1` … `led_index_6` | 0 … 5 (positie in de keten) | FR-001 |
| `niveaubalk_stappen` | 5 | FR-004 |
| `niveaubalk_herstelmarge_pct` | 2,0 procentpunt | FR-008 |
| `led_pomp_aan_ms` / `led_pomp_uit_ms` | 1000 / 1000 (was – / 0) | FR-014 |
| `led_normaal_uit_ms` | 0, continu (was 4900) | FR-002, FR-013 |

**Vervallen constanten**: `led_normaal_aan_ms` en de kleur `led_groen_*` (research R3).

## 2. Signaleringstoestand (`IndicationState`, 001 §7)

| Veld | Was | Wordt | In ESPHome |
|---|---|---|---|
| `led_patroon` | één van 7 patronen op de enige led | keuze nu op **led 1**, met één extra regel: na het opstarten en zonder bruikbaar niveau het patroon storing ("niveau onbekend", direct onder opstarten). "Normaal" is continu blauw, "pomp draait" knippert blauw 1 s / 1 s | `led_patroon_kiezen` → `led_knipperen` op partitie `statusled`; `g_led_patroon` |
| `led_helderheid` | uit `led_helderheid_pct`, minimaal 10 % bij alarm | ongewijzigd, voor led 1 | `g_led_helderheid` |
| `opstarten`, `verbonden` | hulpwaarden | ongewijzigd | `g_opstarten`, `g_verbonden` |
| `niveaubalk_aantal` | — | **nieuw**: aantal brandende balkleds, 0–5 (§3) | `g_niveaubalk_aantal` (`uint8_t`, niet bewaard, begin 0) |
| `niveaubalk_helderheid` | — | **nieuw**: helderheid waarmee de balk het laatst is gezet, 0–1 | `g_niveaubalk_helderheid` (`float`, niet bewaard, begin NAN) |

## 3. Niveaubalk (`LevelBar`): regels

Invoer, elke seconde: vulgraad *v* (`g_vulgraad_pct`), `${niveau_bruikbaar}`, alarmdrempel *D*
(`g_alarmdrempel_pct`), helderheid *h* (`led_helderheid_pct` / 100).

**Grenzen** (FR-004): grens*k* = *k* × *D* / 5, voor *k* = 1 … 5. Grens*k* hoort bij led *k* + 1.

| *D* | led 2 | led 3 | led 4 | led 5 | led 6 |
|---|---|---|---|---|---|
| 80 % (standaard) | 16 % | 32 % | 48 % | 64 % | 80 % |
| 60 % | 12 % | 24 % | 36 % | 48 % | 60 % |
| 10 % (minimum) | 2 % | 4 % | 6 % | 8 % | 10 % |
| 120 % (maximum) | 24 % | 48 % | 72 % | 96 % | 120 % |

**Aantal brandende leds** (FR-005 t/m FR-008), met marge *m* = 2,0 procentpunt:

| Situatie | *N* |
|---|---|
| niveau niet bruikbaar, of *v* is NAN | 0, en het vorige aantal wordt ook 0 |
| anders | max(*N*aan, min(*N*vorig, *N*houd)), met *N*aan = #{*k* : *v* ≥ grens*k*} en *N*houd = #{*k* : *v* ≥ grens*k* − *m*} |

**Uitvoer**: led *k* + 1 brandt blauw op helderheid *h* als *k* ≤ *N*; anders uit. Bij *h* = 0 zijn
alle balkleds uit. De balk knippert nooit.

**Overgangen**: de balk heeft geen eigen toestandsmachine; *N* is een afgeleide waarde met
geheugen van één stap (*N*vorig) voor de hysterese. Gezet wordt alleen als *N* of *h* verandert.

**Voorbeelden** (*D* = 80 %):

| *N*vorig | *v* | *N*aan | *N*houd | *N* | Toelichting |
|---|---|---|---|---|---|
| 0 | 50 % | 3 | 3 | 3 | led 2–4 aan |
| 3 | 47 % | 2 | 3 | 3 | onder de grens van led 4 (48 %), maar binnen de marge: blijft aan |
| 3 | 45,9 % | 2 | 2 | 2 | meer dan 2 procentpunt onder 48 %: led 4 uit |
| 2 | 80 % | 5 | 5 | 5 | led 6 aan op de alarmdrempel, los van de wachttijd van het alarm |
| 5 | NAN | — | — | 0 | meting ongeldig: balk uit |

## 4. Fysieke leds (`LedChain`)

| Led | Index | Licht in ESPHome | Rol |
|---|---|---|---|
| 1 | 0 | `statusled` (partitie) | statuslampje |
| 2–6 | 1–5 | `niveauled_2` … `niveauled_6` (partities) | niveaubalk |
| alle | 0–5 | `ledstrip` (`esp32_rmt_led_strip`, intern, nooit direct gestuurd) | driver |

Volgorde en aansluiting: [contracts/ledbalk.md](contracts/ledbalk.md).
