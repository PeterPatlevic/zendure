# AGENTS Guide for `zendure`

## Scope & Big Picture
- Reines Home-Assistant-YAML-Repo: `template.yml` (abgeleitete Sensoren/Schwellen) und `automation.yml` (Steuer-Loop).
- Datenfluss: SMA-Smartmeter + PV-Forecast + Zendure-Entitäten → Template-Sensoren → Variablen in `automation.yml` → Service-Aufrufe (`number.set_value`, `select.select_option`).
- Hardware: SMA Wechselrichter mit Smartmeter, 6,8 kWp PV, Zendure SolarFlow 2400 AC. Ziel: minimaler Netzbezug, prognose- und tageszeitgesteuert.
- Trigger: `time_pattern` jede Minute (`mode: single`).

## Steuerstrategie (Branches in `automation.yml`)
1. **🌙 Nacht** (`binary_sensor.ist_nacht == on`):
   - **Notladen**: Heute & Morgen schlecht & `soc < soc_min + notlade_offset` → 200 W aus Netz.
   - `prog_morgen_hoch` & `soc > soc_min` → entladen bis `soc_min` (`min(|netz|, max_entlade)`).
   - sonst & `soc > soc_min + reserve_offset` → schonend (`min(|netz|/2, max_entlade)`).
   - default: Limits = 0 (kein Laden aus Netz in der Nacht außer Notladen).
2. **☀️ PV-Überschuss** (`netz > deadband` & `soc < soc_max`) → Mode `input`, `min(¾·netz, max_lade)`.
3. **🟢 Tag, Netzbezug** (`netz < -deadband` & `soc > soc_min`) → Mode `output`, voll bei guter Prognose, halbe Leistung sonst.
4. **Default** (Deadband / SoC-Grenze) → Limits = 0.

## Project-Specific Patterns
- **Vorzeichenkonvention**: `netz > 0` = Einspeisung, `netz < 0` = Bezug (`Nettoleistung Netz` in `template.yml`).
- **Speicherkompensation**: `netz` enthält den Speicher-Effekt. Für Sollwert-Berechnungen wird `netz_bereinigt = netz - speicher_output + speicher_input` verwendet (= „Netz ohne Speicher"). Bedingungen (laden/entladen?) nutzen weiterhin `netz` (IST-Zustand).
- **Prognose-Flags als Strings**: states sind `'True'`/`'False'` (Großschreibung), Vergleich via `is_state(...)`.
- **Hysterese**: `deadband: 50` W gegen Pendeln; immer mit `<= -deadband` / `>= deadband` vergleichen.
- **Leistungs-Capping**: durchgängig `min(abs(...), max_lade|max_entlade) | int`.
- **Mode-first, Limit-second**: erst `select.solarflow_2400_ac_ac_mode` setzen, dann `number.*_limit`.
- **Variablen als Single Source of Truth**: `soc`, `soc_min`, `soc_max`, `max_lade`, `max_entlade`, `speicher_input`, `speicher_output`, `netz_bereinigt`, `prog_*`, `ist_nacht`, `deadband`, `reserve_offset`, `notlade_offset`, `notlade_power`.

## Template-Sensoren (`template.yml`)
- `Nettoleistung Netz` aus `sn_..._metering_power_supplied/absorbed`, geglättet via `statistics`-Sensor (mean, 20 samples / 120 s) → `sensor.nettoleistung_netz_mittelwert`.
- Prognose-Schwellen relativ zu `input_number.pv_peak_leistung` (in W, für 6,8 kWp = 6800):
  - Stunde hoch: `energy_next_hour ≥ peak/2000` (kWh-Vergleich).
  - Heute hoch: `energy_production_today_remaining ≥ peak/1000`.
  - Morgen hoch: `energy_production_tomorrow ≥ 3·peak/1000`.
- `Zendure Max Ladeleistung` / `…Entladeleistung`: dynamische Caps abhängig von SoC + Prognose-Flags (Basis 800 W).
- `binary_sensor.ist_nacht`: Sonnenaufgang +2 h … Sonnenuntergang −3 h.

## Integration Points (harte Entitäts-IDs)
- SMA: `sensor.sn_3007669104_metering_power_supplied`, `sensor.sn_3007669104_metering_power_absorbed`.
- Forecast: `sensor.energy_next_hour`, `sensor.energy_production_today_remaining`, `sensor.energy_production_tomorrow`, `input_number.pv_peak_leistung`.
- Zendure: `sensor.solarflow_2400_ac_electric_level`, `sensor.solarflow_2400_ac_min_soc`, `number.solarflow_2400_ac_soc_set`, `number.solarflow_2400_ac_input_limit`, `number.solarflow_2400_ac_output_limit`, `select.solarflow_2400_ac_ac_mode` (Optionen `input`/`output`), `sensor.solarflow_2400_ac_input_power` (Ist-Ladeleistung), `sensor.solarflow_2400_ac_output_power` (Ist-Entladeleistung).
- Sonne: `sensor.sun_next_rising`, `sensor.sun_next_setting`.
- IDs sind tight gekoppelt – Änderungen IMMER in beiden Dateien synchron pflegen.

## Agent Workflow Notes
- Keine Build-/Test-Tooling im Repo; Validierung über HA „Konfiguration prüfen“ + Entwickler-Tools/Templates.
- Branch-Reihenfolge in `automation.yml` ist semantisch (Nacht zuerst, dann Lade-/Notlade-/Entlade-Zweige) – nicht ohne Grund umsortieren.
- Neue Schwellen als Variable im `variables:`-Block ergänzen, nicht als Magic Number im `choose`.
- Limit-Werte mit `| int` casten (Zendure `number.*` erwartet ganze Zahlen).
- Beim Erweitern prüfen, ob `select.solarflow_2400_ac_ac_mode` die genutzte Option kennt (`input`/`output`).

