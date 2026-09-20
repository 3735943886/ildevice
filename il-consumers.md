# Consuming the IL

> **Informative.** A starting point for whoever writes a consumer of [IL](il.md)
> descriptors. Not part of the IL; each consumer owns its own table.

A consumer maps roles to its own native concepts and treats every property without a
known role generically (a plain switch, number, select or read-only sensor built from the
property's `type`, `unit`, `min`/`max`/`step` and `options`).

| role | Home Assistant | Matter |
|---|---|---|
| `available` | availability | `bridged_device_basic_information.reachable` |
| `on` | on/off | `on_off.on_off` |
| `fan_speed` | fan preset mode | `fan_control.fan_mode` |
| `target_humidity` | `humidifier.target_humidity` | no standard attribute |
| `current_humidity` | `humidifier.current_humidity` | `relative_humidity_measurement.measured_value` |
| `current_temperature` | `climate.current_temperature` | `thermostat.local_temperature` |
| `swing_vertical` | `climate.swing_mode` | `fan_control.rock_setting` |
| `swing_horizontal` | `climate.swing_horizontal_mode` | `fan_control.rock_setting` |
| `action` | `climate.hvac_action` | `thermostat.thermostat_running_state` |
| `target_temperature` | `climate.temperature` | `thermostat.occupied_cooling_setpoint` |

Non-role fields map too. `class` on a number or binary is the Home Assistant `device_class`
(`temperature`, `humidity`, `duration`, `energy`, `power`, `volume`, `pm25`, `problem`,
`running`, `heat`, …); `series: counter` is `state_class: total_increasing` and `gauge`
is `measurement`; `category` is `entity_category`.

Property types map the same way: `trigger` is a button (Home Assistant `button`), and a
property with `requires` is shown unavailable while the property it names is not `true`.

A gap in such a table is fine: it is where a consumer falls back to the generic property.

A consumer reads what a producer emits over a transport such as
[il-mqtt.md](il-mqtt.md), or takes the driver outputs directly when it shares a process
with the producer.
