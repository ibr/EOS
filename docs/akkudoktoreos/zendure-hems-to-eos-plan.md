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
3. Confirm `zendure_ha` still points to `sensor.power_actual` (config entry field `p1meter`).

### Phase 3 - Let EOS produce a plan

1. Configure the battery and the inverter in EOS and let it publish its instruction entity
   `sensor.eos_<device_id>`.
2. Verify that an automatic run succeeds and that the plan contains battery instructions.

### Phase 4 - Build the Home Assistant control layer

Translate the EOS instruction entity and its `operation_mode_factor` attribute into `zendure_ha`
entity writes. The relevant entities are:

- `select.solarflow_2400_pro_ac_mode` (`input` = charge, `output` = discharge)
- `number.solarflow_2400_pro_input_limit` (AC charge limit, up to 3200 W)
- `number.solarflow_2400_pro_output_limit` (AC discharge limit, up to 2400 W)
- `number.solarflow_2400_pro_min_soc` and `number.solarflow_2400_pro_soc_set`

| EOS mode | `ac_mode` | `input_limit` | `output_limit` |
| --- | --- | --- | --- |
| `force_charge` | `input` | `factor * 1200` | `0` |
| `self_consumption` | `output` | `0` | controlled by the loop |
| `non_export` | `output` | `0` | controlled by the loop |
| `peak_shaving` | `output` | `0` | controlled by the loop |
| `idle` | `output` | `0` | `0` |

:::{admonition} Note
:class: note
`zendure_ha` writes `smartMode` together with `acMode` and the limits. If only the number entities
are written, verify that `sensor.solarflow_2400_pro_smart_mode` is `1`; otherwise the device may
ignore the manual limits.
:::

### Phase 5 - Close the zero-export loop

For every discharge mode, run a short control loop (2-5 s) on `sensor.power_actual`:

```text
target = current_output_limit + power_actual
target = clamp(target, 0, max_discharge_power)
number.set_value(number.solarflow_2400_pro_output_limit, target)
```

- `power_actual > 0` (import) increases discharge until the grid is balanced.
- `power_actual < 0` (export) decreases discharge immediately.
- A small dead band (for example 30 W) avoids permanent small adjustments.
- Throttle the Shelly updates; do not send a new setpoint faster than the meter reports.

For `force_charge`, zero export is intentionally not the target. Instead, cap `input_limit` so that
PV surplus is absorbed before the battery starts importing from the grid.

### Phase 6 - Verify

1. `sensor.power_actual` stays within the dead band around `0` for a full day.
2. The measured battery power follows the EOS plan (`sensor.eos_solarflow2400pro`).
3. Grid charging only happens in the planned cheap hours.
4. The battery respects `min_soc`.

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
- **Two masters**: keep the Zendure Manager off while the HA loop runs; only one component may own
  `acMode` and the limits.
- **Rate limits**: the Shelly EM3 and the Zendure cloud API both throttle. A cloud round trip is
  slower than a local (zenSDK/local MQTT) connection, so a local connection is preferred for a
  tight control loop.

<!-- pyml enable no-emphasis-as-heading -->
