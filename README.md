# PrometheusPLC

**An open-source, safety-focused Programmable Logic Controller runtime and development environment for industrial automation.**

[![CI](https://github.com/GeorgeMcIntyre-Web/PrometheusPLC/actions/workflows/ci.yml/badge.svg)](https://github.com/GeorgeMcIntyre-Web/PrometheusPLC/actions/workflows/ci.yml)
[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)

---

## What is PrometheusPLC?

PrometheusPLC is a modern, open-source PLC software platform that brings the reliability and determinism of traditional industrial controllers into a flexible, auditable, and extensible software stack. It targets edge devices, industrial PCs, and embedded Linux platforms to provide:

- **IEC 61131-3 compliant programming** — Structured Text (ST), Ladder Diagram (LD), Function Block Diagram (FBD), Instruction List (IL), and Sequential Function Chart (SFC).
- **Real-time scan cycle engine** — Deterministic task execution with configurable cycle times and watchdog supervision.
- **Industrial protocol support** — Modbus TCP/RTU, OPC UA (client & server), EtherNet/IP, and MQTT for IIoT integration.
- **Web-based IDE** — Browser-accessible development environment with program editing, online monitoring, and diagnostics.
- **Safety-first design** — Built with memory-safe Rust for the runtime core, designed with IEC 62443 cybersecurity principles.

## Who is it for?

- **System integrators** building custom automation solutions without vendor lock-in.
- **OEMs** embedding PLC functionality into their machines and equipment.
- **Plant engineers** seeking auditable, open-source control logic.
- **Educators and researchers** teaching and experimenting with industrial automation.
- **IIoT developers** bridging the gap between operational technology (OT) and IT systems.

## Key Differentiators

| Feature | PrometheusPLC | Traditional PLCs | OpenPLC |
|---------|---------------|------------------|---------|
| Language | Rust (memory-safe) | Proprietary firmware | C/C++ |
| IEC 61131-3 | Full (5 languages) | Full | Partial (ST, LD) |
| OPC UA | Native | Often add-on | No |
| Web IDE | Built-in | Separate software | Basic |
| Open source | Apache-2.0 | No | GPL-3.0 |
| Edge/Cloud ready | Yes | Limited | Limited |

## Architecture Overview

```
┌──────────────────────────────────────────────────┐
│                   Web IDE (React)                 │
│         Program Editor / Monitor / Debug          │
├──────────────────────────────────────────────────┤
│                  REST / WebSocket API             │
├────────────┬─────────────┬───────────────────────┤
│  Compiler  │  Task       │  Protocol             │
│  (ST/LD →  │  Scheduler  │  Drivers              │
│   IR →     │  (real-time │  (Modbus, OPC UA,     │
│   native)  │   cycles)   │   EtherNet/IP, MQTT)  │
├────────────┴─────────────┴───────────────────────┤
│              Runtime Engine (Rust)                │
│     Memory Manager / I/O Image / Watchdog        │
├──────────────────────────────────────────────────┤
│         Hardware Abstraction Layer (HAL)          │
│      GPIO / Fieldbus / Simulated I/O             │
└──────────────────────────────────────────────────┘
```

For the full architecture document, see [`docs/architecture/OVERVIEW.md`](docs/architecture/OVERVIEW.md).

## Project Status

> **Pre-release / Active Development** — PrometheusPLC is in its foundation phase. See the [Roadmap](ROADMAP.md) for milestone details.

## Quick Start

### Prerequisites

- [Rust](https://www.rust-lang.org/tools/install) >= 1.75 (stable)
- [Node.js](https://nodejs.org/) >= 20 LTS (for the Web IDE)
- [just](https://github.com/casey/just) (command runner, optional but recommended)
- Linux (primary target), macOS (development), Windows (WSL2)

### Build from source

```bash
# Clone the repository
git clone https://github.com/GeorgeMcIntyre-Web/PrometheusPLC.git
cd PrometheusPLC

# Build the runtime
cargo build --release

# Run unit tests
cargo test

# Build the Web IDE
cd ide
npm install
npm run build

# Start the runtime with the built-in simulator
cargo run --release -- --simulator
```

### Run with Docker (planned)

```bash
docker run -p 8080:8080 ghcr.io/georgemcintyre-web/prometheusplc:latest
```

## Project Structure

```
PrometheusPLC/
├── runtime/          # PLC runtime engine (Rust)
│   ├── src/
│   │   ├── engine/       # Scan cycle engine & task scheduler
│   │   ├── memory/       # Process image & memory management
│   │   ├── io/           # I/O subsystem & HAL
│   │   └── watchdog/     # Watchdog & fault handling
├── compiler/         # IEC 61131-3 compiler (Rust)
│   ├── src/
│   │   ├── lexer/        # Tokenizer for ST/IL
│   │   ├── parser/       # AST generation
│   │   ├── semantic/     # Type checking & validation
│   │   └── codegen/      # Code generation (IR → native)
├── protocols/        # Industrial protocol drivers (Rust)
│   ├── modbus/           # Modbus TCP & RTU
│   ├── opcua/            # OPC UA client & server
│   ├── ethernet_ip/      # EtherNet/IP (CIP)
│   └── mqtt/             # MQTT for IIoT
├── ide/              # Web-based IDE (TypeScript/React)
│   ├── src/
│   │   ├── editor/       # Program editor (ST, LD, FBD)
│   │   ├── monitor/      # Online variable monitoring
│   │   └── diagnostics/  # System diagnostics & logs
├── hal/              # Hardware Abstraction Layer
│   ├── gpio/             # GPIO drivers
│   ├── fieldbus/         # Fieldbus interfaces
│   └── simulator/        # Simulated I/O for development
├── docs/             # Documentation
│   ├── architecture/     # Architecture documents
│   ├── requirements/     # Requirements specifications
│   ├── adr/              # Architecture Decision Records
│   └── api/              # API documentation
├── tests/            # Test suites
│   ├── unit/             # Unit tests
│   ├── integration/      # Integration tests
│   └── hil/              # Hardware-in-the-loop tests
├── .github/          # GitHub configuration
│   ├── workflows/        # CI/CD pipelines
│   └── ISSUE_TEMPLATE/   # Issue templates
├── .gitignore
├── .editorconfig
├── Cargo.toml        # Rust workspace manifest
├── LICENSE
├── README.md
├── CONTRIBUTING.md
├── ROADMAP.md
├── CHANGELOG.md
└── SECURITY.md
```

## Documentation

| Document | Description |
|----------|-------------|
| [Vision & Scope](docs/VISION.md) | Project vision, target users, and scope boundaries |
| [Architecture Overview](docs/architecture/OVERVIEW.md) | System architecture and component design |
| [Requirements](docs/requirements/REQUIREMENTS.md) | Functional and non-functional requirements |
| [Testing Strategy](docs/TESTING_STRATEGY.md) | Test approach, coverage targets, and HIL plan |
| [Security](SECURITY.md) | Security policy, threat model, and vulnerability reporting |
| [Roadmap](ROADMAP.md) | Development milestones and timeline |
| [Contributing](CONTRIBUTING.md) | How to contribute to PrometheusPLC |
| [Changelog](CHANGELOG.md) | Release history and notable changes |

## Contributing

We welcome contributions! Please read our [Contributing Guide](CONTRIBUTING.md) before submitting pull requests.

All contributors must adhere to our [Code of Conduct](CODE_OF_CONDUCT.md).

## License

PrometheusPLC is licensed under the [Apache License 2.0](LICENSE).

```
Copyright 2026 PrometheusPLC Contributors

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0
```

## Acknowledgments

PrometheusPLC builds on the shoulders of the industrial automation community and is inspired by projects like [OpenPLC](https://openplcproject.com/), the [IEC 61131-3 standard](https://www.plcopen.org/), and the Rust embedded ecosystem.

---

**PrometheusPLC** — _Bringing the fire of open-source to industrial automation._
