# IL over MQTT

> **Draft, version 0.** One transport mapping for the IL messages of [il-messages.md](il-messages.md).
> MQTT is not part of the IL: the model, the messages and the driver interface do not depend on this
> document, and a mapping for UDP, HTTP, a file or a direct call is written the same way (say how each
> message of il-messages.md section 1 is carried, and how W-7..W-12 are met). Keywords as in
> [il.md](il.md) section 0; rules have ids `M-n`.

This mapping does not carry the JSON envelope of il-messages.md. It spreads the same data over
MQTT topics and payloads, because MQTT already has an address (the topic) and stored state (retained
messages). It adds nothing to the model. MQTT 3.1.1 or 5.

## 1. Topics

| what | topic | retained | payload |
|---|---|---|---|
| `descriptor` | `<il_prefix>/<id>` | yes | the descriptor as JSON; an **empty** payload is `remove` |
| `value`, `absent` | see "Locating values" | yes | the value, encoded as in section 3; an empty payload is `absent` |
| `event` | see "Locating values" | **no** | one of the property's `options` |
| `command` | see "Locating values" | no | the same forms as a value |
| `reject` | see "Locating values" | no | `{ "prop", "code", "reason" }` as JSON |
| `presence` | `<il_prefix>/_producer/<source>` | yes | `online` or `offline` |

- **M-1** `il_prefix` defaults to `il` and is the one thing a consumer must be configured with.
  A consumer subscribes to `<il_prefix>/+` and discovers every device.
- **M-2** With the default topic layout, a device `id` MUST NOT contain `/`, `+`, `#` or NUL, MUST NOT
  be empty and MUST NOT start with `_` (reserved for M-12). A producer whose ids do may use `x-mqtt`
  locations that avoid them.
- **M-3** A consumer MUST ignore a retained message on an `event` topic: it is an old occurrence the
  broker replays, not a new one (il-messages.md W-8).
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
  and a producer MUST ignore a retained message on a `set` topic (it would replay an old command;
  il-messages.md W-12).
- **M-10** An empty retained payload on a state topic means *absent* (il.md V-1, V-2); it cannot mean
  the empty string.
- **M-11** A producer that removes a device MUST first clear (empty retained payload) the retained
  values of its properties, then the descriptor (`remove`, il-messages.md W-10). A producer that
  restarts and finds retained values of a property it no longer has SHOULD clear them.
- **M-12** A producer SHOULD publish `online` (retained, QoS 1) on `<il_prefix>/_producer/<source>` when
  it connects, and register `offline` (retained, QoS 1) as its Last Will on the same topic. `<source>`
  is the descriptor `source` of its devices; a producer with several instances uses `<source>` plus a
  suffix it makes unique. A consumer treats it as the `presence` message (il-messages.md W-6).

## 4. Mapping the messages

| message | MQTT |
|---|---|
| `descriptor` | publish (retained) the descriptor on `<il_prefix>/<id>` |
| `remove` | publish an empty retained payload on `<il_prefix>/<id>` (after M-11) |
| `value` | publish (retained) the value on the state topic |
| `absent` | publish an empty retained payload on the state topic |
| `event` | publish `kind` on the state topic, **not retained**. Every message is one occurrence, including a repeat of the same kind |
| `reject` | publish `{ "prop", "code", "reason" }` (not retained) on the `reject` topic |
| `presence` | M-12 |
| `command` | a message on a `set` topic; the producer applies the checks of il.md section 5 |
| `sync` | not used: retained messages already give a late consumer the snapshot (il-messages.md W-7) |

Broker connection events are handled by the host; the driver sees `Connected` / `Disconnected` for
the *device* link only. Timers and device I/O are not MQTT's concern and never appear here.
Delivery order (il-messages.md W-9) holds per topic, not across topics: a consumer that sees a value
before the descriptor holds it by W-5.

## 5. Not covered

Authentication (il.md S-5): the broker's ACLs and the deployment decide who may write a `set` topic.
