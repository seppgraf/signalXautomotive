# Requirements Specification — Template
### Embedded Application: TC4 DRE ↔ Ethernet Data Endpoint
> Part of the **signalXautomotive** Gateway Infrastructure
> This is a **template**. Fill in every `<…>` placeholder and every `TBD`. Sections marked _(AI-fillable)_ may be drafted or expanded by an AI assistant and then reviewed by a human. See [`README.md`](./README.md) for the intended workflow.

---

## How to use this template

- Copy this file to `docs/requirements/<COMPONENT_NAME>_Requirements.md`.
- Replace all `<placeholders>` and resolve all `TBD` markers.
- Keep the requirement **IDs** (`REQ-…`) stable once published — they are used for traceability.
- Each requirement uses **"shall"** for mandatory items, **"should"** for recommended items, and **"may"** for optional items.
- Do **not** delete a requirement row; instead mark its `Status` as `Deprecated` and keep the ID.

### Requirement attributes (column meaning)

| Attribute | Meaning |
|---|---|
| `ID` | Unique, stable identifier (e.g., `REQ-FUN-001`). |
| `Requirement` | A single, testable statement using shall/should/may. |
| `Priority` | `Must` / `Should` / `Could` / `Won't` (MoSCoW). |
| `Verification` | How it is checked: `Test` / `Analysis` / `Inspection` / `Demonstration`. |
| `Status` | `Draft` / `Reviewed` / `Approved` / `Implemented` / `Deprecated`. |
| `Source` | Origin: stakeholder, standard, or upstream document. |

---

## 1. Document Metadata

| Field | Value |
|---|---|
| Component name | `<COMPONENT_NAME>` |
| Document ID | `<DOC-ID>` |
| Version | `<0.1.0>` |
| Status | `Draft` |
| Author(s) | `<name(s)>` |
| Reviewer(s) | `<name(s)>` |
| Created | `<YYYY-MM-DD>` |
| Last updated | `<YYYY-MM-DD>` |
| Related documents | `../CAN_to_Ethernet_Protocol_Overview.md`, `../IEEE_1722_ACF_CAN_BRIEF_Overview.md`, `<links>` |

---

## 2. Purpose & Scope

**Purpose** _(AI-fillable)_: `<Describe what this embedded application does — e.g., "A software endpoint that receives and transmits CAN-over-Ethernet frames (IEEE 1722 ACF_CAN_BRIEF over UDP/IP) produced by an Infineon AURIX TC4 Data Routing Engine (DRE)." >`

**In scope**:
- `<e.g., Receiving IEEE 1722 ACF_CAN_BRIEF frames from the TC4 DRE over Automotive Ethernet>`
- `<e.g., Transmitting frames back toward the DRE / Ethernet backbone>`
- `<…>`

**Out of scope**:
- `<e.g., DRE hardware configuration / routing tables (handled on the TC4)>`
- `<e.g., SOME/IP service exposure (separate component)>`
- `<…>`

---

## 3. System Context

> Reference the architecture in [`../CAN_to_Ethernet_Protocol_Overview.md`](../CAN_to_Ethernet_Protocol_Overview.md).

```
┌───────────────┐   IEEE 1722 ACF / UDP / IP   ┌────────────────────────────┐
│  AURIX TC4    │ ───────────────────────────▶ │  <COMPONENT_NAME>          │
│  DRE Gateway  │ ◀─────────────────────────── │  (this embedded application)│
└───────────────┘   100/1000BASE-T1            └────────────────────────────┘
```

| Item | Value |
|---|---|
| Upstream peer | `Infineon AURIX TC4 DRE` |
| Transport | `<UDP / TCP>` over `<IPv4 / IPv6>` |
| Encapsulation | `IEEE 1722-2016 AVTP ACF — ACF_CAN_BRIEF` |
| Physical layer | `<100BASE-T1 / 1000BASE-T1>` |
| Deployment target | `<ECU / domain controller / test bench / SIL>` |

### 3.1 Actors / Stakeholders

| Actor | Role / Interest |
|---|---|
| `<Integrator>` | `<…>` |
| `<Safety engineer>` | `<…>` |
| `<…>` | `<…>` |

---

## 4. Platform & Environment Requirements

> These constrain *how* and *where* the application is built and run. Fill in each row.

### 4.1 Operating Systems / Runtime

| ID | Requirement | Priority | Verification | Status | Source |
|---|---|---|---|---|---|
| REQ-OS-001 | The application **shall** run on `<e.g., AUTOSAR Adaptive (POSIX PSE51/PSE52)>`. | `Must` | Test | Draft | `<…>` |
| REQ-OS-002 | The application **shall** support `<e.g., Linux (kernel <ver>), QNX <ver>, FreeRTOS <ver>, bare-metal>`. | `<…>` | Test | Draft | `<…>` |
| REQ-OS-003 | The application **should** be portable across the listed OSes via a hardware/OS abstraction layer. | `Should` | Analysis | Draft | `<…>` |

### 4.2 Programming Languages

| ID | Requirement | Priority | Verification | Status | Source |
|---|---|---|---|---|---|
| REQ-LANG-001 | The application **shall** be implemented in `<e.g., C++17 / C11 / Rust>`. | `Must` | Inspection | Draft | `<…>` |
| REQ-LANG-002 | Scripting / tooling **may** use `<e.g., Python 3.x>` for build and test only (not in the runtime image). | `May` | Inspection | Draft | `<…>` |
| REQ-LANG-003 | Use of `<language feature/library>` **shall** be restricted per `<safety subset, e.g., MISRA>`. | `<…>` | Inspection | Draft | `<…>` |

### 4.3 Coding Style & Static Analysis

| ID | Requirement | Priority | Verification | Status | Source |
|---|---|---|---|---|---|
| REQ-STYLE-001 | Code **shall** conform to `<e.g., MISRA C++:2023 / MISRA C:2012 / AUTOSAR C++14 guidelines>`. | `Must` | Inspection | Draft | `<…>` |
| REQ-STYLE-002 | Code **shall** pass `<e.g., clang-tidy / cppcheck / Polyspace / Coverity>` with zero `<severity>` findings. | `Must` | Test | Draft | `<…>` |
| REQ-STYLE-003 | Formatting **shall** be enforced by `<e.g., clang-format with the repo .clang-format>`. | `Should` | Test | Draft | `<…>` |
| REQ-STYLE-004 | Naming conventions **shall** follow `<convention>`. | `Should` | Inspection | Draft | `<…>` |

### 4.4 Toolchain, Build & Dependencies

| ID | Requirement | Priority | Verification | Status | Source |
|---|---|---|---|---|---|
| REQ-TOOL-001 | The build **shall** use `<e.g., CMake ≥ 3.x / Bazel>` and compiler `<e.g., GCC <ver> / Clang <ver> / Tasking>`. | `Must` | Demonstration | Draft | `<…>` |
| REQ-TOOL-002 | Third-party dependencies **shall** be limited to `<list / "none">` and pinned to fixed versions. | `Must` | Inspection | Draft | `<…>` |
| REQ-TOOL-003 | The build **shall** be reproducible and runnable in CI (`<system>`). | `Should` | Demonstration | Draft | `<…>` |

---

## 5. Functional Requirements

| ID | Requirement | Priority | Verification | Status | Source |
|---|---|---|---|---|---|
| REQ-FUN-001 | The application **shall** receive IEEE 1722 ACF_CAN_BRIEF frames from the TC4 DRE over `<UDP/IP>`. | `Must` | Test | Draft | `<…>` |
| REQ-FUN-002 | The application **shall** parse the AVTP common header and ACF_CAN_BRIEF payload (CAN-ID, DLC, data). | `Must` | Test | Draft | `<…>` |
| REQ-FUN-003 | The application **shall** transmit ACF_CAN_BRIEF frames toward the DRE / Ethernet backbone. | `Must` | Test | Draft | `<…>` |
| REQ-FUN-004 | The application **shall** validate `<sequence number / stream ID / length>` and handle malformed frames per REQ-ERR-*. | `Must` | Test | Draft | `<…>` |
| REQ-FUN-005 | The application **shall** support `<N>` concurrent AVTP streams / CAN channels. | `<…>` | Test | Draft | `<…>` |
| REQ-FUN-00x | `<…>` | `<…>` | `<…>` | Draft | `<…>` |

---

## 6. Interface Requirements

| ID | Requirement | Priority | Verification | Status | Source |
|---|---|---|---|---|---|
| REQ-IF-001 | The Ethernet interface **shall** support `<100BASE-T1 / 1000BASE-T1>`. | `Must` | Test | Draft | `<…>` |
| REQ-IF-002 | The application **shall** use `<VLAN 802.1Q / PCP priority <value>>` for `<traffic class>`. | `<…>` | Test | Draft | `<…>` |
| REQ-IF-003 | The application **shall** expose a configuration interface for `<stream IDs, endpoints, filters>` via `<file/format>`. | `Should` | Demonstration | Draft | `<…>` |
| REQ-IF-004 | Time synchronization **shall** use `<gPTP / IEEE 802.1AS>` where timestamps are required. | `<…>` | Test | Draft | `<…>` |

---

## 7. Performance & Timing Requirements

| ID | Requirement | Priority | Verification | Status | Source |
|---|---|---|---|---|---|
| REQ-PERF-001 | End-to-end processing latency (Ethernet RX → app handoff) **shall** be ≤ `<value µs/ms>`. | `Must` | Test | Draft | `<…>` |
| REQ-PERF-002 | The application **shall** sustain `<frames/s or Mbit/s>` without frame loss. | `Must` | Test | Draft | `<…>` |
| REQ-PERF-003 | Worst-case memory footprint **shall** be ≤ `<RAM / ROM budget>`. | `Should` | Analysis | Draft | `<…>` |
| REQ-PERF-004 | CPU utilization **shall** remain ≤ `<value %>` under peak load. | `Should` | Test | Draft | `<…>` |

---

## 8. Safety Requirements

> Reference the relevant functional-safety standard for your project.

| ID | Requirement | Priority | Verification | Status | Source |
|---|---|---|---|---|---|
| REQ-SAF-001 | The application **shall** comply with `<ISO 26262 ASIL-<x>>` for `<safety goal>`. | `<…>` | Analysis | Draft | `<…>` |
| REQ-SAF-002 | On loss of the DRE link, the application **shall** enter `<safe state>` within `<time>`. | `<…>` | Test | Draft | `<…>` |
| REQ-SAF-003 | The application **shall** detect and report `<E2E protection / CRC / counter>` violations. | `<…>` | Test | Draft | `<…>` |

---

## 9. Security Requirements

| ID | Requirement | Priority | Verification | Status | Source |
|---|---|---|---|---|---|
| REQ-SEC-001 | The application **shall** verify `<SecOC>` freshness/MAC on protected frames. | `<…>` | Test | Draft | `<…>` |
| REQ-SEC-002 | Ethernet traffic **shall** support `<MACsec (802.1AE)>` where required. | `<…>` | Test | Draft | `<…>` |
| REQ-SEC-003 | The application **shall not** log secrets, keys, or full payloads at default log level. | `Must` | Inspection | Draft | `<…>` |
| REQ-SEC-004 | Inputs from the network **shall** be treated as untrusted and bounds-checked. | `Must` | Inspection | Draft | `<…>` |

---

## 10. Reliability, Logging & Diagnostics

| ID | Requirement | Priority | Verification | Status | Source |
|---|---|---|---|---|---|
| REQ-REL-001 | The application **shall** recover from `<transient link loss>` without restart. | `Should` | Test | Draft | `<…>` |
| REQ-LOG-001 | The application **shall** provide structured logging at levels `<ERROR/WARN/INFO/DEBUG>`. | `Should` | Demonstration | Draft | `<…>` |
| REQ-DIAG-001 | The application **shall** expose `<counters/health endpoint>` for `<frames RX/TX, drops, errors>`. | `Should` | Demonstration | Draft | `<…>` |

---

## 11. Error Handling

| ID | Requirement | Priority | Verification | Status | Source |
|---|---|---|---|---|---|
| REQ-ERR-001 | Malformed or oversized frames **shall** be dropped and counted, never crash the application. | `Must` | Test | Draft | `<…>` |
| REQ-ERR-002 | Resource exhaustion (buffers, sockets) **shall** be handled deterministically. | `Must` | Test | Draft | `<…>` |

---

## 12. Standards & Compliance

| Standard | Applicability | Notes |
|---|---|---|
| IEEE 1722-2016 (AVTP/ACF) | `<Mandatory>` | ACF_CAN_BRIEF encapsulation |
| ISO 11898 (CAN / CAN FD) | `<…>` | Source frame semantics |
| ISO 26262 | `<ASIL-x>` | Functional safety |
| AUTOSAR (`<Classic / Adaptive>`) | `<…>` | Platform |
| `<MISRA / AUTOSAR C++14>` | `<Mandatory>` | Coding guidelines |
| IEEE 802.1AS / 802.1AE / 802.1Q | `<…>` | Time sync / security / VLAN |
| `<…>` | `<…>` | `<…>` |

---

## 13. Assumptions, Constraints & Open Questions

- **Assumptions**: `<…>`
- **Constraints**: `<…>`
- **Open questions (TBD)**: `<…>`

---

## 14. Traceability

| Requirement ID | Use Case ID(s) | Design / Module | Test Case ID(s) |
|---|---|---|---|
| `REQ-FUN-001` | `UC-001` | `<module>` | `<TC-…>` |
| `<…>` | `<…>` | `<…>` | `<…>` |

---

## 15. Glossary

| Term | Definition |
|---|---|
| ACF | AVTP Control Format (IEEE 1722) |
| AVTP | Audio Video Transport Protocol (IEEE 1722) |
| DRE | Data Routing Engine (AURIX TC4) |
| SecOC | Secure Onboard Communication |
| `<…>` | `<…>` |

---

## 16. References

- [`../CAN_to_Ethernet_Protocol_Overview.md`](../CAN_to_Ethernet_Protocol_Overview.md)
- [`../IEEE_1722_ACF_CAN_BRIEF_Overview.md`](../IEEE_1722_ACF_CAN_BRIEF_Overview.md)
- `<additional standards / datasheets / internal docs>`
