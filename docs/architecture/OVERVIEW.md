# PrometheusPLC — Architecture Overview

## 1. System Context

PrometheusPLC operates at the control layer of an industrial automation system, sitting between field devices (sensors, actuators) and supervisory systems (SCADA, MES, cloud platforms).

```
┌─────────────────────────────────────────────────────────────────┐
│                     Supervisory / Cloud Layer                   │
│              SCADA  │  MES  │  Historian  │  Cloud IoT          │
└──────────┬──────────┴───────┴─────────────┴─────────────────────┘
           │  OPC UA / MQTT / REST
┌──────────▼──────────────────────────────────────────────────────┐
│                       PrometheusPLC                             │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    Web IDE (React/TS)                    │    │
│  │   Program Editor  │  Variable Monitor  │  Diagnostics   │    │
│  └─────────────────────────┬───────────────────────────────┘    │
│                             │ REST / WebSocket                  │
│  ┌──────────────────────────▼──────────────────────────────┐    │
│  │                    API Gateway                          │    │
│  │        Authentication  │  Rate Limiting  │  Routing     │    │
│  └──┬──────────────┬──────────────┬───────────────┬────────┘    │
│     │              │              │               │             │
│  ┌──▼───┐   ┌──────▼─────┐  ┌────▼─────┐  ┌─────▼──────┐      │
│  │Compiler│  │Task        │  │Protocol  │  │Config &    │      │
│  │Service │  │Scheduler   │  │Manager   │  │Persistence │      │
│  └──┬───┘   └──────┬─────┘  └────┬─────┘  └─────┬──────┘      │
│     │              │              │               │             │
│  ┌──▼──────────────▼──────────────▼───────────────▼──────┐      │
│  │                  Runtime Engine                       │      │
│  │  Memory Manager  │  I/O Image  │  Watchdog  │  Clock  │      │
│  └──────────────────────────┬────────────────────────────┘      │
│                             │                                   │
│  ┌──────────────────────────▼────────────────────────────┐      │
│  │              Hardware Abstraction Layer (HAL)          │      │
│  │   GPIO Driver  │  Serial/RS-485  │  SPI/I2C  │  Sim   │      │
│  └──────────────────────────┬────────────────────────────┘      │
└─────────────────────────────┼───────────────────────────────────┘
                              │  Electrical signals
┌─────────────────────────────▼───────────────────────────────────┐
│                        Field Layer                              │
│           Sensors  │  Actuators  │  VFDs  │  Valves             │
└─────────────────────────────────────────────────────────────────┘
```

## 2. Core Components

### 2.1 Runtime Engine

The runtime engine is the heart of PrometheusPLC. Written in Rust, it provides:

- **Scan Cycle Executor** — Repeatedly executes: Read Inputs → Execute Program → Write Outputs.
- **Process Image (I/O Image)** — A consistent snapshot of all I/O values, updated atomically at the start and end of each cycle.
- **Memory Manager** — Manages PLC memory areas: `%I` (inputs), `%Q` (outputs), `%M` (markers/flags), and user-defined variables.
- **Watchdog Supervisor** — Monitors scan cycle duration and triggers fault handling if deadlines are exceeded.
- **Clock & Timekeeping** — Monotonic clock for timers, counters, and cycle timing using `clock_gettime(CLOCK_MONOTONIC)`.

**Key Design Decisions:**
- The runtime uses `SCHED_FIFO` real-time scheduling on Linux with `PREEMPT_RT`.
- Memory is pre-allocated at startup to avoid runtime allocations (no heap allocation in the hot path).
- The process image uses double-buffering to provide consistent snapshots without locking.

### 2.2 Compiler Service

The compiler translates IEC 61131-3 source programs into executable code:

```
Source Code (ST/IL/LD/FBD/SFC)
        │
        ▼
   ┌─────────┐
   │  Lexer   │  → Token stream
   └────┬─────┘
        ▼
   ┌─────────┐
   │  Parser  │  → Abstract Syntax Tree (AST)
   └────┬─────┘
        ▼
   ┌──────────┐
   │ Semantic  │  → Typed AST (type checking, variable resolution)
   │ Analyzer  │
   └────┬──────┘
        ▼
   ┌──────────┐
   │ IR Gen    │  → Intermediate Representation (SSA form)
   └────┬──────┘
        ▼
   ┌──────────┐
   │ Optimizer │  → Optimized IR (constant folding, dead code elimination)
   └────┬──────┘
        ▼
   ┌──────────┐
   │ Code Gen  │  → Native machine code (via Cranelift or LLVM)
   └──────────┘
```

**Key Design Decisions:**
- The compiler produces native machine code (not interpreted) for maximum performance.
- Cranelift is the default code generation backend (fast compilation, no LLVM dependency).
- An interpreter mode is available for debugging and step-through execution.
- Graphical languages (LD, FBD, SFC) are stored as XML and converted to an intermediate textual form before parsing.

### 2.3 Task Scheduler

The task scheduler manages multiple PLC tasks with different priorities and cycle times:

| Task Type | Description | Example |
|-----------|-------------|---------|
| **Cyclic** | Executes at a fixed interval | 10ms main control loop |
| **Event-driven** | Executes on an external trigger (interrupt, variable change) | Emergency stop handler |
| **Freewheeling** | Executes as fast as possible (no fixed interval) | Communication housekeeping |

**Scheduling Algorithm:**
- Rate-monotonic scheduling (RMS) for cyclic tasks.
- Priority-based preemption for event-driven tasks.
- Freewheeling tasks run in the idle time between cyclic tasks.

### 2.4 Protocol Manager

The protocol manager hosts industrial communication drivers:

| Protocol | Role | Transport |
|----------|------|-----------|
| **Modbus TCP** | Server & Client | TCP/IP |
| **Modbus RTU** | Master & Slave | RS-485 Serial |
| **OPC UA** | Server & Client | TCP/IP (binary) |
| **EtherNet/IP** | Adapter & Scanner | TCP/UDP |
| **MQTT** | Publisher & Subscriber | TCP/IP + TLS |

Each protocol driver runs in its own thread and communicates with the runtime via the process image (shared memory with atomic access).

**Key Design Decisions:**
- Protocol threads are lower priority than the scan cycle thread (control logic always takes precedence).
- OPC UA implementation follows the OPC Foundation's specification; we use the `opcua` Rust crate as a foundation.
- All network protocols support TLS/SSL for encrypted communication.
- MQTT integration uses a configurable publish interval and topic mapping.

### 2.5 Web IDE

The web-based IDE is a single-page application (SPA) built with TypeScript and React:

| Feature | Description |
|---------|-------------|
| **ST Editor** | Monaco-based text editor with syntax highlighting, autocomplete, and error markers |
| **LD Editor** | Graphical ladder diagram editor with drag-and-drop rungs, contacts, and coils |
| **FBD Editor** | Graphical function block editor with wired connections |
| **Variable Table** | Declare and manage PLC variables, I/O mappings, and data types |
| **Online Monitor** | Real-time display of variable values during runtime execution |
| **Diagnostics** | System logs, alarm history, performance metrics, and resource usage |
| **Project Manager** | Create, save, open, and version-control PLC projects |

**Key Design Decisions:**
- The IDE communicates with the runtime exclusively via REST and WebSocket APIs (no direct memory access).
- The IDE is stateless — all state lives in the runtime's configuration store.
- Monaco Editor is used for ST editing (same engine as VS Code).
- Graphical editors (LD, FBD) use a custom canvas renderer with React for UI chrome.

### 2.6 Hardware Abstraction Layer (HAL)

The HAL provides a uniform interface for I/O hardware:

```rust
pub trait DigitalInput {
    fn read(&self) -> Result<bool, HalError>;
}

pub trait DigitalOutput {
    fn write(&mut self, value: bool) -> Result<(), HalError>;
}

pub trait AnalogInput {
    fn read(&self) -> Result<f64, HalError>;
    fn get_range(&self) -> (f64, f64);
}

pub trait AnalogOutput {
    fn write(&mut self, value: f64) -> Result<(), HalError>;
    fn get_range(&self) -> (f64, f64);
}
```

**HAL Implementations:**
- `hal-gpio` — Linux GPIO via `/sys/class/gpio` or `libgpiod`.
- `hal-spi` — SPI bus for ADC/DAC expansion boards.
- `hal-serial` — Serial/RS-485 for Modbus RTU.
- `hal-simulator` — Virtual I/O for development and testing (no hardware required).

### 2.7 Configuration & Persistence

All configuration is stored in human-readable TOML files:

```toml
# project.toml — PLC project configuration

[project]
name = "WaterTreatmentPlant"
version = "1.0.0"
author = "Jane Engineer"

[runtime]
default_cycle_time_ms = 10
watchdog_timeout_ms = 50
memory_size_kb = 256

[[tasks]]
name = "MainControl"
type = "cyclic"
interval_ms = 10
priority = 80
program = "main_control.st"

[[tasks]]
name = "Monitoring"
type = "cyclic"
interval_ms = 100
priority = 50
program = "monitoring.st"

[[io.digital_inputs]]
name = "StartButton"
address = "%IX0.0"
hal = "gpio"
pin = 17

[[io.digital_outputs]]
name = "PumpRelay"
address = "%QX0.0"
hal = "gpio"
pin = 27

[[protocols.modbus_tcp]]
enabled = true
bind_address = "0.0.0.0:502"
unit_id = 1

[[protocols.opcua]]
enabled = true
endpoint = "opc.tcp://0.0.0.0:4840"
security_policy = "Basic256Sha256"
```

## 3. Data Flow

### 3.1 Scan Cycle Data Flow

```
                    ┌──────────────────┐
                    │   HAL / Field    │
                    │   Devices        │
                    └────────┬─────────┘
                             │ Read physical inputs
                    ┌────────▼─────────┐
                    │  Input Image     │
                    │  (%I memory)     │
                    └────────┬─────────┘
                             │ Copy to process image
                    ┌────────▼─────────┐
                    │  Program         │
                    │  Execution       │
                    │  (compiled code) │
                    └────────┬─────────┘
                             │ Write results
                    ┌────────▼─────────┐
                    │  Output Image    │
                    │  (%Q memory)     │
                    └────────┬─────────┘
                             │ Write physical outputs
                    ┌────────▼─────────┐
                    │  HAL / Field     │
                    │  Devices         │
                    └──────────────────┘
```

### 3.2 Communication Data Flow

```
┌──────────┐     ┌──────────────┐     ┌───────────────┐
│ External  │────▶│  Protocol    │────▶│  Process      │
│ Client    │◀────│  Driver      │◀────│  Image        │
│ (SCADA)   │     │  (OPC UA)    │     │  (shared mem) │
└──────────┘     └──────────────┘     └───────────────┘
```

Protocol drivers read from and write to the process image. Writes from external clients take effect at the next scan cycle boundary (not mid-cycle).

## 4. Technology Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Runtime Engine | Rust | Memory safety, zero-cost abstractions, real-time capable |
| Compiler Backend | Cranelift | Fast compilation, no heavy LLVM dependency, Rust-native |
| Protocol Drivers | Rust | Consistency with runtime, safe concurrency with `tokio` |
| Web IDE Frontend | TypeScript + React | Industry-standard for web UIs, rich ecosystem |
| Web IDE Backend | Rust (Axum) | Unified language with runtime, high performance |
| Code Editor | Monaco Editor | VS Code quality editing in the browser |
| Configuration | TOML | Human-readable, well-supported in Rust ecosystem |
| Build System | Cargo (Rust), npm (IDE) | Standard tooling for each language |
| Testing | `cargo test`, Vitest | Native test frameworks |
| CI/CD | GitHub Actions | Integrated with repository hosting |

## 5. Deployment Architecture

### 5.1 Embedded / Edge Deployment

```
┌─────────────────────────────────────────┐
│          Industrial PC / Edge Device     │
│          (Linux + PREEMPT_RT)            │
│                                          │
│  ┌──────────────────────────────────┐    │
│  │     PrometheusPLC Runtime        │    │
│  │     (single binary, ~10 MB)      │    │
│  └──────────────────────────────────┘    │
│  ┌──────────────────────────────────┐    │
│  │     Web IDE (static files)       │    │
│  │     served by runtime's HTTP     │    │
│  └──────────────────────────────────┘    │
│                                          │
│  GPIO / SPI / Serial ──▶ Field Devices   │
│  Ethernet ──▶ Modbus TCP / OPC UA        │
└─────────────────────────────────────────┘
```

### 5.2 Development / Simulation Deployment

```
┌─────────────────────────────────────────┐
│          Developer Workstation           │
│          (Linux / macOS / WSL2)          │
│                                          │
│  ┌──────────────────────────────────┐    │
│  │     PrometheusPLC Runtime        │    │
│  │     (--simulator flag)           │    │
│  └──────────────────────────────────┘    │
│  ┌──────────────────────────────────┐    │
│  │     Simulated I/O (hal-sim)      │    │
│  │     Virtual sensors/actuators    │    │
│  └──────────────────────────────────┘    │
│                                          │
│  Browser ──▶ http://localhost:8080       │
└─────────────────────────────────────────┘
```

## 6. Security Architecture

See [SECURITY.md](../../SECURITY.md) for the full security policy and threat model.

**Defense in depth layers:**

1. **Network** — TLS for all protocol communication; firewall rules in deployment guide.
2. **Authentication** — API access requires authentication (token-based); OPC UA uses certificate-based auth.
3. **Authorization** — Role-based access control (Operator, Engineer, Administrator).
4. **Input Validation** — All external inputs (API, protocol messages, uploaded programs) are validated.
5. **Memory Safety** — Rust eliminates entire classes of vulnerabilities (buffer overflow, use-after-free).
6. **Audit Logging** — All configuration changes and program downloads are logged with timestamps and user identity.

## 7. Cross-Cutting Concerns

### 7.1 Logging

- Structured logging via the `tracing` crate.
- Log levels: `ERROR`, `WARN`, `INFO`, `DEBUG`, `TRACE`.
- Logs are written to stdout (for systemd/journald integration) and optionally to file.
- Runtime performance metrics are exposed via a Prometheus-compatible `/metrics` endpoint.

### 7.2 Error Handling

- The runtime distinguishes between **recoverable** errors (retry/degrade) and **fatal** errors (safe shutdown).
- Fault categories: I/O fault, communication fault, program fault, watchdog fault, system fault.
- Each fault category has a configurable response: log, alarm, safe-state, or shutdown.

### 7.3 Configuration Hot-Reload

- Variable value changes can be applied without stopping the runtime.
- Program updates require a controlled stop → download → restart cycle.
- Protocol configuration changes require a restart of the affected protocol driver only.

## 8. Future Architecture Considerations

| Feature | Architecture Impact | Timeline |
|---------|-------------------|----------|
| **Redundancy / Hot Standby** | Requires state synchronization between primary and backup runtimes | Post v1.0 |
| **Distributed I/O** | Remote I/O nodes communicating via EtherCAT or PROFINET | Post v1.0 |
| **Safety PLC (SIL 2/3)** | Requires dual-channel execution, diagnostic coverage, formal verification | Post v2.0 |
| **Container Orchestration** | Kubernetes deployment for multi-PLC management | Post v1.0 |
| **FPGA Acceleration** | Hardware-accelerated I/O processing for sub-millisecond cycles | Research phase |
