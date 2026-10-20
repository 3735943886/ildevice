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

## Releases and compatibility

Current specification release: **v0.1.0** (2026-10-20); descriptor version: **`il: 0`**.
[VERSION](VERSION) records the release number. A Git tag `v<version>` identifies the matching
bundle of documents, schemas, examples and conformance vectors. Published tags are immutable.
Release notes are in the [changelog](il-rationale.md#4-changelog).

The release number and descriptor field `il` serve different purposes:

- `il` identifies the model's compatibility generation. It remains `0` while the model is a draft;
  draft breaking changes are allowed by [X-7](il.md#7-compatibility-and-versioning).
- During `0.x`, breaking specification changes increase the minor release (`0.1.0` → `0.2.0`).
  Backward-compatible additions, clarifications and corrections increase the patch release
  (`0.1.0` → `0.1.1`). Classify a correction by its compatibility impact, not its size.
- Freezing the draft produces release `v1.0.0` with `il: 1`. From then on, releases use semantic
  versioning: breaking changes increase major, compatible additions minor, and corrections patch.
  Breaking model changes also increase `il` as required by X-4; a release number is never sent in
  place of `il`.

Implementations should name the specification release and its commit. Automated conformance
checks should pin the tag's full commit SHA so that the tested schemas and vectors are reproducible.
