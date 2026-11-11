# Contract: Signalering ter plaatse (led en zoemer)

**Status**: ontwerp. Dit beschrijft wat de gebruiker bij de put ziet en hoort (FR-027, FR-028,
SC-009). Het gedrag is ongewijzigd sinds de overstap naar ESPHome (2026-11-11).
- **Uitvoering**: de scripts `led_patroon_kiezen` en `zoemer_patroon_kiezen` vertalen de
  toestand naar een patroon.
- **Led**: elk patroon speelt via het generieke script `led_knipperen`, met niet-blokkerende
  `delay`s.
- **Zoemer**: elk patroon speelt via de scripts `zoemer_herhalen` en `zoemer_eenmalig`, ook
  met niet-blokkerende `delay`s.
- **Controle**: met [quickstart.md](../quickstart.md) §4.

## Statusled (RGBW)

Er is steeds één patroon actief. De **bovenste regel die geldt, wint**. De helderheid volgt
`led_helderheid_pct`.

| Prio | Toestand | Kleur (R,G,B,W) | Patroon |
|---|---|---|---|
| 1 | **Alarm** (HOOGWATER actief) | rood (255,0,0,0) | snel knipperen: 125 ms aan / 125 ms uit |
| 2 | **Waarschuwing** (een pompwaarschuwing actief) | oranje (255,80,0,0) | 500 ms aan / 500 ms uit |
| 3 | **Storing** (sensorstoring of niet ingesteld) | paars (160,0,255,0) | 500 ms aan / 1500 ms uit |
| 4 | **Opstarten** (nog geen geldige niveaumeting, maximaal 10 s) | wit (0,0,0,255) | continu |
| 5 | **Geen verbinding** met Home Assistant (geen API-verbinding) | wit (0,0,0,255) | flits 100 ms, elke 2 s |
| 6 | **Pomp draait** | blauw (0,0,255,0) | continu |
| 7 | **Normaal** | groen (0,255,0,0) | flits 100 ms, elke 5 s |

- Bij een helderheid van 0 % blijft de led uit, behalve bij prio 1 (alarm). Dat gebeurt dan
  op 10 %, zodat een alarm nooit onzichtbaar is.
- Het gekozen patroon is binnen 1 s na een verandering van toestand zichtbaar. Dat is ruim
  binnen SC-009.

## Zoemer (passieve piëzo, PWM 2,7 kHz, duty 50 %)

| Toestand | Patroon |
|---|---|
| Opstarten | 1× 100 ms (zelftest) |
| **Alarm** actief | 3× (150 ms toon / 150 ms stil), elke 10 s herhaald |
| **Waarschuwing** actief (zonder alarm) | 1× 300 ms, elke 60 s herhaald |
| Storing | 2× 100 ms, **eenmalig** bij het ontstaan |
| Melding hersteld (alles weer normaal) | 1× 50 ms |
| Overig | stil |

- Het eerste patroon klinkt binnen 1 s na het actief worden van een melding (SC-009).
- **Zoemer uit** (HA-switch `zoemer` = OFF): nooit geluid, behalve de zelftest bij het
  opstarten.

## Stilzetten (FR-028)

- **Bediening**: de **BOOT-knop** (D9) kort indrukken (debounce 50 ms), of in HA op de button
  `zoemer_stil` drukken.
- **Bevestiging**: 1× een korte piep van 50 ms, en de led knippert 2× wit.
- **Gevolg**: de zoemer is stil zolang de actieve meldingen een **deelverzameling** zijn van
  de meldingen op het moment van stilzetten. Komt er een **nieuwe** melding bij, dan klinkt de
  zoemer weer.
- **Led**: die blijft altijd de werkelijke toestand tonen, ook als de zoemer stil staat.
- **Na een herstart** is "stil" weer opgeheven, want het wordt niet bewaard.
