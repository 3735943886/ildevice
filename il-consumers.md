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
| `fan_speed` | fan percentage derived from ordered levels; climate fan mode | `fan_control.fan_mode` |
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

Non-role fields map too. A property `class` maps to a supported Home Assistant `device_class`
when the platform recognises it; spellings are not always identical (`datetime` maps to
`timestamp`). `series: counter` normally maps to `state_class: total_increasing`, and `gauge`
to `measurement`, subject to that platform's device-class restrictions. `category` maps to
`entity_category`. Unknown classes do not prevent the property from being used.

Property types map the same way: `event` is an event entity (Home Assistant `event`, one `options` entry per event type), `trigger` is a button (Home Assistant `button`), and a
plain control with `requires` can be shown unavailable while its condition does not hold (the
named binary is not `true`, or the named select's value is not one of `in`). For a composite,
check the condition of the specific property being written; one conditional setting need not
make the entire composite unavailable. The producer still validates every write.

A `locked` or `opened` property that is not `rw` is a read-only state and does not form a lock
or valve composite (K-6). A binary sensor or a native read-only presentation can show it.

A gap in such a table is fine: it is where a consumer falls back to the generic property.

A consumer reads the messages of [il-messages.md](il-messages.md) over whatever transport carries them
(for example [il-mqtt.md](il-mqtt.md)), or takes the driver outputs directly when it shares a process
with the producer.

## State recovery and commands

A `sync` snapshot contains the descriptor and one `value` or `absent` for every state-bearing
property (W-7). An identical descriptor is not a command to clear the cache. Removed or
redefined properties must not retain their old cached meaning. Events are occurrences and
are never replayed as state.

MQTT retained replay restores individual topics and does not provide an atomic snapshot or
a replay-complete marker. Follow [the MQTT recovery guidance](il-mqtt.md#4-mapping-the-messages)
when reconnecting; absence of a retained message does not establish freshness of an old value.

Commands have no correlated success acknowledgement or deadline (W-12). A successful trigger
or unchanged write can be silent. Show observed state and explicit rejections without treating
silence as success, and avoid automatically retrying an action whose outcome is uncertain.

## Compatibility evidence for rusthinq

The 2026-10-20 source review found that `rusthinq-scripts/modules/il_common.rhai` publishes
retained descriptors on `<prefix>/<id>/descriptor`, sets `x-mqtt` for property and command
topics, and uses `il: 0`. The `/descriptor` discovery path is an implementation extension
supported by ildevice-homeassistant; it is not the canonical M-1 discovery path.

The Home Assistant consumer's 219 passing tests at that review included stored rusthinq
fixtures, command routing and `/descriptor` discovery. This supports compatibility of that
consumer path, not a claim that the current producer fully conforms to v0.1.0. The review did
not run the rusthinq test suite or exercise a physical device. In particular:

- W-7's revised `sync` snapshot does not directly change the MQTT command or state payloads.
- M-13 needs verification for a property's type, unit or meaning changing on the same topic;
  removing obsolete topics alone does not establish compliance.
- The reviewed helper sets `source` to `rusthinq`. If several instances publish presence under
  the same prefix, each needs a distinct matching source in its descriptors and presence topic
  (M-14). A fixed source alone is not evidence of a failure when presence is unused.
- The helper publishes availability as true during initialisation. Full A-2 compliance requires
  evidence that a complete, current device state is available at that point.

The sans-IO driver interface is a conformance target; a script that performs publication through
its context is not thereby proven to implement that interface. Claims should name the release,
commit and conformance profiles actually checked.
