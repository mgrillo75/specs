# Modbus TCP Multi-Device Simulator — Final Specification

Python service exposing 200+ Modbus TCP slaves on ports 501-700 for PowerMAX FAT testing. Single process, no containers, no Ignition module.

## Overview

Simulator for VG PowerMAX HMI/RTAC Modbus map. 200 identical virtual devices (GEN01-GEN200), each serving the full 4,617-point register map from `full-modbus-map.xlsx`. Used for Factory Acceptance Testing without real plant hardware.

## Stack

| Component | Technology |
|-------------|------------|
| Runtime | Python 3.11/12 |
| Modbus | pymodbus 3.x (`ModbusSimulatorServer`) |
| Concurrency | asyncio (single event loop, 200+ servers) |
| Config | Pydantic models, JSON validation |
| HTTP API | FastAPI + Uvicorn |

## Register Map (Per Slave)

Compiled from `full-modbus-map.xlsx` (4,617 points total). All 200 devices share this map; each has isolated state.

| Function | Range | Count | Data Types | Access |
|----------|-------|-------|------------|--------|
| Coils | 0-657 | 566 | Boolean | R/W per device |
| Discrete Inputs | 10001-11345 | 1,070 | Boolean | Read-only |
| Input Registers | 2970-32968 | 1,273 | Float, Int | Read-only |
| Holding Registers | 1907-41906 | 980 | Float, Int | R/W per device |

**Data Type Breakdown:**
- Float (32-bit): 1,778 points
- Boolean: 1,720 points
- Int (16/32-bit): 475 points

**Writable:** 1,630 points | **Read-only:** 2,343 points

**Signal Categories (from Excel):**
- Generator Tuning (1,288 points)
- Generation Management (1,013 points)
- Voltage Control (876 points)
- Automatic Generation Control (874 points)
- Single Line Diagram (316 points)
- Network Overview, Blackstart Sequence, etc.

## Register Addressing Conventions

Zero-based addressing per Modbus standard:
- Coils: address = register
- Discrete Inputs: address = register - 10001
- Input Registers: address = register - 30000 (or 2970 offset)
- Holding Registers: address = register - 40000 (or 1907 offset)

Swap handling: byteSwap for 16-bit LSB, wordSwap for 32-bit LSR-style values.

## Runtime Modes (Per Device)

| Mode | Behavior | Use Case |
|------|----------|----------|
| `static` | Configured defaults until Modbus client writes | Baseline testing |
| `synthetic` | 1Hz incrementing counters; sine/ramp on Floats | Dynamic signal verification |
| `mirror` | Poll upstream Modbus TCP; replicate to slave | Integration with real RTACs |

Mirror mode optional per-device. Upstream connection: host, port, unit_id, poll_interval_ms.

## HTTP Control Plane

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/health` | GET | Liveness probe — returns `{"status": "ok"}` |
| `/status` | GET | Process health + slave count + mode distribution |
| `/devices` | GET | List all 200 devices: port, mode, unit_id, connection_count |
| `/registers` | GET | Query params: `port`, `function`, `address`. Returns register values |
| `/write` | POST | Body: `{"port": 501, "function": "holding", "address": 1907, "value": 123.4}` |
| `/mode` | POST | Body: `{"port": 501, "mode": "synthetic"}` — runtime mode switch |

HTTP port: 8080 (configurable).

## Config Schema (`config.json`)

```json
{
  "http": {"port": 8080, "host": "0.0.0.0"},
  "devices": [
    {"port": 501, "mode": "static", "unit_id": 1},
    {"port": 502, "mode": "synthetic", "unit_id": 2},
    {"port": 503, "mode": "static", "unit_id": 3},
    ...
    {"port": 700, "mode": "mirror", "unit_id": 200, 
     "upstream": {"host": "192.168.1.10", "port": 502, "poll_ms": 1000}}
  ],
  "register_map": "modbus_map.json",
  "synthetic": {"counter_hz": 1, "sine_amplitude": 100.0}
}
```

## Excel to JSON Compilation

`compile_map.py` converts `full-modbus-map.xlsx` → `modbus_map.json`:

| Excel Column | JSON Field | Notes |
|--------------|------------|-------|
| Signal Name | `name` | Unique identifier |
| MODBUS Type | `function` | Coil/Discrete Input/Input Register/Holding Register |
| Function Code | `func_code` | 0x01, 0x02, 0x04, 0x05, 0x06, 0x10 |
| MODBUS Register Start Address | `address` | Integer address |
| Register Count | `count` | 1 for Boolean, 2 for Float/32-bit |
| Data Type | `datatype` | bool, float32, int16, int32 |
| Writable | `writable` | boolean |
| Engineering Units | `units` | MW, kV, Hz, etc. |
| Scale / State | `scale` | Conversion factor or state mapping |
| Category | `category` | For filtering/organization |

Validation: Duplicate addresses, overlapping ranges, and missing required fields fail compilation with line-specific errors.

## Deployment

**Windows (Primary Target):**
```powershell
# Install as Windows Service
.\install_service.ps1 -Install
# Service name: ModbusSimulator
# Logs: C:\Logs\modbus-sim\service.log
```
- NSSM wraps Python process
- Auto-starts on boot
- Headless (no GUI)
- Restart on crash

**Linux (Dev/Testing):**
```bash
sudo systemctl enable modbus-sim
sudo systemctl start modbus-sim
```

**Ignition Connection:**
- 200 Modbus TCP device connections
- Each pointing to `localhost:501` through `:700`
- Same configuration as production RTACs (only IP/port differs)

## File Structure

```
modbus-sim/
├── simulator.py           # asyncio launcher; HTTP API
├── device.py              # ModbusSimulatorServer wrapper per slave
├── config.py              # Pydantic models; validation
├── compiler/
│   ├── compile_map.py     # Excel → JSON converter
│   └── validators.py        # Address conflict detection
├── maps/
│   ├── full-modbus-map.xlsx   # Source of truth (read-only)
│   └── modbus_map.json        # Compiled output
├── config.json            # 200 device definitions
├── install_service.ps1    # NSSM install/uninstall
├── tests/
│   ├── test_device.py
│   ├── test_compiler.py
│   └── test_api.py
└── README.md              # Start/stop/verify commands
```

## Acceptance Criteria

| ID | Criteria | Verification |
|----|----------|--------------|
| A1 | 200 slaves bind on ports 501-700 | `GET /devices` returns 200 entries |
| A2 | Per-device isolated state | Write to port 501; verify port 502 unchanged |
| A3 | Full 4,617-point map per device | Spot-check all 4 function areas via Modbus reads |
| A4 | Float, Int, Boolean data types | Read holding register 1907 (Float); verify encoding |
| A5 | Ignition reads from all 200 devices | Connection count = 200 in Ignition gateway status |
| A6 | HTTP `/registers?port=501` returns values | JSON array with register metadata + current values |
| A7 | Runtime mode switch without restart | `POST /mode` → slave changes static→synthetic |
| A8 | Mirror mode propagates upstream within 1s | Change upstream value; verify reflected in slave |
| A9 | Service auto-starts on Windows boot | Reboot; verify `GET /health` returns ok |
| A10 | <500 MB RAM for 200 active slaves | Task Manager / `psutil` memory report |
| A11 | Excel compilation validates duplicates | `compile_map.py` exits non-zero on address conflict |

## Verification Commands

```powershell
# Quick health check
curl http://localhost:8080/health

# List all devices
curl http://localhost:8080/devices | ConvertFrom-Json

# Read specific register from slave on port 501
python -c "from pymodbus.client import ModbusTcpClient; c=ModbusTcpClient('localhost', 501); c.connect(); r=c.read_holding_registers(0, 2); print(r.registers); c.close()"

# Switch mode at runtime
Invoke-RestMethod -Uri http://localhost:8080/mode -Method Post -ContentType 'application/json' -Body '{"port": 501, "mode": "synthetic"}'

# Verify synthetic mode (counter incrementing)
curl http://localhost:8080/registers?port=501&function=holding&address=1907
```

## Out of Scope

- Modbus RTU/serial transport
- Kubernetes/container deployment (user requirement: no IT group)
- Per-device unique register maps (all 200 share compiled map)
- SSL/TLS on Modbus TCP (production RTACs don't use it)
- Authentication on HTTP API (runs on localhost only)
