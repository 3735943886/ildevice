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
| `mode` | `climate.hvac_mode` / `humidifier.mode` / fan preset mode | no standard attribute |
| `fan_speed` | fan preset mode | `fan_control.fan_mode` |
| `speed` | `fan.percentage` | `fan_control.percent_setting` |
| `oscillate` | `fan.oscillating` | `fan_control.rock_setting` |
| `direction` | `fan.direction` | `fan_control.airflow_direction` |
| `target_humidity` | `humidifier.target_humidity` | no standard attribute |
| `current_humidity` | `humidifier.current_humidity` | `relative_humidity_measurement.measured_value` |
| `current_temperature` | `climate.current_temperature` | `thermostat.local_temperature` |
| `swing_vertical` | `climate.swing_mode` | `fan_control.rock_setting` |
| `swing_horizontal` | `climate.swing_horizontal_mode` | `fan_control.rock_setting` |
| `action` | `climate.hvac_action` | `thermostat.thermostat_running_state` |
| `target_temperature` | `climate.temperature` | `thermostat.occupied_cooling_setpoint` |
| `brightness` | `light.brightness` (0..255) | `level_control.current_level` (1..254) |
| `color_temperature` | `light.color_temp_kelvin` | `color_control.color_temperature_mireds` (`1e6 / K`) |
| `color` | `light.rgb_color` / `hs_color` | `color_control.current_hue` / `current_saturation` |
| `color_mode` | `light.color_mode` | `color_control.color_mode` |
| `position` | `cover.current_position` | `window_covering.current_position_lift_percent100ths` (inverted: `0` is open) |
| `tilt` | `cover.current_tilt_position` | `window_covering.current_position_tilt_percent100ths` (inverted) |
| `cover_state` | `cover.is_opening` / `is_closing` / `is_closed` | `window_covering.operational_status` |
| `open` `close` `stop` | `cover.open_cover` / `close_cover` / `stop_cover` | `up_or_open` / `down_or_close` / `stop_motion` |
| `locked` | `lock.lock` / `unlock` | `door_lock.lock_door` / `unlock_door` |
| `lock_state` | `lock` state (`locked`, `unlocked`, `locking`, `unlocking`, `jammed`, `open`) | `door_lock.lock_state` |
| `unlatch` | `lock.open` | `door_lock.unlatch_door` |
| `opened` | `valve` (`is_closed` is its negation) | `valve_configuration_and_control.current_state` |
| `alarm_state` | `alarm_control_panel` state | no standard attribute |
| `arm_home` `arm_away` `arm_night` `disarm` | `alarm_control_panel` arm / disarm services | no standard attribute |
| `vacuum_state` | `vacuum` activity | `rvc_operational_state.operational_state` |
| `start` `pause` `return_home` `locate` | `vacuum` start / pause / return_to_base / locate | `rvc_operational_state` commands |
| `battery` | `sensor` with `device_class: battery` | `power_source.bat_percent_remaining` |

Non-role fields map too. `class` on a number or binary is the Home Assistant `device_class`
(`temperature`, `humidity`, `duration`, `energy`, `power`, `volume`, `pm25`, `problem`,
`running`, `heat`, …); `series: counter` is `state_class: total_increasing` and `gauge`
is `measurement`; `category` is `entity_category`.

Property types map the same way: `event` is an event entity (Home Assistant `event`, one `options` entry per event type), `trigger` is a button (Home Assistant `button`), and a
property with `requires` is shown unavailable while its condition does not hold (the named binary is
not `true`, or the named select's value is not one of `in`).

A `locked` property that is not `rw` is a read-only state, not a lock: a consumer shows it as a
binary sensor rather than offering a lock it cannot operate.

A gap in such a table is fine: it is where a consumer falls back to the generic property.

A consumer reads the messages of [il-messages.md](il-messages.md) over whatever transport carries them
(for example [il-mqtt.md](il-mqtt.md)), or takes the driver outputs directly when it shares a process
with the producer.
