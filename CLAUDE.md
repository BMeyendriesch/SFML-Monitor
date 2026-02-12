# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Home Assistant dashboard cards for monitoring the learning progress of a **Solar Forecast ML (SFML)** system. The cards visualize AI model performance, forecast accuracy, weather expert weights, drift detection, and physics calibration — all queried from the SFML SQLite database via HA's SQL integration.

All UI labels are in **German**.

## Architecture

### Data Source

The SFML integration maintains a SQLite database on the HA system at:
`/config/solar_forecast_ml/solar_forecast.db`

A sample copy (`solar_forecast.db`) is included in this repo for developing and testing SQL queries locally. Use `sqlite3 solar_forecast.db` to explore.

SQLAlchemy URL for HA SQL sensors: `sqlite:////config/solar_forecast_ml/solar_forecast.db`

### Key Database Tables for Monitoring

| Table | Purpose |
|---|---|
| `daily_summaries` | Daily predicted vs. actual kWh with accuracy % — primary performance metric |
| `hourly_predictions` | Hourly prediction detail: method (blended/physics/AI+Physics), accuracy, confidence |
| `ai_learned_weights_meta` | Active AI model info: type (tiny_lstm), samples, R², RMSE |
| `drift_metrics_rolling` | Rolling 7/14/30-day metrics: MAE, RMSE, bias, coverage |
| `drift_events` | Detected drift events (bias_shift, coverage_drop) |
| `weather_expert_weights` | Expert weights per cloud type (5 experts × 9 cloud types) |
| `weather_expert_learning` | Daily weight update history per expert |
| `physics_calibration_groups` | Per-panel-group calibration factor and confidence |
| `physics_calibration_history` | Historical calibration data points |
| `method_performance_learning` | Physics vs. AI advantage per condition bucket |
| `ai_seasonal_factors` | Monthly seasonal correction factors |

### System Context

- **Location:** Northern Germany (lat 52.75, lon 8.76)
- **Installed capacity:** 15.3 kWp, 3 panel groups (Gruppe 1, 2, 3)
- **AI model:** tiny_lstm with attention, 267 training samples
- **Weather sources:** open_meteo, wttr_in, ecmwf_layers, bright_sky, pirate_weather
- **Prediction methods:** physics, AI+Physics, blended

### Implementation Pattern

Each dashboard card consists of:
1. **SQL sensor(s)** in `sql_sensors.yaml` — queries the SFML database, returns current value as state and historical data as JSON attribute via `json_group_array(json_object(...))`
2. **Card YAML** — `custom:apexcharts-card` using `data_generator` to parse JSON attributes into chart data

### Card Types

- `custom:apexcharts-card` — charts (line, bar, area) using `data_generator` for SQL-sourced data
- Standard HA `entities` card — for displaying current scalar values (model type, RMSE, etc.)

## Conventions

- SQL sensor names use prefix `sfml_` (e.g., `sensor.sfml_daily_accuracy`)
- Chart data passed via JSON attributes using SQLite's `json_group_array(json_object(...))`
- Color scheme: green=accuracy/good, red=error/drift, orange=actual production, grey=predicted, blue=AI metrics
- `data_generator` parses JSON: `var data = JSON.parse(entity.attributes.chart_data);`
- Filter low-production days (`actual_kwh > 0.5`) to avoid skewed accuracy percentages

## No Build System

Raw YAML configuration files consumed directly by Home Assistant. No build, lint, or test pipeline. Deploy by pasting YAML into HA dashboard card configuration (manual YAML mode) and adding SQL sensors to HA's `configuration.yaml`.

## Required Home Assistant Setup

- **SQL Integration** — already configured, add sensors from `sql_sensors.yaml`
- **Frontend** — [apexcharts-card](https://github.com/RomRider/apexcharts-card) custom Lovelace card
- **SFML Integration** — provides the SQLite database being queried
