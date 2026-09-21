# IL over MQTT

> **Draft, version 0.** A transport mapping for the [IL](il.md). It is a separate layer:
> the IL model and driver interface do not depend on this document, and another mapping
> (HTTP, files) can be written the same way. This document is specification only.

This document says only how IL data is carried on MQTT. It adds nothing to the model.

## 1. Topics

| what | topic | retained | payload |
|---|---|---|---|
| descriptor | `<il_prefix>/<id>` | yes | the descriptor as JSON; an **empty** payload removes the device |
| property value | see "Locating values" | yes | the value: `true`/`false`, a JSON number, or a string |
| event occurrence | see "Locating values" | **no** | one of the property's `options`. A consumer ignores a retained message on an `event` topic: it is an old occurrence the broker replays, not a new one |
| property write | see "Locating values" | no | the same forms |

`il_prefix` defaults to `il` and is the one thing a consumer must be configured with.
A consumer subscribes to `<il_prefix>/+` and discovers every device.

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

The driver never writes this block: it does not know topics. The MQTT layer of the host
adds `x-mqtt` to the descriptor when it publishes it.

`{id}` is the descriptor's `id` and `{prop}` the property name. A property may carry its
own `x-mqtt` with `state` / `set` to override. Because it is an `x-` field, a consumer
that does not speak MQTT ignores it, and the model stays free of topics.

If `x-mqtt` is absent the defaults are `<il_prefix>/<id>/<prop>`,
`<il_prefix>/<id>/<prop>/set` and `<il_prefix>/<id>/reject`.

## 3. Mapping the driver interface

| IL output | MQTT |
|---|---|
| `Descriptor` | publish (retained) the descriptor |
| `Value { prop, value }` | publish (retained) the value on the state topic |
| `Absent { prop }` | publish an empty retained payload on the state topic |
| `Event { prop, kind }` | publish `kind` on the state topic, **not retained**. Every message is one occurrence, including a repeat of the same kind |
| `Reject { prop, reason }` | publish `{ "prop", "reason" }` (not retained) on the `reject` topic from `x-mqtt`, default `<il_prefix>/<id>/reject` |

| MQTT | IL input |
|---|---|
| a message on a `set` topic | `Command { prop, value }` |
| broker connection events | handled by the host; the driver sees `Connected` / `Disconnected` for the *device* link only |

A `trigger` has no state topic (it is never published); only its `set` topic exists. An
`event` has no `set` topic (it is never written). An empty retained payload means *absent*, so a
`text` property cannot report the empty string as a value: a producer treats it as absent.

Timers and device I/O are not MQTT's concern and never appear here.
