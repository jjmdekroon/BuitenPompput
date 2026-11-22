# Contract: Hardware-aansluitingen

**Status**: ontwerp. Dit is de gezaghebbende bron voor pintoewijzing en elektrische aannames
(constitutie, Embedded Platform Constraints). In de firmware staan de pinnen alleen in het
`substitutions:`-blok van `esphome/pompput.yaml` (`pin_trig`, `pin_echo`, `pin_led`), en dat
blok volgt deze tabel. Een wijziging gebeurt hier en daar tegelijk.
D1/A1 is vrij sinds 004-stroomsensor-verwijderen: de stroomsensor (ACS712) en de pompbewaking
zijn vervallen ([specs/004-stroomsensor-verwijderen](../../004-stroomsensor-verwijderen/spec.md)).
D5 is vrij sinds 002-zoemer-verwijderen: de piëzo-zoemer en de functie van de BOOT-knop zijn
vervallen ([specs/002-zoemer-verwijderen](../../002-zoemer-verwijderen/spec.md)).
Gewijzigd door [003](../../003-niveau-ledbalk/spec.md): zes leds in plaats van één. Het aantal
leds en hun volgorde staan in `led_aantal` en `led_index_1` … `led_index_6`.

## Board

- **Seeed Studio XIAO ESP32C3**: ESP32-C3 (RISC-V, 160 MHz, 400 KB SRAM, 4 MB flash), WiFi
  2,4 GHz met externe antenne.
- GPIO-niveau **3,3 V, niet 5 V-tolerant**.
- ADC: alleen **ADC1** (GPIO0–4) gebruiken. Het bruikbare bereik is ongeveer 0–2,5 V bij een
  demping van 11/12 dB.
- Strapping-pinnen: GPIO2, GPIO8 en GPIO9. Ze worden niet gebruikt voor signalen die bij het
  opstarten van buitenaf worden aangestuurd. GPIO9 is de BOOT-knop op het board; de firmware
  gebruikt die pin niet.

## Pintoewijzing

| XIAO-pin | GPIO | Richting | Functie | Aansluiting |
|---|---|---|---|---|
| D1 / A1 | 3 | — | vrij (was ACS712 OUT, vervallen met 004) | — |
| D2 | 4 | uit | HC-SR04 TRIG | rechtstreeks (3,3 V, bankverificatie) |
| D3 | 5 | in | HC-SR04 ECHO | via deler 1 kΩ (serie) / 2 kΩ (naar GND) → ≈ 3,33 V (of 1,8 kΩ → ≈ 3,21 V) |
| D4 | 6 | uit | DIN van led 1 van een keten van 6 SK6812-RGBW-leds | 330 Ω in serie |
| D5 | 7 | — | vrij (was piëzo-zoemer, vervallen met 002) | — |
| D9 | 9 | — | BOOT-knop op het board, geen functie in de firmware | alleen voor de downloadmodus bij het opstarten |
| 5V | — | voeding | HC-SR04 VCC, VDD van de zes leds via diode 1N4001 | 5 V van USB-C |
| GND | — | — | gemeenschappelijke massa | alle modules |
| D0, D6, D7, D8, D10 | 2, 21, 20, 8, 10 | — | vrij | niet gebruikt |

## Schakeling per onderdeel

### HC-SR04 (afstand)

```text
5V ───────────── VCC
D2 (GPIO4) ───── TRIG
ECHO ──[1 kΩ]──┬── D3 (GPIO5)
               └─[2 kΩ]── GND
GND ──────────── GND
```

- Monteer de sensor loodrecht naar beneden, met vrij zicht (bundel van ongeveer 15°).
- Het vol-niveau ligt minstens 20 cm onder de sensor.
- Bescherm de printplaat tegen vocht, met alleen de transducers open.

### Een bestaande stroomsensor weghalen

Tot en met firmware 0.4.0 zat er een ACS712 in de fasedraad van de pomp, met OUT via een deler
10 kΩ / 10 kΩ op A1. Sinds 004 leest de firmware die sensor niet meer (004 FR-012, FR-015).

1. **Blijven zitten mag.** De firmware configureert A1 niet; de pin krijgt via de deler
   hoogstens ongeveer 2,5 V, binnen het bereik van een 3,3 V-ingang (004 research R5).
2. **Weghalen is werk aan netspanning.** Dat gebeurt door of onder toezicht van een vakbekwaam
   persoon, met de pompgroep spanningsloos.
3. **Verbind de fasedraad weer door.** De fase naar de pomp liep door de sensor (IP+ → IP−).
   Maak een geschikte, geïsoleerde verbinding in een afgesloten behuizing met trekontlasting.
   N en PE blijven ongewijzigd.
4. **Controleer de pomp.** Zet de groep weer aan en controleer dat de pomp op zijn eigen vlotter
   aanslaat en afslaat (001 SC-007).
5. **Laagspanningskant.** Haal de deler en de draden naar A1, 5V en GND van de sensor weg, of
   laat ze los en geïsoleerd achter.

### Ledketen (6 × SK6812 RGBW, ledbalk)

```text
5V ──|>|──┬──────────┬──────── … ───┬── VDD van led 1 t/m 6 (≈ 4,3 V)
   1N4001 │          │              │
        100 µF     100 nF         100 nF        (100 nF per led; vaak al op de strip)
          │          │              │
GND ──────┴──────────┴──────── … ───┴── GND van led 1 t/m 6

D4 (GPIO6) ──[330 Ω]── DIN led 1   DOUT led 1 ── DIN led 2   …   DOUT led 5 ── DIN led 6
```

- Led 1 is de eerste in de keten (het dichtst bij het apparaat) en is het statuslampje; led 2
  t/m 6 vormen de niveaubalk, led 6 hangt bovenaan (003 contracts/ledbalk.md).
- **Diode**: 1N4001 (1 A). De 1N4148 is niet geschikt voor zes leds: bij ongeveer 150 mA zakt
  VDD te ver voor een betrouwbaar 3,3 V-datasignaal (003 research R4).
- Door de diode is VDD ongeveer 4,3 V; de ingangsdrempel van led 1 ligt dan rond 3,0 V, dus de
  3,3 V-data van de XIAO is betrouwbaar. De volgende leds krijgen een opgefrist signaal van hun
  voorganger.
- Elko van 100 µF tussen VDD en GND bij led 1, en 100 nF per led.

## Voeding

- 5 V USB-C-adapter van minimaal 1 A.
- Verbruik: XIAO met WiFi ongeveer 100 mA (pieken tot ongeveer 350 mA), HC-SR04 ongeveer
  15 mA, de zes leds samen hoogstens ongeveer 146 mA (led 1 paars en vijf blauwe balkleds op
  100 %; de firmware gebruikt nooit volledig wit op meerdere leds).
  Totaal ongeveer 510 mA piek; een adapter van 1 A blijft voldoende (003 research R4).
- De adapter hangt aan een groep die **niet** uitvalt als de beveiliging van de pomp afslaat.

## Veilige begintoestand (constitutie III)

Bij het opstarten zijn de uitgangen in een veilige toestand voordat de logica draait:
- TRIG laag: de component `ultrasonic` zet de pin bij het opstarten als uitgang, en geeft
  alleen een puls van 10 µs bij een meting;
- alle leds uit (`restore_mode: ALWAYS_OFF` op de strip `ledstrip` en op de zes partities,
  nooit uit flash hersteld); daarna led 1 wit als opstartindicatie via `on_boot`, en de
  niveaubalk uit tot er een geldige niveaumeting is (003 FR-007, FR-012);
- GPIO7 (D5) wordt niet geconfigureerd; een nog aangesloten passieve zoemer blijft stil
  (002 research R2);
- GPIO3 (A1) wordt niet geconfigureerd; een nog aangesloten ACS712 geeft via de deler
  hoogstens ongeveer 2,5 V (004 research R5).

De firmware heeft geen uitgang die de pomp kan beïnvloeden (FR-007, constitutie III).
GPIO9 (BOOT-knop) is een strapping-pin. De firmware leest de pin niet meer (sinds 002); de
knop werkt alleen nog bij het opstarten, voor de downloadmodus.
