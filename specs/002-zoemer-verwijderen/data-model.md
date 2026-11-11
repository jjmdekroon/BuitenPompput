# Data Model: Zoemer verwijderen

**Feature**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md) | **Datum**: 2026-11-11

Deze feature voegt geen gegevens toe. Dit document legt vast wat er verandert ten opzichte van
het datamodel van 001 ([data-model.md](../001-pompput-monitor/data-model.md)). Alles wat hier
niet staat, blijft gelijk.

## 1. Bewaarde instellingen (`Settings`, 001 §1)

| Instelling | Was | Wordt | Spec |
|---|---|---|---|
| `zoemer_aan` (switch, bewaard, standaard aan) | instelling | **vervalt**. De oude waarde blijft ongelezen in flash staan (research R3) | FR-004, FR-009 |
| `led_helderheid_pct` | instelling | ongewijzigd | FR-006 |
| alle overige instellingen | — | ongewijzigd, waarde blijft behouden na de update | FR-009, SC-005 |

**Constanten (001 §1, "vaste waarden")**: de rij "Zoemertoon 2700 Hz, 50 %" vervalt, net als
alle zoemerpatronen, de pauze na een eenmalig patroon, de debounce van de knop en het
bevestigingspatroon van de led.

## 2. Signaleringstoestand (`IndicationState`, 001 §7)

Na de wijziging bestaat de signaleringstoestand alleen nog uit het led-patroon.

| Veld | Was | Wordt |
|---|---|---|
| `led_patroon` | één van 7 patronen, bovenste geldige regel wint; tijdelijk onderbroken door de bevestiging van "stil" | één van 7 patronen, bovenste geldige regel wint; **nooit onderbroken** |
| `led_helderheid` | uit `led_helderheid_pct`, minimaal 10 % bij alarm | ongewijzigd |
| `opstarten`, `verbonden` | hulpwaarden voor de led | ongewijzigd |
| `zoemer_patroon` | alarm, waarschuwing of stil; plus eenmalige piepjes | **vervalt** |
| `stil_voor` | set actieve meldingen op het moment van "stil" | **vervalt** |
| `vorige_set` | set actieve meldingen bij de vorige beoordeling (voor "nieuw" en "hersteld") | **vervalt** (alleen de zoemer gebruikte dit) |

### Overgangen van het led-patroon

Ongewijzigd ten opzichte van [lokale-signalering.md](../001-pompput-monitor/contracts/lokale-signalering.md)
"Statusled": na elke beoordeling (elke 1 s) en bij elke verandering van verbinding, pompstatus
of opstartfase wordt het patroon opnieuw gekozen. Zonder bevestigingspatroon is er geen
tijdelijke toestand meer waarin de keuze wordt overgeslagen.

## 3. Entiteiten in Home Assistant (001 contract "Bediening")

| Entiteit | Wordt |
|---|---|
| `switch.pompput_zoemer` | vervalt |
| `button.pompput_zoemer_stil` | vervalt |
| `button.pompput_niveaumeting_opnieuw_starten` | ongewijzigd |
| `number.pompput_led_helderheid` | ongewijzigd |

Aantal entiteiten: 37 → 35 (14 numbers, 0 switches, 1 button; de rest ongewijzigd).

## 4. Gebeurtenissen (001 §6)

Ongewijzigd. Er was geen gebeurtenis voor de zoemer of het stilzetten.

## 5. Validatieregels

- Er zijn geen nieuwe validatieregels.
- Het apparaat stuurt GPIO7 en GPIO9 nergens aan of uit; zie research R2.
