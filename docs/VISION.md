# PrometheusPLC — Vision & Scope

## 1. Vision Statement

PrometheusPLC aims to be the leading open-source PLC platform that combines the determinism and reliability expected by industrial automation professionals with the transparency, extensibility, and modern engineering practices of open-source software.

We believe that industrial control logic should be:
- **Auditable** — Every line of control logic should be inspectable, version-controlled, and testable.
- **Portable** — Control programs should not be locked to a single vendor's hardware.
- **Secure** — Industrial systems must be defended with the same rigor as any internet-facing service.
- **Accessible** — World-class PLC tooling should be available to everyone, from Fortune 500 plants to university labs.

## 2. Problem Statement

Traditional PLC ecosystems suffer from:

1. **Vendor lock-in** — Programs written for Siemens, Rockwell, or Schneider PLCs are not portable. Switching vendors means rewriting control logic.
2. **Opaque firmware** — Proprietary runtimes cannot be audited for security vulnerabilities or logic errors beyond what the vendor exposes.
3. **Costly tooling** — Engineering software licenses (TIA Portal, Studio 5000, Unity Pro) cost thousands of dollars per seat.
4. **Limited IIoT integration** — Legacy PLCs were not designed for cloud connectivity, MQTT, or REST APIs.
5. **Security gaps** — Many fielded PLCs have no authentication, no encryption, and no firmware signing.

PrometheusPLC addresses these problems by providing a modern, open-source alternative built on safe foundations.

## 3. Target Users

### Primary Users

| Persona | Description | Key Needs |
|---------|-------------|-----------|
| **System Integrator** | Designs and commissions automation systems for end customers | Vendor-independent runtime, standard IEC 61131-3 programming, easy protocol configuration |
| **OEM Engineer** | Embeds control logic into machines and equipment | Lightweight runtime for embedded Linux, deterministic cycle times, flexible I/O mapping |
| **Plant Engineer** | Maintains and troubleshoots running automation systems | Online monitoring, diagnostics, audit trail, hot-swap program updates |

### Secondary Users

| Persona | Description | Key Needs |
|---------|-------------|-----------|
| **Educator / Researcher** | Teaches or studies industrial automation concepts | Free tooling, simulation mode, clear documentation |
| **IIoT Developer** | Integrates OT data with IT/cloud systems | MQTT, OPC UA, REST API, data export capabilities |
| **Security Auditor** | Assesses industrial control system security | Open-source code, documented threat model, compliance mapping |

## 4. Scope

### 4.1 In Scope

| Area | Description |
|------|-------------|
| **PLC Runtime Engine** | Deterministic scan cycle execution on Linux-based platforms (x86_64, ARM64) |
| **IEC 61131-3 Compiler** | Compilation of Structured Text (ST), Ladder Diagram (LD), Function Block Diagram (FBD), Instruction List (IL), and Sequential Function Chart (SFC) |
| **Standard Function Libraries** | IEC 61131-3 standard function blocks (timers, counters, math, string, type conversion) |
| **Protocol Drivers** | Modbus TCP, Modbus RTU, OPC UA (client + server), EtherNet/IP (CIP), MQTT |
| **Web IDE** | Browser-based development environment for program editing, compilation, download, and monitoring |
| **Hardware Abstraction Layer** | GPIO, SPI, I2C interfaces for embedded Linux; simulated I/O for development |
| **REST & WebSocket API** | Programmatic access to runtime state, configuration, and diagnostics |
| **Configuration & Persistence** | Project files, variable declarations, I/O mappings, task configurations stored in human-readable formats (TOML/YAML) |
| **Logging & Diagnostics** | Structured logging, alarm/event history, performance metrics |

### 4.2 Out of Scope (v1.0)

| Area | Rationale |
|------|-----------|
| **Safety PLC (SIL 2/3)** | Requires formal verification and TUV certification; planned for future phases |
| **Motion control** | Servo/stepper coordination is a separate domain; may be added post-v1.0 |
| **Custom FPGA/ASIC runtime** | Initial focus is on software runtime on commodity hardware |
| **Windows CE / VxWorks support** | Linux-first; RTOS support may be explored later |
| **DCS (Distributed Control System) features** | Batch management, recipe management, historian integration are future scope |
| **Physical hardware manufacturing** | PrometheusPLC is software-only; users bring their own hardware |

### 4.3 Assumptions

1. Target hardware runs a Linux-based OS with kernel >= 5.10.
2. Real-time performance is achieved via `PREEMPT_RT` kernel patches or similar mechanisms (not a hard RTOS guarantee).
3. Users have basic familiarity with IEC 61131-3 programming concepts.
4. Network connectivity is available for protocol communication (Modbus TCP, OPC UA, MQTT).
5. I/O hardware is accessible via standard Linux interfaces (GPIO, SPI, I2C, serial).

### 4.4 Constraints

1. **Determinism** — Scan cycle jitter must be < 1ms for cycle times >= 10ms on supported hardware.
2. **Memory safety** — The runtime core must be written in Rust to eliminate buffer overflows, use-after-free, and data races.
3. **Standards compliance** — All IEC 61131-3 implementations must pass the PLCopen conformance test suite (where publicly available).
4. **Backward compatibility** — Once v1.0 is released, the configuration file format and public API must follow SemVer for breaking changes.

## 5. Success Criteria

| Metric | Target | Timeframe |
|--------|--------|-----------|
| Can execute a basic ST program with simulated I/O | Yes | Milestone 1 (MVP) |
| Scan cycle jitter < 1ms at 10ms cycle time | Yes | Milestone 2 |
| Modbus TCP and OPC UA functional | Yes | Milestone 2 |
| Web IDE can edit, compile, and download ST programs | Yes | Milestone 3 |
| Passes IEC 61131-3 conformance tests for ST | Yes | Milestone 4 |
| 10+ external contributors | Yes | 12 months post-v1.0 |
| Deployed in at least 1 production environment | Yes | 18 months post-v1.0 |

## 6. Competitive Landscape

| Platform | Type | License | IEC 61131-3 | OPC UA | Web IDE | Rust/Memory-safe |
|----------|------|---------|-------------|--------|---------|-------------------|
| **PrometheusPLC** | Software PLC | Apache-2.0 | Full (planned) | Yes | Yes | Yes |
| OpenPLC | Software PLC | GPL-3.0 | Partial (ST, LD) | No | Basic | No (C/C++) |
| CODESYS | Soft PLC + IDE | Proprietary | Full | Yes | No (native IDE) | No |
| Siemens TIA Portal | Hardware PLC IDE | Proprietary | Full | Yes | No | No |
| Beckhoff TwinCAT | Soft PLC (Windows) | Proprietary | Full | Yes | No | No |
| Ignition (Inductive) | SCADA/HMI | Proprietary | No | Yes | Yes | No (Java) |

## 7. Glossary

| Term | Definition |
|------|------------|
| **PLC** | Programmable Logic Controller — a ruggedized computer for industrial automation |
| **IEC 61131-3** | International standard defining PLC programming languages |
| **ST** | Structured Text — a high-level, Pascal-like PLC programming language |
| **LD** | Ladder Diagram — a graphical PLC programming language resembling relay logic |
| **FBD** | Function Block Diagram — a graphical PLC programming language using connected blocks |
| **IL** | Instruction List — a low-level, assembly-like PLC programming language |
| **SFC** | Sequential Function Chart — a graphical language for sequential/state-based logic |
| **Scan Cycle** | One complete execution of the PLC program: read inputs → execute logic → write outputs |
| **OPC UA** | Open Platform Communications Unified Architecture — an industrial communication standard |
| **Modbus** | A serial/TCP communication protocol widely used in industrial automation |
| **HAL** | Hardware Abstraction Layer — an interface isolating hardware-specific code from application logic |
| **PREEMPT_RT** | Linux kernel patches enabling real-time scheduling guarantees |
| **SIL** | Safety Integrity Level — a measure of safety system reliability (IEC 61508) |
| **IEC 62443** | International standard for cybersecurity in industrial automation and control systems |
