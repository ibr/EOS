% SPDX-License-Identifier: Apache-2.0
(zendure-hems-to-eos-plan-page)=

<!-- pyml disable no-emphasis-as-heading -->

# Plan: From Zendure HEMS to EOS Control with Zero Export via Home Assistant

This is a migration and decision plan for a Zendure SolarFlow 2400 Pro that is currently managed by
Zendure's own HEMS. The target state is:

- HEMS disabled, Zendure's `zendure_ha` manager left as the low-level actuator, and
- EOS planning the battery schedule, executed by Home Assistant while the Shelly grid meter keeps
  grid export at zero.

The plan collects only the Zendure Manager and HEMS considerations. General EOS pitfalls are
documented separately in the integration pitfalls guide.

## 1. Why HEMS and the Zendure Manager block each other

`zendure_ha` only treats a device as controllable when its internal `connectionStatus` is `>= 10`.
That value is derived in `ZendureDevice.setStatus()` and is **not** a network reachability flag:

| `connectionStatus` | Condition in `setStatus()` | Manager usable |
| --- | --- | --- |
| 0 | no recent data | no |
| 1 | `socStatus == 1` | no |
| 2 | `hemsState.is_on` (vendor HEMS active) | no |
| 3 | `fuseGroup == 0` (`unused`) | no |
| 10 | cloud connection | yes |
| 11 | local MQTT server | yes |
| 12 | `zenSDK` | yes |

Both conditions are checked **before** the actual connection type, so:

- a device with **HEMS enabled** reports `2`,
- a device with **fuse group `unused`** reports `3`,

and the manager refuses to start with:

```text
No devices online, not possible to start the operation
```

This is the single most important prerequisite: as long as HEMS is on or the fuse group is
`unused`, neither the Zendure Manager nor a custom Home Assistant control loop can steer the device.

## 2. Architecture options

| Option | Control of power | Price optimization | Grid export limit |
| --- | --- | --- | --- |
| A. Zendure Manager `smart` | `zendure_ha` manager | no | yes (P1 based) |
| B. EOS + Home Assistant | HA automations | yes (EOS plan) | yes (Shelly based) |
| C. Hybrid | `zendure_ha` manager | EOS selects the mode | yes (P1 based) |

Option B is the target described here. Option A is the quick fallback. Option C is the pragmatic
compromise if a closed-loop zero-export automation is too much maintenance.

:::{admonition} Warning
:class: warning
Never run option A and option B at the same time. The Zendure Manager and a custom HA automation
would write conflicting `acMode`, `inputLimit` and `outputLimit` values.
:::

## 3. Prerequisite: a trustworthy grid meter

The zero-export loop needs the **net grid power at the point of common coupling**, with
`positive = import` and `negative = export`.

The Shelly EM3 `192.168.1.249` (`shellyem3-485519D702A7`) measures the three phases. Its phase
sensors are `sensor.strom_phase_a_leistung`, `..._b_...` and `..._c_...`. A template sensor turns
them into the signed total. The name must be `sensor.power_actual`, because that is the entity the
`zendure_ha` configuration already references as the P1 meter.

```yaml
- sensor:
    - name: "Power Actual"
      unique_id: "eos_power_actual"
      unit_of_measurement: "W"
      device_class: power
      state_class: measurement
      state: >-
        {{ (states("sensor.strom_phase_a_leistung") | float(0))
         + (states("sensor.strom_phase_b_leistung") | float(0))
         + (states("sensor.strom_phase_c_leistung") | float(0)) }}
```

Sanity check: while the battery is discharging, `sensor.power_actual` should sit near `0`, and it
should track the utility meter (for example the Tibber Pulse) within a few watts.

## 4. Step-by-step migration

### Phase 0 - Prepare and roll back safely

1. Create a Home Assistant backup and export the current Zendure app settings.
2. Record the current values of `select.zendure_manager_operation`, both fuse groups, the SoC
   limits and the input/output limits.
3. Keep `select.zendure_manager_operation = off` until the new path is verified.

### Phase 1 - Leave HEMS in the Zendure app

Do this in the Zendure app, not in Home Assistant:

1. Disable **HEMS** for the SolarFlow 2400 Pro (and for the AIO 2400 if it should be controlled).
2. Assign a **fuse group** to every device (for example `owncircuit` or the matching `groupXXXX`
   value) instead of `unused`.

Verify in Home Assistant:

- `binary_sensor.solarflow_2400_pro_hems_state` is `off`
- `sensor.solarflow_2400_pro_connection_status` is `10`, `11` or `12`
- the same holds for every device you want to control

Do not enable the Zendure Manager yet.

### Phase 2 - Establish the grid meter

1. Add the `sensor.power_actual` template sensor from section 3.
2. Confirm the sign convention: force a short export (or read a known situation) and check that
   `sensor.power_actual` goes negative.
3. Confirm `zendure_ha` points to `sensor.power_actual` (config entry field `p1meter`). The bridge
   in Phase 5 later changes this to `sensor.eos_grid_setpoint`.

### Phase 3 - Let EOS produce a plan

1. Configure the battery and the inverter in EOS and let it publish its instruction entity
   `sensor.eos_<device_id>`.
2. Verify that an automatic run succeeds and that the plan contains battery instructions.

### Phase 4 - Understand the EOS battery modes

EOS publishes an S2 battery mode plus an `operation_mode_factor` (0-1) on the instruction entity.
The two most common modes differ only in whether the battery may discharge:

<!-- pyml disable line-length -->

| EOS mode | EOS simulation mapping | Meaning |
| --- | --- | --- |
| `self_consumption` | `dc_charge > 0` and `discharge_allowed = true` | charge from PV surplus and discharge to cover the local load (classic zero export / self use) |
| `non_export` | `dc_charge > 0` and `discharge_allowed = false` | charge from PV surplus, **do not discharge** (store instead of feeding in) |

<!-- pyml enable line-length -->

This is the EOS mapping in `_battery_operation_from_solution()` and matches the simulation table in
`resource.md`. The broader S2 definition of `NON_EXPORT` only forbids export to the grid; the EOS
simulation is stricter and maps it to "no discharge at all". The bridge below follows the EOS
simulation.

Other modes that can appear: `idle` (nothing), `forced_charge` / `grid_support_import` (grid
charging). `peak_shaving` and `grid_support_export` require explicit battery export and are
deliberately **not** mapped here.

### Phase 5 - Build the bridge (virtual P1 + mode mapping)

The Zendure Manager has exactly one rule: it regulates its P1 sensor to `0 W`. To make it execute an
EOS target, feed it a **virtual** P1 sensor:

```text
sensor.eos_grid_setpoint = sensor.power_actual - eos_target
```

The Manager then drives `eos_grid_setpoint -> 0`, which means the real grid follows `eos_target`.
The selected manager mode decides the allowed direction:

| EOS mode | Zendure Manager mode | `eos_target` | Effect |
| --- | --- | --- | --- |
| `self_consumption` | `smart` | `0` | zero export: discharge for load, absorb surplus |
| `non_export` | `store_solar` | `0` | absorb surplus, never discharge |
| `forced_charge`, `grid_support_import` | `smart_charging` | `factor * 1200` | grid charge up to the target |
| `idle` | `off` | `0` | battery holds |
| anything else | `smart` | `0` | safe default (zero export) |

Implementation in Home Assistant:

1. Template sensor `sensor.eos_grid_setpoint` (snippet below).
2. Automation `EOS SolarFlow Manager Bridge` that maps the EOS mode to the manager mode.
3. Zendure integration option `p1meter` changed from `sensor.power_actual` to
   `sensor.eos_grid_setpoint`. The Energy Dashboard keeps using the real `sensor.power_actual`.

<!-- pyml disable line-length -->

```yaml
- sensor:
    - name: "Eos Grid Setpoint"
      unique_id: "eos_grid_setpoint"
      unit_of_measurement: "W"
      device_class: power
      state_class: measurement
      state: >-
        {% set mode = states("sensor.eos_solarflow2400pro") %}
        {% set factor = state_attr("sensor.eos_solarflow2400pro", "operation_mode_factor") | float(0) %}
        {% set grid = states("sensor.power_actual") | float(0) %}
        {% if mode in ["force_charge", "grid_support_import"] %}
          {{ grid - (factor * 1200) | round(0) }}
        {% else %}
          {{ grid | round(0) }}
        {% endif %}
```

```yaml
- id: eos_zendure_manager_bridge
  alias: "EOS SolarFlow Manager Bridge"
  triggers:
    - trigger: state
      entity_id: sensor.eos_solarflow2400pro
    - trigger: homeassistant
      event: start
  actions:
    - variables:
        eos_mode: "{{ states('sensor.eos_solarflow2400pro') }}"
        target_mode: >-
          {% if eos_mode in ['force_charge','grid_support_import'] %}smart_charging
          {% elif eos_mode == 'non_export' %}store_solar
          {% elif eos_mode == 'idle' %}off
          {% else %}smart{% endif %}
    - condition: template
      value_template: "{{ states('select.zendure_manager_operation') != target_mode }}"
    - action: select.select_option
      target:
        entity_id: select.zendure_manager_operation
      data:
        option: "{{ target_mode }}"
  mode: restart
```

<!-- pyml enable line-length -->

:::{admonition} Note
:class: note
The manager only accepts a mode other than `off` when at least one device is online
(`connectionStatus >= 10`). HEMS on (status 2) or fuse group `unused` (status 3) block it.
:::

### Phase 6 - Verify

1. `sensor.eos_grid_setpoint` equals `sensor.power_actual` while the EOS target is `0` and is
   offset by `factor * 1200` during `forced_charge`.
2. `select.zendure_manager_operation` follows the EOS mode (`smart`, `store_solar`,
   `smart_charging`, `off`).
3. During `non_export` the battery does **not** discharge; during `self_consumption` the real grid
   stays near `0`.
4. Grid charging only happens in the planned `forced_charge` windows.
5. The battery respects `min_soc`.

### Phase 7 - Rollback

1. Set `select.zendure_manager_operation = off` and disable the HA automations.
2. Re-enable HEMS in the Zendure app.
3. Restore the fuse groups if needed.

## 5. Open questions and risks

- **Vendor features**: disabling HEMS also disables Zendure's own AI scheduling in the app. That is
  intended, but the app may show warnings.
- **`smartMode` interplay**: confirm on the device whether manual limits are honored while the
  vendor smart mode is active.
- **Meter orientation**: the zero-export loop is only as good as the Shelly CT direction and phase
  assignment.
- **Fuse group choice**: the group caps charge/discharge power. Pick one that matches the house
  fuses, not the largest value.
- **One master**: the Zendure Manager owns `acMode` and the limits; the EOS bridge only sets the
  manager mode and the virtual P1. Do not add a second component that writes the limits.
- **Rate limits**: the Shelly EM3 and the Zendure cloud API both throttle. A cloud round trip is
  slower than a local (zenSDK/local MQTT) connection, so a local connection is preferred for a
  tight control loop.

<!-- pyml enable no-emphasis-as-heading -->
