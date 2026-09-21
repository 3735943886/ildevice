# IL: a neutral device description (sans-IO)

> **Draft, version 0.** A specification shared by every project that turns a device into
> data (rusthinq, rustuya, and consumers of either). It depends on no project, no
> transport and no programming language. No code lives in this repository.

This document is **normative** except where a section says otherwise. History, worked
examples and design reasoning are in [il-rationale.md](il-rationale.md) (informative).

## 0. Conventions

The words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT** and **MAY** are to be read as in
RFC 2119. Every normative rule has an id (`D-1`, `V-3`, …) so that tests and reviews can point
at it; an id is never reused for a different rule.

| term | meaning |
|---|---|
| device | a physical or logical thing described by one descriptor |
| descriptor | the JSON document that describes a device (section 2) |
| property | one named, typed value (or action) of a device |
| producer | software that emits descriptors and values for devices, and accepts writes |
| driver | the sans-IO state machine inside a producer that speaks to one device (section 8) |
| consumer | software that reads descriptors and values and may write properties |
| host | the software that runs a driver and carries out its outputs |
| composite | a set of properties, found by role, that together form a light, a cover, a climate device, … (section 10) |
| absent | a property has no value right now (section 4) |

JSON here means RFC 8259 JSON, and every string is UTF-8.

## 1. Overview

The IL describes what a device *is*, what values it has, and which of them can be
written, using no vocabulary of Home Assistant, Matter, Tuya, LG or any one consumer.

**The IL is sans-IO.** It is data types plus pure functions. It never opens a socket,
reads a clock, spawns a thread, or knows a topic. Whoever implements it feeds it events and
carries out what it asks for. The IL defines only the **shape of the data** (descriptor,
messages, driver inputs and outputs). How it travels (a function call, UDP, HTTP, a file, MQTT) is a
transport mapping and adds nothing to the model: the messages are in
[il-messages.md](il-messages.md), and [il-mqtt.md](il-mqtt.md) is one mapping.

```
                 ┌──────────── IL (this document): data + pure logic ─────────────┐
 host events ──► │ Driver::handle(Input) ──► Vec<Output>          Descriptor,     │ ──► host effects
 (frame, time,   │ (a state machine, no I/O)                      Value, Role     │    (send, timer,
  command)       └────────────────────────────────────────────────────────────────┘     publish)
```

Design goals, in priority order (informative):

1. No I/O in the IL: every effect is a returned value, every input an argument.
2. Producer and consumer never need to know each other: everything a consumer needs to use a
   device is in the descriptor.
3. Additive growth: new devices add properties, roles and fields without breaking anyone.
4. Small: six value types, one descriptor, no per-device-kind schemas.

## 2. Descriptor

A device is a set of **typed properties**. A property with a standard meaning carries a
`role`; one without is a property without a role, and is fully usable.

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
    "humidity": { "type": "number", "role": "current_humidity", "unit": "%" }
  }
}
```

(The example is abbreviated; a complete one is [examples/dhum_056905_ww.json](examples/dhum_056905_ww.json).)

| field | required | meaning |
|---|---|---|
| `il` | yes | IL version the descriptor was written against (non-negative integer). |
| `id` | yes | Globally stable device id, a non-empty string. |
| `props` | yes | Property name → definition (section 3). |
| `source` | no | Name of the producer (`rusthinq`, `rustuya`, …). Free string. |
| `kind` | no | Coarse category, an open string; the kinds with a composite are in section 10. |
| `class` | no | Refinement of `kind`. Not the same as a property's `class`. |
| `label` `vendor` `model` | no | Human-facing identity. `label` is the name a person calls the device: a producer that knows the owner's own name for it puts that here and falls back to a generic one ("Dehumidifier") only when it has none. |
| `identifiers` | no | Map of other ids (MAC, cloud id, Tuya device id) for matching with other systems. Values are strings. |
| `groups` | no | Map from a `group` name to `{ "kind", "class", "label" }` (section 10). |

- **D-1** `id` MUST be the same for the same device across restarts and reconfiguration.
- **D-2** Property names MUST match `[a-z0-9_]+`.
- **D-3** A descriptor MUST NOT contain a topic, URL or network address of the transport (a
  mapping's private `x-` block is the only place for one), and MUST NOT contain a secret
  (a local key, token, password or session id).
- **D-4** `identifiers` SHOULD hold only what a consumer needs to match the device with another
  system; a producer SHOULD NOT copy personal data (an owner's name, address, account id) into
  any field.
- **D-5** A producer MUST emit a new `Descriptor` whenever any part of the descriptor changes.

## 3. Property definition

| field | applies to | meaning |
|---|---|---|
| `type` | all (required) | `binary`, `number`, `select`, `text`, `trigger`, `event`. |
| `rw` | all | `true` if the property accepts writes. Default `false`. |
| `role` | all | Standard meaning, section 9. |
| `requires` | all | A condition on another property of the device that must currently hold for this property to accept writes. A string names a `binary` property that must be `true` (a control that only works while the panel has granted remote start). An object `{ "prop": "mode", "in": ["cool"] }` names a `select` property whose current value must be one of the listed options (a setting that only counts while cooling). |
| `group` | all | Names the sub-unit or composite the property belongs to (section 10). |
| `unit` | number | Unit label (section 11.2). |
| `min` `max` `step` | number | Range (both inclusive) and increment. |
| `options` | select, event | Allowed values (for an `event`, the kinds of occurrence), in a stable order. |
| `label` | all | Human-readable name. |
| `class` | all | What the value is (section 11.1). Not the same as the descriptor's `class`. |
| `series` | number | `gauge` (default): the value goes up and down. `counter`: a running total that only grows and starts again from zero (energy of a cycle, water dispensed today). |
| `category` | all | `diagnostic` or `config` (section 11.3). Absent means an ordinary property. |
| `src` | all | Opaque origin hint (a protocol tag, a Tuya dp id). |

- **P-1** `select` and `event` properties MUST have a non-empty `options` list of distinct strings.
- **P-2** `options` MUST NOT be given on other types, and `min` `max` `step` `unit` `series` MUST NOT
  be given on other types than `number`. (A consumer ignores them there.)
- **P-3** A `trigger` is always writable: `rw` MAY be omitted and MUST NOT be `false`. An `event`
  MUST NOT be `rw`.
- **P-4** If both are present, `min` MUST be less than or equal to `max`, and `step` MUST be greater
  than zero.
- **P-5** `requires` MUST refer to a property of the same descriptor other than itself: the string form to a `binary` property, the object form to a `select` property with `in` a non-empty list of that property's `options`.
- **P-6** A consumer MUST NOT interpret `src`.

## 4. Values

| type | value |
|---|---|
| `binary` | boolean |
| `number` | a finite number **already in the declared `unit`**. Scaling and unit conversion are the producer's job. |
| `select` | one of `options`, a string |
| `text` | a non-empty string |
| `trigger` | never a value. Writing to it (any payload) performs the action. |
| `event` | never a value: the mirror of a `trigger`. Each time the device does the thing (a button pressed, a door bell rung) the producer emits **one occurrence**, one of `options`. The occurrence *is* the time it happened; there is no timestamp, no state to recover, and the same kind twice in a row is two occurrences. |

- **V-1** A property that the device is not currently reporting is *absent*. A producer MUST NOT
  publish a placeholder (`0`, `""`, `null`, `unknown`) for it and MUST emit `Absent` when a value
  it published stops being known.
- **V-2** A `text` value MUST NOT be the empty string; an empty text is absent.
- **V-3** A producer MUST NOT emit a `select` value that is not in `options`. When a device reports
  a value the options do not list, the producer either adds it to `options` (and emits a new
  descriptor) or treats the property as absent. A consumer that receives an unlisted value anyway
  MUST tolerate it (shows it as it is) and MUST NOT fail.
- **V-4** A writable `select` MAY list fewer options than a read-only sibling that reports the
  same underlying thing, when some values can be reported but not selected.
- **V-5** A `number` whose value is integral MUST be written without a fraction when the transport
  has a distinction (`5`, not `5.0`).

## 5. Commands

A consumer writes a property with a value of the forms in section 4. A producer MUST check
every write against its own descriptor **before** acting on it, in this order, and a write that
fails a step MUST be answered with a `Reject` carrying that step's code (below); nothing is
sent to the device for it.

| # | check | code |
|---|---|---|
| 1 | the property exists | `unknown_property` |
| 2 | the property is `rw` (a `trigger` always is; an `event` never is) | `read_only` |
| 3 | if it has `requires`, the condition holds now (the named binary is `true`; the named select's value is in `in`); a value the producer has not reported counts as not satisfied | `requires_unmet` |
| 4 | the payload is of the property's type: `binary` `true`/`false` (also `on`/`off`/`1`/`0`, normalised to `true`/`false`); `number` a finite number; `select` one of `options`; `text` a non-empty string; `trigger` any payload | `invalid_value` |
| 5 | `number` within `min`..`max`, inclusive | `out_of_range` |
| 6 | `number` a multiple of `step` counted from `min` (from 0 if there is no `min`) | `bad_step` |

- **C-1** A value outside a range or off a step MUST be refused, never clamped or rounded:
  silently writing a different value than was asked for is worse than saying no.
- **C-2** The driver that acts on a valid write MUST receive it in canonical form (`true`/`false`,
  an integer written as an integer).
- **C-3** Step 6 is exact in decimal arithmetic. An implementation on binary floating point MUST
  treat a value within 1e-9 of a step as on the step.
- **C-4** Further codes: `unavailable` (the device link is down and the driver cannot queue the
  write), `refused` (the device answered that it will not do it), `unsupported` (the driver
  deliberately does not offer the action). A producer MAY use these after step 6. A consumer that
  meets a code it does not know MUST treat it as a generic failure.
- **C-5** `Reject.reason` is free human-readable text; a consumer MUST NOT parse it.

## 6. Availability

- **A-1** Availability is a property with the role `available` (binary, read only).
- **A-2** A producer that has the property MUST publish `false` when the device link is not up and
  from a fresh start until the driver has the device's first complete state, and `true` while the
  link is up and values are current.
- **A-3** A producer with no notion of availability omits the property. A consumer then assumes the
  device is available while values keep arriving.

## 7. Compatibility and versioning

- **X-1** A consumer MUST ignore an unknown field, role, class, unit, kind, `category` value, group
  kind or `Reject` code, and treat what it belongs to generically.
- **X-2** A property with an unknown `type` MUST be skipped, not treated as an error.
- **X-3** Fields private to one project MUST be prefixed `x-` (`x-tuya`) so that they cannot clash
  with a later standard field. Transport mappings use the same mechanism.
- **X-4** `il` is bumped only for a **breaking** change: removing or renaming a field, role or type;
  changing the type, unit or meaning of a role; making something optional required; narrowing
  what a value may be. Adding a field, type, role, class, unit, kind, category or code is not
  breaking.
- **X-5** A consumer that sees a higher `il` than it knows SHOULD still try, and MUST refuse only if
  it cannot honour the fields it needs.
- **X-6** A name in this document is not removed from a registry once assigned; it may be marked
  deprecated in [il-rationale.md](il-rationale.md) and keeps its meaning.

## 8. The driver interface (sans-IO)

A producer is built on a **driver**: a state machine that turns what happened into what should be
done, defined as data in, data out:

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
| `Command { prop, value }` | a consumer asked to write a property (already checked, section 5) |
| `Timer(name)` | a timer the driver set has fired |

**Outputs**

| output | meaning |
|---|---|
| `Descriptor(desc)` | the device's description |
| `Value { prop, value }` | a property now has this value |
| `Absent { prop }` | a property no longer has a value |
| `Event { prop, kind }` | an `event` property happened once |
| `SendFrame(bytes)` / `SendMessage { channel, json }` | write to the device |
| `SetTimer { name, after }` / `CancelTimer(name)` | schedule or cancel a wake-up |
| `Reject { prop, code, reason }` | a command could not be honoured (`code` from section 5) |

- **R-1** A driver MUST be pure: it MUST NOT perform I/O, read a clock, or use randomness the host
  did not hand in. The same state, `now` and input MUST give the same outputs.
- **R-2** A driver MUST emit `Descriptor` on `Connected` if it has not emitted one since the host
  started it, and whenever the descriptor changes (D-5). A host that keeps the last descriptor MAY
  replay it.
- **R-3** A driver MUST emit `Descriptor` before the first `Value`, `Absent` or `Event` for a
  property that the descriptor introduces.
- **R-4** A driver MUST emit `Absent` for every property whose value it can no longer vouch for
  when it receives `Disconnected`, except `available`, which it sets `false` (A-2).
- **R-5** `Value` MUST NOT be emitted for a `trigger` or an `event`; `Event` MUST be emitted only for
  an `event`.
- **R-6** A timer name is unique per driver; `SetTimer` with an existing name replaces it.
  Handshake retries, periodic refresh and a debounced re-query after a command are all timers.
- **R-7** A driver MAY refuse to do a thing at all: an action the device does not truly support is
  not offered as a property, or is answered `unsupported`.

The messages a producer and a consumer exchange are defined in [il-messages.md](il-messages.md).

Nothing here is asynchronous: a host that is async, threaded or blocking all drive it the same
way. A driver is testable with no environment: feed a list of inputs, assert the list of outputs.
Pure helpers a driver may use (no I/O): frame parse and build, checksums, hex, the model's value
conversions.

## 9. Roles

A role says "this property is the standard X", so a consumer that understands it can build a native
control instead of a generic one.

- **O-1** Role names are neutral (`target_humidity`, not a Home Assistant or Matter name).
- **O-2** A property MUST NOT need a role to be published or controlled.
- **O-3** A role is added to the registry only with a stated meaning, type, unit and access.
- **O-4** A role MUST appear on at most one property within a composite (section 10).
- **O-5** A property that carries a role MUST have the type the registry gives it. A consumer MUST
  ignore the role of a property that does not.

Access: `ro` read only; `rw` writable, and a producer MUST NOT declare it read only;
`rw?` writable when the producer and the owner allow it (section 12).

| role | type | access | unit / values | meaning |
|---|---|---|---|---|
| `available` | binary | ro | | device reachable |
| `on` | binary | rw | | main power |
| `mode` | select | rw | device's own | operating mode |
| `fan_speed` | select | rw | device's own | fan speed or strength |
| `target_humidity` | number | rw | `%` | humidity setpoint |
| `current_humidity` | number | ro | `%` | measured relative humidity |
| `current_temperature` | number | ro | `°C` | measured room temperature |
| `target_temperature` | number | rw | `°C` | temperature setpoint |
| `swing_vertical` | binary | rw | | vertical air-flow sweep on / off |
| `swing_horizontal` | binary | rw | | horizontal air-flow sweep on / off |
| `action` | select | ro | device's own (`off`, `idle`, `cooling`, …) | what a climate device is doing now |
| `brightness` | number | rw | `%`, up to `100` | light level. A device with a lowest usable level says so in `min` (Tuya's 10 of 1000 is `min: 1`); the producer scales |
| `color_temperature` | number | rw | `K`, device's own `min`/`max` | white colour temperature; warmer is lower. The producer converts from mireds or a device scale |
| `color` | text | rw | `#rrggbb` | colour at full brightness (brightness is its own property, so hue and saturation survive a dimmer). One property, so a colour change is one atomic write |
| `color_mode` | select | ro | `white`, `color`, others the device's own (`scene`, `music`) | which mode of the light is showing. Writing `color` or `color_temperature` switches it |
| `position` | number | rw? | `%`, `0`..`100` | how far a cover is open, `100` fully open, `0` fully closed. A device whose wire counts the other way is inverted by the producer |
| `tilt` | number | rw? | `%`, `0`..`100` | slat angle of a cover, `100` fully open |
| `motion` | select | ro | `opening`, `closing`, `stopped` | what a cover is doing now |
| `open` `close` `stop` | trigger | rw | | move a cover fully open, fully closed, or halt it |
| `locked` | binary | rw? | | a lock's bolt: `true` locked, `false` unlocked |
| `unlatch` | trigger | rw? | | release a lock's latch without leaving it unlocked (open the door) |
| `opened` | binary | rw? | | a valve: `true` open, `false` closed |
| `alarm_state` | select | ro | `disarmed`, `armed_home`, `armed_away`, `armed_night`, `arming`, `pending`, `triggered` | state of an alarm panel |
| `arm_home` `arm_away` `arm_night` | trigger | rw? | | arm the panel in that mode |
| `disarm` | trigger | rw? | | disarm the panel |
| `vacuum_state` | select | ro | `cleaning`, `docked`, `paused`, `returning`, `idle`, `error` | what a robot vacuum is doing |
| `start` `pause` `return_home` `locate` | trigger | rw | | start or resume cleaning, pause, send to the dock, make it announce itself |
| `battery` | number | ro | `%` | battery charge of the device |

- **O-6** `available`, `current_*`, `action`, `color_mode`, `motion`, `alarm_state`, `vacuum_state` and `battery` are read only, and a producer
  MUST NOT declare them `rw`.
- **O-7** Positions are normalised by the producer to percent with `100` open, whatever the wire,
  its `control_back_mode` or the consumer (Matter counts `0` as open) does.
- **O-8** A `select` role whose registry values are listed (`motion`, `color_mode`, `alarm_state`, `vacuum_state`) MUST use those
  tokens for what they name; the device's own tokens are additions.

## 10. Kinds and composites

A **composite** is the set of a device's properties, found by role, that forms one native thing.
A device's `kind` says which composite its properties outside any kinded group form. The table gives
what each composite REQUIRES and what it MAY add; a kind not in the table has no composite and its
properties are only plain properties.

| kind | required roles | optional roles |
|---|---|---|
| `light` | `on` | `brightness`, `color_temperature`, `color`, `color_mode` |
| `cover` | `position`, or `open` and `close` | `tilt`, `motion`, `stop`, `open`, `close`, `position` |
| `lock` | `locked` | `unlatch` |
| `valve` | `opened` | |
| `siren` | `on` | |
| `switch` | `on` | |
| `climate` | `target_temperature` | `on`, `mode`, `fan_speed`, `current_temperature`, `current_humidity`, `swing_vertical`, `swing_horizontal`, `action` |
| `humidifier` | `on`, `target_humidity` | `mode`, `fan_speed`, `current_humidity`, `current_temperature` |
| `fan` | `on` | `mode`, `fan_speed` |
| `alarm` | `alarm_state` | `arm_home`, `arm_away`, `arm_night`, `disarm` |
| `vacuum` | `vacuum_state` | `start`, `pause`, `return_home`, `locate`, `fan_speed`, `battery` |

`class` refines a kind (`curtain`, `blind`, `garage_door`, `gate`, `door_lock`, `air_conditioner`,
`dehumidifier`, `air_purifier`); it changes no requirement. Other kinds seen so far
(`dispenser`, `washer`, `dryer`, `clothing_care`, `cooktop`, `camera`) have no composite. A camera's motion and recording switches are plain properties; its stream is outside the IL.

- **K-1** A light is `on` plus whichever of `brightness`, `color_temperature`, `color` it has; a
  consumer derives the supported modes from which roles are present, and none is on/off only.
  Effects and scenes are an ordinary `select` without a role until two devices agree.
- **K-2** A cover with no `position` is fine: `open`, `close` and `stop` alone. A cover with
  `position` and no `open`/`close` is opened by writing `100` and closed by writing `0`.
- **K-3** Roles are looked up **within the composite's own properties**; a role is not shared across
  composites.
- **K-4** A set of properties that lacks a required role does not form its composite: they are plain
  properties. This is not an error.
- **K-5** Power and mode are separate: a device that reports a mode while off carries the wire's mode
  in `mode` and whether it runs in `on`. A consumer that has a single "off" mode derives it.

### Groups

The descriptor's `groups` map names a `group` used by properties. A group without a `kind` is only a
label for the sub-unit (`ch1`, `right`, `zone_a`). A group with a `kind` is a composite of its own:

```json
"kind": "cover",
"groups": { "light": { "kind": "light", "label": "Light" } },
"props": {
  "position":   { "type": "number", "rw": true, "role": "position", "unit": "%", "min": 0, "max": 100 },
  "switch_led": { "type": "binary", "rw": true, "role": "on",         "group": "light" },
  "brightness": { "type": "number", "rw": true, "role": "brightness", "group": "light", "unit": "%", "min": 1, "max": 100 }
}
```

- **G-1** The device's own `kind` composite is made of the properties that are in no kinded group.
- **G-2** Two kinded groups MAY have the same `kind` (two lights); each is looked up separately (K-3).
- **G-3** A consumer names a group's composite after the group's `label`; its name for the device's
  main composite is the device's.
- **G-4** A descriptor without `groups` has the meaning it had before `groups` existed, and a
  consumer that does not know `groups` builds the main composite from every property.
- **G-5** A `group` that `groups` does not list is a plain label.

How a consumer maps roles to its own concepts is outside the IL; a starting point is in
[il-consumers.md](il-consumers.md).

## 11. Classes, units and categories

### 11.1 Property classes

A property `class` says nothing about function, only about the kind of value, so any property can
carry one and a consumer MAY ignore it. It is the difference between a number that is a temperature
and one that is merely `°C`, and between a `binary` that is a fault and one that is a lock.

- **L-1** A consumer MUST ignore a class it does not know.
- **L-2** A class is added to the registry once a device needs it, with the type it applies to and a
  meaning.

| applies to | class | meaning |
|---|---|---|
| number | `temperature` | a temperature |
| number | `humidity` | relative humidity |
| number | `duration` | a length of time |
| number | `energy` | energy |
| number | `power` | power |
| number | `volume` | volume of liquid or gas |
| number | `pm1` `pm25` `pm10` | particulate concentration |
| binary (read only) | `problem` | true means there is a fault |
| binary (read only) | `running` | true means it is running |
| binary (read only) | `heat` | true means it is hot |
| binary (read only) | `door` | true means open |
| binary (writable) | `outlet` `switch` | the kind of load it switches |
| text | `datetime` | an instant as RFC 3339 with an offset (`2026-09-21T14:30:00+09:00`) |
| trigger | `restart` `identify` `update` | what the action does |

### 11.2 Units

- **U-1** A unit is a plain string. A producer SHOULD use the spelling below when the quantity is one
  of these, and MUST NOT put a conversion into the string.
- **U-2** Micro is written `μ` (U+03BC). A consumer that compares units MUST treat U+00B5 and U+03BC
  as the same.

`%` `°C` `K` `W` `Wh` `kWh` `V` `A` `Hz` `s` `min` `h` `mL` `L` `m³` `μg/m³` `ppm` `lx` `Pa` `hPa`
`m/s` `km/h` `dB`

### 11.3 Category

- **T-1** `diagnostic`: read only, or a `trigger`, of interest when troubleshooting (fault codes, locks,
  counters, a factory reset). `config`: a setting rather than an everyday control.
- **T-2** A consumer MAY hide or group by category; nothing else changes.

## 12. Safety

- **S-1** A producer MUST publish `locked`, `opened`, `unlatch`, `disarm` and any control that starts heating or
  motion of a hazardous device (a cooktop ring, a garage door) read only or not at all, unless the
  owner has asked for remote control. Unlocking or opening is never implied by a role's presence.
- **S-2** A producer SHOULD tie such a control to a property the device reports when remote operation
  is allowed, using `requires`.
- **S-3** A `locked` property that is not `rw` is a read-only state, not a lock; a consumer MUST NOT
  offer a lock it cannot operate.
- **S-4** Fault and door-open state are properties with `class: "problem"` / `"door"`, not roles.
- **S-5** Authentication and authorisation of writes belong to the transport and the deployment; a
  producer that receives a write it is not allowed to accept from that sender SHOULD answer with
  `refused`.

## 13. Conformance

An implementation conforms as one or more of these; each is a list of the rules above.

- **Descriptor**: valid against [schema/descriptor.schema.json](schema/descriptor.schema.json) and
  D-1..D-4, P-1..P-5, O-5, O-6, G-5.
- **Producer**: emits conforming descriptors (D-5); V-1..V-3, V-5; checks every write as in section
  5 (C-1..C-4); A-1..A-3; S-1..S-3.
- **Driver**: R-1..R-7, and section 5 for the writes it receives; passes the vectors in
  [vectors/](vectors/).
- **Consumer**: X-1..X-5, V-3 (tolerance), P-6, L-1, U-2, K-3, K-4, G-3..G-4, S-3.
- **Messages**: [il-messages.md](il-messages.md) W-1..W-13, checked against
  [schema/message.schema.json](schema/message.schema.json). A producer also produces snapshots (W-7).
- **Transport mapping**: says how each message of il-messages.md section 1 is carried and how W-7..W-12
  are met (absence, descriptor removal, replay of an event, ordering); it defines no field of the model
  ([il-mqtt.md](il-mqtt.md) is one).

Producers, consumers and mappings claim conformance to an IL version (`il`) and to this document's
date or revision.
