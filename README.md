# SFML Monitor — Home Assistant Dashboard

Dashboard-Karten für das Monitoring des **Solar Forecast ML (SFML)**-Systems in Home Assistant. Die Karten visualisieren KI-Modellperformance, Vorhersagegenauigkeit, Wetterexperten-Gewichte, Drift-Erkennung und Physik-Kalibrierung — alle Daten kommen direkt aus der SFML-SQLite-Datenbank über die HA SQL-Integration.

## Enthaltene Karten

| Datei | Inhalt |
|---|---|
| `card_lernfortschritt.yaml` | Lernfortschritt-Übersicht (Genauigkeit, Modellstatus) |
| `card_accuracy_trend.yaml` | Genauigkeitstrend über Zeit |
| `card_forecast_vs_actual.yaml` | Prognose vs. tatsächliche Produktion |
| `card_fehlermetriken.yaml` | Fehlermetriken (MAE, RMSE, Bias) |
| `card_drift_timeline.yaml` | Drift-Ereignisse und Rolling-Metriken |
| `card_gruppen_genauigkeit.yaml` | Genauigkeit pro Panelgruppe |
| `card_kalibrierung.yaml` | Physik-Kalibrierungsfaktoren |
| `card_method_comparison.yaml` | Vergleich Physik vs. KI+Physik vs. Blended |
| `card_model_status.yaml` | Aktueller Modellstatus (Typ, Samples, R², RMSE) |
| `card_weather_experts.yaml` | Wetterexperten-Gewichte pro Wolkentyp |
| `card_experten_lernen.yaml` | Lernverlauf der Wetterexperten |
| `card_ensemble_vertrauen.yaml` | Ensemble-Vertrauen und Konfidenz |
| `card_ki_vorteil.yaml` | KI-Vorteil gegenüber reiner Physik |
| `card_lstm_lernmetriken.yaml` | LSTM-interne Lernmetriken |

## Voraussetzungen

- **Home Assistant** mit aktivierter [SQL-Integration](https://www.home-assistant.io/integrations/sql/)
- **apexcharts-card** (via HACS): [github.com/RomRider/apexcharts-card](https://github.com/RomRider/apexcharts-card)
- **SFML-Integration** — stellt die SQLite-Datenbank bereit unter:
  `/config/solar_forecast_ml/solar_forecast.db`

## Installation

### 1. SQL-Sensoren einrichten

Die Sensoren aus `sql_sensors.yaml` in die HA-Konfiguration übernehmen:

```yaml
# configuration.yaml
sql:
  - # Inhalt aus sql_sensors.yaml einfügen
```

Detaillierte Schritt-für-Schritt-Anleitung: [SQL_SENSOREN_ANLEITUNG.md](SQL_SENSOREN_ANLEITUNG.md)

Nach dem Einfügen HA neu starten und prüfen, ob die `sensor.sfml_*`-Entitäten erscheinen.

### 2. Dashboard-Karten einfügen

Jede `card_*.yaml`-Datei ist eine eigenständige Lovelace-Karte:

1. Dashboard öffnen → **Bearbeiten** → **Karte hinzufügen**
2. Unten **„Manuell"** wählen
3. Inhalt der gewünschten YAML-Datei einfügen
4. Speichern

## Systemkontext

- **Standort:** Norddeutschland (52,75° N, 8,76° O)
- **Anlagenleistung:** 15,3 kWp, 3 Panelgruppen
- **KI-Modell:** tiny_lstm mit Attention-Mechanismus
- **Wetterquellen:** open_meteo, wttr_in, ecmwf_layers, bright_sky, pirate_weather
- **Prognosemethoden:** Physik, KI+Physik, Blended

## Datenbankstruktur (Auszug)

| Tabelle | Inhalt |
|---|---|
| `daily_summaries` | Tägliche Prognose vs. Ist-Wert mit Genauigkeit |
| `hourly_predictions` | Stündliche Vorhersagedetails |
| `ai_learned_weights_meta` | Aktiver KI-Modellzustand |
| `drift_metrics_rolling` | Rollende 7/14/30-Tage-Fehlermetriken |
| `drift_events` | Erkannte Drift-Ereignisse |
| `weather_expert_weights` | Expertengewichte pro Wolkentyp |
| `physics_calibration_groups` | Kalibrierungsfaktoren pro Panelgruppe |

## Konventionen

- Sensorpräfix: `sfml_` (z.B. `sensor.sfml_daily_accuracy`)
- Chartdaten via JSON-Attribut: `json_group_array(json_object(...))`
- Farbschema: Grün = Genauigkeit/gut, Rot = Fehler/Drift, Orange = Ist-Produktion, Grau = Prognose, Blau = KI-Metriken
