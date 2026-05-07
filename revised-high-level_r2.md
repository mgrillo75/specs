# Modbus TCP Multi-Device Simulator — Rev 2 Specification

Python service that polls a single upstream Modbus TCP source, translates data through multiple device-type maps, and exposes 200+ unique Modbus TCP slaves on ports 1501-1701.

## Overview

**Architecture Components:**
1. **SEL RTAC** — Upstream Modbus TCP source (192.168.40.100:502)
2. **Modbus TCP Client** — Polls upstream every 250ms, maintains 10,000+ register cache
3. **Translator** — Applies device-specific maps (700G, GenSet, Black-Start)
4. **Device Maps** — JSON map files per device type (700g_map.json, genset_map.json, blackstart_map.json)
5. **Asyncio Engine** — Powers 200+ concurrent Modbus TCP servers with passthrough/static/synthetic modes
6. **HTTP Control Plane** — REST API for health, status, device control, and register access

**Data Flow:** Poll → Translate → Serve | 200+ devices on ports 1501-1701

Use case: FAT testing where multiple downstream systems need to connect to simulated individual devices, all sourced from a single upstream Modbus source.

## Stack

| Component | Technology |
|-----------|------------|
| Runtime | Python 3.11/12 |
| Modbus Client | pymodbus 3.x (`ModbusTcpClient`) |
| Modbus Servers | pymodbus 3.x (`ModbusSimulatorServer`) |
| Concurrency | asyncio (single event loop, 200+ servers) |
| Config | Pydantic models, JSON validation |
| HTTP API | FastAPI + Uvicorn |

## 3-Step Data Flow

### Step 1: Modbus TCP Client Polling

| Parameter | Value | Description |
|-----------|-------|-------------|
| Upstream Host | 192.168.40.100 | SEL RTAC or other Modbus TCP server |
| Upstream Port | 502 | Standard Modbus TCP port |
| Poll Interval | 250ms | Configurable refresh rate |
| Function | Read Holding Registers | Primary data source |
| Read Size | 10,000+ registers | Full upstream dataset |
| Library | pymodbus 3.x | ModbusTcpClient |

The client maintains an in-memory **register cache** of all upstream register values. This cache is the single source of truth for all downstream translations.

### Step 2: Translator

The translator **maps to device types**, converting upstream register data into device-specific layouts. Each device type has its own map file:

| Device Type | Map File | Port Range | Count |
|-------------|----------|------------|-------|
| 700G | `700g_map.json` | 1501-1600 | 100 |
| GenSet | `genset_map.json` | 1601-1650 | 50 |
| Black-Start | `blackstart_map.json` | 1651-1701 | 51 |
| (Extensible) | Custom maps | — | Additional device types |

**Translation Rules:**
- Source register address → Target register address mapping
- Data type conversion (if needed)
- Scaling factor application
- Boolean/bitfield extraction
- Default value injection for missing registers

### Step 3: Asyncio Engine + Modbus TCP Servers

**Asyncio Engine:**
- Powers 200+ concurrent servers on single event loop
- pymodbus 3.x server implementation
- Supports three runtime modes per device:
  - `passthrough` — Direct translation from upstream cache
  - `static` — Frozen values, ignore upstream changes
  - `synthetic` — Generated values (counters, sine waves)

**200+ Modbus TCP Servers:**
- Unique IP:Port (192.168.40.100:1501 through :1701)
- Device-specific register map (from Step 2)
- Isolated datastore per device
- Individual mode control per device

## Port Allocation

| Port Range | Count | Purpose |
|------------|-------|---------|
| 1501-1600 | 100 | 700G standard devices |
| 1601-1650 | 50 | GenSet controllers |
| 1651-1701 | 51 | Black-Start GenSet |

**Note**: 201 total devices (including Server 200 on port 1701). Port ranges map to device types for organization.

## Register Maps (Per Device Type)

Each device type has a different register layout. All compiled from `full-modbus-map.xlsx`.

### 700G Map (Standard Device)

| Function | Range | Count | Data Types |
|----------|-------|-------|------------|
| Coils | 0-100 | 100 | Boolean |
| Discrete Inputs | 10001-10100 | 100 | Boolean |
| Input Registers | 30001-30500 | 500 | Float, Int |
| Holding Registers | 40001-40500 | 500 | Float, Int |

### GenSet Map

| Function | Range | Count | Data Types |
|----------|-------|-------|------------|
| Coils | 0-50 | 50 | Boolean |
| Discrete Inputs | 10001-10080 | 80 | Boolean |
| Input Registers | 30001-30200 | 200 | Float, Int |
| Holding Registers | 40001-40250 | 250 | Float, Int |

### Black-Start GenSet Map

| Function | Range | Count | Data Types |
|----------|-------|-------|------------|
| Coils | 0-75 | 75 | Boolean |
| Discrete Inputs | 10001-10150 | 150 | Boolean |
| Input Registers | 30001-30800 | 800 | Float, Int |
| Holding Registers | 40001-41000 | 1000 | Float, Int |

## Runtime Modes (Per Device)

| Mode | Behavior | Use Case |
|------|----------|----------|
| `passthrough` | Direct translation from upstream cache | Normal operation |
| `static` | Frozen values, ignore upstream changes | Testing/debugging |
| `synthetic` | Generated values (counters, sine waves) | Offline simulation |

Mode can be switched per-device at runtime via HTTP API.

## HTTP Control Plane

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/health` | GET | Liveness probe |
| `/status` | GET | Client status, translator state, server counts |
| `/upstream` | GET | Current upstream connection status, last poll time |
| `/devices` | GET | List all 200+ devices: port, type, mode, connection_count |
| `/registers` | GET | Query: `port`, `function`, `address`. Returns register values |
| `/write` | POST | Inject register value: `{"port": 1501, "address": 40001, "value": 123.4}` |
| `/mode` | POST | Switch device mode: `{"port": 1501, "mode": "synthetic"}` |
| `/refresh` | POST | Force immediate upstream poll |

**Admin/Tester HTTP API Client** connects on port **8080** (configurable).

## Config Schema (`config.json`)

```json
{
  "http": {"port": 8080, "host": "0.0.0.0"},
  "upstream": {
    "host": "192.168.40.100",
    "port": 502,
    "unit_id": 1,
    "poll_interval_ms": 250,
    "read_start": 40001,
    "read_count": 10000
  },
  "devices": [
    {"port": 1501, "type": "700g", "mode": "passthrough"},
    {"port": 1502, "type": "700g", "mode": "passthrough"},
    ...
    {"port": 1601, "type": "genset", "mode": "passthrough"},
    ...
    {"port": 1651, "type": "blackstart", "mode": "passthrough"}
  ],
  "maps": {
    "700g": "700g_map.json",
    "genset": "genset_map.json",
    "blackstart": "blackstart_map.json"
  }
}
```

## Excel to JSON Compilation

`compile_maps.py` converts `full-modbus-map.xlsx` → multiple device-type JSON files:

| Excel Column | JSON Field | Notes |
|--------------|------------|-------|
| Signal Name | `name` | Unique identifier |
| Category | `device_type` | 700G, GenSet, Black-Start, etc. |
| MODBUS Type | `function` | Coil/Discrete/Input/Holding |
| Register Address | `source_address` | Address in upstream data |
| Target Address | `target_address` | Address in simulated device |
| Data Type | `datatype` | bool, float32, int16, int32 |
| Scale | `scale` | Multiplier for value translation |

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

## File Structure

```
modbus-sim/
├── simulator.py           # Main asyncio launcher
├── client.py              # Modbus TCP client (Step 1)
├── translator.py          # Device-type translator (Step 2)
├── server.py              # Modbus TCP server factory (Step 3)
├── device.py              # Per-server device instance
├── config.py              # Pydantic models
├── compiler/
│   ├── compile_maps.py    # Excel → multiple JSON maps
│   └── validators.py      # Address conflict detection
├── maps/
│   ├── full-modbus-map.xlsx   # Source of truth
│   ├── 700g_map.json          # Compiled 700G map
│   ├── genset_map.json        # Compiled GenSet map
│   └── blackstart_map.json    # Compiled Black-Start map
├── config.json            # 200+ device definitions
├── install_service.ps1    # NSSM install/uninstall
├── tests/
│   ├── test_client.py
│   ├── test_translator.py
│   ├── test_server.py
│   └── test_api.py
└── README.md              # Start/stop/verify commands
```

## Acceptance Criteria

| ID | Criteria | Verification |
|----|----------|--------------|
| A1 | Client polls upstream every 250ms | `GET /upstream` shows last_poll < 300ms |
| A2 | 200+ servers bind on ports 1501-1701 | `GET /devices` returns 200+ entries |
| A3 | Per-device isolated state | Write to port 1501; verify 1502 unchanged |
| A4 | Translator maps to correct device types | Verify port 1501 uses 700g_map, 1601 uses genset_map |
| A5 | Passthrough mode reflects upstream | Change upstream value; verify propagated to slaves within 500ms |
| A6 | HTTP `/registers?port=1501` returns values | JSON array with register metadata + current values |
| A7 | Runtime mode switch without restart | `POST /mode` → device changes passthrough→synthetic |
| A8 | Service auto-starts on Windows boot | Reboot; verify `GET /health` returns ok |
| A9 | <500 MB RAM for 200 active servers | Task Manager / `psutil` memory report |
| A10 | Excel compilation validates all maps | `compile_maps.py` exits non-zero on conflict |

## Verification Commands

```powershell
# Quick health check
curl http://localhost:8080/health

# Check upstream polling status
curl http://localhost:8080/upstream

# List all devices with types
curl http://localhost:8080/devices | ConvertFrom-Json

# Read specific register from simulated device on port 1501
python -c "from pymodbus.client import ModbusTcpClient; c=ModbusTcpClient('192.168.40.100', 1501); c.connect(); r=c.read_holding_registers(0, 2); print(r.registers); c.close()"

# Switch mode at runtime
Invoke-RestMethod -Uri http://localhost:8080/mode -Method Post -ContentType 'application/json' -Body '{"port": 1501, "mode": "synthetic"}'

# Force upstream refresh
Invoke-RestMethod -Uri http://localhost:8080/refresh -Method Post
```

