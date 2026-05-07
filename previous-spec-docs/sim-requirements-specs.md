# Software Requirements Specification — Modbus TCP Device Simulator

## 1. Purpose

This document specifies requirements for a **Modbus TCP simulator** that exposes many logically identical virtual devices from a **single deployable application**, with optional HTTP control/observability and optional mirroring of an upstream Modbus source.

## 2. Scope

### 2.1 In scope

- Modbus TCP server behavior per simulated device (listener + address space).
- Shared register/coil map definition compiled from a structured source (e.g. Excel) to normalized JSON.
- Per-device volatile state for writable registers and coils.
- Configurable runtime value generation: static defaults, synthetic signals, or mirror from another Modbus TCP server.
- Containerized deployment; Kubernetes compatibility.
- HTTP endpoints for health, status, and device discovery (minimum set defined in §6).

### 2.2 Out of scope (initial release)

- One Kubernetes pod per simulated device (see §7.3).
- Modbus RTU/serial transport.
- Distinct per-device *behaviors* or map variants unless later specified (all instances share one compiled map template).

## 3. Definitions

| Term | Meaning |
|------|---------|
| **Simulated device** | One Modbus TCP endpoint (host + port) with its own address space state. |
| **Compiled map** | Normalized JSON (or equivalent) describing registers/coils and metadata, produced from the authoritative map source. |
| **Golden box** | Single container/process hosting the simulator application and all simulated device listeners for this deployment. |

## 4. Assumptions and constraints

- **A-1:** Many simulated devices are **logically identical**; one map template applies to all instances.
- **A-2:** Target execution environment is **Linux containers** (e.g. Docker) with optional **Kubernetes** orchestration.
- **A-3:** Python **3.11 or 3.12** is the implementation baseline unless formally changed.
- **C-1 (Linux ports):** TCP ports below **1024** are **privileged** for non-root processes. External “low” port numbers **MUST** be satisfied via Service/port mapping to high internal bind ports unless an approved security exception grants low-port bind capability.

## 5. Functional requirements

### 5.1 Modbus TCP serving

| ID | Requirement |
|----|-------------|
| **FR-01** | The system **MUST** implement Modbus TCP **server** functionality compatible with client read/write operations against each simulated device’s listener. |
| **FR-02** | The system **MUST** support **multiple** concurrent simulated devices within **one** application process, each with a distinct TCP listen address (host/port). |
| **FR-03** | Each simulated device **MUST** use the **same** compiled map template to define its address space layout. |
| **FR-04** | Writable coils and holding/input registers **MUST** be **isolated per simulated device**: a write on one device **MUST NOT** alter another device’s state. |
| **FR-05** | The system **SHOULD** use an **asynchronous** Modbus server implementation suitable for many concurrent listeners (e.g. pymodbus async patterns). |

### 5.2 Configuration and map compilation

| ID | Requirement |
|----|-------------|
| **FR-06** | Device instances, listener bindings, and runtime options **MUST** be driven by **validated** configuration (e.g. Pydantic models) using **YAML and/or JSON**. |
| **FR-07** | The authoritative register/coil map **MAY** be maintained in **Excel (`.xlsx`)**; the system **MUST** provide a compilation step that produces **normalized JSON** consumed at runtime. |
| **FR-08** | Invalid or incomplete map or config input **MUST** fail at startup or compile time with actionable errors (no partial silent defaults for required fields). |

### 5.3 Runtime modes

| ID | Requirement |
|----|-------------|
| **FR-09** | The system **MUST** support a **`static`** mode: values **MUST** match configured defaults until modified by a Modbus client write (where applicable). |
| **FR-10** | The system **MUST** support a **`synthetic`** mode: selected values **MUST** change over time according to documented, simple generation rules (exact algorithms **MAY** be implementation-defined but **MUST** be stable and configurable). |
| **FR-11** | The system **SHOULD** support a **`mirror`** mode: an **optional** Modbus TCP **client** **MUST** poll an upstream server and map received values into per-device state according to configuration. |
| **FR-12** | Upstream polling (**mirror**) **MUST** be **disableable** so a deployment can run with only static/synthetic behavior (no upstream dependency). |

### 5.4 HTTP control plane

| ID | Requirement |
|----|-------------|
| **FR-13** | The system **MUST** expose **`GET /health`** suitable for liveness/readiness probes. |
| **FR-14** | The system **MUST** expose **`GET /status`** reporting process-level simulator health and high-level runtime state. |
| **FR-15** | The system **MUST** expose **`GET /devices`** (or equivalent) listing configured simulated devices and their listen endpoints. |
| **FR-16** | Additional control or admin endpoints **MAY** be added; they **MUST** be documented in the implementation’s API reference. |

## 6. Interface requirements

### 6.1 External Modbus

- **IF-01:** Simulated devices **MUST** present Modbus TCP on the host/interface and ports defined in configuration.
- **IF-02:** In **`mirror`** mode, the system **MUST** document upstream connection parameters (host, port, unit/slave id as applicable, scan rate, and mapping rules).

### 6.2 HTTP

- **IF-03:** The HTTP API **MUST** be served over TCP (typical deployment: FastAPI with **Uvicorn**). TLS termination **MAY** be delegated to the platform (ingress/service mesh).

## 7. Non-functional requirements

### 7.1 Implementation stack

| ID | Requirement |
|----|-------------|
| **NFR-01** | The implementation **MUST** use Python **3.11 or 3.12**, pymodbus (async client/server), FastAPI, Uvicorn, and Pydantic unless formally waived. |
| **NFR-02** | The application **MUST** ship as a **container image** suitable for Kubernetes deployment. |
| **NFR-03** | The repository **MUST** include **pytest**-based **smoke tests** that exercise startup and minimal Modbus read/write paths. |

### 7.2 Deployment and security

| ID | Requirement |
|----|-------------|
| **NFR-04** | By default, the container **SHOULD** bind Modbus listeners to **non-privileged** high ports internally; **Kubernetes Service** (or equivalent) **SHOULD** map external port ranges (e.g. 501–700) to those internal targets. |
| **NFR-05** | If low-port bind inside the container is required, the deployment **MUST** document the exact Linux capability or security context and obtain approval; this **SHOULD** be avoided by default. |

### 7.3 Scalability and topology

| ID | Requirement |
|----|-------------|
| **NFR-06** | The default deployment model **MUST** be **one pod (or container)** running **one** application process that manages **all** simulated devices for that replica, **not** one pod per device. |
| **NFR-07** | A future **one-pod-per-device** topology **MAY** be introduced only if required for heterogeneous device behavior, strict failure isolation, or independent lifecycle per device; such a change **MUST** be specified in a revision to this document. |

## 8. Traceability note

Discussion-derived recommendations (single template, many listeners, optional upstream, port mapping on Linux) are captured above as **FR-**, **NFR-**, **IF-**, **A-**, and **C-** statements for review and test planning.
