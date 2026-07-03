# DataFarm-Tech/proto

Shared [Protocol Buffers](https://protobuf.dev/) schema for the CoAP telemetry protocol between the Green Gauge device firmware ([green-gauge-core](https://github.com/DataFarm-Tech/green-gauge-core)) and the backend. Keeping the schema in its own repo means both sides regenerate bindings from the exact same source of truth instead of copy-pasting field definitions.

## Contents

| File | Purpose |
| --- | --- |
| `telemetry.proto` | Message definitions |
| `telemetry.options` | [nanopb](https://jpa.kapsi.fi/nanopb/) field-size bounds (`max_size`/`max_count`) used when generating the fixed-size, allocation-free C structs the firmware needs |
| `telemetry.pb.c` / `telemetry.pb.h` | Committed nanopb-generated C code (see [Regenerating](#regenerating) below) — this is what the firmware actually compiles against |

## Messages

| Message | Direction | Sent on |
| --- | --- | --- |
| `ActivateRequest` | device → server | `POST /activate` |
| `GpsUpdateRequest` | device → server | `PUT /gps-update` |
| `ReadingRequest` | device → server | `POST /reading` |
| `StringValue` | server → device | `GET /firmware-version`, `GET /firmware-download` responses |

`StringValue` is a generic single-field wrapper reused for both OTA response endpoints, which each return a single opaque string (a version string or an HTTPS URL, respectively).

## CoAP Content-Format

Protocol Buffers has no IANA-assigned CoAP Content-Format number. Both sides of this protocol use `65500`, a value from the RFC 7252 "Reserved for Experimental Use" range (`65000`–`65535`). This is a private convention specific to this protocol, not a public/registered value — if either side changes it, the other must be updated in lockstep.

## Consumers

- **Firmware** ([green-gauge-core](https://github.com/DataFarm-Tech/green-gauge-core)) — pulls this repo in as a git submodule at `main/proto/`, and compiles the committed `telemetry.pb.c` directly (see that repo's README for details, and the [Regenerating](#regenerating) section below for why the generated code is committed rather than built from `.proto` at compile time).
- **Backend** — regenerate bindings for whatever language the backend is written in using a standard `protoc` invocation against `telemetry.proto` (no nanopb-specific tooling needed on that side; `telemetry.options` only affects the nanopb/C output).

## Regenerating

`telemetry.pb.c`/`telemetry.pb.h` are nanopb-generated C code, committed to this repo rather than generated at firmware build time — the ESP-IDF `nikas-belogolov/nanopb` component only vendors the nanopb runtime (`pb_encode.c`/`pb_decode.c`/`pb_common.c`), not the code generator, so there is no build-time `.proto` → `.pb.c` step available on the firmware build machine.

After editing `telemetry.proto` or `telemetry.options`:

```bash
pip install --user nanopb
nanopb_generator -I. telemetry.proto
```

This overwrites `telemetry.pb.c`/`telemetry.pb.h`. Review the diff, then commit and push all changed files together (`.proto`, `.options` if changed, and the regenerated `.pb.c`/`.pb.h`).

If you're only regenerating bindings for a non-firmware consumer (e.g. the backend), use a plain `protoc --<lang>_out=...` invocation against `telemetry.proto` instead — you don't need nanopb or `telemetry.options` for that.

## Updating firmware to a new schema version

After pushing changes here, bump the submodule pointer in `green-gauge-core`:

```bash
cd main/proto && git checkout main && git pull
cd ../.. && git add main/proto
git commit -m "Bump proto submodule"
```
