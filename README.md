# Zendure SolarFlow 2400 AC – Home Assistant Automatik

Steuerung eines Zendure SolarFlow 2400 AC Speichers über Home Assistant.  
Hardware: SMA Wechselrichter + Smartmeter, 6,8 kWp PV, Zendure SolarFlow 2400 AC.

## Ziele

- **Möglichst kein Netzbezug** – Eigenverbrauch maximieren
- **PV-Überschuss → Speicher füllen** – nichts verschenken
- **Nacht → Speicher entleeren** – Platz für den nächsten Tag
- **Gute Prognose → Speicher voll nutzen** – aggressiv ent-/beladen
- **Schlechte Prognose → Speicher schonen** – Reserve für schlechte Tage

---

## Architektur / Datenfluss

```
Smartmeter (Einspeisung - Bezug)
  ↓  minus Speicher-Entladung + Speicher-Ladung  (Kompensation)
sensor.nettoleistung_netz              ← template.yml (bereinigter Momentanwert)
  ↓  statistics mean (20 samples / 120s)
sensor.nettoleistung_netz_mittelwert   ← template.yml (geglätteter Wert)
  ↓
Variable "netz" in automation.yml      → Entscheidungslogik → Service-Aufrufe
```

Die Speicherkompensation stellt sicher, dass `netz` den **wahren Haushaltsbedarf** zeigt  
(nicht den bereits vom Speicher veränderten Wert am Smartmeter).

---

## Dateien

| Datei | Verantwortung |
|---|---|
| `template.yml` | Datenaufbereitung: Sensoren, Prognose-Flags, dynamische Leistungs-Caps |
| `automation.yml` | Steuerlogik: Entscheidungs-Branches, Service-Aufrufe |
| `AGENTS.md` | Konventionen & Hinweise für KI-Coding-Agents |

---

## Manuell anzulegende Helper

Vor dem ersten Start müssen folgende Helper in Home Assistant angelegt werden  
(**Einstellungen → Geräte & Dienste → Helfer → Helfer erstellen**):

| Typ | Entity-ID                                     | Name                           | Werte / Einstellung                                          | Zweck                                                                                                                                                           |
|---|-----------------------------------------------|--------------------------------|--------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 🔢 Zahl | `input_number.pv_peak_leistung`               | PV Peak Leistung               | min 0, max 20000, Schritt 100, Einheit `W`, Default **6800** | Bezugsgröße für Prognose-Schwellen (PV-Anlagengröße in W)                                                                                                       |
| 🔘 Schalter | `input_boolean.zendure_verbrauchsprognose`    | Zendure Verbrauchsprognose     | Default an                                                   | Aktiviert das zusätzliche Verbrauchs-Cap im Template-Sensor `Zendure Max Entladeleistung` (drosselt bei knapper Reserve auf 200 W). Aus = nur SoC + PV-Prognose |
| 📝 Text | `input_text.zendure_status`                   | Zendure Status                 | min 0, max 60                                                | Wird von der Automation pro Branch gesetzt; `sensor.zendure_steuerung_status` liest hieraus                                                                     |
| 🔢 Zahl | `input_number.zendure_grenze_ladeleistung`    | Zendure Grenze Ladeleistung    | min 100, max 1200, Schritt 50, Einheit `W`, Default **800**  | Obergrenze für Ladeleistung                                                                                                                                     |
| 🔢 Zahl | `input_number.zendure_grenze_entladeleistung` | Zendure Grenze Entladeleistung | min 100, max 1000, Schritt 50, Einheit `W`, Default **800**  | Obergrenze für Entladeleistung                                                                                                                                  |

> **Wichtig**: Solange `input_text.zendure_status` nicht existiert, schlägt jeder `set_value`-Aufruf in der Automation fehl und der Branch bricht ab. Helper **zuerst** anlegen, dann YAML neu laden.

### Schnellanlage via YAML (`configuration.yaml`)

Alternativ direkt per YAML statt UI:

```yaml
input_number:
  pv_peak_leistung:
    name: PV Peak Leistung
    min: 0
    max: 20000
    step: 100
    unit_of_measurement: W
    initial: 6800

input_boolean:
  zendure_verbrauchsprognose:
    name: Zendure Verbrauchsprognose
    icon: mdi:chart-timeline-variant
    initial: true

input_text:
  zendure_status:
    name: Zendure Status
    min: 0
    max: 60
    initial: "🟰 Passiv (Deadband)"

input_number:
  zendure_grenze_ladeleistung:
    name: Zendure Grenze Ladeleistung
    min: 100
    max: 1200
    step: 50
    unit_of_measurement: W
    initial: 800

input_number:
  zendure_grenze_entladeleistung:
    name: Zendure Grenze Entladeleistung
    min: 100
    max: 1000
    step: 50
    unit_of_measurement: W
    initial: 800
```

---

## Steuerlogik (`automation.yml`)

Trigger: jede Minute (`time_pattern /1`), `mode: single`.

### Branches (Reihenfolge = Priorität)

| # | Branch | Bedingung | Aktion                                            |
|---|---|---|---------------------------------------------------|
| 1 | 🌙 Nacht – Notladen | `ist_nacht` + heute & morgen schlecht + `soc < soc_min + 10` | Mode `input`, 200 W aus Netz                      |
| 2 | 🌙 Nacht – Platz schaffen | `ist_nacht` + `prog_morgen_hoch` + `soc > soc_min` + `netz < -50` | Mode `output`, `min(\|110% netz\|, max_entlade)` |
| 3 | 🌙 Nacht – Schonen | `ist_nacht` + morgen schlecht + `soc > soc_min + 15` + `netz < -50` | Mode `output`, `min(\|80% netz\|, max_entlade)` |
| 4 | 🌙 Nacht – Default | `ist_nacht` (kein Match oben) | Limits = 0 (passiv)                               |
| 5 | ☀️ Tag – Überschuss | `netz > 50` + `soc < soc_max` | Mode `input`, `min(\|netz\|, max_lade)` |
| 6 | 🟢 Tag – Bezug | `netz < -50` + `soc > soc_min` | Mode `output`, `min(\|netz\|, max_entlade)` |
| 7 | 🟰 Default | Deadband oder SoC-Grenze | Limits = 0 (passiv)                               |

**Tag-Entladung (Branch 6):**  
Einheitliche Regel: `min(|netz|, max_entlade)`. Die gesamte Drosselung (SoC, Prognose, Reserve, optional Verbrauchsprognose) lebt im Template-Sensor `Zendure Max Entladeleistung` – siehe unten. Der Branch ist damit „dumm" und nur Ausführer.

### Tuning-Variablen

| Variable | Default | Funktion |
|---|---|---|
| `deadband` | 50 W | Hysterese gegen Pendeln |
| `reserve_offset` | 15 % | Nacht-Reserve bei schlechter Prognose |
| `notlade_offset` | 10 % | Schwelle über `soc_min` für Notladen |
| `notlade_power` | 200 W | Notlade-Leistung aus dem Netz |

---

## Dynamische Leistungs-Caps (`template.yml`)

Wert + Status + Icon werden im **gleichen Template-Sensor** berechnet. Der Wert ist der `state`, der Status-Text liegt als Attribut `status`, das Icon als Attribut `icon`. Die separaten Status-Sensoren sind nur dünne Reader (`state_attr(... 'status')` / `state_attr(... 'icon')`).

Innerhalb eines Wert-Sensors steht die if-elif-Kette dreimal (state, attr.status, attr.icon) – HA evaluiert jedes Template eigenständig. Alle drei Blöcke sind aber **zeilengleich strukturiert**: jeder Branch setzt `ns.cap`, `ns.status` und `ns.icon` in einer Zeile, sodass Drift sofort sichtbar wäre.

| Wert-Sensor (W) | Status-Sensor (Text + Icon) |
|---|---|
| `sensor.zendure_max_ladeleistung` (Attr.: `status`, `icon`) | `sensor.zendure_max_ladeleistung_status` |
| `sensor.zendure_max_entladeleistung` (Attr.: `status`, `icon`) | `sensor.zendure_max_entladeleistung_status` |

### Max Ladeleistung

Kernfrage: *"Wie schnell laden?"* – Bei schlechter Prognose aggressiv, bei guter sanfter (schont Batterie).

| Bedingung | Leistung | Status |
|---|---|---|
| Basiswert | 800 W | – |
| `soc >= soc_max` | 0 W | 🚫 Voll – kein Laden |
| Tagesprognose schlecht | 1/1 | 🚀 Aggressiv – schlechte Prognose |
| Gute Prognose, fast voll (< 10% Platz) | 1/3 | 🐢 Sanft – fast voll |
| Gute Prognose + gerade viel PV | 2/3 | 🐇 Moderat – viel PV erwartet |
| Gute Prognose, noch Platz, gerade wenig PV | 1/1 | ⚡ Normal – Standardrate |

### Max Entladeleistung

Kernfrage: *"Wie viel können wir uns leisten?"* – Prognose bestimmt Aggressivität, SoC-Reserve drosselt.

| Bedingung | Leistung | Status |
|---|---|---|
| Basiswert | 800 W | – |
| `soc <= soc_min` | 0 W | 🚫 Leer – kein Entladen |
| Morgen kommt PV & Reserve > 20% | 1/1 | 🚀 Voll – Morgen kommt PV |
| Heute kommt noch PV & Reserve > 15% | 3/4 | 🟢 Moderat – Heute kommt PV |
| Schlechte Prognose, Reserve > 30% | 1/2 | 🟡 Halb – viel Reserve |
| Schlechte Prognose, Reserve > 10% | 1/4 | 🟠 Vorsichtig – wenig Reserve |
| Knapp über soc_min | 1/6 | 🔴 Notreserve – knapp über Min |
| Verbrauchs-Cap aktiv (s.u.) | max 200 W (= base/4) | 🛡️ Verbrauchs-Cap (25%) |

> `reserve` = `soc - soc_min` (nutzbarer Bereich über dem Minimum)

**Optionaler zusätzlicher Cap durch Verbrauchsprognose:**

Wenn `input_boolean.zendure_verbrauchsprognose = on` UND keine PV-Prognose-Hilfe (heute & morgen schlecht) UND die Speicher-Reserve in Wh kleiner ist als der erwartete Bedarf der **nächsten** Tageszeit-Periode (7-Tage-Mittel × Periodendauer) UND der Basis-Cap > 200 W liegt, wird zusätzlich auf maximal `base/4` (= 200 W) gedeckelt. Status: `🛡️ Verbrauchs-Cap (25%)`.

| Toggle | Verhalten |
|---|---|
| **AN** (Default) | Volle Logik inkl. Verbrauchs-Vorausschau – schont den Speicher wenn Reserve knapp für den nächsten Block |
| **AUS** | Nur SoC + PV-Prognose – einfacher, aggressiver |

---

## Prognose-Sensoren (`template.yml`)

Alle relativ zu `input_number.pv_peak_leistung` (für 6,8 kWp = **6800**):

| Sensor | Schwelle | Datenquelle |
|---|---|---|
| `PV Prognose Stunde hoch` | `energy_next_hour >= peak / 2000` | Forecast Solar |
| `PV Prognose Heute hoch` | `energy_production_today_remaining >= peak / 1000` | Forecast Solar |
| `PV Prognose Morgen hoch` | `energy_production_tomorrow >= 3 · peak / 1000` | Forecast Solar |

States sind Strings `'True'` / `'False'` (Großschreibung!).

---

## Nachtsensor

`binary_sensor.ist_nacht`: **on**, wenn Sonnenaufgang + 2h noch vor Sonnenuntergang − 3h liegt.  
→ Nacht beginnt früher (3h vor Sunset) und endet später (2h nach Sunrise).

---

## Vorzeichenkonvention

| Wert | Bedeutung |
|---|---|
| `netz > 0` | Einspeisung ins Netz (PV-Überschuss) |
| `netz < 0` | Bezug aus dem Netz (Verbrauch > Erzeugung) |

---

## Wichtige Entity-IDs

**SMA Smartmeter:**
- `sensor.sn_3007669104_metering_power_supplied`
- `sensor.sn_3007669104_metering_power_absorbed`

**Zendure SolarFlow 2400 AC:**
- `sensor.solarflow_2400_ac_electric_level` (SoC)
- `sensor.solarflow_2400_ac_min_soc`
- `number.solarflow_2400_ac_soc_set` (Max-SoC)
- `number.solarflow_2400_ac_input_limit` / `output_limit`
- `select.solarflow_2400_ac_ac_mode` (`input` / `output`)
- `sensor.solarflow_2400_ac_output_pack_power` (Ist-Entladung)
- `sensor.solarflow_2400_ac_pack_input_power` (Ist-Ladung)

**Forecast Solar:**
- `sensor.energy_next_hour`
- `sensor.energy_production_today_remaining`
- `sensor.energy_production_tomorrow`

**Sonstige:**
- `input_number.pv_peak_leistung` (6800 für 6,8 kWp)
- `sensor.sun_next_rising` / `sensor.sun_next_setting`

---

## Verbrauchsanalyse nach Tageszeit

### Konzept

Der historische Stromverbrauch wird in 5 Tageszeit-Perioden erfasst und über 7 Tage gemittelt.
So kann die Automation vorausschauend entscheiden: „Reicht die Speicherreserve für die nächste Periode?"

### Zeitfenster

| Periode | Stunden | Typischer Verbrauch |
|---|---|---|
| Morgen | 06:00 – 09:59 | Frühstück, Warmwasser |
| Vormittag | 10:00 – 11:59 | Haushalt, Büro |
| Nachmittag | 12:00 – 16:59 | Kochen, Haushalt |
| Abend | 17:00 – 21:59 | Kochen, Licht, Unterhaltung |
| Nacht | 22:00 – 05:59 | Standby, Kühlschrank |

### Sensor-Kette

```
sensor.sn_..._metering_power_absorbed  (Momentan-Bezug in W)
  ↓  gefiltert nach Zeitfenster (availability-Template)
sensor.verbrauch_morgen / …vormittag / …nachmittag / …abend / …nacht
  ↓  statistics mean, max_age: 7 Tage
sensor.durchschnitt_verbrauch_morgen_7d / …vormittag_7d / …nachmittag_7d / …abend_7d / …nacht_7d
  ↓  Lookup nach nächster Tageszeit
sensor.erwarteter_verbrauch_naechste_periode  (W)
```

### Hilfs-Sensoren

| Sensor | Funktion |
|---|---|
| `sensor.aktuelle_tageszeit` | Aktuelle Periode (Morgen/Vormittag/…) |
| `sensor.naechste_tageszeit` | Die darauffolgende Periode |
| `sensor.erwarteter_verbrauch_naechste_periode` | 7d-Durchschnitt der nächsten Periode (W) |

### Integration in die Entlade-Entscheidung

Die Verbrauchsprognose wirkt **ausschließlich als zusätzliches Cap** im Template-Sensor `Zendure Max Entladeleistung` – nicht in der Automation. Aktiviert wird sie über den Helper `input_boolean.zendure_verbrauchsprognose`.

Berechnete Werte (in `template.yml`):

```yaml
erwartet:        # Erwarteter Verbrauch nächste Periode (W, 7d-Mittel)
reserve_wh:      # Verfügbare Energie über soc_min (kapazität × reserve%)
naechste_dauer:  # Dauer der nächsten Periode (h)
bedarf_wh:       # erwartet × Dauer = geschätzter Gesamtbedarf (Wh)
```

**Cap-Logik:**

| Toggle | PV-Prognose | Reserve vs. Bedarf | Effekt |
|---|---|---|---|
| AN | gut (heute oder morgen) | – | kein Verbrauchs-Cap, normale Logik |
| AN | schlecht | Reserve ≥ Bedarf | kein Verbrauchs-Cap, normale Logik |
| AN | schlecht | Reserve < Bedarf | **zusätzliches Cap auf 200 W** |
| AUS | – | – | nie ein Verbrauchs-Cap |

### Hinweise

- Die Statistics-Sensoren brauchen **7 Tage Anlaufzeit** für aussagekräftige Daten.
- Bis dahin gelten Default-Werte: Morgen 300W, Vormittag 400W, Nachmittag 350W, Abend 500W, Nacht 200W.
- Die Reserve-Berechnung nutzt `sensor.solarflow_2400_ac_total_kwh` (Gesamtkapazität in kWh) – passt sich automatisch an, wenn Module ergänzt werden.
- Die Perioden-Dauer ist fest hinterlegt: Morgen 4h, Vormittag 2h, Nachmittag 5h, Abend 5h, Nacht 8h.
- `sampling_size: 10000` bei den 7d-Statistics, sonst werden nur die letzten 20 Samples gemittelt.

