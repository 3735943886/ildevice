# IL: a neutral device description (sans-IO)

> **Draft, version 0.** A specification shared by every project that turns a device into
> data (rusthinq, rustuya, and consumers of either). It depends on no project, no
> transport and no programming language. No code lives in this repository; the first drivers
> (rusthinq) follow it from their own.

The IL describes what a device *is*, what values it has, and which of them can be
written, using no vocabulary of Home Assistant, Matter, Tuya, LG or any one consumer.

**The IL is sans-IO.** It is data types plus pure functions. It never opens a socket,
reads a clock, spawns a thread, or knows a topic. Whoever implements it (a rusthinq host, a
rustuya bridge, a test) feeds it events and carries out what it asks for. How the
data leaves a producer (MQTT, in-process calls, a file) is separate: the MQTT mapping is
[il-mqtt.md](il-mqtt.md), and it adds nothing to the model.

```
                 ┌──────────── IL (this document): data + pure logic ─────────────┐
 host events ──► │ Driver::handle(Input) ──► Vec<Output>          Descriptor,     │ ──► host effects
 (frame, time,   │ (a state machine, no I/O)                      Value, Role     │    (send, timer,
  command)       └────────────────────────────────────────────────────────────────┘     publish)
```

Design goals, in priority order:

1. **No I/O in the IL.** Every effect is a returned value; every input is an argument.
2. **Producer and consumer never need to know each other.** Everything a consumer needs
   to use a device is in the descriptor.
3. **Additive growth.** New devices add properties, roles and fields without breaking
   anyone who already runs.
4. **Small.** Six value types, one descriptor, no per-device-kind schemas.

## 1. Descriptor

A device is a list of **typed properties**. Every value is described the same way. A
property with a standard meaning carries a `role`; one without is simply a property
without a role, and is fully usable.

```json
{
  "il": 0,
  "id": "00000000-0000-0000-0000-000000000000",
  "source": "rusthinq",
  "kind": "humidifier",
  "class": "dehumidifier",
  "label": "Dehumidifier",
  "vendor": "LG",
  "model": "DHUM_056905_WW",
  "identifiers": { "thinq_id": "00000000-0000-0000-0000-000000000000" },
  "props": {
    "available":{ "type": "binary", "role": "available" },
    "power":    { "type": "binary", "rw": true,  "role": "on" },
    "mode":     { "type": "select", "rw": true,  "role": "mode",
                  "options": ["smart", "jet", "silent", "spot", "laundry"] },
    "target":   { "type": "number", "rw": true,  "role": "target_humidity",
                  "min": 30, "max": 70, "step": 1, "unit": "%" },
    "humidity": { "type": "number", "role": "current_humidity", "unit": "%" },
    "fan":      { "type": "select", "rw": true,  "role": "fan_speed", "options": ["low", "high"] },
    "tank_full":{ "type": "binary" },
    "tank_state":{ "type": "select", "options": ["not_full", "full_stopped", "full_fan_mode"] },
    "uvnano":   { "type": "binary", "rw": true, "src": { "tlv_tag": "0x2a2" } }
  }
}
```

The example shows a subset of the device's properties (the full list is in section 5).

| field | meaning |
|---|---|
| `il` | IL version the descriptor was written against (integer). |
| `id` | Globally stable device id. Same device, same id, across restarts. |
| `source` | Name of the producer (`rusthinq`, `rustuya`, …). Free string. |
| `kind` | Coarse category, an open string (`humidifier`, `climate`, `switch`, …). A hint, not a schema selector. |
| `class` | Optional refinement of `kind`. |
| `label` `vendor` `model` | Human-facing identity. `label` is the name a person calls the device: a producer that knows the owner's own name for it (an account alias, a name typed into a setup screen) puts that here, and only falls back to a generic one ("Dehumidifier") when it has none. A consumer names the device, and so its entities, after it. |
| `identifiers` | Free map of other ids (MAC, cloud id, Tuya device id) for matching with other systems. |
| `props` | Property name → definition. Names are `[a-z0-9_]+`. |
| `groups` | Optional map from a `group` name to `{ "kind", "class", "label" }`. A group that has a `kind` is a composite of its own (see below); one without is only a label. |

The descriptor has no topics, URLs or addresses. Where a value physically travels is
never part of the model.

### Property definition

| field | applies to | meaning |
|---|---|---|
| `type` | all | `binary`, `number`, `select`, `text`, `trigger`, `event`. |
| `rw` | all | `true` if the property accepts writes. Default `false`. |
| `role` | all | Optional standard meaning, see section 3. |
| `requires` | all | Optional name of a `binary` property that must currently be `true` for this property to accept writes (a control that only works while the panel has granted remote start). A consumer greys the control out otherwise; a producer still rejects a write made anyway. |
| `group` | all | Optional string grouping properties of one sub-unit (`ch1`, `zone_a`) for devices with several of the same thing. If the device's `groups` gives the group a `kind`, the group is a composite of its own. |
| `unit` | number | Plain-text unit label (`%`, `°C`, `min`). |
| `min` `max` `step` | number | Range and increment. |
| `options` | select, event | Allowed values (for an `event`, the kinds of occurrence), stable order. |
| `label` | all | Optional human-readable name. |
| `class` | all | Optional open string saying *what* the value is, for a consumer that classifies values: for a number the quantity (`temperature`, `humidity`, `duration`, `energy`, `power`, `volume`, `pm1`, `pm25`, `pm10`), for a binary what being true means (`problem`, `running`, `heat`, `door`), for a writable binary the kind of load it switches (`outlet`, `switch`), for a trigger what it does (`restart`, `identify`, `update`). Unknown classes are ignored. Not the same as the descriptor's `class`. |
| `series` | number | `gauge` (default): the value goes up and down. `counter`: a running total that only grows and starts again from zero (energy of a cycle, water dispensed today). |
| `category` | all | `diagnostic`: read only, or a `trigger`, of interest when troubleshooting (fault codes, locks, counters, a factory reset); `config`: a setting rather than an everyday control. Absent means an ordinary property. A consumer may hide or group by it; nothing else changes. |
| `src` | all | Optional opaque origin hint (a protocol tag, a Tuya dp id). Never interpreted by a consumer. |

### Values

| type | value |
|---|---|
| `binary` | boolean |
| `number` | a number **already in the declared `unit`**. Scaling and unit conversion are the producer's job. |
| `select` | one of `options`, a string |
| `text` | a string |
| `trigger` | never a value. Writing to it (any payload) performs the action; it is never published and is always `rw`. |
| `event` | never a value, and never `rw`: the mirror of a `trigger`. Each time the device does the thing (a button pressed, a door bell rung) the producer publishes **one occurrence**, one of `options` as a string. The occurrence *is* the time it happened; there is no timestamp, no state to recover, and the same kind twice in a row is two occurrences. |

A value the device is not currently reporting is *absent*, never a placeholder.

### Commands

A consumer writes a property in the value forms above. A producer checks every write
against its own descriptor **before** acting on it, and reports a failure as a `Reject`;
nothing is sent to the device for a write that fails:

1. the property must exist, and be `rw` (a `trigger` is always writable);
2. if it has `requires`, the named `binary` property must currently be `true` (a value the
   producer has not reported counts as not satisfied);
3. `binary`: `true`/`false` (also `on`/`off`/`1`/`0`, normalised to `true`/`false`);
   `number`: a finite number within `min`/`max` and a multiple of `step` counted from `min`
   (from 0 if there is none); `select`: one of `options`; `text` and `trigger`: any payload.

A value outside a range or step is **refused, not clamped**: silently writing a different
value than was asked for is worse than saying no. The driver that acts on a valid write
receives it in canonical form (`true`/`false`, an integer written as an integer).

### Availability

Availability is an ordinary property with the role `available` (binary, read only). A
producer with no notion of availability omits it; a consumer then assumes the device is
available while values keep arriving.

### Compatibility rules

- **Unknown fields are ignored.** Fields private to one project are prefixed `x-`
  (`x-tuya`) so they cannot clash with a later standard field. Transport bindings use
  the same mechanism for their own hints.
- **Unknown roles are ignored**; the property is treated generically.
- **A property with an unknown `type` is skipped**, not an error.
- **`il` is bumped only for a breaking change** (removing or redefining something). A
  consumer that sees a higher `il` than it knows still tries, and only refuses if it
  cannot honour the fields it needs.

## 2. The driver interface (sans-IO)

A producer is a **driver**: a state machine that turns what happened into what should be
done. It is defined as data in, data out:

```
Driver::handle(now, Input) -> Vec<Output>
```

`now` is a timestamp handed in by the host; the driver never reads a clock.

**Inputs**

| input | meaning |
|---|---|
| `Connected` / `Disconnected` | the device link came up or dropped |
| `Frame(bytes)` | bytes received from the device |
| `Message { channel, json }` | a structured message from the device on a named channel, for protocols that have them besides raw frames |
| `Command { prop, value }` | a consumer asked to write a property |
| `Timer(name)` | a timer the driver set has fired |

**Outputs**

| output | meaning |
|---|---|
| `Descriptor(desc)` | the device's description (emit on connect and when it changes) |
| `Value { prop, value }` | a property now has this value |
| `Absent { prop }` | a property no longer has a value |
| `Event { prop, kind }` | an `event` property happened once |
| `SendFrame(bytes)` / `SendMessage { channel, json }` | write to the device |
| `SetTimer { name, after }` / `CancelTimer(name)` | schedule or cancel a wake-up |
| `Reject { prop, reason }` | a command could not be honoured |

Consequences:

- **Timers are outputs, not a facility.** The host owns the clock and the scheduler; a
  driver asks for `SetTimer("refresh", 5s)` and later receives `Timer("refresh")`.
  Handshake retries, periodic refresh and a debounced re-query after a command are all
  this. Setting a timer with an existing name replaces it.
- **A driver is testable with no environment**: feed a list of inputs, assert the list of
  outputs. Captured device frames become fixtures directly.
- **Deterministic**: the same inputs at the same `now` give the same outputs.
- **Nothing here is asynchronous.** A host that is async, threaded or blocking all drive
  it the same way.

Pure helpers a driver may use (no I/O): frame parse and build for the protocols in
question, checksums, hex, and the model's value conversions.

## 3. Roles

A role says "this property is the standard X", so a consumer that understands it can
build a native control instead of a generic one. The vocabulary is **open**:

- Names are neutral (`target_humidity`, not a Home Assistant or Matter name). Each
  consumer keeps its own table from role to its native concept.
- A property never *needs* a role to be published or controlled.
- A role is added to the registry only with a stated meaning, type and unit.

Registry (grows with each device):

| role | type | meaning |
|---|---|---|
| `available` | binary | device reachable |
| `on` | binary | main power |
| `mode` | select | operating mode |
| `fan_speed` | select | fan speed / strength; the options are the device's own |
| `target_humidity` | number | humidity setpoint, percent |
| `current_humidity` | number | measured relative humidity, percent |
| `current_temperature` | number | measured room temperature, °C |
| `target_temperature` | number | temperature setpoint, °C |
| `swing_vertical` | binary | vertical air-flow sweep on / off |
| `swing_horizontal` | binary | horizontal air-flow sweep on / off |
| `action` | select | what a climate device is doing now; the options are the device's own (`off`, `idle`, `cooling`, …), read only |
| `brightness` | number | light level, percent, up to `100`. A device with a lowest usable level says so in `min` (Tuya's 10 of 1000 is `min: 1`); the producer does the scaling |
| `color_temperature` | number | white colour temperature in kelvin (`unit: "K"`), with the device's own `min`/`max`; warmer is lower. The producer converts from mireds or a device scale |
| `color` | text | colour as `#rrggbb` at full brightness (brightness is its own property, so hue and saturation survive a dimmer). One property, so a colour change is one atomic write |
| `color_mode` | select | which of the light's modes is showing: `white` (`color_temperature`) or `color` (`color`); other tokens (`scene`, `music`) are the device's own. A consumer only reads it: writing `color` or `color_temperature` switches the mode |
| `position` | number | how far a cover is open, percent, `100` fully open, `0` fully closed. A device whose wire counts the other way is inverted by the producer |
| `tilt` | number | slat angle of a cover, percent, `0`..`100`, `100` fully open |
| `motion` | select | what a cover is doing now: `opening`, `closing`, `stopped`; read only |
| `open` `close` `stop` | trigger | move a cover fully open, fully closed, or halt it |
| `locked` | binary | a lock's bolt: `true` locked, `false` unlocked. `rw` only if the producer allows remote locking (see below) |
| `unlatch` | trigger | release a lock's latch without leaving it unlocked (open the door) |
| `opened` | binary | a valve: `true` open, `false` closed. Read-only unless the owner allows remote operation, as for `locked` |

#### Lights, covers, locks, valves and sirens

Each is a device `kind` (`light`, `cover`, `lock`, `valve`, `siren`; a siren is `on` and nothing more) whose roles are used together; `class`
refines it (`curtain`, `blind`, `garage_door`, `gate`, `door_lock`). The vocabulary was
taken from what Tuya's standard instruction set (`switch_led`, `bright_value`, `temp_value`,
`colour_data`, `control`, `percent_control`, `lock_motor_state`), Matter (`OnOff`,
`LevelControl`, `ColorControl`, `WindowCovering`, `DoorLock`) and Home Assistant all share.
Rules that keep the model small:

- **A light is `on` plus whichever of `brightness`, `color_temperature`, `color` it has.**
  A consumer derives the supported modes from which roles are present: none is on/off only.
  Effects and scenes are an ordinary `select` without a role until two devices agree.
- **Positions are normalised by the producer** to percent with `100` open, whatever the
  wire, its `control_back_mode` or the consumer (Matter counts `0` as open) does.
- **A cover with no `position` is fine**: `open`, `close` and `stop` alone. A cover with
  `position` and no `open`/`close` is opened by writing `100` / `0`.
- **Locks are the dangerous case.** A producer publishes `locked` read only unless the
  owner has asked for remote control; unlocking is never implied by a role's presence.
  Fault and door-open state are properties with `class: "problem"` / `"door"`, not roles.
  A `requires` may tie `locked` to a property the device reports when remote operation is
  allowed.

#### More than one composite on a device

A device has one `kind`, which is the composite made of its properties **outside any kinded
group**. A device that is also something else (a curtain motor with a light, a fan with a
light, a humidifier with a fan, two lights on one relay board) declares each extra composite as
a group with a `kind`, and puts that composite's properties in the group:

```json
"kind": "cover",
"groups": { "light": { "kind": "light", "label": "Light" } },
"props": {
  "position":   { "type": "number", "rw": true, "role": "position", "unit": "%", "min": 0, "max": 100 },
  "switch_led": { "type": "binary", "rw": true, "role": "on",         "group": "light" },
  "brightness": { "type": "number", "rw": true, "role": "brightness", "group": "light", "unit": "%", "min": 1, "max": 100 }
}
```

- Roles are looked up **within the composite's own properties**, so two lights each have their
  own `on`. A role is not shared across composites.
- A consumer names the group's composite after the group's `label` (its own name for the
  device's main composite is the device's).
- A descriptor without `groups` means exactly what it meant before; a consumer that does not
  know `groups` ignores it and builds the main composite from every property, as it did.
- A kinded group whose properties do not form that kind's composite (a light without `on`)
  is not one: its properties are plain properties.

How a consumer maps roles to its own concepts is outside the IL; a starting point is in
[il-consumers.md](il-consumers.md).

### Classes

A property `class` is a hint like a role, but says nothing about function, only about the
kind of value, so any property can carry one and a consumer may ignore it. It is the
difference between a number that is a temperature and one that is merely `°C`, and between a
`binary` that is a fault and one that is a lock. The vocabulary is open and grows the same way
as roles: a class is written down once a device needs it.

Units are plain strings. Producers write micro as `μ` (U+03BC), which is what the sensor
ecosystems use for `μg/m³`; a consumer that compares units should treat U+00B5 and U+03BC
as the same.

## 4. Mapping from other property models

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

## 5. Worked example: DHUM_056905_WW (LG, via rusthinq)

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
- **Proposed here, since adopted:** the "dispensed today" numbers reset every day, which a
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
  fit a condition on a mode. Proposal, not adopted: a `requires` that can name a
  `select` value.
- **Two properties from one tag.** `filter_remaining` and `filter_used` are computed from
  the same pair of tags (hours left, rated life).

## 6. Evolving the IL

- **Extras first, roles later.** A device-specific value starts as a property without a
  role. When a second device, in any project, has the same thing, promote it: add a role
  with meaning, type and unit, and tag both. No property name has to change.
- **Gaps are proposed, not patched around.** Anything a device needed that the model
  cannot say is written down here as a proposed change with the device that raised it.
- **The spec follows the drivers.** A declarative driver definition is deferred until two
  devices show which operations actually repeat.
- **Found while writing the first driver (proposed, not yet in the model):**
  - A `select` can hold a value the device reports but the options do not list (this
    appliance sends mode 22, an ionizer mode the app does not expose). For now a driver
    leaves such a value unpublished. Proposal: a consumer tolerates an unlisted `select`
    value, and a driver may choose to list it.
  - `Reject` needs a place to go, so `x-mqtt` gained `reject`.
  - A driver publishes a value only once it has one; whether `available` should be
    published `false` at connect until the first values arrive is what the first driver
    does, and is the intended reading of the role.
- **Reserved for later, deliberately absent from v0:** structured values. (One-shot events
  are no longer reserved: they are the `event` type.)

### Changelog

- `0` (draft): property type `event` (an occurrence, the mirror of `trigger`); role `opened`
  (valve); kind `siren` (role `on`); property `class` also applies to a writable binary and a
  trigger, and `category: diagnostic` to a trigger.
- `0` (draft): descriptor field `groups`: a group with a `kind` is a composite of its own, so a
  device can be a cover and a light, or have two lights (13 of 285 Tuya fixture devices did).

- `0` (draft, unreleased addition): roles for lights (`brightness`, `color_temperature`,
  `color`, `color_mode`), covers (`position`, `tilt`, `motion`, `open`, `close`, `stop`) and
  locks (`locked`, `unlatch`), added ahead of the first Tuya producer (rustuya).
- `0` (draft; drivers for all ten appliances of the first installation; roles `fan_speed`, `current_temperature`, `target_temperature`, `swing_vertical`, `swing_horizontal`, `action`, type `trigger` and fields `requires`, `class`, `series`, `category` added along the way): descriptor with typed properties, roles registry, `group`, `src`, `x-`
  extension prefix, availability as a role, sans-IO driver interface. No transport in
  the model.

## 7. This repository

Only specification lives here: this document, the transport mappings, informative notes
for consumers, JSON Schemas under `schema/`, and example descriptors under `examples/`.
There is deliberately no reference implementation and no code in any language. Each
project implements the types and the driver interface in its own language and follows
the schema. Language-neutral conformance vectors (a list of inputs with the list of
outputs a driver must produce) are still to be added under `vectors/`.

## 8. Hosting

How a particular host runs drivers is that host's own documentation, not part of the
IL.

## 9. Open points

- Whether a driver is written as Rhai returning `Output` values, as native Rust, or both
  behind the same interface. The interface allows either.
- Whether `group` is enough for multi-component devices or needs a real nested form. For
  devices with several composites, `groups` with a `kind` is the answer so far (found by the
  Tuya spike, see [notes/tuya-spike-gaps.md](notes/tuya-spike-gaps.md)); a nested form is not
  needed yet.
