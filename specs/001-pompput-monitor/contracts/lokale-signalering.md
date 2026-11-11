# Contract: Signalering ter plaatse (statusled)

**Status**: ontwerp. Dit beschrijft wat de gebruiker bij de put ziet (FR-027, FR-028 voor de
helderheid, SC-009; 002 FR-002, FR-003). De zoemer, het stilzetten en de functie van de
BOOT-knop zijn vervallen met
[specs/002-zoemer-verwijderen](../../002-zoemer-verwijderen/spec.md); ter plaatse signaleert
alleen nog de statusled.
- **Uitvoering**: het script `led_patroon_kiezen` vertaalt de toestand naar een patroon.
- **Led**: elk patroon speelt via het generieke script `led_knipperen`, met niet-blokkerende
  `delay`s.
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
