# IL: rationale, worked examples and history

> **Informative.** Nothing here adds a requirement to [il.md](il.md); where the two differ,
> il.md wins. This is where the reasons live, so that the specification itself can stay short:
> how properties of real devices were mapped, what each device changed in the model, and the
> changelog.

ecosystems use for `μg/m³`; a consumer that compares units should treat U+00B5 and U+03BC
as the same.

## 1. Mapping from other property models

The types are the intersection of what devices express and what consumers render,
so producers on other stacks map in directly:

| source model | IL |
|---|---|
| Tuya `bool` | `binary` |
| Tuya `value` (with `scale`) | `number` (the producer applies the scale and reports the real unit) |
| Tuya `enum` | `select` |
| Tuya `string` | `text` |
| Tuya `bitmap` / `fault` | `number` (raw) or one `binary` per bit |
| Tuya `raw` | not represented; keep it private under `x-` |
| LG TLV tag | per the tag's meaning; the tag goes in `src` |

## 2. Worked examples

### First device: DHUM_056905_WW (LG dehumidifier, via rusthinq)

| wire tag | property | type | role | note |
|---|---|---|---|---|
| (connection) | `available` | binary | `available` | |
| 0x1f7 | `power` | binary | `on` | |
| 0x1f9 | `mode` | select | `mode` | smart, jet, silent, spot, laundry |
| 0x253 | `target` | number | `target_humidity` | 30..70, step 1 |
| 0x336 | `humidity` | number | `current_humidity` | |
| 0x1fa | `fan` | select | `fan_speed` | low, high |
| 0x1fd | `temperature` | number | `current_temperature` | wire is half degrees; reported in °C |
| 0x186 | `tank_full` | binary | | true for wire 1 or 2 |
| 0x186 | `tank_state` | select | | not_full, full_stopped, full_fan_mode |
| 0x360 | `sterilize` | binary | | |
| 0x2a2 | `uvnano` | binary | | |
| 0x21e | `bucket_light` | binary | | |
| 0x21b | `off_timer` | number | | minutes, 0 = none; the appliance takes whole hours, so `step` is 60 |
| 0x221 | `error` | number | | raw fault code |

The tank has three wire states; `tank_full` reports a boolean, and what the third state
means is to be settled on the real device. That is the intended way the IL is refined.

### Second device: AIR_910604_WW (LG air purifier, via rusthinq)

`kind: "fan"`, `class: "air_purifier"`. Properties: `available`, `power` (`on`), `mode`
(`mode`: circulator_clean, baby_care, dual_clean, auto), `fan` (`fan_speed`: low, mid, high,
power, auto), `pm1`, `pm25`, `pm10` (μg/m³), `air_quality`, `odor` (1..4), `filter_life`,
`top_filter_life` (percent, computed from two wire tags: hours left over the budget),
`light`, `sterilize`, `sleep_timer` (min), `error`.

What the second device changed:

- **`fan_speed` was promoted to a role.** Both appliances have a fan-strength select, with
  different options (`low, high` here, `low, mid, high, power, auto` there), which is what
  roles are for: the meaning is standard, the options stay the device's own. The
  dehumidifier's `fan` gained the role; no property name changed.
- `sterilize` (wire 0x360) is the same thing on both appliances and already had the same
  name. It is a candidate for a role once a third device has it.
- A property may be **derived from several wire tags** (`filter_life` from hours left and
  the budget), and one wire tag may feed several properties (`tank_full`, `tank_state`).
  Both are driver business and needed nothing from the model.

### Third device: 1WPU4CIGCR__2 (LG water purifier, via rusthinq)

`kind: "dispenser"`, `class: "water_purifier"`; an AABB (non-TLV) appliance, which the model
did not care about: `status`, `tap_uv`, `water_selection`, `water_amount` (selects),
`default_water` and `default_amount` (writable selects), `auto_care`, `button_sound`,
`not_use_notice` (writable binaries), four "dispensed today" numbers, and `self_clean_next`.

- **The `text` type is now used**, by `self_clean_next` (a month-day and time the wire
  carries without a year, in UTC). It stays: nothing else in the model fits a date-like
  value, and turning it into local time is a consumer's job (it needs a calendar and a
  timezone), so the driver reports it as it is.
- **Proposed then, adopted since:** the "dispensed today" numbers reset every day, which a
  consumer wants to know (Home Assistant calls it a total that may reset). An optional
  hint on a `number` would say so. This became the property field `series: "counter"`.
- `select` options are free tokens per device (`120ml`, `250ml`, `continuous`), a writable
  select may list fewer options than its read-only sibling (`default_amount` has no
  `continuous`), and nothing in the model had to change for that.

### Fourth and fifth devices: D140110 (dishwasher) and WBEY3GT (cooktop)

The dishwasher (`kind: "washer"`, `class: "dishwasher"`) is read-only: no `rw` property at
all, which the model already allowed. The cooktop (`kind: "cooktop"`) is the first device
whose controls are dangerous and conditional, and it changed the model twice:

- **`trigger` type.** "Switch this ring off" and "switch the cooktop off" are actions, not
  values. A `binary` with `rw` would invent a state that is never reported; `trigger` says
  what it is (a button). Consumers that do not know it skip it, per the compatibility rules.
- **`requires`.** The appliance obeys a network command only while the panel's own remote
  button has been pressed; that permission is itself reported (`remote_start`). `requires`
  ties a control to a binary property, so a consumer can disable it and a producer rejects
  it otherwise. rethink did this with a per-entity availability topic; here it is data.
- **`group` was needed and sufficient.** Each of the three rings has its own state, level
  and timers: `right_state`, `right_power_level`, ... all with `group: "right"`.
- **A driver may refuse to do a thing at all.** No command here lights a ring: the app never
  sent one, so it is not invented. That is a driver decision and needs no model support.

### Sixth to ninth devices: the laundry family (mini washer, dryer, styler, washer)

Four appliances on one protocol (`kind: "washer"`, `"dryer"`, `"clothing_care"`), so this is
where the model got its repeated use:

- **A writable property that is driver state, not appliance state.** The appliance reports
  where the dial sits (`course`, read only) but has no notion of a pending course, while a
  Start command has to name one. `course_select` is a writable `select` the driver keeps
  itself: it follows the dial when the dial moves, keeps a choice across the appliance's
  status repeats, and is what `start` asks for. Two properties for two facts, and nothing
  in the model was needed for it; the descriptor's label ("Course for Start") says which is
  which.
- **`trigger` and `requires` carried all of it.** Start, Pause and Power off are triggers
  that require `remote_start`, the flag the appliance reports for "armed at the panel".
- **A command the appliance acknowledges and ignores is not a feature.** The styler answers
  a power-on with a plain acknowledgement and does nothing, so none is offered; an
  acknowledgement only says the frame was well formed. A refusal (status 0xff) is reported
  as a `Reject` against the property whose command it answers.
- **Enumerations are free tokens per device** (`heavy_duty`, `kids_wear`), a device may
  carry several selects that share a token set (`course`, `op_course`, `download_course`),
  and a select may list options the appliance reports but no command can select
  (`course` lists the dial's downloaded-course position, `course_select` does not).

### Tenth device: CST_570004_WW (LG ceiling-cassette air conditioner, via rusthinq)

`kind: "climate"`, `class: "air_conditioner"`; one indoor unit of a cooling-only multi-split.
Properties: `available`, `power` (`on`), `mode` (`mode`: cool, dry, fan_only, auto), `fan`
(`fan_speed`: auto, very_low, low, medium, high, power), `target` (`target_temperature`,
16..30, step 0.5), `temperature` (`current_temperature`), `humidity` (`current_humidity`),
`action` (off, idle, cooling, drying, fan, active), `swing_vertical`, `swing_horizontal`,
`energy_save`, `comfort_saving`, `wind_mode` (off, manner, long_power, study, auto_temp),
`auto_dry` (off, 10min, 30min, 60min, smart), `auto_dry_remaining`, `sleep_timer` (min),
`display` (off, 50%, 100%), `power_draw` (W), `filter_remaining` (%), `filter_used`,
`filter_life` (h), `error`.

What the tenth device changed:

- **`current_temperature` was promoted to a role** (the dehumidifier's `temperature` had
  none), and **`target_temperature` was added** with it: a climate device cannot be
  described without them. Both are in °C whatever the wire scale (half degrees here).
- **Power and mode stay separate.** The wire reports a mode even while the unit is off, so
  `mode` always carries the wire's mode and `power` says whether it runs; a consumer that
  has a single "off" mode derives it (`power` false).
- **`action` is a plain read-only select**, not a role: what the unit is doing right now,
  derived from power, mode and, when the unit reports it, whether it is really cooling.
  `active` means running in `auto`, where the wire does not say which it is.
- **A value that only counts in some state.** `energy_save` takes effect only while
  cooling and the appliance forgets it over a power cycle. The driver keeps the wanted
  value, publishes it, and writes it again when the unit next enters cool; the descriptor
  cannot express "only in cool", and `requires` (a binary that must be true) does not
  fit a condition on a mode. A `requires` that can name a
  `select` value is deferred (section 6 below).
- **Two properties from one tag.** `filter_remaining` and `filter_used` are computed from
  the same pair of tags (hours left, rated life).

## 3. Evolving the IL

- **Extras first, roles later.** A device-specific value starts as a property without a
  role. When a second device, in any project, has the same thing, promote it: add a role
  with meaning, type and unit, and tag both. No property name has to change.
- **Gaps are proposed, not patched around.** Anything a device needed that the model
  cannot say is written down here as a proposed change with the device that raised it.
- **The spec follows the drivers.** A declarative driver definition is deferred until two
  devices show which operations actually repeat.
- **Found while writing the first driver:** all three points were settled in the specification pass
  (section 5 below): unlisted `select` values (V-3), where `Reject` goes (il-mqtt), and the initial
  value of `available` (A-2).
- **Reserved for later, deliberately absent from v0:** structured values. (One-shot events
  are no longer reserved: they are the `event` type.)

## 4. Changelog

- `0` (draft): [il-messages.md](il-messages.md): the transport-independent message forms, snapshot and
  `sync`, `presence`; il-mqtt.md is now one mapping of them. `requires` may name a `select` and its
  allowed values; roles and kinds for `alarm` and `vacuum`, role `battery`; property class `datetime`.

- `0` (draft): il.md rewritten as a normative document (RFC 2119 keywords, rule ids, glossary,
  per-kind required roles, class and unit registries, safety and conformance sections); history
  moved here. Additions: `Reject.code` and its list, the rules V-2 (empty text is absent), V-3
  (unlisted select values), A-2 (initial availability), C-3 (step tolerance); kind `switch`.

- `0` (draft): property type `event` (an occurrence, the mirror of `trigger`); role `opened`
  (valve); kind `siren` (role `on`); property `class` also applies to a writable binary and a
  trigger, and `category: diagnostic` to a trigger.
- `0` (draft): descriptor field `groups`: a group with a `kind` is a composite of its own, so a
  device can be a cover and a light, or have two lights (13 of 285 Tuya fixture devices did).

- `0` (draft, unreleased addition): roles for lights (`brightness`, `color_temperature`,
  `color`, `color_mode`), covers (`position`, `tilt`, `motion`, `open`, `close`, `stop`) and

## 5. Decisions taken in the specification pass

The first restructuring into a normative document settled these points, which had been open
proposals. Each is a default that a later device can overturn by the rules of section 3.

| point | decision | rule |
|---|---|---|
| A `select` value the options do not list | a producer lists it or treats the property as absent; a consumer tolerates one it receives | V-3 |
| Empty `text` vs absent (found by the MQTT mapping, where an empty retained payload means absent) | an empty text is not a value | V-2 |
| Initial value of `available` | `false` until the first complete state, and after `Disconnected` | A-2 |
| Why a write failed | `Reject` carries a `code` from a closed list, plus free text | C-4, C-5 |
| Step and range check | inclusive range, exact decimal step, 1e-9 tolerance for floats | C-3 |
| What a composite needs | a required and an optional role list per kind | section 10 |
| `requires` naming a `select` value | adopted as an object form `{ prop, in }`; the string form stays | P-5 |
| Structured / date-time value | no new type; a `text` with class `datetime` (RFC 3339 with offset). A driver whose wire has no year or zone reports plain text without the class | section 11.1 |
| `camera`, `alarm_control_panel`, `vacuum` | `alarm` and `vacuum` composites with provisional roles taken from Home Assistant and Matter, not yet tried on a device; `camera` has no composite (motion and recording switches are plain properties) | section 10 |
| Producer presence | a `presence` message (MQTT: a retained topic with a Last Will) | il-messages.md W-6 |
| Message size | descriptor 64 KiB, text 1 KiB | W-13 |
| Transport | none in the model; the data shape is in il-messages.md and MQTT is one mapping | il-messages.md |

## 6. Open points

- Whether a driver is written as Rhai returning `Output` values, as native Rust, or both behind the
  same interface. The interface allows either.
- Whether `group` is enough for multi-component devices or needs a real nested form. `groups` with a
  `kind` is the answer so far (found by the Tuya spike, see
  [notes/tuya-spike-gaps.md](notes/tuya-spike-gaps.md)).
- Roles for `alarm` and `vacuum` are provisional until a device uses them; `camera` roles wait for
  a device that needs one.
- Conformance vectors so far cover command validation only ([vectors/](vectors/)); driver
  input/output sequences wait for a first driver's captured frames.

## Fan roles `speed`, `oscillate`, `direction`

Found by comparing a Tuya producer with Home Assistant core's tuya fans: a fan there is one entity with a
percentage, an oscillation switch and a direction. The IL had only the select `fan_speed`, so a producer had to
publish the three as unrelated properties and a consumer showed them as three extra entities beside the fan.
`speed` (number, percent), `oscillate` (binary) and `direction` (select, `forward`/`reverse`) are optional roles of
the `fan` kind; `fan_speed` stays for devices whose speeds are named levels.
