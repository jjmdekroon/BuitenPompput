# Contract: Ledbalk (signalering ter plaatse)

**Status**: ontwerp. Dit beschrijft wat de gebruiker bij de put ziet en hoe de zes leds zijn
aangesloten (003 FR-001 t/m FR-015). Bij de uitvoering vervangt dit de inhoud van
[001 contracts/lokale-signalering.md](../../001-pompput-monitor/contracts/lokale-signalering.md),
en worden de led-onderdelen van
[001 contracts/hardware.md](../../001-pompput-monitor/contracts/hardware.md) hierop aangepast.

- **Uitvoering**: het script `led_patroon_kiezen` kiest het patroon van led 1; het script
  `niveaubalk_bijwerken` zet led 2 t/m 6. Beide draaien elke seconde vanuit `status_bijwerken`.
- **Controle**: met [quickstart.md](../quickstart.md) §4.

## 1. Led 1: statuslampje

Er is steeds één patroon actief. De **bovenste regel die geldt, wint**. De helderheid volgt
`led_helderheid_pct`.

| Prio | Toestand | Kleur (R,G,B,W) | Patroon | Wijziging |
|---|---|---|---|---|
| 1 | **Alarm** (HOOGWATER actief) | rood (255,0,0,0) | snel knipperen: 125 ms aan / 125 ms uit | — |
| 2 | **Waarschuwing** (een pompwaarschuwing actief) | oranje (255,80,0,0) | 500 ms aan / 500 ms uit | — |
| 3 | **Storing** (sensorstoring of niet ingesteld) | paars (160,0,255,0) | 500 ms aan / 1500 ms uit | — |
| 4 | **Opstarten** (nog geen geldige niveaumeting, maximaal 10 s) | wit (0,0,0,255) | continu | — |
| 5 | **Niveau onbekend** (na het opstarten is het niveau niet bruikbaar: onbekend, verouderd, storing of niet ingesteld) | paars (160,0,255,0) | 500 ms aan / 1500 ms uit (zelfde patroon als storing) | **nieuw** (FR-017) |
| 6 | **Geen verbinding** met Home Assistant | wit (0,0,0,255) | flits 100 ms, elke 2 s | was prio 5 |
| 7 | **Pomp draait** | blauw (0,0,255,0) | **1000 ms aan / 1000 ms uit** | was prio 6, continu blauw (FR-014) |
| 8 | **Normaal** | **blauw (0,0,255,0)** | **continu** | was prio 7, groen, flits per 5 s (FR-002, FR-013) |

- Prio 5 vult het gat tussen het wegvallen van de meting (na 30 s "verouderd") en de
  storingsmelding (na de wachttijd, standaard 5 min). Zonder deze regel zou led 1 blauw zijn
  met de balk uit, net als bij een lege put. Prio 3 en prio 5 delen hetzelfde patroonnummer,
  zodat de overgang van "niveau onbekend" naar "storing" het knipperen niet onderbreekt.

- Bij een helderheid van 0 % blijft led 1 uit, behalve bij prio 1 (alarm). Die brandt dan op
  10 %, zodat een alarm nooit onzichtbaar is.
- Een nieuw patroon is binnen 1 s na de verandering van toestand zichtbaar (eis ≤ 5 s, FR-009).

## 2. Led 2 t/m 6: niveaubalk

- **Kleur**: blauw (0,0,255,0), **continu**; nooit knipperen.
- **Grenzen**: led *n* (n = 2 … 6) gaat aan bij vulgraad ≥ (*n* − 1) / 5 × alarmdrempel. Bij de
  standaarddrempel 80 %: 16, 32, 48, 64 en 80 %. Tabel en rekenregel:
  [data-model.md §3](../data-model.md#3-niveaubalk-levelbar-regels).
- **Uitgaan**: pas als de vulgraad 2 procentpunt onder de grens van de led is gezakt.
- **Aaneengesloten**: brandt led *n*, dan branden ook led 2 … *n* − 1.
- **Led 6** volgt alleen het niveau, niet het alarm: hij gaat aan op de drempel, ook als de
  wachttijd van het alarm nog loopt, en uit als het niveau zakt, ook als het alarm nog actief is.
- **Ongeldig niveau** (opstarten, verouderd, storing, niet ingesteld, meting uitgevallen): led
  2 t/m 6 uit, en led 1 nooit continu blauw (§1 prio 1, 4 of 5, of een hogere melding).
- **Helderheid**: dezelfde `led_helderheid_pct` als led 1; bij 0 % alle balkleds uit, ook bij een
  alarm.
- Een grensovergang is binnen ongeveer 2 s zichtbaar (eis ≤ 5 s, FR-009).

## 3. Samen gezien (voorbeelden, alarmdrempel 80 %)

`●` = blauw continu, `◐` = blauw knipperend 1 s / 1 s, `R` = rood snel knipperend,
`P` = paars knipperend, `W` = wit, `○` = uit. Led 1 links, led 6 rechts.

| Situatie | Led 1 2 3 4 5 6 |
|---|---|
| Leeg, pomp stil, verbonden | ● ○ ○ ○ ○ ○ |
| 50 %, pomp draait | ◐ ● ● ● ○ ○ |
| 82 %, alarm nog in wachttijd | ● ● ● ● ● ● |
| 82 %, alarm actief | R ● ● ● ● ● |
| Alarm actief, meting uitgevallen | R ○ ○ ○ ○ ○ |
| Meting weg > 30 s, nog geen storingsmelding | P ○ ○ ○ ○ ○ |
| Sensorstoring niveau | P ○ ○ ○ ○ ○ |
| Opstarten | W ○ ○ ○ ○ ○ |

## 4. Aansluiting (vervangt de led in 001 hardware.md)

```text
5V ──|>|──┬──────────┬──────── … ───┬── VDD van led 1 t/m 6 (≈ 4,3 V)
   1N4001 │          │              │
        100 µF     100 nF         100 nF        (100 nF per led; vaak al op de strip)
          │          │              │
GND ──────┴──────────┴──────── … ───┴── GND van led 1 t/m 6

D4 (GPIO6) ──[330 Ω]── DIN led 1   DOUT led 1 ── DIN led 2   …   DOUT led 5 ── DIN led 6
```

- Led 1 zit het dichtst bij het apparaat; led 6 hangt bovenaan.
- **Diode**: 1N4001 (1 A). De 1N4148 is niet meer geschikt voor zes leds (research R4).
- **Verbruik**: hoogstens ≈ 146 mA voor de leds (led 1 paars en vijf blauwe balkleds op 100 %);
  totaal met XIAO en sensoren ≈ 520 mA piek. De 5 V-adapter van minimaal 1 A blijft voldoende.
- **Veilige begintoestand** (constitutie III): de strip en alle zes partities hebben
  `restore_mode: ALWAYS_OFF`; vanaf het opstarten zijn alle leds uit, daarna toont led 1 wit
  (opstarten) en blijft de balk uit tot er een geldige niveaumeting is.

## 5. Home Assistant

Geen wijziging: de strip en de partities zijn intern. De enige bediening blijft de
`number` "Led-helderheid".
