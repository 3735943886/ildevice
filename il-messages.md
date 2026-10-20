# IL messages: the data that crosses the boundary

> **Draft, version 0.** Normative. Keywords and rule ids as in [il.md](il.md) section 0 (ids here
> are `W-n`). Depends on no transport: it defines **only the shape of the data** a producer and a
> consumer exchange. Carrying it (UDP, HTTP, a file, MQTT, or a plain function call) is a
> transport mapping ([il-mqtt.md](il-mqtt.md) is one), and a mapping adds nothing to the model.

The IL is sans-IO: the driver ([il.md](il.md) section 8) takes inputs and returns outputs as
values, and never sends or receives anything. This document names the values that leave a
producer for a consumer, and those that come back, so that any two implementations can talk
over any channel that can move them.

## 1. Messages

A message is a JSON object with a string field `msg` that names its kind. Field names are
exactly as below. `id` is a descriptor `id`, `prop` a property name.

**Producer to consumer** (each is the data of the driver output of the same name, addressed to a
device):

| `msg` | fields | meaning |
|---|---|---|
| `descriptor` | `descriptor` | the device's description (il.md section 2) |
| `remove` | `id` | the device is gone; forget it and its values |
| `value` | `id`, `prop`, `value` | the property now has this value |
| `absent` | `id`, `prop` | the property no longer has a value |
| `event` | `id`, `prop`, `kind` | an `event` property happened once; `kind` is one of its `options` |
| `reject` | `id`, `prop`, `code`, `reason`? | a write could not be honoured (il.md section 5) |
| `presence` | `source`, `online` | the producer named `source` is reachable (`true`) or not (`false`) |

**Consumer to producer:**

| `msg` | fields | meaning |
|---|---|---|
| `command` | `id`, `prop`, `value`? | write a property. `value` is omitted for a `trigger` |
| `sync` | `id`? | send the current state (section 3) of the device, or of every device if `id` is omitted |

`?` marks an optional field.

- **W-1** A message MUST be a single JSON object, UTF-8, RFC 8259. A transport MAY wrap it (a
  frame, an HTTP body, a line of a file) but MUST hand the consumer or producer exactly this object.
- **W-2** `value` in a `value` or `command` message is a JSON value of the property's type: a boolean,
  a number, or a string. (`select`, `text` and `event` are strings.) A `command` MAY use the wider
  forms of il.md section 5 (`"on"`, `1`); the producer normalises them.
- **W-3** A receiver MUST ignore a message whose `msg` it does not know, and a field it does not
  know (il.md X-1).
- **W-4** A `value` message MUST NOT name a `trigger` or an `event`; an `event` message MUST name an
  `event`. (il.md R-5.)
- **W-5** A message about a device the receiver has no descriptor for is not an error: a consumer
  MAY drop it or hold it until the descriptor arrives (a `sync` fetches one).
- **W-6** A `presence` message with `online: false` makes a consumer treat every device whose
  descriptor `source` equals `source` as unavailable until an `online: true` arrives, whatever the
  devices' own `available` last said. It does not delete the devices. Absence of any `presence`
  message means the producer is assumed present.

Independent message examples (JSON Lines; not one JSON document or a complete session):

```jsonl
{ "msg": "descriptor", "descriptor": { "il": 0, "id": "d1", "props": { "power": { "type": "binary", "rw": true, "role": "on" } } } }
{ "msg": "value",   "id": "d1", "prop": "power", "value": true }
{ "msg": "absent",  "id": "d1", "prop": "power" }
{ "msg": "event",   "id": "d2", "prop": "press", "kind": "double" }
{ "msg": "command", "id": "d1", "prop": "power", "value": "off" }
{ "msg": "reject",  "id": "d1", "prop": "target", "code": "out_of_range", "reason": "max is 70" }
{ "msg": "presence", "source": "rusthinq", "online": true }
{ "msg": "sync" }
```

## 2. Relation to the driver interface

The driver's outputs `Descriptor`, `Value`, `Absent`, `Event` and `Reject` are the producer-to-consumer
messages above with the device `id` added by the host; the driver's input `Command` is the `command`
message after the checks of il.md section 5. `remove`, `presence` and `sync` belong to the host, not
the driver: the driver does not know its own `id` in the world, whether the producer is alive, or who
is joining. `SendFrame`, `SendMessage`, `SetTimer`, `CancelTimer` are between the driver and its host
and never leave the producer.

A producer that calls a consumer directly, in one process, passes these same objects (or the
language's equivalent of them) and needs no serialisation; the data model is the interface.

## 3. Current state, replay and ordering

A consumer that starts after a producer needs the current state. A transport may restore stored
state itself (MQTT uses retained messages); otherwise the consumer requests a snapshot with `sync`.
A transport mapping must state whether replay is atomic and how absent values clear cached state;
MQTT's per-topic replay is not an atomic snapshot (il-mqtt.md section 4).

- **W-7** A producer MUST be able to produce, for a device, its **snapshot**: the `descriptor` message,
  then exactly one `value` or `absent` message for each state-bearing property (`binary`, `number`,
  `select`, `text`), in any order. A missing value MUST be represented by `absent`, so a consumer
  that already has state can discard stale values. A snapshot MUST NOT include `event` or `reject`.
  A producer MUST answer `sync` with the snapshots of the devices asked for and then resume live
  updates. For each device, the snapshot MUST represent one logical point in time; live updates
  for that device MUST NOT interleave with its snapshot. A descriptor alone is not a cache reset.
- **W-8** An `event` is an occurrence, not state: a transport MUST NOT replay it, and a snapshot never
  contains one. A consumer that is handed an event it knows to be a replay MUST ignore it.
- **W-9** Messages of one device from one producer are delivered in the order they were produced, and a
  `descriptor` precedes the first `value`, `absent` or `event` for the properties it introduces
  (il.md R-3). A transport that cannot guarantee order MUST say so in its mapping, and a consumer
  on it SHOULD treat a message for a property it has no definition for by W-5.
- **W-10** A `remove` is followed by no further message of that device until a new `descriptor` for the
  same `id` arrives.
- **W-11** Delivery guarantees (at most once, at least once) belong to the transport. A consumer MUST
  tolerate a repeated `value` or `absent`; a repeated `event` is a repeated occurrence (il.md section
  4), so a transport that duplicates messages MUST NOT be used for events without saying so.
- **W-12** A write's observable effect is reported through ordinary state updates; a failed write
  is reported by `reject`. The IL defines no success acknowledgement, command correlation or
  completion deadline. A successful trigger, or a write of the value already held, MAY produce no
  state update. Silence MUST NOT be interpreted as success or failure, and a state update alone
  does not identify which command caused it. A `command` MUST NOT be replayed: a transport that
  stores messages MUST NOT store or replay it. A consumer SHOULD NOT automatically retry a trigger
  after an uncertain outcome; the action might already have happened.

## 4. Size

- **W-13** A `descriptor` message SHOULD be at most 64 KiB and a `text` value at most 1 KiB. A receiver
  MAY drop a larger message.

## 5. Security

Who may send a `command` is decided by the transport and the deployment (il.md S-5), never by a
field of the message; a message carries no credential.
