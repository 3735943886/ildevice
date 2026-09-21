# Intermediate Layer Device Model

A neutral, transport-independent description of a device: what it is, what values it
has, and which of them can be written. Specification only; no code in any language.

- [il.md](il.md): the model (descriptor, typed properties, roles) and the sans-IO driver
  interface (inputs and outputs as data).
- [il-mqtt.md](il-mqtt.md): how the IL is carried on MQTT.
- [il-consumers.md](il-consumers.md): informative notes for writing a consumer.
- [schema/descriptor.schema.json](schema/descriptor.schema.json): JSON Schema of the
  descriptor.
- [examples/](examples/): example descriptors.
- [notes/](notes/): informative notes (parity with an existing integration, gaps found by the
  Tuya spike).

Status: draft, version 0.
