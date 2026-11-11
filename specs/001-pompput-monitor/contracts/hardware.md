# Contract: Hardware-aansluitingen

**Status**: ontwerp. Dit is de gezaghebbende bron voor pintoewijzing en elektrische aannames
(constitutie, Embedded Platform Constraints). In de firmware staan de pinnen alleen in het
`substitutions:`-blok van `esphome/pompput.yaml` (`pin_trig`, `pin_echo`, `pin_acs712`,
`pin_led`, `pin_zoemer`, `pin_knop`), en dat blok volgt deze tabel. Een wijziging gebeurt hier
en daar tegelijk. (De hardware is ongewijzigd sinds de overstap naar ESPHome op 2026-11-11.)

## Board

- **Seeed Studio XIAO ESP32C3**: ESP32-C3 (RISC-V, 160 MHz, 400 KB SRAM, 4 MB flash), WiFi
  2,4 GHz met externe antenne.
- GPIO-niveau **3,3 V, niet 5 V-tolerant**.
- ADC: alleen **ADC1** (GPIO0–4) gebruiken. Het bruikbare bereik is ongeveer 0–2,5 V bij een
  demping van 11/12 dB.
- Strapping-pinnen: GPIO2, GPIO8 en GPIO9. Ze worden niet gebruikt voor signalen die bij het
  opstarten van buitenaf worden aangestuurd. Uitzondering is GPIO9 (BOOT-knop, alleen aan de
  gebruikerskant).

## Pintoewijzing

| XIAO-pin | GPIO | Richting | Functie | Aansluiting |
|---|---|---|---|---|
| D1 / A1 | 3 | analoog in | ACS712 OUT | via deler 10 kΩ (serie) / 10 kΩ (naar GND); optioneel 10 nF naar GND |
| D2 | 4 | uit | HC-SR04 TRIG | rechtstreeks (3,3 V, bankverificatie) |
| D3 | 5 | in | HC-SR04 ECHO | via deler 1 kΩ (serie) / 2 kΩ (naar GND) → ≈ 3,33 V (of 1,8 kΩ → ≈ 3,21 V) |
| D4 | 6 | uit | SK6812 RGBW DIN | 330 Ω in serie |
| D5 | 7 | uit (PWM) | Piëzo-zoemer | 100 Ω in serie, andere pool naar GND |
| D9 | 9 | in (pull-up op het board) | BOOT-knop = "zoemer stil" | knop op het board; een externe knop mag parallel naar GND |
| 5V | — | voeding | HC-SR04 VCC, ACS712 VCC, led-VDD via diode | 5 V van USB-C |
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

### ACS712, 30 A-variant (pompstroom), netspanning

```text
230 V fase (L) ──▶ IP+ ┤ ACS712 ├ IP− ──▶ fase naar de pomp      (N en PE ongewijzigd)
5V ── VCC         OUT ──[10 kΩ]──┬── A1 (GPIO3)
GND ── GND                       ├─[10 kΩ]── GND
                                 └─[10 nF]── GND (optioneel)
```

- Alleen de **fasedraad** gaat door de sensor. N en PE blijven ongewijzigd.
- De netspanningskant komt in een aparte, afgesloten behuizing met trekontlasting. De
  pompgroep is beveiligd met een aardlekschakelaar.
- Aansluiten op netspanning gebeurt door of onder toezicht van een vakbekwaam persoon.
- Verwacht signaal op A1:
  - bij 0 A ongeveer 1,25 V;
  - de AC-amplitude is gevoeligheid × √2 × I_rms × 0,5. Voorbeeld: de 30 A-variant
    (66 mV/A) bij 3 A geeft ongeveer ±0,14 V.

### Adresseerbare RGBW-led (statusled, SK6812-compatibel)

```text
5V ──|>|── VDD (≈ 4,3 V, diode 1N4001 of 1N4148)
D4 (GPIO6) ──[330 Ω]── DIN
GND ─────────────────── GND        (100 nF tussen VDD en GND dicht bij de led)
```

### Piëzo-zoemer (passief, PWM)

```text
D5 (GPIO7) ──[100 Ω]── piëzo + ;  piëzo − ── GND
```

Is het volume te laag, gebruik dan een NPN-transistor (bijvoorbeeld BC547): de basis via 1 kΩ
aan D5, de piëzo met 1 kΩ parallel tussen 5 V en de collector, en de emitter aan GND.

## Voeding

- 5 V USB-C-adapter van minimaal 1 A.
- Verbruik: XIAO met WiFi ongeveer 100 mA (pieken tot ongeveer 350 mA), HC-SR04 ongeveer
  15 mA, ACS712 ongeveer 10 mA, SK6812 maximaal ongeveer 80 mA (volledig wit; de firmware
  begrenst de helderheid).
- De adapter hangt aan een groep die **niet** uitvalt als de beveiliging van de pomp afslaat.

## Veilige begintoestand (constitutie III)

Bij het opstarten zijn de uitgangen in een veilige toestand voordat de logica draait:
- TRIG laag: de component `ultrasonic` zet de pin bij het opstarten als uitgang, en geeft
  alleen een puls van 10 µs bij een meting;
- de zoemer uit: de `ledc`-output begint uit, en alleen een script zet de toon aan;
- de led uit (`restore_mode: ALWAYS_OFF`, nooit uit flash hersteld), en daarna wit als
  opstartindicatie via `on_boot`.

De firmware heeft geen uitgang die de pomp kan beïnvloeden (FR-007, constitutie III).
GPIO9 (BOOT-knop) is een strapping-pin. In de YAML is die alleen ingang, met
`ignore_strapping_warning: true`.
