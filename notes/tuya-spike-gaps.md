# Tuya spike: what the IL and il_ha cannot yet express

Informative, not part of the specification. Result of a throwaway spike (2026-09-21):
tuya2ha v2's `classify()` output for the 285 fixture devices of Home Assistant core's `tuya`
integration was turned into IL descriptors, planned with il_ha's model-blind `plan_entities`,
and compared with core's own entity snapshots (1205 entities, 17 platforms). The script is
`rustuya-homeassistant/scripts/spike_tuya2il.py` (uncommitted).

Scope of the comparison, so the numbers are not over-read: **static classification only**
(platform, key, device class, entity category, unit, state class, options, min/max/step, entity
count). Not compared: values (read conversion), commands (write), availability, entity names
(core uses translation keys), and the `climate`, `humidifier` and `fan` platforms (42 entities),
whose descriptors the spike did not attempt.

## The three categories chosen for the spike (`kg`, `dj`, `cl`: 59 entities)

Reproduced except four differences, all in the list below (`switch` device class ×3, one device
with two lights).

## Gaps over all categories

| # | gap | size | kind | note |
|---|---|---|---|---|
| 1 | **PARTLY RESOLVED: `valve` (role `opened`), `siren` (role `on`) and `event` (new property type `event`) done. Still open: `camera` 9, `alarm_control_panel` 2, `vacuum` 2 (13 entities).** Was: platforms with no IL role or il_ha platform: `valve` 14, `event` 14, `camera` 9, `siren` 5, `alarm_control_panel` 2, `vacuum` 2 | 46 entities | IL + il_ha | Needs roles or kinds first. `event` is not a property value (it is an occurrence); `valve`, `siren`, `alarm_control_panel`, `vacuum` are composites like `cover`. |
| 2 | **RESOLVED (draft): `groups` with a `kind`.** Was: `kind` is singular, one composite per kind: a device needing two composites, or two of the same | 13 devices | IL | `clkg` cover + light (3), `cs` fan + humidifier (6), `fs` fan + light, `kt` climate + light; `dj_hpc8ddyfv85haxa7` has two lights (`switch_led`, `switch_1`). The second composite is silently dropped or degrades to plain properties. Needs a per-property or per-group way to say which composite a role belongs to (`group` already exists for sub-units). |
| 3 | **RESOLVED (draft, il_ha + `il.md` `class` row).** Was: **`class` on a writable binary is ignored** (`switch` device class `outlet`) | 89 | il_ha | The IL text says a binary `class` means what being true means, so an outlet is a stretch, but core sets it on every `kg`/`cz`/`pc` switch. Either allow `class` on `rw` binary in the spec or leave it to il_ha's rule. |
| 4 | **RESOLVED (`il.md` `category` row, il_ha).** Was: **`category` is dropped for a trigger** (`diagnostic` button: `factory_reset`) | 2 | il_ha | il_ha applies `diagnostic` only to read-only properties, and a trigger is always writable. |
| 5 | **RESOLVED (`il.md` `class` row, il_ha).** Was: **`class` on a trigger** (`restart` button device class) | 3 | il_ha | il_ha gives buttons no device class. |
| 6 | **RESOLVED (il_ha rule).** Was: **A read-only `select` has no `enum` device class** (Home Assistant needs `device_class: enum` plus options) | 33 | il_ha | A consumer rule, derivable: a read-only select is an enumeration sensor. |
| 7 | **Harness only, fixed in the spike.** Was: **Empty unit vs no unit** (`''` in core) | 22 | harness | Not a real difference; core's snapshot writes an empty string. |

## What did not show up as a gap

- Number ranges, units and steps (`min`/`max`/`step`/`unit`) reproduce exactly.
- `series` (`counter`/`gauge`) gives the same state class for every sensor.
- `category: config` on writable properties and `diagnostic` on read-only ones.
- Enumeration options.
- `light` and `cover` for every single-composite device.

## Also seen

- **Light values are not the same numbers**: core exposes brightness 0..255 and hue/saturation;
  the IL says percent and `#rrggbb`. The producer converts, so this is not a gap, but a value
  comparison (not done here) is where any loss would show.
- **Entity names** come from translation keys (`indexed_switch` with an index). The IL has
  `label` only; a producer would have to render the text, and a consumer loses the translations.
- **Composite unique ids**: il_ha uses `<id>-light`; core uses `tuya.<id><dp code>`. il_ha's
  `aliases` option covers it for one composite per device.

## Status after the first change

Gap 2 is done in the draft: `il.md` (descriptor field `groups`, section 10 "Kinds and
composites", subsection Groups), `schema/descriptor.schema.json`, and il_ha's `plan.py` (a kinded group is planned as its
own composite; 4 new tests). Re-running the spike: the light-count differences (`clkg`,
`dj_hpc8ddyfv85haxa7`) and the `kt` stray switch are gone; the remaining differences are gaps 3-7.
Not yet exercised: fan + humidifier / fan + light (the spike does not emit fan or humidifier
descriptors), and the il_ha *entity* classes for a second composite (only planning is covered).

Gaps 3-7 done the same day (il_ha: `class` passes through for a writable binary and a trigger,
`diagnostic` applies to a trigger, a read-only select is an `enum` sensor; 3 new tests, 84 pass).
The one difference the spike still prints over all 1205 entities is `qxj/windspeed_avg`
(`m/s` vs `km/h`): Home Assistant converts to the unit system's display unit, which is the
host's doing and was already known from the v2 golden harness, not an IL gap.

### The `event` type (decided with the owner)

No timestamp field: an occurrence is a non-retained message, and its arrival is the time. Each
message is one occurrence (a repeat of the same kind is another), and a consumer ignores a
retained message on an event topic (a replayed old occurrence). il_ha tests cover exactly those
three rules. A residue: core's doorbell `alarm_message` event also carries the decoded message
text as an attribute (`message`); an IL `event` carries only the kind, so a producer that wants to
keep the text publishes it as a sibling `text` property.

## Suggested order

1. Gap 1, what is left: `camera` (motion/recording switches only; the stream is
   out of scope), `alarm_control_panel`, `vacuum`.
