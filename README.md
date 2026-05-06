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

## Steuerlogik (`automation.yml`)

Trigger: jede Minute (`time_pattern /1`), `mode: single`.

### Branches (Reihenfolge = Priorität)

| # | Branch | Bedingung | Aktion |
|---|---|---|---|
| 1 | 🌙 Nacht – Notladen | `ist_nacht` + heute & morgen schlecht + `soc < soc_min + 10` | Mode `input`, 200 W aus Netz |
| 2 | 🌙 Nacht – Platz schaffen | `ist_nacht` + `prog_morgen_hoch` + `soc > soc_min` + `netz < -50` | Mode `output`, `min(\|netz\|, max_entlade)` |
| 3 | 🌙 Nacht – Schonen | `ist_nacht` + morgen schlecht + `soc > soc_min + 15` + `netz < -50` | Mode `output`, `min(\|netz/2\|, max_entlade)` |
| 4 | 🌙 Nacht – Default | `ist_nacht` (kein Match oben) | Limits = 0 (passiv) |
| 5 | ☀️ Tag – Überschuss | `netz > 50` + `soc < soc_max` | Mode `input`, `min(¾·netz, max_lade)` |
| 6 | 🟢 Tag – Bezug | `netz < -50` + `soc > soc_min` | Mode `output`, prognoseabhängig (s.u.) |
| 7 | 🟰 Default | Deadband oder SoC-Grenze | Limits = 0 (passiv) |

**Tag-Entladung (Branch 6):**
- Gute Prognose (heute ODER morgen): `min(|netz|, max_entlade)` – voll decken
- Schlechte Prognose: `min(|netz/2|, max_entlade)` – nur halb decken

### Tuning-Variablen

| Variable | Default | Funktion |
|---|---|---|
| `deadband` | 50 W | Hysterese gegen Pendeln |
| `reserve_offset` | 15 % | Nacht-Reserve bei schlechter Prognose |
| `notlade_offset` | 10 % | Schwelle über `soc_min` für Notladen |
| `notlade_power` | 200 W | Notlade-Leistung aus dem Netz |

---

## Dynamische Leistungs-Caps (`template.yml`)

### Max Ladeleistung

Kernfrage: *"Wie schnell laden?"* – Bei schlechter Prognose aggressiv, bei guter sanfter (schont Batterie).

| Bedingung | Leistung | Begründung |
|---|---|---|
| `soc >= soc_max` | 0 W | Voll, nicht überladen |
| Tagesprognose schlecht | 800 W | Jede kWh mitnehmen die kommt |
| Gute Prognose, fast voll (< 10% Platz) | 266 W | Sanft vollladen, PV kommt noch |
| Gute Prognose + gerade viel PV | 533 W | Moderat, es kommt noch mehr |
| Gute Prognose, noch Platz, gerade wenig PV | 800 W | Normal laden wenn's da ist |

### Max Entladeleistung

Kernfrage: *"Wie viel können wir uns leisten?"* – Prognose bestimmt Aggressivität, SoC-Reserve drosselt.

| Bedingung | Leistung | Begründung |
|---|---|---|
| `soc <= soc_min` | 0 W | Leer, Schutz |
| Morgen kommt PV & Reserve > 20% | 800 W | Wird morgen sicher aufgefüllt |
| Heute kommt noch PV & Reserve > 15% | 600 W | Wird heute teilweise aufgefüllt |
| Schlechte Prognose, Reserve > 30% | 400 W | Viel Puffer, maßvoll nutzen |
| Schlechte Prognose, Reserve > 10% | 200 W | Wenig Puffer, sparsam |
| Knapp über soc_min | 133 W | Notreserve dehnen |

> `reserve` = `soc - soc_min` (nutzbarer Bereich über dem Minimum)

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

`binary_sensor.ist_nacht`: **on** wenn Sonnenaufgang + 2h noch vor Sonnenuntergang − 3h liegt.  
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

Im Tag-Entlade-Branch (`🟢 TAG – Netzbezug`) werden zusätzliche Variablen berechnet:

```yaml
erwartet:           # Erwarteter Verbrauch nächste Periode (W, 7d-Mittel)
reserve_wh:         # Verfügbare Energie über soc_min (ca. 24 Wh pro %-Punkt)
naechste_dauer:     # Dauer der nächsten Periode in Stunden
naechste_bedarf_wh: # erwartet × Dauer = geschätzter Gesamtbedarf (Wh)
```

**Entscheidungsmatrix:**

| Bedingung | Entladeleistung | Begründung |
|---|---|---|
| Gute PV-Prognose (heute/morgen) | 100% von `|netz|` | Wird sicher wieder aufgefüllt |
| Reserve > Bedarf nächste Periode | 67% von `|netz|` | Genug Puffer, moderat nutzen |
| Reserve ≤ Bedarf nächste Periode | 33% von `|netz|` | Strecken für später |

### Hinweise

- Die Statistics-Sensoren brauchen **7 Tage Anlaufzeit** für aussagekräftige Daten.
- Bis dahin gelten Default-Werte: Morgen 300W, Vormittag 400W, Nachmittag 350W, Abend 500W, Nacht 200W.
- Die Reserve-Berechnung `(soc - soc_min) × 24 Wh` basiert auf ~2,4 kWh nutzbarer Kapazität des SolarFlow 2400 AC.
- Die Perioden-Dauer ist fest hinterlegt: Morgen 4h, Vormittag 2h, Nachmittag 5h, Abend 5h, Nacht 8h.

