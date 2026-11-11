# Pompput-monitor

Bewaking van een buitenpompput. Een HC-SR04 meet het waterniveau, en een ACS712 ziet aan de
stroom of de pomp draait. Daarmee bepaalt het apparaat zelf de meldingen (hoogwater, pomp start
niet, pomp pompt niet af, pomp draait te lang, sensorstoringen, niet ingesteld) en geeft alles
via de versleutelde native API van ESPHome door aan Home Assistant. Een RGBW-led en een
piëzo-zoemer signaleren ter plaatse, ook zonder Home Assistant. Het apparaat stuurt de pomp
**nooit** aan.

Ontwerp en eisen: [specs/001-pompput-monitor/](specs/001-pompput-monitor/) (spec, plan,
data-model, contracten, quickstart, verificatie).

> **⚠️ Veiligheid: 230 V**
>
> De ACS712 zit in serie met de **fasedraad** van de pomp en staat dus op netspanning.
> - Alleen de fase gaat door de sensor; N en PE blijven ongewijzigd.
> - Monteer de netspanningskant in een aparte, afgesloten behuizing met trekontlasting, op een
>   groep met aardlekschakelaar.
> - Laat het aansluiten op netspanning doen of controleren door een **vakbekwaam persoon**.
>   Goedkope ACS712-modules hebben beperkte kruipwegen.
>
> Zie [contracts/hardware.md](specs/001-pompput-monitor/contracts/hardware.md) en research R7.

## Platform

| Onderdeel | Versie of keuze |
|---|---|
| Firmware | **ESPHome 2026.10.1**, exact vastgepind in [requirements.txt](requirements.txt), in een eigen `.venv` |
| Framework | `esp-idf` (de eigen ESP-IDF-toolchain van ESPHome; PlatformIO is niet nodig) |
| Board | Seeed Studio XIAO ESP32C3 (`esp32: variant: esp32c3`) |
| Koppeling met HA | ESPHome native API met versleuteling; updates via WiFi (OTA) met dezelfde sleutel |
| Componenten | `ultrasonic`, `adc`, `ct_clamp`, `esp32_rmt_led_strip`, `ledc`, `template`, `globals`, `script`, `interval`, `debug`, `wifi_signal`, `uptime` |
| Externe componenten | geen |

Het hele apparaat staat in één bestand: [esphome/pompput.yaml](esphome/pompput.yaml), met
genummerde secties (instellingen, basis, niveau, pomp, meldingen, instellingen in HA,
signalering, gebeurtenissen).

**Pinnen en elektrische aannames**: de gezaghebbende bron is
[specs/001-pompput-monitor/contracts/hardware.md](specs/001-pompput-monitor/contracts/hardware.md).
In de firmware staan de pinnen alleen in het `substitutions:`-blok bovenaan
[esphome/pompput.yaml](esphome/pompput.yaml) (`pin_trig`, `pin_echo`, `pin_acs712`, `pin_led`,
`pin_zoemer`, `pin_knop`); wijzig ze altijd op beide plaatsen tegelijk.

## Aansluiten

Samenvatting voor het solderen. De gezaghebbende bron blijft
[contracts/hardware.md](specs/001-pompput-monitor/contracts/hardware.md); wijzig pinnen daar
én in het `substitutions:`-blok van [esphome/pompput.yaml](esphome/pompput.yaml).

- **Board**: de pinnamen (D0–D10, A0–A2) zijn de opdruk van de **Seeed XIAO ESP32C3**, met
  de bijbehorende GPIO-nummers. Andere ESP32-C3-boards (SuperMini, DevKitM-1, …) hebben een
  andere opdruk en indeling.
- **Niet 5 V-tolerant**: de GPIO's van de ESP32-C3 kunnen geen 5 V aan. De 5 V-signalen van de
  HC-SR04 (ECHO) en de ACS712 (OUT) gaan daarom via een weerstandsdeler.

### Overzicht

| Onderdeel | Aansluiting | XIAO-pin | GPIO | Via |
|---|---|---|---|---|
| HC-SR04 | VCC | 5V | — | rechtstreeks |
| HC-SR04 | TRIG | D2 | 4 | rechtstreeks (veilig; of 3,3 V als "hoog" telt, op de bank controleren) |
| HC-SR04 | ECHO | D3 | 5 | deler: 1 kΩ in serie, 2 kΩ naar GND |
| HC-SR04 | GND | GND | — | |
| ACS712 30 A | VCC | 5V | — | |
| ACS712 30 A | OUT | D1 / A1 | 3 | deler: 10 kΩ in serie, 10 kΩ naar GND (optioneel 10 nF naar GND) |
| ACS712 30 A | GND | GND | — | |
| ACS712 30 A | IP+ / IP− | — | — | in serie met de **fase** naar de pomp (**230 V**) |
| RGBW-led (SK6812) | VDD | 5V | — | via een diode (1N4001 of 1N4148); 100 nF naar GND bij de led |
| RGBW-led (SK6812) | DIN | D4 | 6 | 330 Ω in serie |
| RGBW-led (SK6812) | GND | GND | — | |
| Piëzo-zoemer | + | D5 | 7 | 100 Ω in serie |
| Piëzo-zoemer | − | GND | — | |
| Knop "zoemer stil" | — | D9 | 9 | de BOOT-knop op het board; een externe knop mag parallel naar GND |

Vrij: D0 (GPIO2), D6 (GPIO21), D7 (GPIO20), D8 (GPIO8), D10 (GPIO10).

### Pinnen van de XIAO

Bovenaanzicht, USB-C boven. Controleer de opdruk op je board.

```text
                          USB-C
                     ┌────┴┴┴┴────┐
          (vrij)  D0 │ ●        ● │ 5V    → 5 V-rail: HC-SR04, ACS712, led (via diode)
  ACS712 OUT ──▶  D1 │ ●        ● │ GND   → GND-rail: alle onderdelen
  HC-SR04 TRIG ◀─ D2 │ ●        ● │ 3V3     (niet gebruikt)
  HC-SR04 ECHO ─▶ D3 │ ●        ● │ D10     (vrij)
  led DIN      ◀─ D4 │ ●        ● │ D9      BOOT-knop (op het board)
  zoemer       ◀─ D5 │ ●        ● │ D8      (vrij)
          (vrij)  D6 │ ●        ● │ D7      (vrij)
                     └────────────┘
```

### Per onderdeel

**HC-SR04 (afstand).** Monteer hem loodrecht naar beneden, met vrij zicht (bundel van ongeveer
15°). Het vol-niveau ligt minstens 20 cm onder de sensor.

```text
5V ───────────── VCC
D2 (GPIO4) ───── TRIG
ECHO ──[1 kΩ]──┬── D3 (GPIO5)        deler: 5 V × 2/3 ≈ 3,33 V
               └─[2 kΩ]── GND        (1,8 kΩ geeft ≈ 3,21 V, meer marge)
GND ──────────── GND
```

Waarom TRIG zonder deler kan: TRIG is een ingang van de HC-SR04, en de XIAO stuurt hem met
3,3 V aan. Er komt dus geen 5 V op D2. Alleen ECHO, waar de sensor zelf 5 V terugstuurt, heeft
een deler nodig. Of de HC-SR04 3,3 V als "hoog" herkent, verschilt per module. Bij de meeste
werkt het, maar controleer het op de bank: "Afstand (ruw)" moet kloppende waarden geven.
Werkt het niet betrouwbaar, gebruik dan een HC-SR04P of RCWL-1601. Die werkt op 3,3 V: voed
hem uit de 3V3-pin en laat de ECHO-deler weg.

**ACS712, 30 A-variant (pompstroom), op netspanning.** Lees eerst de veiligheidswaarschuwing
bovenaan. Alleen de fasedraad gaat door de sensor.

```text
230 V fase (L) ──▶ IP+ ┤ ACS712 ├ IP− ──▶ fase naar de pomp     (N en PE ongewijzigd)

5V ── VCC         OUT ──[10 kΩ]──┬── D1/A1 (GPIO3)
GND ── GND                       ├─[10 kΩ]── GND
                                 └─[10 nF]── GND (optioneel, tegen ruis)
```

**RGBW-statusled (SK6812).** Door de diode is VDD ongeveer 4,3 V. Daardoor is het
3,3 V-datasignaal van de XIAO een geldig "hoog", zonder level shifter.

```text
5V ──|>|── VDD                       diode 1N4001 of 1N4148, streep (kathode) aan de led-kant
D4 (GPIO6) ──[330 Ω]── DIN
GND ─────────────────── GND
                                     100 nF tussen VDD en GND, dicht bij de led
```

**Piëzo-zoemer (passief).** De firmware maakt de toon (2,7 kHz) met PWM.

```text
D5 (GPIO7) ──[100 Ω]── piëzo +
                       piëzo − ── GND
```

Is het volume te laag, gebruik dan een NPN-transistor (bijvoorbeeld een BC547). Basis via 1 kΩ
aan D5, emitter aan GND, en de piëzo met 1 kΩ parallel tussen 5 V en de collector.

**Knop "zoemer stil".** Dit is de BOOT-knop op de XIAO (D9), dus er is niets aan te sluiten.
Een externe knop mag parallel tussen D9 en GND. Houd hem niet ingedrukt tijdens het opstarten.

### Voeding

- Een 5 V USB-C-adapter van minimaal 1 A in de USB-C van de XIAO. De 5V-pin van de XIAO voedt
  de HC-SR04, de ACS712 en (via de diode) de led.
- Alle GND's komen samen op de GND van de XIAO.
- Hang de adapter aan een groep die **niet** uitvalt als de beveiliging van de pomp afslaat.

### Controle voor het eerste gebruik

1. Meet zonder stroom door de ACS712 ongeveer **1,25 V** op D1/A1. Meet je 2,5 V, dan
   ontbreekt de deler. Meet je 0 V, dan krijgt de ACS712 geen voeding.
2. Kijk na of de 2 kΩ van de ECHO-deler echt aan **GND** zit. Zonder die verbinding komt er
   bijna 5 V op D3, en dat kan de pin beschadigen. Zitten de 1 kΩ en de 2 kΩ omgekeerd, dan
   komt er maar ongeveer 1,7 V op D3: niet schadelijk, maar de sensor meet dan niets.
3. Kijk na of de diode van de led goed om zit: de streep wijst naar de led.

## Bouwen en flashen

Alle commando's in PowerShell, vanuit de map van de repository.

1. **ESPHome installeren**, eenmalig, in een eigen omgeving met de vastgepinde versie:
   ```powershell
   python -m venv .venv
   .venv\Scripts\python -m pip install -r requirements.txt
   .venv\Scripts\esphome version          # verwacht: Version: 2026.10.1
   ```
2. **Geheimen**: kopieer [esphome/secrets.example.yaml](esphome/secrets.example.yaml) naar
   `esphome/secrets.yaml` (staat in `.gitignore`).
   - Vul `wifi_ssid` en `wifi_password` in; alleen 2,4 GHz.
   - Maak een sleutel voor `api_encryption_key` met het commando dat in het voorbeeldbestand
     staat. Bewaar hem: je hebt hem nodig in Home Assistant, en hij beveiligt ook de updates
     via WiFi.
3. **Controleren en bouwen** (constitutie, gate 1 en 2; noteer RAM en flash):
   ```powershell
   .venv\Scripts\esphome config esphome/pompput.yaml
   .venv\Scripts\esphome compile esphome/pompput.yaml
   ```
4. **De eerste keer flashen, via USB**:
   ```powershell
   .venv\Scripts\esphome run esphome/pompput.yaml --device COM5
   ```
   Vervang COM5 door jouw poort (Apparaatbeheer). Lukt de verbinding niet: houd **B** (BOOT)
   ingedrukt, druk kort op **R** (RESET), laat B los en probeer opnieuw. Druk na het flashen
   op R.
5. **Latere updates via WiFi**, en het log lezen:
   ```powershell
   .venv\Scripts\esphome run esphome/pompput.yaml --device pompput.local
   .venv\Scripts\esphome logs esphome/pompput.yaml
   ```

De bank- en veldvalidatie staan in
[quickstart.md](specs/001-pompput-monitor/quickstart.md). De resultaten, en wat nog niet op
hardware geverifieerd is, staan in
[verificatie.md](specs/001-pompput-monitor/verificatie.md).

## Opbouw

```text
esphome/pompput.yaml           het hele apparaat (ESPHome), in genummerde secties
esphome/secrets.example.yaml   voorbeeld van de geheimen; secrets.yaml staat in .gitignore
homeassistant/                 HA-pakket (meldingen, herinneringen, onbereikbaar) en dashboard
specs/001-pompput-monitor/     spec, plan, research, data-model, contracten, quickstart, taken
requirements.txt               de vastgepinde ESPHome-versie
```

## Home Assistant

1. Het apparaat verschijnt onder *Instellingen → Apparaten en diensten* als "Pompput"
   (ESPHome). Voeg het toe met de sleutel uit `esphome/secrets.yaml`.
2. Zet `packages` aan in `configuration.yaml`, bijvoorbeeld:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
3. Kopieer [homeassistant/packages/pompput.yaml](homeassistant/packages/pompput.yaml) naar de
   map `packages` van Home Assistant.
4. **Vul de notifier in**: vervang in dat bestand elke `REPLACE_ME` door je eigen
   notify-service zonder `notify.`, bijvoorbeeld `mobile_app_mijn_telefoon`.
5. Herstart Home Assistant. Het apparaat staat op `niet_ingesteld`.
6. Vul onder *Configuratie* van het apparaat **Afstand leeg**, **Afstand vol** en
   **Inschakelniveau** in (zie quickstart §5 voor de kalibratie in de put).
7. Optioneel: het voorbeelddashboard
   [homeassistant/dashboard-pompput.yaml](homeassistant/dashboard-pompput.yaml).

Herinneringen (`repeat`, standaard elk uur) en de wachttijd voor "onbereikbaar" (`delay_on`,
standaard 5 minuten) stel je in het pakket in; alle andere drempels zijn `number`-entiteiten van
het apparaat.

## Signalering ter plaatse

| Led | Betekenis |
|---|---|
| rood, snel knipperen | alarm (hoogwater) |
| oranje, 1 Hz | waarschuwing (pomp) |
| paars, kort per 2 s | storing of niet ingesteld |
| wit, continu | opstarten |
| wit, flits per 2 s | geen verbinding met Home Assistant |
| blauw, continu | pomp draait |
| groen, flits per 5 s | normaal |

De BOOT-knop zet de zoemer stil tot er een nieuwe melding bijkomt. Houd de knop **niet**
ingedrukt tijdens het opstarten, want dan gaat de chip in de downloadmodus. Details:
[contracts/lokale-signalering.md](specs/001-pompput-monitor/contracts/lokale-signalering.md).
