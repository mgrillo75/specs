# Modbus Multi-Port Simulator Module (MVP)

## Overview
Gateway-only Ignition 8.3.3 module that launches multiple Modbus TCP slaves on unique ports to simulate device endpoints for FAT testing.

## Core Requirements

- **Gateway-only module** — no Designer, Client, or Vision scope
- **5 Modbus TCP slaves** — bound to ports 5020-5024 on 0.0.0.0
- **Canned register data** — each slave serves distinct values encoding its port number
- **j2mod library** — handles Modbus TCP protocol
- **Lifecycle integration** — start on Gateway startup, clean shutdown on Gateway stop

## Register Map (Per Slave)

| Register | Type | Value |
|----------|------|-------|
| Holding Register 0 | uint16 | Port number (5020-5024) |
| Holding Register 1 | uint16 | 1Hz incrementing counter |
| Holding Register 2 | uint16 | 0xCAFE (sentinel) |
| Input Register 0 | uint16 | Slave index (0-4) |
| Coil 0 | bool | 1Hz toggle |
| Discrete Input 0 | bool | Always true |

## Acceptance Criteria

1. Module builds to unsigned `.modl` via Gradle
2. Installs on Ignition 8.3.3 Gateway
3. Log shows all 5 slaves bound successfully
4. External Modbus client can read unique port numbers from each slave
5. Clean shutdown with no socket leaks

## Deliverables

- Gradle project with Gateway hook class
- `ModbusSimulatorGatewayHook.java` — lifecycle management
- `SlaveBundle.java` — per-slave wrapper
- README with pymodbus verification one-liner
