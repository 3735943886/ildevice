# Intermediate Layer Device Model

A neutral, transport-independent description of a device: what it is, what values it
has, and which of them can be written. Specification only; no code in any language.

**Normative**

- [il.md](il.md): the model (descriptor, typed properties, roles, kinds, classes, units), the
  command checks, safety rules, the sans-IO driver interface, and conformance. Rules have ids
  (`D-1`, `V-3`, …).
- [il-messages.md](il-messages.md): the messages a producer and a consumer exchange, as data only
  (no transport: a function call, UDP, HTTP, a file or MQTT can all carry them).
- [il-mqtt.md](il-mqtt.md): one transport mapping, MQTT.
- [schema/descriptor.schema.json](schema/descriptor.schema.json): JSON Schema of the descriptor
  (structure, type/role consistency); [schema/message.schema.json](schema/message.schema.json): the messages.
- [vectors/](vectors/): language-neutral conformance vectors: command validation (`commands.json`), payloads on the wire (`wire-values.json`), topics (`topics.json`), composites (`composites.json`).

**Informative**

- [il-rationale.md](il-rationale.md): worked examples per device, design history, decisions,
  changelog, open points.
- [il-consumers.md](il-consumers.md): notes for writing a consumer.
- [examples/](examples/): example descriptors (illustrative, not a device catalogue);
  [examples/invalid/](examples/invalid/) holds descriptors the schema must reject, each naming
  the rule it breaks in `x-violates`.
- [notes/](notes/): parity with an existing integration, gaps found by the Tuya spike.

Status: draft, version 0.
