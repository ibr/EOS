% SPDX-License-Identifier: Apache-2.0
(integration-pitfalls-page)=

<!-- pyml disable no-emphasis-as-heading -->

# Integration Pitfalls: Home Assistant, Tibber and a Battery

This page collects practical pitfalls that show up when EOS is first wired to Home Assistant (HA),
a Tibber tariff and a home battery such as a Zendure SolarFlow 2400 Pro. It targets the automatic
`GENETIC` optimizer, but most points also apply to a Docker or source installation.

Each section follows the same pattern: **symptom -> cause -> fix**.

:::{admonition} Scope
:class: note
EOS never controls hardware directly. It only produces an energy management plan and publishes
device instructions to Home Assistant entities. The actual charger/discharger control stays in HA.
:::

## 1. `GENETIC` refuses to start without an inverter

**Symptom**

`POST /v1/optimize` returns `503` and the log contains:

```text
Configure an inverter to model PV and grid energy flows.
```

**Cause**

The `GENETIC` planner models the AC/DC coupling through an inverter device. A battery alone is not
enough, even when it is AC coupled.

**Fix**

Configure both a battery and an inverter, and link the inverter to the battery via `battery_id`.
`inverter.max_power_w` and `devices.max_inverters` are required.

```json
{
  "devices": {
    "batteries": {
      "storage": {
        "device_id": "storage",
        "capacity_wh": 5280,
        "max_charge_power_w": 2400
      }
    },
    "max_batteries": 1,
    "inverters": {
      "storage_inverter": {
        "device_id": "storage_inverter",
        "max_power_w": 2400,
        "battery_id": "storage",
        "ac_to_dc_efficiency": 0.94,
        "dc_to_ac_efficiency": 0.94,
        "max_ac_charge_power_w": 1200
      }
    },
    "max_inverters": 1
  }
}
```

## 2. A fresh battery SoC factor is mandatory

**Symptom**

```text
Fresh SoC missing for storage; supply soc or a measurement.
```

**Cause**

Without a request override, `GENETIC` reads the state of charge from a measurement channel, by
default `<device_id>-soc-factor`, and expects a **factor between 0.0 and 1.0** (not percent).
The value must be no older than `optimization.genetic.measurement_max_age_seconds`.

**Fix**

- Map the Home Assistant adapter to a sensor that yields a factor, for example
  `sensor.storage-soc-factor = electric_level / 100`.
- Make sure the sensor is numeric and available.

```json
{
  "adapter": {
    "homeassistant": {
      "device_measurement_entity_ids": {
        "storage-soc-factor": "sensor.storage_soc_factor",
        "storage-power-3-phase-sym-w": "sensor.storage_battery_power"
      }
    }
  }
}
```

## 3. The default SoC freshness limit is too short for hourly slots

**Symptom**

The first optimization works, then later runs fail again with a stale SoC message.

**Cause**

The HA adapter samples the SoC at the start of the EMS slot. With an hourly
`optimization.genetic.interval_sec` the newest sample can be almost one hour old, while the
default `measurement_max_age_seconds` is only `300`.

**Fix**

Raise the limit, for example to `3600` or `5400`:

```json
{
  "optimization": {
    "genetic": {
      "measurement_max_age_seconds": 5400
    }
  }
}
```

## 4. The PV forecast must cover the night hours

**Symptom**

```text
Missing or invalid PV within the control horizon.
```

**Cause**

`bounded_forecast_array` only fills a target slot when the source series **fully covers** it.
Providers that return only daylight samples (for example `PVForecastForecastSolar`) leave the
night slots `NaN`, and the optimizer rejects the run.

**Fix**

Use a provider that returns a continuous series with `0` at night, or supply a complete series via
`PVForecastImport`. The Akkudoktor provider with `backend: local` does this.

## 5. The Akkudoktor remote PV backend may answer with HTTP 400

**Symptom**

```text
400 Client Error: Bad Request for url: https://api.akkudoktor.net/forecast?...
```

**Cause**

The remote endpoint has rejected the single-plane scalar parameters EOS sends
(`lat`, `lon`, `power`, `azimuth`, `tilt`). The local `pvlib` backend computes the same forecast
from Open-Meteo data and is not affected.

**Fix**

```json
{
  "pvforecast": {
    "provider": "PVForecastAkkudoktor",
    "akkudoktor": { "backend": "local", "resolution_minutes": 60 }
  }
}
```

## 6. Akkudoktor PV planes need `inverter_paco`

**Symptom**

```text
PVForecastAkkudoktor is missing plane inverter_paco configuration
```

**Cause**

The provider clips the AC power to the inverter rating and requires the field on every plane.

**Fix**

Add `inverter_paco` (in watts) to each `pvforecast.planes` entry.

## 7. A missing weather provider breaks the EOSdash prediction page

**Symptom**

In EOSdash the prediction page shows:

```text
Can not retrieve predictions from http://...: 404 Client Error:
Not Found for url: .../v1/prediction/series?...
```

**Cause**

The EOSdash prediction page requests weather series
(`weather_relative_humidity`, `weather_temp_air`, `weather_ghi`, `weather_dni`, `weather_dhi`)
unconditionally. If no weather provider is configured these keys do not exist and the page aborts
with a `404` instead of skipping the missing series.

**Fix**

Configure a weather provider, for example:

```json
{
  "weather": { "provider": "OpenMeteo" }
}
```

## 8. Do not double count Tibber fees

**Symptom**

Optimization results suggest grid charging although the all-in price is already high, or the
computed cost is roughly twice the real invoice.

**Cause**

`ElecPriceTibber` already stores the Tibber **total** price (energy + grid fees + taxes + margin)
in `elecprice_marketprice_wh`. Adding `ElecFeeFixed` on top applies the fees a second time.

**Fix**

Leave `elecfee.provider` empty when Tibber is the price source. Only use `ElecFeeFixed` with
fee-free spot providers such as Akkudoktor, Energy Charts or SMARD.

## 9. A feed-in tariff must be configured

**Symptom**

```text
Missing or invalid feed-in tariff within the control horizon.
```

**Cause**

`GENETIC` always reads `feed_in_tariff_wh`. Without a provider the series stays empty and the run
is cancelled.

**Fix**

Configure a feed-in tariff. Use `FeedInTariffFixed` for a fixed EEG rate, or `FeedInTariffTibber`
for Tibber's spot energy component. The Tibber feed-in provider shares the credentials with
`elecprice.tibber`.

## 10. Tibber token and home id

**Symptom**

```text
Tibber access_token is required
```

or the prices come from the wrong home.

**Cause**

- EOS expects a Tibber API access token as a **plain string**.
- The Home Assistant Tibber integration stores it as a dictionary
  (`data["token"] = {"access_token": "...", "token_type": "Bearer"}`) inside
  `.storage/core.config_entries`. Copying the whole dictionary does not work.
- One Tibber account can contain several homes. Without `home_id` the first home with a
  subscription is used, which may not be the one with the battery.

**Fix**

Set the token and, if needed, the home id:

```json
{
  "elecprice": {
    "provider": "ElecPriceTibber",
    "tibber": {
      "access_token": "tibber_pat_...",
      "home_id": "00000000-0000-0000-0000-000000000000"
    }
  }
}
```

Home ids can be listed with a small GraphQL query against `https://api.tibber.com/v1-beta/gql`:

```graphql
{ viewer { homes { id appNickname } } }
```

## 11. Tables keyed by device id, not by list index

**Symptom**

`config_entity_ids` entries such as `devices/batteries/0/capacity_wh` have no effect.

**Cause**

Devices are stored as a map keyed by `device_id`. The `/0/` example in the adapter docstring
still references the old list layout.

**Fix**

Use the device id in the path:

```json
{
  "devices/batteries/storage/min_soc_percentage": "number.storage_min_soc"
}
```

## 12. Battery power sign convention

**Symptom**

The optimizer charges while the battery actually discharges.

**Cause**

EOS defines **positive battery power as discharge** and negative as charge.

**Fix**

Map a sensor that follows the same convention, or invert it with a template sensor. Most Zendure
integrations expose a signed power value that already matches.

## 13. EOS does not control devices and uses S2 operation modes

**Symptom**

The optimizer runs, but the battery does not react.

**Cause**

EOS only writes instructions to entities. In addition, the hardware-independent modes follow the
S2 standard, not a simple idle/charge/discharge enum. Typical battery instructions are
`IDLE`, `SELF_CONSUMPTION`, `NON_EXPORT`, `PEAK_SHAVING` and `FORCED_CHARGE`.

**Fix**

- Enable `adapter.provider = ["HomeAssistant"]` and register the instruction entity
  `sensor.eos_<device_id>` in `device_instruction_entity_ids`.
- Build HA automations that translate the instruction entity and its `operation_mode_factor`
  attribute into your device's controls.

:::{admonition} Warning
:class: warning
Entities created by the EOS adapter have **no unique id**. Wrap them in template sensors if you
need them in the UI, history or long-term statistics.
:::

## 14. Runtime configuration is not persistent by default

**Symptom**

A configuration change disappears after a restart.

**Cause**

`PUT /v1/config` only updates the runtime settings.

**Fix**

Save explicitly with `PUT /v1/config/file`, or set `general.config_save_mode` to `AUTOMATIC` so
EOS persists changes itself.

## 15. Quick verification

Use the REST API of a running instance (`http://<host>:8503`) to validate a fresh setup:

```bash
# 1. Are the prediction keys present?
curl -s http://localhost:8503/v1/prediction/keys

# 2. Does the price look like an all-in price (around 0.0003 .. 0.0006 EUR/Wh)?
curl -s "http://localhost:8503/v1/prediction/series?key=elecprice_marketprice_wh"

# 3. Does the PV forecast cover night and day (48 hourly values)?
curl -s "http://localhost:8503/v1/prediction/series?key=pvforecast_ac_power"

# 4. Does an optimization run succeed and publish a plan?
curl -s -X POST http://localhost:8503/v1/optimize
curl -s http://localhost:8503/v1/energy-management/plan
```

## Minimal working configuration

The following (redacted) configuration combines all fixes above and is known to produce a valid
plan for a Home Assistant add-on with a Tibber tariff and an AC-coupled battery.

```json
{
  "general": {
    "latitude": 51.0,
    "longitude": 13.0,
    "timezone": "Europe/Berlin"
  },
  "ems": { "interval": 900.0, "mode": "OPTIMIZATION" },
  "devices": {
    "batteries": {
      "storage": {
        "device_id": "storage",
        "capacity_wh": 5280,
        "charging_efficiency": 0.89,
        "discharging_efficiency": 0.89,
        "max_charge_power_w": 2400,
        "min_soc_percentage": 10,
        "max_soc_percentage": 100
      }
    },
    "max_batteries": 1,
    "inverters": {
      "storage_inverter": {
        "device_id": "storage_inverter",
        "max_power_w": 2400,
        "battery_id": "storage",
        "ac_to_dc_efficiency": 0.94,
        "dc_to_ac_efficiency": 0.94,
        "max_ac_charge_power_w": 1200
      }
    },
    "max_inverters": 1
  },
  "elecprice": {
    "provider": "ElecPriceTibber",
    "tibber": { "access_token": "<redacted>", "home_id": "<redacted>" }
  },
  "feedintariff": {
    "provider": "FeedInTariffFixed",
    "feedintarifffixed": {
      "feed_in_tariff_amt_kwh": {
        "windows": [
          {
            "start_time": "00:00:00.000000",
            "duration": "1 day",
            "value": 0.078
          }
        ]
      }
    }
  },
  "load": {
    "provider": "LoadAkkudoktor",
    "loadakkudoktor": { "loadakkudoktor_year_energy_kwh": 3000.0 }
  },
  "pvforecast": {
    "provider": "PVForecastAkkudoktor",
    "akkudoktor": { "backend": "local", "resolution_minutes": 60 },
    "planes": [
      {
        "surface_tilt": 24.0,
        "surface_azimuth": 180.0,
        "peakpower": 1.1,
        "inverter_paco": 1200
      }
    ],
    "max_planes": 1
  },
  "weather": { "provider": "OpenMeteo" },
  "adapter": {
    "provider": ["HomeAssistant"],
    "homeassistant": {
      "device_measurement_entity_ids": {
        "storage-soc-factor": "sensor.storage_soc_factor",
        "storage-power-3-phase-sym-w": "sensor.storage_battery_power"
      },
      "device_instruction_entity_ids": ["sensor.eos_storage"],
      "grid_import_emr_entity_ids": ["sensor.grid_import_total_kwh"],
      "grid_export_emr_entity_ids": ["sensor.grid_export_total_kwh"],
      "pv_production_emr_entity_ids": ["sensor.pv_production_total_kwh"]
    }
  },
  "optimization": {
    "algorithm": "GENETIC",
    "genetic": {
      "interval_sec": 3600,
      "horizon_hours": 24,
      "measurement_max_age_seconds": 5400
    }
  }
}
```

<!-- pyml enable no-emphasis-as-heading -->
