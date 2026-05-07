# VG Modbus Simulator Specs and Requirements

## Purpose and Scope

This repository implements a Python-based Modbus simulation harness for the S32
HMI/RTAC Modbus map plus legacy per-generator Modbus TCP compatibility servers.
It is intended to support local integration checks for HMI/RTAC-facing tags,
generator/load-bank behavior, register-map resolution, and Modbus read
verification without requiring the real upstream plant hardware.

The simulator has two main surfaces:

- A FastAPI control/read API for simulator state, tag metadata, and supported
  writes into the simulator model.
- Modbus TCP servers for a rich HMI/RTAC map and for legacy GEN01-GEN40 slices.

Raw Modbus writes to writable rich HMI/RTAC tags are a supported control path.
They are intercepted in the rich server datastore, decoded with the configured
tag schema, applied through the same app-level model write path as API writes,
and then mirrored back from `SimulationModel`. Raw writes to non-writable tags
and legacy GEN server writes do not mutate simulator state.

## System Architecture

The runtime entry point is `app.py`. It owns FastAPI route definitions, lifespan
startup/shutdown, the rich HMI/RTAC Modbus server, legacy GEN server startup,
the upstream poller, and datastore mirror loops.

Core module boundaries:

- `simulation_model.py`: pure simulated hardware and control behavior. It owns
  generator state, load-bank state, setpoint/load behavior, burn-in, manual/auto
  mode behavior, test mode, caps, limits, and derived outputs.
- `register_map.py`: configuration loading for `modbus.config.json` into typed
  `ModbusConfig` and `ModbusTag` objects, including config path resolution and
  grouping tags by function area.
- `modbus_codec.py`: datatype encode/decode, byte swap, word swap, 40001-style
  Modbus register mapping, and OPC item path parsing.
- `verify_modbus.py`: command-line verification tool for showing config,
  listing tags, resolving reads, dry-run plans, and live Modbus reads.
- `tests/`: standard-library `unittest` coverage for model behavior, register
  config loading, codec behavior, API helpers/endpoints, and verification CLI
  formatting/resolution.

On application startup, the FastAPI lifespan starts:

- Rich HMI/RTAC Modbus map server on `MODBUS_SIM_HOST` / `MODBUS_SIM_PORT`,
  default `0.0.0.0:1502`.
- Legacy GEN01-GEN40 holding-register servers on ports `505-544`.
- An upstream poller that reads holding registers from `127.0.0.1:502` by
  default.
- Mirror loops that republish cached legacy generator values and rich
  `SimulationModel` values into pymodbus datastores every 0.25 seconds.

FastAPI runs under uvicorn on the port selected at launch, normally `8000`.

## Simulated Hardware and Device Model

The current rich model represents a load-bank control system and three default
generator objects:

- Default generator names: `GEN01`, `GEN02`, `GEN03`.
- Each generator has `kw`, `breaker_closed`, `running`, `voltage`, and
  `frequency`.
- Generator names are normalized from forms such as `1`, `GEN1`, and
  `GENERATOR1` into `GEN01`.
- The model can create additional generator entries through `set_generator`,
  but the configured rich map currently exposes only three generator kW inputs
  and three breaker status pairs.

The load-bank state includes:

- Target generator kW, default `2050`.
- Load-bank cap, default `2000`.
- Generator kW high limit, default `2150`.
- Load-bank step, default `50` kW.
- Burn-in time, default `30.0` seconds.
- Test mode, default enabled.
- Auto/manual control mode, default manual.
- Manual setpoint, active setpoint, requested resistive load, applied load,
  measured active power, available resistive load, nominal voltage/frequency,
  connection mode, network rescan, status/alarms, and derived `lb_max_kw`.

Important behavior:

- During burn-in, the model reports burn-in progress and does not apply normal
  control until after the burn-in sequence.
- In manual mode, `setpoint_kw` follows `manual_setpoint_kw`.
- In auto mode, the model compares target kW, average online generator kW,
  highest generator kW, measured load, cap, and high limit to increase, hold,
  or decrease load-bank setpoint.
- When leaving auto mode, the model performs a bumpless transfer by copying the
  current setpoint into the manual setpoint.
- Test mode suppresses applied load even when requested load changes.
- Live mode (`Test_Mode_Input` false) applies setpoint load after burn-in.
- Optional physics simulation can gradually move measured active power toward
  the active load target.

Derived outputs include number of generators online, average generator kW,
highest generator kW, generator kW difference from target, load-bank setpoint,
requested resistive load, burn-in state, and last control action.

## Device Inventory and Device Type Conventions

The runtime has several device categories:

- Rich model generator objects: three default simulated generators (`GEN01` to
  `GEN03`) used by the rich HMI/RTAC map.
- Legacy generator cache/server slices: forty `GEN01` to `GEN40` cache entries,
  each exposed on a dedicated Modbus TCP server.
- Load bank and control system: simulated setpoint/cap/limit/control-mode,
  permissive, status, load, and derived process values.
- HMI/RTAC Modbus server: one rich Modbus TCP server populated from
  `SimulationModel` through `modbus.config.json`.
- Upstream source: a Modbus TCP source polled at `127.0.0.1:502` by default for
  the legacy generator cache.
- IED/device identifier list: the workbook
  `PowerMAX-modbus-unique-IED-list.xlsx`.

The workbook is a read-only source file in this repo. Inspection of the current
one-column workbook found 278 identifiers. Prefix examples and counts are:

- `ICB`: 95, examples `ICB-01` through `ICB-05`.
- `GEN`: 54, examples `GEN02` through `GEN06`.
- `GCB`: 46, examples `GCB-01` through `GCB-05`.
- `G`: 35, examples `G2`, `G20`, `G21`, `G22`, `G23`.
- `CB`: 21, examples `CB-101-01` through `CB-101-05`.
- Smaller groups include `AUX` (6), `SC` (6), `FCB` (5), `F` (3),
  `Main` (2), and singletons such as `EDG`, `EM-1`, `GCSA`, `N/A`, and `Tie`.

The workbook is not currently loaded by the runtime code. It should be treated
as inventory/source data unless a future task explicitly assigns runtime
integration.

## Modbus Architecture

### Rich HMI/RTAC Server

The rich server is created from `modbus.config.json` and defaults to
`MODBUS_SIM_HOST=0.0.0.0` and `MODBUS_SIM_PORT=1502`. It exposes all four
Modbus function areas:

- Coils
- Discrete inputs
- Holding registers
- Input registers

The current map contains 43 tags:

- Coils: 2
- Discrete inputs: 10
- Holding registers: 12
- Input registers: 19

The current writable tag count is 8. Writable rich tags can be written through
the API endpoints or through raw Modbus writes that fully cover the configured
tag address range.

### Legacy GEN Servers

The legacy compatibility layer creates forty holding-register Modbus TCP
servers:

- `GEN01` runs on port `505`.
- `GEN40` runs on port `544`.
- Each generator exposes a five-register slice:
  `kw`, `voltage`, `frequency`, `breaker_closed`, `running`.

The upstream poller reads `40 * 250` holding registers from upstream address
`40001` mapped to zero-based address `0`, then slices the result into
forty generator blocks of 250 registers each. Only the first five values in
each block are mapped into the legacy `GenState`.

### Register, Datatype, and Swap Conventions

`modbus.config.json` is the authoritative rich HMI/RTAC map. Its `_note` says
the core map is derived from `SEL_RTAC/Devices/HMI_Modbus_Server_MODBUS.xml` and
`HMI_Modbus_Server_Tags.xml`.

Tag fields include:

- `function`: one of `coil`, `discrete`, `holding`, or `input`.
- `address`: zero-based Modbus address.
- `length`: bit/register count.
- `datatype`: current map uses `bool`, `uint16`, and `uint32`.
- `writable`: whether the API accepts writes for the tag.
- `path`: simulation or source path used by runtime mapping.
- `wordSwap`: 32-bit LSR-style values use word swap.
- `byteSwap`: 16-bit LSB-style values use byte swap.

`modbus_codec.py` maps human register ranges as follows:

- `1-9999`: coils, zero-based address `register - 1`.
- `10001-19999`: discrete inputs, zero-based address `register - 10001`.
- `30001-39999`: input registers, zero-based address `register - 30001`.
- `40001-49999`: holding registers, zero-based address `register - 40001`.

Supported decode/encode datatypes include `bool`, `uint16`, `int16`, `uint32`,
`int32`, and `float32`.

Supported OPC item tokens include:

- `C`: coil bool.
- `DI`: discrete input bool.
- `HRUI`: holding `uint32` with `wordSwap`.
- `HRUS`: holding `uint16` with `byteSwap`.
- `IRUI`: input `uint32` with `wordSwap`.
- `IRUS`: input `uint16` with `byteSwap`.

## REST/API Surface

The FastAPI app is titled `Generator Modbus Simulator rev0`.

Endpoints:

- `GET /health`: returns `{"status": "ok"}`.
- `GET /generators`: returns the forty legacy generator cache entries.
- `GET /generators/{gen_id}`: returns one legacy generator cache entry,
  case-normalized to uppercase, or 404 if missing.
- `GET /state`: returns a full rich `SimulationModel` snapshot.
- `GET /tags`: syncs the rich context and returns tag metadata plus current
  values for all configured rich tags.
- `GET /tags/{tag_name}`: syncs the rich context and returns one tag, or 404.
- `POST /write`: writes a configured rich tag by `tag` or `path`.
- `PUT /tags/{tag_name}`: writes a configured rich tag by route name.

Write request body shape:

```json
{
  "tag": "HMI_Modbus_Server_Tags_Target_kW_Input",
  "value": 1800
}
```

or:

```json
{
  "path": "cfg.targetGenKW",
  "value": 1800
}
```

API write requirements:

- The request must include `tag` or `path`.
- The tag/path must resolve to a configured tag.
- The tag must be marked `writable`.
- The value must be accepted by `SimulationModel.write_path` or by the explicit
  special-case handling in `app.py`.

Special write behavior:

- `lb_mode_req_auto` writes `Control_Mode_Input`.
- `ctrl_state_req_enable` has no current `SimulationModel` input, so the raw
  request value is preserved in `rich_tag_values` for explicit API behavior.
- Other writable tags are passed through `SimulationModel.write_path`.

## Data and Configuration Sources

Primary data/configuration sources:

- `modbus.config.json`: authoritative rich HMI/RTAC map in this repo.
- `SimulationModel`: runtime source of rich process values and supported
  simulator inputs.
- Upstream Modbus source: legacy generator values read from `127.0.0.1:502`.
- `PowerMAX-modbus-unique-IED-list.xlsx`: read-only one-column IED/device
  identifier inventory.

`register_map.py` resolves config path candidates in this order:

- `MODBUS_CONFIG_PATH`, when set.
- `modbus.config.json` next to the module/base path.
- `modbus.config.json` in the parent of the base path.
- `modbus.config.json` in the current working directory.

The current `modbus.config.json` includes connection metadata for an external
configured slave (`192.168.48.108:502`), unit id `1`, HTTP port `4000`, timeout
`3000` ms, and poll interval `1000` ms. The simulator runtime defaults in
`app.py` are separate from that external map metadata.

## Runtime Behavior

Startup:

1. Global rich config/model/context objects are initialized at import time.
2. FastAPI lifespan starts legacy GEN servers, the rich server, upstream
   polling, legacy cache mirroring, and rich context mirroring.
3. Uvicorn hosts the FastAPI app, normally on port `8000`.

Polling and mirroring:

- The upstream poller reads the legacy upstream source once per second.
- The legacy mirror updates each GEN server context every 0.25 seconds.
- The rich mirror repopulates all rich datastore values from
  `SimulationModel` every 0.25 seconds.

Read behavior:

- Rich tag reads try `SimulationModel` by tag name and configured path.
- Breaker tags are derived from `GEN1/2/3` breaker state and inverted for
  `_open` paths.
- Permissive/control-state tags are synthesized where the pure model does not
  directly own them.
- Otherwise, the runtime falls back to `rich_tag_values` or `0`.

Write behavior:

- Supported simulator writes go through FastAPI `POST /write` or
  `PUT /tags/{tag_name}`.
- Raw Modbus writes to writable rich tags are intercepted by the rich datastore,
  decoded through `modbus_codec.decode_value` using `tag.to_schema()`, and
  applied through the same app-level write helper used by API writes.
- Raw writes to non-writable rich tags do not mutate `SimulationModel`; because
  the rich mirror loop republishes model values every 0.25 seconds, those
  datastore-only values are overwritten.
- Partial raw writes that do not fully cover a configured writable tag remain
  datastore-only until the next mirror sync.
- Legacy GEN server raw writes remain datastore-only and do not mutate simulator
  state.

## Known Limitations

- Raw Modbus write interception is implemented only for full writes to writable
  tags in the rich HMI/RTAC server.
- The rich map exposes three generator model objects, while the legacy
  compatibility surface exposes forty generator cache slices.
- The workbook inventory is not currently part of runtime loading.
- Tests avoid live network servers; live port conflicts are possible when
  running the full uvicorn app because startup opens one rich server plus
  forty legacy GEN servers.
- Pymodbus API compatibility matters. The repository currently depends on
  `pymodbus==3.8.6`.
- The upstream poller defaults are constants in `app.py`
  (`127.0.0.1:502`, start register `40001`, 250 registers per generator).
  They are not currently driven by environment variables.
- `SimulationModel` currently handles a defined subset of tag/path writes.
  Unknown paths raise errors through the API.

## Requirements

Functional requirements:

- The simulator shall expose a rich HMI/RTAC Modbus TCP server on
  `MODBUS_SIM_HOST` / `MODBUS_SIM_PORT`, default `0.0.0.0:1502`.
- The simulator shall expose legacy GEN01-GEN40 Modbus TCP servers on ports
  `505-544`.
- The FastAPI app shall expose health, generator cache, rich model state, tag
  metadata/value, and supported write endpoints.
- API writes shall validate tag/path existence, writeability, and model input
  compatibility before mutating simulator state.
- API writes shall synchronize the rich Modbus datastore after applying changes.
- Raw Modbus writes to writable rich HMI/RTAC tags shall decode values through
  the configured tag schema and apply them to `SimulationModel` through the
  same app-level write path as API writes.
- Raw Modbus writes to non-writable rich tags and legacy GEN server tags shall
  not mutate `SimulationModel`.
- Partial raw Modbus writes that do not fully cover a configured writable rich
  tag may remain datastore-only.
- The rich register map shall remain driven by `modbus.config.json`.
- Register-map and codec logic shall use `register_map.py` and
  `modbus_codec.py`; implementations shall not duplicate endian/register
  conversion rules elsewhere.
- The verification CLI shall support tag-based, register-based, and manual
  function/address read resolution, plus dry-run output.

Quality and maintenance requirements:

- Code and docs should remain ASCII.
- Tests should use standard-library `unittest`.
- Pure model, codec, and register-map behavior should stay testable without
  opening sockets.
- Runtime network behavior should be isolated from unit tests unless a task
  explicitly targets live startup.
- Future changes to `modbus.config.json` must account for app behavior, CLI
  output, and register-map test expectations.

## Verification Plan

Baseline commands:

```powershell
python -m unittest discover -s tests
python -m py_compile verify_modbus.py
```

Useful verification CLI commands:

```powershell
python verify_modbus.py --show-config
python verify_modbus.py --list-tags
python verify_modbus.py --tag HMI_Modbus_Server_Tags_Target_kW --dry-run
python verify_modbus.py --register 40013 --datatype uint16 --byte-swap --dry-run
```

Useful live-read checks after starting uvicorn:

```powershell
uvicorn app:app --host 0.0.0.0 --port 8000
python verify_modbus.py --host 127.0.0.1 --port 1502 --tag HMI_Modbus_Server_Tags_Target_kW
python verify_modbus.py --host 127.0.0.1 --port 1502 --function holding --address 0 --length 2 --datatype uint32 --word-swap
python verify_modbus.py --host 127.0.0.1 --port 1502 --tag HMI_Modbus_Server_Tags_Target_kW --duration 10 --interval 1
```

Recommended API write check:

```powershell
Invoke-RestMethod -Method Post -Uri http://127.0.0.1:8000/write -ContentType application/json -Body '{"tag":"HMI_Modbus_Server_Tags_Target_kW_Input","value":1800}'
```

Expected stable unit-test expectations:

- Register map loads 43 tags.
- Function counts are 2 coils, 10 discrete inputs, 12 holding registers, and
  19 input registers.
- Writable tag count is 8.
- Datatype counts are 12 `bool`, 23 `uint32`, and 8 `uint16`.
- Rich `/tags` returns 43 tags.
- API write to `HMI_Modbus_Server_Tags_Target_kW_Input` updates the writable
  input tag, the readback target tag, and the encoded holding-register values.
- Raw rich Modbus writes to full writable holding-register and coil tags update
  the same model paths as API writes and survive the rich mirror sync.
- Raw rich Modbus writes to non-writable tags do not mutate the model, and
  partial raw writes to writable tags remain datastore-only.
- Generator endpoint semantics continue to return forty legacy generator cache
  entries.

## Open Questions and Future Work

- Should upstream poller defaults be configurable through environment variables
  or `modbus.config.json`?
- Should the workbook IED/device inventory become a runtime source, a generated
  config artifact, or remain reference-only?
- Should the rich model grow from three mapped generators to match the forty
  legacy generator slices?
- Should live integration tests be added for uvicorn startup and real Modbus
  reads when ports are known to be free?
- Should `ctrl_state_req_enable` become a first-class `SimulationModel` input
  instead of an app-level preserved value?
