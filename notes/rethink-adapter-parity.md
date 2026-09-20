# Parity with the rethink adapter's Home Assistant entities

Informative, not part of the specification. A consumer that replaces an existing integration
wants the same entities under the same ids; this records, for the first installation, how the
entities the rethink adapter created in Home Assistant line up with the IL descriptors of the
rusthinq drivers, and what the comparison changed in the IL.

Method: the adapter's entities were read from Home Assistant's entity registry (unique id,
domain, device class, unit, entity category, state class), one device per model, and each was
matched to a descriptor property, by name where it is the same and by the table below where it
is not. The adapter builds its `unique_id` as `<device id>-<suffix>`.

## What the comparison changed in the IL

| gap | resolution |
|---|---|
| Device class (`duration`, `energy`, `power`, `volume`, `humidity`, `temperature`, `problem`, `running`, `heat`, …) has no IL counterpart; a unit alone cannot say it (`%` is a humidity and a filter life) | property field `class` |
| State class (`measurement` / `total_increasing`) | property field `series` (`gauge` / `counter`) |
| Entity category (`diagnostic` / `config`) | property field `category` |
| A climate device's swing axes and current action cannot be found without a role | roles `swing_vertical`, `swing_horizontal`, `action` |
| `μg/m³` written with U+00B5 in the driver, U+03BC in Home Assistant | drivers use U+03BC; the spec says consumers treat them alike |

Deliberately not added: a display precision (the values are integers; how many digits to show is
the consumer's business), icons (presentation), and labels for enumeration values (the option
tokens are stable identifiers; a consumer makes them readable, `wool_knitwear` to "Wool Knitwear").

## Differences that remain (decisions for the consumer / cutover)

- **Composite entities.** The adapter's `climate` (air conditioner) and `humidifier`
  (dehumidifier) entities are built from the properties with the roles `on`, `mode`,
  `fan_speed`, `target_*`, `current_*`, `swing_*`, `action`. Their `unique_id`s (`<id>-climate`,
  `<id>-humidifier`) have no descriptor property; a consumer that wants to keep them names them itself.
- **`off_timer` of the dehumidifier.** The adapter shows hours (0..9, step 1); the IL says minutes
  (0..540, step 60), because the wire is minutes and the IL carries the wire's unit. A consumer that
  keeps the old entity converts.
- **Read-only twins.** For the water purifier's settings the adapter created both a diagnostic
  `binary_sensor` / `sensor` and a writable `switch` / `select` under the same suffix. The IL has one
  property (`rw`); a consumer creates the control, and the diagnostic twin is dropped.
- **Water purifier counters.** The adapter also has `mineral_water_total` and `sparkling_water_total`
  (always 0: this model has no such tap). The driver does not report them.
- **Air purifier.** The adapter's `fan` entity carries power, mode and fan strength; its
  `sterilization` is the IL's `sterilize`.
- **Enumeration option labels** differ in form (`Smart` vs `smart`, `very low` vs `very_low`,
  `long power` vs `long_power`); entity ids and units do not.

## Suffix table

Where the adapter's suffix (the part after the device id in `unique_id`) differs from the IL
property name. All other entities have the same name in both.

| model | adapter suffix | IL property |
|---|---|---|
| DHUM_056905_WW | `bucket_full` | `tank_full` |
| DHUM_056905_WW | `bucket_state` | `tank_state` |
| DHUM_056905_WW | `current_humidity` | `humidity` |
| DHUM_056905_WW | `fan_speed` | `fan` |
| DHUM_056905_WW | `sterilization` | `sterilize` |
| DHUM_056905_WW | `uv_nano` | `uvnano` |
| CST_570004_WW | `autodry_setting` | `auto_dry` |
| CST_570004_WW | `autodryremain` | `auto_dry_remaining` |
| CST_570004_WW | `energy_current` | `power_draw` |
| CST_570004_WW | `energysave` | `energy_save` |
| CST_570004_WW | `sleeptimer` | `sleep_timer` |
| 1WPU4CIGCR__2 | `uvnano` | `tap_uv` |
| 1WPU4CIGCR__2 | `sterilize_schedule` | `self_clean_next` |
| 1WPU4CIGCR__2 | `default_water_amount` | `default_amount` |
| 1WPU4CIGCR__2 | `cold_water_total` | `cold_water_today` |
| 1WPU4CIGCR__2 | `hot_water_total` | `hot_water_today` |
| 1WPU4CIGCR__2 | `normal_water_total` | `normal_water_today` |
| 1WPU4CIGCR__2 | `sterilised_water_total` | `sterilized_water_today` |
| AIR_910604_WW | `sterilization` | `sterilize` |
| S3BF_POD_DN4 | `error-message` | `error_message` |
| RH14_N_KR | `error-message` | `error_message` |
| Pd0F_F | `error-message` | `error_message` |
| F24VDD | `error-message` | `error_message` |
