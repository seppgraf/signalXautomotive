# Use Case Specification — Template
### Embedded Application: TC4 DRE ↔ Ethernet Data Endpoint
> Part of the **signalXautomotive** Gateway Infrastructure
> This is a **template**. Copy it once per use case and fill in every `<…>` placeholder. Sections marked _(AI-fillable)_ may be drafted or expanded by an AI assistant and then reviewed by a human. See [`README.md`](./README.md).

---

## How to use this template

- Copy this file to `docs/use-cases/UC-<NNN>_<short-name>.md`.
- Give the use case a **stable ID** (`UC-NNN`) — referenced from the requirements traceability table.
- Keep flows numbered so steps can be cited individually (e.g., `UC-001 / 3.2`).
- Link every use case back to the requirements it satisfies under **Traceability**.

---

## UC-`<NNN>`: `<Use case name>`

| Field | Value |
|---|---|
| Use case ID | `UC-<NNN>` |
| Name | `<short, action-oriented name, e.g., "Receive CAN frame over Ethernet">` |
| Version | `<0.1.0>` |
| Status | `Draft` |
| Author(s) | `<name(s)>` |
| Created / Updated | `<YYYY-MM-DD>` / `<YYYY-MM-DD>` |
| Priority | `<Must / Should / Could>` |
| Related requirements | `REQ-FUN-001, <…>` |

---

### 1. Summary _(AI-fillable)_

`<One or two sentences describing the goal of this use case from the actor's perspective. E.g., "The application receives an IEEE 1722 ACF_CAN_BRIEF frame emitted by the TC4 DRE, validates it, and makes the decoded CAN data available to consumers.">`

### 2. Actors

| Actor | Type | Role in this use case |
|---|---|---|
| `<TC4 DRE>` | `Primary / External system` | `<source/sink of Ethernet frames>` |
| `<COMPONENT_NAME>` | `System under design` | `<…>` |
| `<Consumer / Operator>` | `Secondary` | `<…>` |

### 3. Goal & Value

- **Goal**: `<what success looks like>`
- **Trigger**: `<what initiates the use case, e.g., "An ACF_CAN_BRIEF UDP datagram arrives on the configured port.">`
- **Frequency / Rate**: `<e.g., up to N frames/s>`

### 4. Preconditions

- `<e.g., Ethernet link to the TC4 DRE is up (100/1000BASE-T1).>`
- `<e.g., Stream/endpoint configuration is loaded.>`
- `<…>`

### 5. Postconditions

- **Success (guarantee)**: `<e.g., The CAN frame (ID, DLC, data) is decoded and delivered; counters updated.>`
- **Failure (minimal guarantee)**: `<e.g., The frame is dropped, the error counter is incremented, and the system remains operational.>`

---

### 6. Main Success Flow

| Step | Actor | Action |
|---|---|---|
| 1 | `<TC4 DRE>` | `<Sends an ACF_CAN_BRIEF frame over UDP/IP.>` |
| 2 | `<COMPONENT_NAME>` | `<Receives the datagram on the configured socket.>` |
| 3 | `<COMPONENT_NAME>` | `<Validates AVTP header (subtype, version, sequence number, length).>` |
| 4 | `<COMPONENT_NAME>` | `<Parses ACF_CAN_BRIEF payload (CAN-ID, DLC, data).>` |
| 5 | `<COMPONENT_NAME>` | `<Delivers the decoded frame to <consumer> and updates RX counters.>` |
| n | `<…>` | `<…>` |

### 7. Alternate Flows

| ID | Branch point | Condition | Steps |
|---|---|---|---|
| A1 | Step `<3>` | `<Sequence number gap detected>` | `<Log gap, update counter, continue.>` |
| A2 | Step `<4>` | `<Unknown stream ID>` | `<Route to default handler / drop per policy.>` |
| `<…>` | `<…>` | `<…>` | `<…>` |

### 8. Exception Flows

| ID | Branch point | Condition | Handling |
|---|---|---|---|
| E1 | Step `<2>` | `<Malformed / oversized datagram>` | `<Drop, count, do not crash (REQ-ERR-001).>` |
| E2 | Step `<2>` | `<DRE link down>` | `<Enter <safe state>, attempt reconnect (REQ-SAF-002).>` |
| E3 | Step `<3>` | `<SecOC verification fails>` | `<Reject frame, raise security event (REQ-SEC-001).>` |
| `<…>` | `<…>` | `<…>` | `<…>` |

---

### 9. Data / Interface Details

| Item | Value |
|---|---|
| Transport | `<UDP / TCP>` over `<IPv4 / IPv6>`, port `<…>` |
| Encapsulation | `IEEE 1722 ACF_CAN_BRIEF` |
| Key fields used | `<Stream ID, Sequence Num, CAN-ID, DLC, Data, Timestamp>` |
| Config inputs | `<stream map, endpoints, filters>` |
| Outputs | `<decoded frame to consumer, counters, log/diag events>` |

### 10. Non-Functional Constraints

| Aspect | Constraint |
|---|---|
| Latency | `<≤ value µs/ms>` |
| Throughput | `<frames/s or Mbit/s>` |
| Safety | `<ASIL-x behavior, safe state>` |
| Security | `<SecOC/MACsec expectations>` |
| Resource budget | `<RAM/ROM/CPU>` |

---

### 11. Acceptance Criteria _(AI-fillable)_

- `<Given … When … Then …>`
- `<Given a valid ACF_CAN_BRIEF frame, when received, then the decoded CAN-ID and data match the source and RX counter increments by 1.>`
- `<Given a malformed frame, when received, then it is dropped and the error counter increments without affecting other streams.>`

### 12. Open Questions / Assumptions

- `<TBD …>`

### 13. Traceability

| Use Case Step | Requirement ID(s) | Test Case ID(s) |
|---|---|---|
| `6 / step 2–3` | `REQ-FUN-001, REQ-FUN-002` | `<TC-…>` |
| `8 / E1` | `REQ-ERR-001` | `<TC-…>` |
| `<…>` | `<…>` | `<…>` |
