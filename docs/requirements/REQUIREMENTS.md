# PrometheusPLC — Requirements Specification

**Document Version:** 0.1.0
**Last Updated:** 2026-02-08
**Status:** Draft

---

## 1. Functional Requirements

### 1.1 Runtime Engine

| ID | Requirement | Priority | Acceptance Criteria |
|----|-------------|----------|---------------------|
| RT-001 | The runtime SHALL execute a cyclic scan: read inputs → execute program → write outputs | Must | Scan cycle completes within configured cycle time |
| RT-002 | The runtime SHALL support configurable cycle times from 1ms to 10,000ms | Must | Cycle time is set via configuration file and honored at runtime |
| RT-003 | The runtime SHALL support multiple concurrent tasks with independent cycle times and priorities | Must | At least 8 concurrent tasks with different intervals |
| RT-004 | The runtime SHALL provide a watchdog that detects scan cycle overruns | Must | Watchdog triggers fault if cycle exceeds configured timeout |
| RT-005 | The runtime SHALL transition to a safe state on unrecoverable fault | Must | Outputs go to configured safe values; fault is logged |
| RT-006 | The runtime SHALL support hot online variable value changes without stopping | Should | Variable values can be changed via API during execution |
| RT-007 | The runtime SHALL pre-allocate all memory at startup (no heap allocation in scan cycle) | Must | Verified by custom allocator instrumentation in tests |
| RT-008 | The runtime SHALL use real-time scheduling (SCHED_FIFO) on Linux | Must | Runtime thread runs at configured RT priority |
| RT-009 | The runtime SHALL expose a Prometheus-compatible metrics endpoint | Should | `/metrics` returns scan cycle time, jitter, memory usage |
| RT-010 | The runtime SHALL support graceful shutdown with output safe-state | Must | SIGTERM/SIGINT triggers safe shutdown sequence |

### 1.2 Compiler

| ID | Requirement | Priority | Acceptance Criteria |
|----|-------------|----------|---------------------|
| CC-001 | The compiler SHALL parse and compile Structured Text (ST) per IEC 61131-3 | Must | All ST language features from IEC 61131-3 Ed. 3 |
| CC-002 | The compiler SHALL parse and compile Ladder Diagram (LD) | Must | Standard contacts, coils, and function block invocations |
| CC-003 | The compiler SHALL parse and compile Function Block Diagram (FBD) | Should | Standard blocks with wired connections |
| CC-004 | The compiler SHALL parse Instruction List (IL) | Could | Basic IL instruction set |
| CC-005 | The compiler SHALL parse Sequential Function Chart (SFC) | Should | Steps, transitions, actions, divergences/convergences |
| CC-006 | The compiler SHALL produce native machine code via Cranelift | Must | Compiled programs run without interpreter overhead |
| CC-007 | The compiler SHALL provide an interpreter mode for debugging | Should | Step-through execution with variable inspection |
| CC-008 | The compiler SHALL report clear error messages with source locations | Must | Errors include file, line, column, and description |
| CC-009 | The compiler SHALL implement all IEC 61131-3 standard data types | Must | BOOL, BYTE, WORD, DWORD, LWORD, SINT, INT, DINT, LINT, USINT, UINT, UDINT, ULINT, REAL, LREAL, TIME, DATE, STRING |
| CC-010 | The compiler SHALL implement standard function blocks (TON, TOF, TP, CTU, CTD, CTUD, SR, RS) | Must | Behavior matches IEC 61131-3 specification |

### 1.3 Protocol Drivers

| ID | Requirement | Priority | Acceptance Criteria |
|----|-------------|----------|---------------------|
| PD-001 | Modbus TCP server SHALL expose PLC variables as Modbus registers | Must | Standard Modbus function codes (01, 02, 03, 04, 05, 06, 15, 16) |
| PD-002 | Modbus TCP client SHALL read/write registers from remote Modbus devices | Must | Configurable polling interval and register mapping |
| PD-003 | Modbus RTU master SHALL communicate over RS-485 serial | Should | Configurable baud rate, parity, stop bits |
| PD-004 | OPC UA server SHALL expose PLC variables as OPC UA nodes | Must | Browsable address space with read/write access |
| PD-005 | OPC UA server SHALL support security policies (Basic256Sha256) | Must | Certificate-based authentication and encrypted transport |
| PD-006 | OPC UA client SHALL read/write variables from remote OPC UA servers | Should | Configurable subscription and polling |
| PD-007 | MQTT client SHALL publish PLC variables to configurable topics | Should | Configurable publish interval, QoS, and topic mapping |
| PD-008 | MQTT client SHALL subscribe to topics and write values to PLC variables | Should | Configurable topic-to-variable mapping |
| PD-009 | EtherNet/IP adapter SHALL respond to CIP explicit and implicit messages | Could | Basic CIP object model |
| PD-010 | All network protocols SHALL support TLS encryption | Must | TLS 1.2+ for TCP-based protocols |

### 1.4 Web IDE

| ID | Requirement | Priority | Acceptance Criteria |
|----|-------------|----------|---------------------|
| IDE-001 | The IDE SHALL provide a Structured Text editor with syntax highlighting | Must | Keywords, types, comments, and strings highlighted |
| IDE-002 | The IDE SHALL provide autocomplete for ST keywords, variables, and function blocks | Should | Autocomplete suggestions appear on typing |
| IDE-003 | The IDE SHALL display compilation errors inline in the editor | Must | Error markers at the correct line/column |
| IDE-004 | The IDE SHALL provide a graphical Ladder Diagram editor | Should | Drag-and-drop rungs, contacts, coils |
| IDE-005 | The IDE SHALL provide online variable monitoring | Must | Real-time variable values updated via WebSocket |
| IDE-006 | The IDE SHALL support program download to the runtime | Must | Compile → stop → download → start workflow |
| IDE-007 | The IDE SHALL provide a variable declaration table | Must | Declare name, type, address, initial value, comment |
| IDE-008 | The IDE SHALL provide a diagnostics/log viewer | Should | Structured log display with filtering |
| IDE-009 | The IDE SHALL work in modern browsers (Chrome, Firefox, Edge) | Must | No browser plugins required |
| IDE-010 | The IDE SHALL support user authentication | Must | Login required; role-based access (Operator/Engineer/Admin) |

### 1.5 Hardware Abstraction Layer

| ID | Requirement | Priority | Acceptance Criteria |
|----|-------------|----------|---------------------|
| HAL-001 | The HAL SHALL provide a uniform trait interface for digital and analog I/O | Must | `DigitalInput`, `DigitalOutput`, `AnalogInput`, `AnalogOutput` traits |
| HAL-002 | The HAL SHALL include a GPIO driver for Linux GPIO subsystem | Must | Works with `libgpiod` on Raspberry Pi and industrial PCs |
| HAL-003 | The HAL SHALL include a simulator driver for development without hardware | Must | Configurable virtual I/O points |
| HAL-004 | The HAL SHALL include a serial driver for RS-485 communication | Should | Standard Linux serial interface |
| HAL-005 | The HAL SHALL support I/O point configuration via TOML files | Must | I/O mapping defined in project configuration |

---

## 2. Non-Functional Requirements

### 2.1 Performance

| ID | Requirement | Target | Measurement Method |
|----|-------------|--------|-------------------|
| NF-PERF-001 | Scan cycle jitter | < 1ms at 10ms cycle time | Statistical analysis over 10,000 cycles |
| NF-PERF-002 | Maximum scan cycle jitter | < 5ms at 10ms cycle time | Worst-case measurement over 1 hour |
| NF-PERF-003 | Program compilation time (1000 LOC ST) | < 2 seconds | Benchmark on reference hardware |
| NF-PERF-004 | Runtime memory footprint (idle) | < 50 MB RSS | Measured via `/proc/pid/status` |
| NF-PERF-005 | Runtime startup time | < 5 seconds | Time from process start to first scan cycle |
| NF-PERF-006 | Web IDE initial load time | < 3 seconds | Lighthouse performance audit |
| NF-PERF-007 | API response time (variable read) | < 10ms p99 | Load test with 100 concurrent requests |

### 2.2 Reliability

| ID | Requirement | Target |
|----|-------------|--------|
| NF-REL-001 | Runtime uptime without restart | > 30 days continuous operation |
| NF-REL-002 | Graceful degradation on I/O fault | Runtime continues with fault indication; affected I/O uses last known or safe value |
| NF-REL-003 | Automatic recovery from transient communication faults | Reconnect with exponential backoff |
| NF-REL-004 | No data loss on power failure | Configuration persisted to non-volatile storage; runtime state is ephemeral by design |

### 2.3 Security

| ID | Requirement | Priority |
|----|-------------|----------|
| NF-SEC-001 | All API endpoints SHALL require authentication | Must |
| NF-SEC-002 | Passwords SHALL be stored as salted hashes (Argon2) | Must |
| NF-SEC-003 | All network communication SHALL support TLS 1.2+ | Must |
| NF-SEC-004 | OPC UA SHALL use certificate-based authentication | Must |
| NF-SEC-005 | The runtime SHALL validate all external inputs (API, protocol, uploaded programs) | Must |
| NF-SEC-006 | The runtime SHALL log all authentication attempts and configuration changes | Must |
| NF-SEC-007 | Default installation SHALL have no anonymous access enabled | Must |
| NF-SEC-008 | The system SHALL support role-based access control (RBAC) | Must |

### 2.4 Portability

| ID | Requirement | Target |
|----|-------------|--------|
| NF-PORT-001 | Primary target: Linux x86_64 with PREEMPT_RT | Must support |
| NF-PORT-002 | Secondary target: Linux aarch64 (ARM64) | Must support |
| NF-PORT-003 | Development: macOS (Apple Silicon), Linux (non-RT), Windows (WSL2) | Should support (no RT guarantees) |
| NF-PORT-004 | Cross-compilation from x86_64 host to aarch64 target | Should support |

### 2.5 Maintainability

| ID | Requirement | Target |
|----|-------------|--------|
| NF-MAINT-001 | Code coverage (runtime + compiler) | > 80% line coverage |
| NF-MAINT-002 | All public APIs documented with rustdoc | 100% of public items |
| NF-MAINT-003 | CI pipeline passes on every pull request | Required for merge |
| NF-MAINT-004 | No `unsafe` Rust code without documented justification and safety invariant | 100% compliance |

---

## 3. Regulatory & Standards Compliance

### 3.1 IEC 61131-3 (PLC Programming Languages)

| Aspect | Target Compliance | Notes |
|--------|-------------------|-------|
| Structured Text (ST) | Full conformance | Primary language; all features implemented |
| Ladder Diagram (LD) | Full conformance | Graphical editor + IL/ST interop |
| Function Block Diagram (FBD) | Substantial conformance | Standard blocks; custom block support |
| Instruction List (IL) | Basic conformance | IL is deprecated in Ed. 3 but widely used |
| Sequential Function Chart (SFC) | Substantial conformance | Steps, transitions, actions |
| Standard data types | Full conformance | All elementary and derived types |
| Standard function blocks | Full conformance | Timers, counters, edge detection, bistables |

### 3.2 IEC 62443 (Industrial Cybersecurity)

| Security Level | Target | Notes |
|----------------|--------|-------|
| SL 1 — Protection against casual violation | v1.0 | Authentication, logging, input validation |
| SL 2 — Protection against intentional violation with low resources | v2.0 | Encrypted communication, RBAC, audit trail |
| SL 3 — Protection against sophisticated attack | Future | Formal security audit, penetration testing |

### 3.3 IEC 61508 / Safety (Future)

PrometheusPLC v1.0 is **not** designed for safety-critical applications (SIL-rated). Safety PLC functionality is planned for post-v2.0 and will require:
- Dual-channel program execution with comparison.
- Diagnostic coverage analysis.
- Formal verification of safety logic.
- Third-party certification (e.g., TUV).

---

## 4. Constraints & Dependencies

### 4.1 Technical Constraints

1. The runtime core MUST be implemented in Rust (memory safety requirement).
2. The runtime MUST NOT use heap allocation during scan cycle execution.
3. Configuration files MUST be in TOML format (human-readable, version-control friendly).
4. The Web IDE MUST be a single-page application served by the runtime's HTTP server (no separate deployment).

### 4.2 External Dependencies

| Dependency | Purpose | License | Risk |
|-----------|---------|---------|------|
| Cranelift | Code generation | Apache-2.0 | Low — Bytecode Alliance maintained |
| Tokio | Async runtime (protocols, API) | MIT | Low — widely adopted |
| Axum | HTTP framework | MIT | Low — Tokio ecosystem |
| opcua (Rust crate) | OPC UA implementation | MPL-2.0 | Medium — check compatibility |
| React | IDE frontend | MIT | Low |
| Monaco Editor | Code editor | MIT | Low — Microsoft maintained |

### 4.3 Hardware Dependencies

- Development: Any modern x86_64 or ARM64 machine.
- Production: Linux-based industrial PC or SBC with:
  - Kernel >= 5.10 (preferably with PREEMPT_RT patches)
  - At least 256 MB RAM
  - At least 100 MB storage
  - GPIO/SPI/I2C/Serial interfaces as needed
