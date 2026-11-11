# Quickstart: Zoemer verwijderen valideren

**Feature**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md) | **Datum**: 2026-11-11

Deze gids controleert dat het apparaat zonder zoemer werkt en dat er niets anders is veranderd.
Installeren, bouwen en flashen gaan zoals in
[001 quickstart §2](../001-pompput-monitor/quickstart.md#2-installeren-bouwen-en-de-eerste-keer-flashen).
De uitkomsten komen in `specs/002-zoemer-verwijderen/verificatie.md`.

## 1. Benodigdheden

- Het bestaande, ingestelde apparaat (met alle instellingen gezet) en Home Assistant met de
  ESPHome-integratie.
- **Noteer vóór de update** de waarden van alle instellingen in HA (de 14 `number`-entiteiten),
  voor §3 stap 3.
- Voor scenario #8 en #9: een tweede apparaat of hetzelfde apparaat zonder zoemer gemonteerd,
  en één met de zoemer nog aangesloten.
- De testopstelling van 001 (testbak of object onder de afstandsmeter, belasting op de
  stroomsensor) voor de scenario's in §4.

## 2. Compile- en resource-gate

1. **Compile-gate** (gate 1):
   ```powershell
   .venv\Scripts\esphome config esphome/pompput.yaml
   .venv\Scripts\esphome compile esphome/pompput.yaml
   ```
   **Verwacht**: geen fouten en geen waarschuwingen uit de eigen lambdas.
2. **Resource-gate** (gate 2): noteer RAM en flash. **Verwacht**: RAM ≤ 75 %, flash ≤ 90 %, en
   beide gelijk aan of lager dan de vorige meting in
   [001 verificatie.md](../001-pompput-monitor/verificatie.md) (research R7).
3. **Geen zoemerresten in de configuratie**:
   ```powershell
   Select-String -Path esphome/pompput.yaml -Pattern 'zoemer|ledc|pin_knop|boot_knop|g_stil|bevestiging|signalering_bijwerken|g_vorige_set|g_eenmalig' -CaseSensitive:$false | Where-Object { $_.Line -notmatch '002-zoemer-verwijderen' }
   ```
   **Verwacht**: geen uitvoer (FR-001, FR-004, FR-005). De verwijzing naar
   `specs/002-zoemer-verwijderen/` in de kopcommentaar wordt bewust overgeslagen.

## 3. Bijwerken van een bestaand apparaat

1. Flash via WiFi: `.venv\Scripts\esphome run esphome/pompput.yaml --device pompput.local`.
2. **Verwacht**: het apparaat start opnieuw zonder opstartpiep; de led is wit en daarna groen
   (of blauw als de pomp draait). In HA toont de apparaatinfo versie `0.3.0`.
3. **Instellingen behouden** (FR-009, SC-005): vergelijk de 14 instellingen in HA met de
   genoteerde waarden. **Verwacht**: alle gelijk, inclusief Led-helderheid.
4. **Entiteiten opgeruimd** (FR-004, SC-004): open in HA het apparaat "Pompput".
   **Verwacht**: geen "Zoemer" en geen "Zoemer stil", ook niet als onbeschikbaar. Staan ze er
   toch, verwijder ze dan onder Instellingen → Entiteiten en noteer dit in de verificatie.
5. **Dashboard en meldingen** (FR-010): laad het bijgewerkte pakket en dashboard (HA opnieuw
   starten of YAML herladen). **Verwacht**: de kaart "Bediening" toont alleen "Instellingen
   tonen"; geen rode "entiteit niet gevonden"-regels.

## 4. Scenario's op het apparaat (hardware-gate)

Luister bij elk scenario mee: het apparaat maakt **nooit** geluid (FR-001, SC-001).

| # | Scenario | Handeling | Verwacht | Spec |
|---|---|---|---|---|
| 1 | Opstarten | Onderbreek de voeding en zet hem terug | Geen piep. Led wit, daarna groen; binnen 1 min actuele waarden in HA | US1 scen. 1, FR-001 |
| 2 | Hoogwater | Breng het niveau > 30 s boven de alarmdrempel | Hoogwater-alarm in HA; led snel rood **binnen 5 s** na het actief worden; geen geluid | US1 scen. 2, FR-002, FR-003, FR-008 |
| 3 | Alarm bij lage helderheid | Zet Led-helderheid op 0 % en herhaal #2 | Led rood op minimaal 10 %; andere toestanden blijven uit | FR-002, FR-006 |
| 4 | Pompwaarschuwing | Laat "pomp start niet" ontstaan (001 quickstart §4 #10) | Waarschuwing in HA; led oranje; geen geluid | US1 scen. 3 |
| 5 | Sensorstoring | Onderbreek de voeding van de ACS712 (001 §4 #13) | Stroomsensor storing; led paars; geen piepjes | US1 scen. 3 |
| 6 | Herstel | Laat de melding van #2, #4 of #5 verdwijnen | Herstelmelding in HA; led terug naar groen of blauw; geen herstelpiep | US1 scen. 4, FR-008 |
| 7 | Zonder Home Assistant | Zet de API-verbinding uit (HA stoppen of WiFi weg) en herhaal #2 | Led wit flitsend, bij het alarm snel rood binnen 5 s; geen geluid | US1 scen. 5, FR-003, SC-002 |
| 8 | Zonder zoemer gemonteerd | Herhaal #1, #2, #4 en #5 op een apparaat zonder zoemer | Zelfde uitkomst als hierboven | SC-003 |
| 9 | Oude zoemer nog aangesloten | Herhaal #1, #2 en #5 met de zoemer nog op D5 | De zoemer blijft stil, ook bij opstarten | FR-007, randgeval |
| 10 | BOOT-knop tijdens bedrijf | Druk tijdens een actief alarm kort op de BOOT-knop | Niets verandert: geen witte flitsen, led blijft rood, melding blijft actief | US2 scen. 4, FR-005 |
| 11 | Led-helderheid | Zet Led-helderheid op 80 % en daarna op 10 % | De led volgt binnen 1 s | US2 scen. 5, FR-006 |

## 5. Documentatie (FR-011, SC-006)

```powershell
Select-String -Path README.md, specs/001-pompput-monitor/spec.md, specs/001-pompput-monitor/data-model.md, specs/001-pompput-monitor/quickstart.md, specs/001-pompput-monitor/contracts/*.md, homeassistant/packages/pompput.yaml, homeassistant/dashboard-pompput.yaml -Pattern 'zoemer|piëzo|piep' -CaseSensitive:$false
```

**Verwacht**: alleen treffers die zeggen dat de zoemer is vervallen (verwijzingen naar 002),
geen beschrijving van de zoemer als onderdeel van het apparaat. Dat geldt ook voor de
historische clarifications in `specs/001-pompput-monitor/spec.md`: die blijven staan, maar
eindigen op "(zoemer vervallen met 002)". De testprocedure van 001 beschrijft stilte als "het
apparaat blijft stil", zonder het woord "piep", en geeft dus geen treffers. In het
aansluitschema van de README staat D5 als vrij.

## 6. Traceerbaarheid: requirement → scenario

| Requirement | Scenario's |
|---|---|
| FR-001 | §2 stap 3; §4 #1, #2, #4–#7 |
| FR-002 | §4 #2, #3 |
| FR-003 | §4 #2, #7 |
| FR-004 | §2 stap 3; §3 stap 4 |
| FR-005 | §2 stap 3; §4 #10 |
| FR-006 | §4 #3, #11 |
| FR-007 | §4 #9 |
| FR-008 | §4 #2, #6; §3 stap 2 |
| FR-009 | §3 stap 3 |
| FR-010 | §3 stap 5; §5 |
| FR-011 | §5 |
| SC-001 | §4 alle |
| SC-002 | §4 #2, #7 |
| SC-003 | §4 #8 |
| SC-004 | §3 stap 4, 5 |
| SC-005 | §3 stap 3 |
| SC-006 | §5 |
