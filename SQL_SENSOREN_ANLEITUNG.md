# SQL-Sensoren manuell in Home Assistant anlegen

## Wichtige Hinweise

- SQL-Sensoren lassen sich über die UI anlegen oder in `sql_sensors.yaml`, eingebunden mit `sql: !include sql_sensors.yaml`; der YAML-Weg braucht danach einen Kern-Neustart. Bis zum 11.10.2026 stand hier, YAML gehe nicht — das war falsch, am 11.10.2026 liefen 8 der 29 SQL-Sensoren aus YAML.
- Die Sensoren dieser Anleitung sind über die Home Assistant UI angelegt: **Einstellungen → Geräte & Dienste → Integration hinzufügen → SQL**
- Die Datenbankverbindung ist immer gleich: `sqlite:////config/solar_forecast_ml/solar_forecast.db`
- Die Spalte ist immer: `state`
- Alle Sensoren haben aktuell das Suffix `_2` (z. B. `sensor.sfml_daily_accuracy_2`). Die Card-YAML-Dateien sind bereits entsprechend angepasst.
- Der Sensor **„SFML Ensemble Gewichte“** (sfml_ensemble_gewichte) wird **NICHT benötigt** — es gibt keine Card, die ihn nutzt. Diese Card wurde entfernt, da die Daten keine Zeitreihe darstellen.

---

## 1. SFML Tägliche Genauigkeit

**Entity-ID:** `sensor.sfml_tagliche_genauigkeit_2`

**Zu füllende Felder:**
- **Datenbankadresse:** `sqlite:////config/solar_forecast_ml/solar_forecast.db`
- **Spalte:** `state`
- **Einheit:** `%`

**SQL-Query (eine Zeile – zum Kopieren – AKTUALISIERT):**
```sql
SELECT ROUND(AVG(acc), 1) as state, json_group_array(json_object('date', date, 'accuracy', acc, 'predicted', predicted, 'actual', actual)) as chart_data FROM (SELECT date, ROUND(accuracy_percent, 1) as acc, ROUND(predicted_total_kwh, 2) as predicted, ROUND(actual_total_kwh, 2) as actual FROM daily_summaries WHERE actual_total_kwh > 0.5 ORDER BY date DESC LIMIT 30)
```

**Wichtiger Hinweis:** Falls der Sensor in HA einen falschen State-Wert liefert (z. B. 15,8 % statt ~61 %): Der Sensor wurde mit einer veralteten Query angelegt. Die obige Query ergibt laut lokaler DB einen korrekten Durchschnitt von 61,6 %. Sensor in HA neu konfigurieren: Einstellungen → Geräte & Dienste → SQL → Sensor bearbeiten → Query ersetzen.

**Verwendung:** Cards `card_accuracy_trend.yaml` und `card_forecast_vs_actual.yaml`

---

## 2. SFML Modell Status

**Entity-ID:** `sensor.sfml_model_status_2`

**Zu füllende Felder:**
- **Datenbankadresse:** `sqlite:////config/solar_forecast_ml/solar_forecast.db`
- **Spalte:** `state`
- **Einheit:** _(leer lassen)_

**SQL-Query (eine Zeile – zum Kopieren):**
```sql
SELECT m.active_model as state, m.training_samples as samples, ROUND(m.accuracy * 100, 1) as r2, ROUND(m.rmse, 4) as rmse, ROUND(d.mae, 4) as mae_30d, ROUND(d.bias, 4) as bias_30d, m.last_trained FROM ai_learned_weights_meta m LEFT JOIN (SELECT mae, bias FROM drift_metrics_rolling WHERE scope = 'global' AND window_days = 30 AND season IS NULL ORDER BY calculated_at DESC LIMIT 1) d ON 1=1 WHERE m.id = 1
```

**Verwendung:** Card `card_model_status.yaml` — AI-Modell-Typ, Training-Samples, R², RMSE, 30-Tage-MAE/Bias

---

## 3. SFML Drift Events

**Entity-ID:** `sensor.sfml_drift_events_2`

**Zu füllende Felder:**
- **Datenbankadresse:** `sqlite:////config/solar_forecast_ml/solar_forecast.db`
- **Spalte:** `state`
- **Einheit:** _(leer lassen)_

**SQL-Query (eine Zeile – zum Kopieren):**
```sql
SELECT (SELECT COUNT(*) FROM drift_events) as state, json_group_array(json_object('date', event_date, 'scope', scope, 'type', drift_type, 'severity', severity, 'value', ROUND(metric_value, 4), 'description', description)) as events_data FROM (SELECT * FROM drift_events ORDER BY event_date DESC, id DESC LIMIT 10)
```

**Verwendung:** Card `card_model_status.yaml` — Anzahl Drift-Events in der Entities-Card

---

## 4. SFML Wetter Experten

**Entity-ID:** `sensor.sfml_weather_experts_2`

**Zu füllende Felder:**
- **Datenbankadresse:** `sqlite:////config/solar_forecast_ml/solar_forecast.db`
- **Spalte:** `state`
- **Einheit:** _(leer lassen)_

**SQL-Query (eine Zeile – zum Kopieren):**
```sql
SELECT (SELECT expert_name FROM (SELECT expert_name, AVG(weight) as avg_w FROM weather_expert_weights GROUP BY expert_name ORDER BY avg_w DESC LIMIT 1)) as state, json_group_array(json_object('expert', expert_name, 'weight', ROUND(avg_w * 100, 1))) as chart_data FROM (SELECT expert_name, AVG(weight) as avg_w FROM weather_expert_weights GROUP BY expert_name ORDER BY avg_w DESC)
```

**Verwendung:** Card `card_weather_experts.yaml` — horizontales Balkendiagramm der Experten-Gewichtung

---

## 5. SFML Methoden Performance

**Entity-ID:** `sensor.sfml_method_performance_2`

**Zu füllende Felder:**
- **Datenbankadresse:** `sqlite:////config/solar_forecast_ml/solar_forecast.db`
- **Spalte:** `state`
- **Einheit:** _(leer lassen)_

**SQL-Query (eine Zeile – zum Kopieren):**
```sql
SELECT CASE WHEN AVG(ai_mae) < AVG(physics_mae) THEN 'AI+Physics' ELSE 'Physics' END as state, json_group_array(json_object('bucket', cloud_bucket || '/' || hour_bucket, 'physics_mae', ROUND(physics_mae, 3), 'ai_mae', ROUND(ai_mae, 3), 'samples', sample_count)) as method_data, (SELECT json_group_array(json_object('group', group_name, 'factor', ROUND(global_factor, 4), 'confidence', ROUND(confidence, 3), 'samples', sample_count)) FROM physics_calibration_groups) as calibration_data FROM method_performance_learning WHERE season IS NULL AND sample_count > 0
```

**Verwendung:** Card `card_method_comparison.yaml` — Physics vs. AI MAE pro Bedingung + Kalibrierungsfaktoren

---

## 6. SFML Lernfortschritt

**Entity-ID:** `sensor.sfml_lernfortschritt_2`

**Zu füllende Felder:**
- **Datenbankadresse:** `sqlite:////config/solar_forecast_ml/solar_forecast.db`
- **Spalte:** `state`
- **Einheit:** `%`

**SQL-Query (eine Zeile – zum Kopieren – AKTUALISIERT):**
```sql
SELECT ROUND(AVG(acc), 1) as state, json_group_array(json_object('date', date, 'accuracy', acc, 'error', err, 'predicted', predicted, 'actual', actual)) as chart_data, (SELECT ROUND(AVG(e), 2) FROM (SELECT ABS(error_kwh) as e FROM daily_summaries WHERE actual_total_kwh > 0.5 AND error_kwh IS NOT NULL ORDER BY date DESC LIMIT 7)) as error_7d FROM (SELECT date, ROUND(accuracy_percent, 1) as acc, ROUND(ABS(error_kwh), 2) as err, ROUND(predicted_total_kwh, 2) as predicted, ROUND(actual_total_kwh, 2) as actual FROM daily_summaries WHERE actual_total_kwh > 0.5 ORDER BY date)
```

**Wichtiger Hinweis:** Die Query enthält jetzt zusätzlich `predicted` und `actual` im chart_data-JSON. Diese Felder werden für die R²-Berechnung in der Card benötigt.

**Verwendung:** Card `card_lernfortschritt.yaml` — Genauigkeit 7d-Ø, R² Live 7d-Ø, MAE 7d-Ø + R² Training aus Modell-Meta

---

## 7. SFML Kalibrierung Verlauf

**Entity-ID:** `sensor.sfml_kalibrierung_verlauf_2`

**Zu füllende Felder:**
- **Datenbankadresse:** `sqlite:////config/solar_forecast_ml/solar_forecast.db`
- **Spalte:** `state`
- **Einheit:** _(leer lassen)_

**SQL-Query (eine Zeile – zum Kopieren):**
```sql
SELECT ROUND((SELECT AVG(global_factor) FROM physics_calibration_groups), 3) as state, json_group_array(json_object('date', date, 'group_name', group_name, 'factor', factor)) as chart_data FROM (SELECT date, group_name, ROUND(AVG(avg_ratio), 3) as factor FROM physics_calibration_history GROUP BY date, group_name ORDER BY date, group_name)
```

**Verwendung:** Card `card_kalibrierung.yaml` — Kalibrierungsfaktor-Konvergenz pro Panelgruppe

---

## 8. SFML Fehlermetriken

**Entity-ID:** `sensor.sfml_fehlermetriken_2`

**Zu füllende Felder:**
- **Datenbankadresse:** `sqlite:////config/solar_forecast_ml/solar_forecast.db`
- **Spalte:** `state`
- **Einheit:** `kWh`

**SQL-Query (eine Zeile – zum Kopieren – AKTUALISIERT):**
```sql
SELECT ROUND(AVG(abs_err), 2) as state, json_group_array(json_object('date', date, 'error', abs_err, 'predicted', predicted, 'actual', actual)) as chart_data FROM (SELECT date, ROUND(ABS(error_kwh), 3) as abs_err, ROUND(predicted_total_kwh, 2) as predicted, ROUND(actual_total_kwh, 2) as actual FROM daily_summaries WHERE actual_total_kwh > 0.5 AND error_kwh IS NOT NULL ORDER BY date)
```

**Wichtiger Hinweis:** Die alte Query hatte die Bedingung `ml_rmse IS NOT NULL`, die zu unvollständigen Daten führte. Die obige aktualisierte Query gibt mehr historische Daten zurück (alle Tage mit gültigen Fehlermetriken).

**Entity-Suffixe:** Die Card `card_fehlermetriken.yaml` referenziert derzeit `sensor.sfml_fehlermetriken_2`. Falls HA diesen Sensor mit einem anderen Suffix anlegt, muss die Card entsprechend angepasst werden.

**Verwendung:** Card `card_fehlermetriken.yaml` — MAE/RMSE 7-Tage-Trend berechnet aus täglichen Fehlern

---

## 9. SFML Drift Timeline

**Entity-ID:** `sensor.sfml_drift_timeline_2`

**Zu füllende Felder:**
- **Datenbankadresse:** `sqlite:////config/solar_forecast_ml/solar_forecast.db`
- **Spalte:** `state`
- **Einheit:** _(leer lassen)_

**SQL-Query (eine Zeile – zum Kopieren):**
```sql
SELECT COUNT(*) as state, json_group_array(json_object('date', event_date, 'scope', scope, 'type', drift_type, 'severity', severity, 'value', ROUND(metric_value, 3), 'threshold', ROUND(threshold_value, 3), 'action', response_action, 'desc', description)) as chart_data FROM (SELECT * FROM drift_events ORDER BY event_date, id)
```

**Verwendung:** Card `card_drift_timeline.yaml` — Drift-Events als Punkte nach Schweregrad und Scope

---

## 10. SFML Ensemble Gewichte
**⚠️ WIRD NICHT BENÖTIGT — überspringen**

**Entity-ID:** `sensor.sfml_ensemble_gewichte_2` _(nicht anlegen)_

Grund: Diese Query wurde aus den Dashboard-Cards entfernt. Es gibt keine Card, die diesen Sensor nutzt. Die Ensemble-Gewichte haben keine aussagekräftige Zeitreihe und sind daher für die Monitoring-Ansicht nicht relevant.

---

## 11. SFML Gruppen Genauigkeit

**Entity-ID:** `sensor.sfml_gruppen_genauigkeit_2`

**Zu füllende Felder:**
- **Datenbankadresse:** `sqlite:////config/solar_forecast_ml/solar_forecast.db`
- **Spalte:** `state`
- **Einheit:** `%`

**SQL-Query (eine Zeile – zum Kopieren):**
```sql
SELECT ROUND(AVG(avg_acc), 1) as state, json_group_array(json_object('date', target_date, 'group', group_name, 'accuracy', avg_acc, 'hours', hours)) as chart_data FROM (SELECT hp.target_date, ppg.group_name, ROUND(AVG((1.0 - MIN(ABS(ppg.prediction_kwh - ppg.actual_kwh) / ppg.actual_kwh, 1.0)) * 100), 1) as avg_acc, COUNT(*) as hours FROM prediction_panel_groups ppg JOIN hourly_predictions hp ON ppg.prediction_id = hp.prediction_id WHERE hp.is_production_hour = 1 AND ppg.actual_kwh > 0.01 GROUP BY hp.target_date, ppg.group_name ORDER BY hp.target_date, ppg.group_name)
```

**Verwendung:** Card `card_gruppen_genauigkeit.yaml` — Prognose-Treffsicherheit pro Panelgruppe über Zeit

---

## 12. SFML Experten Lernen

**Entity-ID:** `sensor.sfml_experten_lernen_2`

**Zu füllende Felder:**
- **Datenbankadresse:** `sqlite:////config/solar_forecast_ml/solar_forecast.db`
- **Spalte:** `state`
- **Einheit:** _(leer lassen)_

**SQL-Query (eine Zeile – zum Kopieren):**
```sql
SELECT (SELECT expert_name FROM (SELECT expert_name, AVG(weight_after) as avg_w FROM weather_expert_learning GROUP BY expert_name ORDER BY avg_w DESC LIMIT 1)) as state, json_group_array(json_object('date', date, 'expert', expert_name, 'weight', ROUND(avg_weight * 100, 1))) as chart_data FROM (SELECT date, expert_name, AVG(weight_after) as avg_weight FROM weather_expert_learning GROUP BY date, expert_name ORDER BY date, expert_name)
```

**Verwendung:** Card `card_experten_lernen.yaml` — Gewichtsentwicklung der 5 Wetterquellen über Zeit

---

## 13. SFML KI Vorteil

**Entity-ID:** `sensor.sfml_ki_vorteil` _(kein `_2`-Suffix laut Benutzer)_

**Zu füllende Felder:**
- **Datenbankadresse:** `sqlite:////config/solar_forecast_ml/solar_forecast.db`
- **Spalte:** `state`
- **Einheit:** `kWh`

**SQL-Query (eine Zeile – zum Kopieren):**
```sql
SELECT ROUND(AVG(physics_mae - ai_mae), 3) as state, json_group_array(json_object('date', target_date, 'physics_mae', physics_mae, 'ai_mae', ai_mae, 'blend_mae', blend_mae, 'stunden', stunden)) as chart_data FROM (SELECT target_date, ROUND(AVG(ABS(actual_kwh - physics_kwh)), 3) as physics_mae, ROUND(AVG(ABS(actual_kwh - ai_kwh)), 3) as ai_mae, ROUND(AVG(ABS(actual_kwh - prediction_kwh)), 3) as blend_mae, COUNT(*) as stunden FROM hourly_predictions WHERE is_production_hour = 1 AND actual_kwh > 0.01 AND physics_kwh IS NOT NULL AND ai_kwh IS NOT NULL GROUP BY target_date ORDER BY target_date)
```

**State-Bedeutung:** Ø (Physik-MAE − KI-MAE) in kWh — positiv = KI im Schnitt genauer, negativ = Physik im Schnitt genauer

**Verwendung:** Card `card_ki_vorteil.yaml` — Zeitlicher Verlauf des täglichen MAE (kWh) für Physik, KI (LSTM) und Blend aus `hourly_predictions`

---

## 14. SFML LSTM Tagesmetriken

**Entity-ID:** `sensor.sfml_lstm_tagesmetriken_2`

**Zu füllende Felder:**
- **Datenbankadresse:** `sqlite:////config/solar_forecast_ml/solar_forecast.db`
- **Spalte:** `state`
- **Einheit:** `kWh`

**SQL-Query (eine Zeile – zum Kopieren):**
```sql
SELECT ROUND(AVG(ml_mae), 3) as state, json_group_array(json_object('date', date, 'mae', ROUND(ml_mae, 3), 'rmse', ROUND(ml_rmse, 3), 'r2', ROUND(ml_r2_score, 3))) as chart_data FROM (SELECT date, ml_mae, ml_rmse, ml_r2_score FROM daily_summaries WHERE ml_rmse IS NOT NULL AND actual_total_kwh > 0.5 ORDER BY date)
```

**Verwendung:** Card `card_lstm_lernmetriken.yaml` — R²-Score des LSTM-Modells im zeitlichen Verlauf (geclampt auf [-1, 1])

---

## 15. SFML Ensemble Vertrauen

**Entity-ID:** `sensor.sfml_ensemble_vertrauen_2`

**Zu füllende Felder:**
- **Datenbankadresse:** `sqlite:////config/solar_forecast_ml/solar_forecast.db`
- **Spalte:** `state`
- **Einheit:** `%`

**SQL-Query (eine Zeile – zum Kopieren):**
```sql
SELECT ROUND(AVG(lstm_weight * 100), 1) as state, json_group_array(json_object('group', group_name, 'bucket', cloud_bucket || '/' || hour_bucket, 'lstm_w', ROUND(lstm_weight * 100, 1), 'ridge_w', ROUND(ridge_weight * 100, 1), 'lstm_mae', ROUND(lstm_mae, 4), 'ridge_mae', ROUND(ridge_mae, 4), 'samples', sample_count)) as chart_data FROM (SELECT * FROM ensemble_group_weights WHERE season = 'winter' AND cloud_bucket = 'overcast' ORDER BY group_name, CASE hour_bucket WHEN 'morning' THEN 1 WHEN 'midday' THEN 2 WHEN 'afternoon' THEN 3 END)
```

**Verwendung:** Card `card_ensemble_vertrauen.yaml` — 100%-gestapelter Balken: LSTM- vs. Ridge-Gewicht pro Panelgruppe und Tageszeit (nur bedeckte Bedingungen)

---

## Anleitung zum Anlegen eines Sensors in der Home Assistant UI

1. **Home Assistant öffnen** und zum Bereich **Einstellungen** navigieren
2. **Geräte & Dienste** → **Integrationen** auswählen
3. **Integration hinzufügen** klicken und nach **SQL** suchen
4. Falls noch nicht vorhanden: SQL-Integration konfigurieren (mit dem bereits bestehenden HA-System sollte sie bereits vorhanden sein)
5. Neue SQL-Sensoren hinzufügen:
   - **Name:** Aus obiger Liste (z. B. „SFML Tägliche Genauigkeit“)
   - **Datenbankadresse:** `sqlite:////config/solar_forecast_ml/solar_forecast.db`
   - **Query:** Aus obiger Liste kopieren (komplette einzeilige Query)
   - **Spalte:** `state`
   - **Einheit:** Wenn vorhanden, aus obiger Liste eintragen (z. B. `%`, `kWh`)
6. **Speichern** klicken
7. **Entity-ID überprüfen:** HA zeigt die neue Entity im Integrations-Panel. Notieren Sie die Entity-ID (meist mit dem Sensor-Namen generiert, z. B. `sensor.sfml_tägliche_genauigkeit`)
8. Falls HA ein Suffix anhängt (z. B. `_2`): Die entsprechende Card-YAML-Datei anpassen (entity-ID in der Card ändern)
9. **Dashboard-Card hinzufügen:** Card-YAML-Datei in den manuellen Modus kopieren und die Entity-IDs aktualisieren

---

## Tipps und Fehlerbehebung

- **Query-Fehler:** Falls beim Speichern eine Fehlermeldung erscheint, überprüfen Sie, dass die komplette Query ohne Zeilenumbrüche vorhanden ist.
- **Datenbank nicht erreichbar:** Überprüfen Sie, dass der HA-Container Zugriff auf `/config/solar_forecast_ml/solar_forecast.db` hat.
- **Attribute werden nicht angezeigt:** Die JSON-Attribute (z. B. `chart_data`, `events_data`) werden automatisch erzeugt. Überprüfen Sie in der Entwickler-Tools → States, dass das Attribut vorhanden ist.
- **Card zeigt keine Daten:** Überprüfen Sie, dass die Entity-ID in der Card mit der tatsächlichen Entity-ID des Sensors übereinstimmt.
- **Performance:** Falls die Queries zu lange dauern (mehr als 10 Sekunden), kann die Limit-Clause (z. B. `LIMIT 30`) reduziert werden.

---

## Zusammenfassung

Es sind **14 Sensoren** zu erstellen (Nr. 10 „SFML Ensemble Gewichte“ wird nicht benötigt):

| # | Sensor | Entity-ID | Einheit | Card |
|---|--------|-----------|---------|------|
| 1 | SFML Tägliche Genauigkeit | `sensor.sfml_tagliche_genauigkeit_2` | `%` | card_accuracy_trend, card_forecast_vs_actual |
| 2 | SFML Modell Status | `sensor.sfml_model_status_2` | — | card_model_status |
| 3 | SFML Drift Events | `sensor.sfml_drift_events_2` | — | card_model_status |
| 4 | SFML Wetter Experten | `sensor.sfml_weather_experts_2` | — | card_weather_experts |
| 5 | SFML Methoden Performance | `sensor.sfml_method_performance_2` | — | card_method_comparison |
| 6 | SFML Lernfortschritt | `sensor.sfml_lernfortschritt_2` | `%` | card_lernfortschritt |
| 7 | SFML Kalibrierung Verlauf | `sensor.sfml_kalibrierung_verlauf_2` | — | card_kalibrierung |
| 8 | SFML Fehlermetriken | `sensor.sfml_fehlermetriken_2` | `kWh` | card_fehlermetriken |
| 9 | SFML Drift Timeline | `sensor.sfml_drift_timeline_2` | — | card_drift_timeline |
| 10 | ~~SFML Ensemble Gewichte~~ | ~~`sensor.sfml_ensemble_gewichte_2`~~ | — | **nicht anlegen** |
| 11 | SFML Gruppen Genauigkeit | `sensor.sfml_gruppen_genauigkeit_2` | `%` | card_gruppen_genauigkeit |
| 12 | SFML Experten Lernen | `sensor.sfml_experten_lernen_2` | — | card_experten_lernen |
| 13 | SFML KI Vorteil | `sensor.sfml_ki_vorteil` | `kWh` | card_ki_vorteil |
| 14 | SFML LSTM Tagesmetriken | `sensor.sfml_lstm_tagesmetriken` | `kWh` | card_lstm_lernmetriken |
| 15 | SFML Ensemble Vertrauen | `sensor.sfml_ensemble_vertrauen_2` | `%` | card_ensemble_vertrauen |
