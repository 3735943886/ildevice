# IL over MQTT

> **Draft, version 0.** A transport mapping for the [IL](il.md). It is a separate layer:
> the IL model and driver interface do not depend on this document, and another mapping
> (HTTP, files) can be written the same way. This document is specification only.
> Keywords (MUST, SHOULD, …) are as in [il.md](il.md) section 0; rules have ids `M-n`.

This document says only how IL data is carried on MQTT (3.1.1 or 5). It adds nothing to the model.

## 1. Topics

| what | topic | retained | payload |
|---|---|---|---|
| descriptor | `<il_prefix>/<id>` | yes | the descriptor as JSON; an **empty** payload removes the device |
| property value | see "Locating values" | yes | the value, encoded as in section 3 |
| event occurrence | see "Locating values" | **no** | one of the property's `options` |
| property write | see "Locating values" | no | the same forms as a value |
| reject | see "Locating values" | no | `{ "prop", "code", "reason" }` as JSON |

- **M-1** `il_prefix` defaults to `il` and is the one thing a consumer must be configured with.
  A consumer subscribes to `<il_prefix>/+` and discovers every device.
- **M-2** With the default topic layout, a device `id` MUST NOT contain `/`, `+`, `#` or NUL, and
  MUST NOT be empty. (A producer whose ids can may use `x-mqtt` locations that avoid them.)
- **M-3** A consumer MUST ignore a retained message on an `event` topic: it is an old occurrence the
  broker replays, not a new one.
- **M-4** A `trigger` has no state topic (it is never published); only its `set` topic exists. An
  `event` has no `set` topic (it is never written).

## 2. Locating values

A producer often already has a topic layout (rusthinq: `<prefix>/<id>/<prop>`; rustuya:
user-configured templates), and should not have to change it. The descriptor may carry
the private extension `x-mqtt` to say where its values are:

```json
"x-mqtt": {
  "state": "rusthinq/{id}/{prop}",
  "set":   "rusthinq/{id}/{prop}/set",
  "reject": "rusthinq/{id}/reject"
}
```

- **M-5** The driver never writes this block: it does not know topics. The MQTT layer of the host
  adds `x-mqtt` to the descriptor when it publishes it.
- **M-6** `{id}` is the descriptor's `id` and `{prop}` the property name. A property MAY carry its own
  `x-mqtt` with `state` / `set` to override. A consumer that does not speak MQTT ignores `x-mqtt`.
- **M-7** If `x-mqtt` is absent the defaults are `<il_prefix>/<id>/<prop>`,
  `<il_prefix>/<id>/<prop>/set` and `<il_prefix>/<id>/reject`.

## 3. Payloads and delivery

| type | payload (UTF-8, no JSON quoting of strings) |
|---|---|
| `binary` | `true` or `false` (a write may also send `on` / `off` / `1` / `0`) |
| `number` | a decimal number in JSON number syntax; an integral value has no fraction |
| `select`, `text` | the string as it is, not wrapped in quotes |
| `event` | one of the options, as for `select` |
| `trigger` (write) | any payload, including an empty one |

- **M-8** A descriptor, a value and a removal MUST be published with QoS 1 or 2, and a value or
  descriptor with the retain flag. An event occurrence and a reject MUST NOT be retained.
- **M-9** A write SHOULD be sent with QoS 1. A consumer MUST NOT send a write with the retain flag,
  and a producer MUST ignore a retained message on a `set` topic (it would replay an old command).
- **M-10** An empty retained payload on a state topic means *absent* (il.md V-1, V-2); it cannot mean
  the empty string.
- **M-11** A producer that removes a device MUST first clear (empty retained payload) the retained
  values of its properties, then the descriptor. A producer that restarts and finds retained values
  of a property it no longer has SHOULD clear them.

## 4. Mapping the driver interface

| IL output | MQTT |
|---|---|
| `Descriptor` | publish (retained) the descriptor |
| `Value { prop, value }` | publish (retained) the value on the state topic |
| `Absent { prop }` | publish an empty retained payload on the state topic |
| `Event { prop, kind }` | publish `kind` on the state topic, **not retained**. Every message is one occurrence, including a repeat of the same kind |
| `Reject { prop, code, reason }` | publish `{ "prop", "code", "reason" }` (not retained) on the `reject` topic |

| MQTT | IL input |
|---|---|
| a message on a `set` topic | `Command { prop, value }`, after the checks of il.md section 5 |
| broker connection events | handled by the host; the driver sees `Connected` / `Disconnected` for the *device* link only |

Timers and device I/O are not MQTT's concern and never appear here.

## 5. Not covered

A presence signal for a producer as a whole (a Last Will covers one connection, not each device
on it), payload size limits, and authentication (il.md S-5: the broker's ACLs and the deployment
decide who may write a `set` topic). See [il-rationale.md](il-rationale.md) section 6.
