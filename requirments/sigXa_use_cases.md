# Use Case Specification — signalXautomotive
### Embedded Application: TC4 DRE ↔ Ethernet CAN Data Endpoint (Linux / QNX SOC)
> Part of the **signalXautomotive** Gateway Infrastructure
> Filled from [`../docs/templates/Use_Case_Template.md`](../docs/templates/Use_Case_Template.md). Sections marked _(AI-fillable)_ have been drafted by an AI assistant and require human review.

For background on the transport path and frame format, see:
- [`../docs/CAN_to_Ethernet_Protocol_Overview.md`](../docs/CAN_to_Ethernet_Protocol_Overview.md)
- [`../docs/IEEE_1722_ACF_CAN_BRIEF_Overview.md`](../docs/IEEE_1722_ACF_CAN_BRIEF_Overview.md)

---

## Scope & System Context

These use cases specify a portable software endpoint — the **signalXautomotive CAN-over-Ethernet endpoint** (`sigXa-endpoint`) — that runs as an application **on a System-on-Chip (SOC)** and exchanges CAN / CAN FD frames with vehicle CAN buses. The CAN traffic is routed by an **Infineon AURIX TC4 Data Routing Engine (DRE)**, which encapsulates each CAN frame as an **IEEE 1722-2016 AVTP ACF_CAN_BRIEF (`acf_format = 0x03`)** message and forwards it over Automotive Ethernet to the SOC. The same path is used in reverse for transmit.

```
┌───────────────┐   ACF_CAN_BRIEF over                ┌────────────────────────────┐
│  AURIX TC4    │   { UDP/IP } or { raw AVTP L2 }      │  SOC                       │
│  DRE Gateway  │ ──────────────────────────────────▶ │  sigXa-endpoint            │
│  (CAN ⇄ ETH)  │ ◀────────────────────────────────── │  (Linux or QNX application)│
└───────────────┘   100 / 1000BASE-T1                  └────────────────────────────┘
        │
   Physical CAN / CAN FD buses
```

**Transport options (both shall be supported):**

| Mode | Encapsulation | Destination | Typical use |
|---|---|---|---|
| **UDP** | `AVTP / ACF_CAN_BRIEF` inside `UDP / IP` (default UDP port `17220`) | IP unicast / multicast | Routed networks, easy host integration, SIL/test bench |
| **AVTP L2 (raw Ethernet)** | `AVTP / ACF_CAN_BRIEF` directly in an IEEE 802.3 frame (`EtherType 0x22F0`) | MAC unicast / multicast (optionally VLAN 802.1Q) | Lowest overhead/latency on the AVB/TSN backbone |

**In scope:** receiving and transmitting ACF_CAN_BRIEF frames on the SOC over UDP **or** AVTP L2; parsing/serializing the AVTP common header and ACF_CAN_BRIEF payload; delivering decoded CAN data to local consumers and accepting CAN data from local producers; operating identically on **Linux** and **QNX** via an OS/transport abstraction layer.

**Out of scope:** DRE hardware configuration and routing tables (handled on the TC4); SOME/IP / Signal-to-Service exposure (separate component); CAN signal-level decoding (factor/offset/byte-order — handled above this endpoint).

### Use Case Index

| ID | Name | Priority |
|---|---|---|
| [UC-001](#uc-001-receive-can-data-over-ethernet-dre--soc-application) | Receive CAN data over Ethernet (DRE → SOC application) | Must |
| [UC-002](#uc-002-transmit-can-data-over-ethernet-soc-application--dre--can) | Transmit CAN data over Ethernet (SOC application → DRE → CAN) | Must |
| [UC-003](#uc-003-run-the-endpoint-portably-on-linux-and-qnx) | Run the endpoint portably on Linux and QNX | Must |

---

## UC-001: Receive CAN data over Ethernet (DRE → SOC application)

| Field | Value |
|---|---|
| Use case ID | `UC-001` |
| Name | Receive CAN frame over Ethernet |
| Version | `0.1.0` |
| Status | `Draft` |
| Author(s) | signalXautomotive team (AI-drafted) |
| Created / Updated | `2026-06-06` / `2026-06-06` |
| Priority | `Must` |
| Related requirements | `REQ-FUN-001, REQ-FUN-002, REQ-FUN-004, REQ-OS-002, REQ-IF-001, REQ-ERR-001` |

---

### 1. Summary _(AI-fillable)_

The `sigXa-endpoint` application, running on the SOC (Linux or QNX), receives an IEEE 1722 ACF_CAN_BRIEF frame originated by a vehicle CAN bus and routed by the TC4 DRE over Automotive Ethernet. The frame arrives either as a UDP datagram or as a raw AVTP Layer-2 Ethernet frame. The endpoint validates the AVTP/ACF headers, decodes the CAN-ID, DLC and data, and delivers the decoded CAN frame to local consumers.

### 2. Actors

| Actor | Type | Role in this use case |
|---|---|---|
| AURIX TC4 DRE | Primary / External system | Source of ACF_CAN_BRIEF frames (encapsulates CAN → Ethernet) |
| `sigXa-endpoint` | System under design | Receives, validates, decodes and dispatches CAN frames on the SOC |
| Application consumer | Secondary | On-SOC software that uses the decoded CAN data (e.g., ADAS, logging, control) |

### 3. Goal & Value

- **Goal**: A CAN frame placed on the vehicle bus is made available, intact, to a consumer running on the SOC with bounded latency.
- **Trigger**: An ACF_CAN_BRIEF message arrives at the SOC — as a UDP datagram on the configured port, or as an AVTP L2 frame (EtherType `0x22F0`) on the bound interface.
- **Frequency / Rate**: Up to the aggregate CAN/CAN FD frame rate of all routed buses (design target: configurable, e.g., ≥ 20 000 frames/s aggregate).

### 4. Preconditions

- The Ethernet link between the TC4 DRE and the SOC is up (100BASE-T1 / 1000BASE-T1).
- The endpoint is configured with the transport mode (UDP or AVTP L2), the listening socket/interface, and the stream map (Stream ID → CAN channel).
- For UDP mode: the listening IP/port (and any multicast group) is configured; for AVTP L2 mode: the network interface and optional VLAN/MAC filter are configured.
- gPTP (IEEE 802.1AS) is available if AVTP timestamps are to be consumed (optional).

### 5. Postconditions

- **Success (guarantee)**: The CAN frame (CAN-ID, DLC, data, flags `eff/rtr/fdf/brs/esi`) is decoded and delivered to the consumer; the per-stream RX counter is incremented and, if present and valid, the AVTP timestamp is attached.
- **Failure (minimal guarantee)**: The offending frame is dropped, the relevant error/drop counter is incremented, and the endpoint and all other streams remain operational.

---

### 6. Main Success Flow

| Step | Actor | Action |
|---|---|---|
| 1 | AURIX TC4 DRE | Receives a CAN/CAN FD frame on a physical bus and encapsulates it as ACF_CAN_BRIEF over UDP/IP or AVTP L2. |
| 2 | `sigXa-endpoint` | Receives the datagram (UDP) or raw frame (AVTP L2) on the configured socket/interface. |
| 3 | `sigXa-endpoint` | Validates the AVTP common header (subtype, version, `sv`, `sequence_num`, `stream_id`, `acf_msg_length`). |
| 4 | `sigXa-endpoint` | Validates the ACF_CAN_BRIEF message header (`acf_format = 0x03`, length, `pad`, `can_bus_id`, optional `tv`/timestamp). |
| 5 | `sigXa-endpoint` | Parses the CAN frame descriptor: `eff`, `rtr`, `fdf`, `brs`, `esi`, `dlc`, `can_identifier` and the 0–64 B payload. |
| 6 | `sigXa-endpoint` | Resolves `stream_id` → logical CAN channel via the stream map and updates the per-stream RX counter / sequence tracking. |
| 7 | `sigXa-endpoint` | Delivers the decoded CAN frame (with optional timestamp) to the registered consumer(s). |

### 7. Alternate Flows

| ID | Branch point | Condition | Steps |
|---|---|---|---|
| A1 | Step 3 | Sequence number gap detected (`sequence_num` discontinuity or `mr=1`) | Log gap, increment gap counter, deliver current frame, continue. |
| A2 | Step 4 | `tv=1` (optional ACF 32-bit timestamp present) | Extract timestamp and attach it to the delivered frame. |
| A3 | Step 5 | `rtr=1` (remote frame, no payload) | Deliver frame with `dlc` as requested length and empty payload. |
| A4 | Step 6 | Unknown / unconfigured `stream_id` | Route to default handler or drop per configured policy; increment unmapped-stream counter. |
| A5 | Step 2 | Frame delivered to a multicast group (UDP or L2) | Accept if the endpoint has joined the group/MAC filter; otherwise ignore at the socket layer. |

### 8. Exception Flows

| ID | Branch point | Condition | Handling |
|---|---|---|---|
| E1 | Step 2 | Malformed, truncated, or oversized datagram/frame | Drop, increment error counter, do not crash (`REQ-ERR-001`). |
| E2 | Step 2 | DRE link down / no frames within timeout | Enter defined safe/degraded state, attempt reconnect, raise health event (`REQ-SAF-002`, `REQ-REL-001`). |
| E3 | Step 3 | AVTP header invalid (bad subtype/version/length) | Reject frame, increment header-error counter, continue. |
| E4 | Step 4 | `acf_format ≠ 0x03` (e.g., ACF_CAN `0x02`) | Reject as unsupported format, count, log once per stream. |
| E5 | Step 3–4 | SecOC verification fails (when SecOC is enabled) | Reject frame, raise security event (`REQ-SEC-001`). |
| E6 | Step 2 | Receive-buffer / socket-queue exhaustion under peak load | Drop deterministically, increment overflow counter (`REQ-ERR-002`, `REQ-PERF-002`). |

---

### 9. Data / Interface Details

| Item | Value |
|---|---|
| Transport | **UDP** over IPv4/IPv6 (default port `17220`, configurable) **or** **raw AVTP L2** (IEEE 802.3, EtherType `0x22F0`, optional VLAN 802.1Q) |
| Encapsulation | `IEEE 1722-2016 AVTP — ACF_CAN_BRIEF (acf_format = 0x03)` |
| Key fields used | `stream_id`, `sequence_num`, `avtp_timestamp` (if `tv=1`), `can_bus_id`, `eff/rtr/fdf/brs/esi`, `dlc`, `can_identifier`, payload |
| Config inputs | transport mode, listen socket / interface, multicast group / MAC filter, VLAN, stream map (Stream ID → CAN channel), policies |
| Outputs | decoded CAN frame to consumer(s), RX/drop/gap/error counters, log & diagnostic events |

### 10. Non-Functional Constraints

| Aspect | Constraint |
|---|---|
| Latency | Ethernet RX → consumer handoff ≤ target (e.g., ≤ 1 ms; tighter for safety-critical streams) — to be set in requirements. |
| Throughput | Sustain aggregate routed CAN/CAN FD rate without loss (e.g., ≥ 20 000 frames/s). |
| Safety | Defined safe state on link loss; per-stream isolation so one bad stream cannot affect others (ASIL level project-defined). |
| Security | Optional SecOC verification and MACsec termination supported; network input treated as untrusted and bounds-checked. |
| Resource budget | RAM/ROM/CPU budget per deployment target (project-defined). |

---

### 11. Acceptance Criteria _(AI-fillable)_

- Given a valid ACF_CAN_BRIEF frame received over **UDP**, when it is processed, then the decoded CAN-ID, DLC and data match the source CAN frame and the RX counter increments by 1.
- Given the identical CAN frame received over **AVTP L2**, when it is processed, then the decoded result is identical to the UDP case (transport-independent decoding).
- Given a CAN FD frame (`fdf=1`, up to 64 B), when received, then the full payload and `brs/esi` flags are delivered intact.
- Given a malformed or oversized frame, when received, then it is dropped, the error counter increments, and other streams continue to be delivered without interruption.
- Given a sequence-number gap, when detected, then the gap counter increments and the current frame is still delivered.

### 12. Open Questions / Assumptions

- TBD: default UDP port and whether unicast, multicast, or both are used per deployment.
- TBD: exact latency and throughput budgets per platform (Linux vs QNX) and per stream class.
- Assumption: the DRE emits only ACF_CAN_BRIEF (`0x03`), per the protocol overview; ACF_CAN (`0x02`) is treated as unsupported on RX.

### 13. Traceability

| Use Case Step | Requirement ID(s) | Test Case ID(s) |
|---|---|---|
| `6 / steps 2–5` | `REQ-FUN-001, REQ-FUN-002` | `TBD-TC-…` |
| `6 / step 6` | `REQ-FUN-004, REQ-FUN-005` | `TBD-TC-…` |
| `8 / E1, E6` | `REQ-ERR-001, REQ-ERR-002` | `TBD-TC-…` |
| `8 / E2` | `REQ-SAF-002, REQ-REL-001` | `TBD-TC-…` |
| `8 / E5` | `REQ-SEC-001` | `TBD-TC-…` |

---

## UC-002: Transmit CAN data over Ethernet (SOC application → DRE → CAN)

| Field | Value |
|---|---|
| Use case ID | `UC-002` |
| Name | Transmit CAN frame over Ethernet |
| Version | `0.1.0` |
| Status | `Draft` |
| Author(s) | signalXautomotive team (AI-drafted) |
| Created / Updated | `2026-06-06` / `2026-06-06` |
| Priority | `Must` |
| Related requirements | `REQ-FUN-003, REQ-FUN-004, REQ-OS-002, REQ-IF-001, REQ-IF-004, REQ-ERR-001` |

---

### 1. Summary _(AI-fillable)_

A producer on the SOC hands a CAN frame (CAN-ID, DLC, data, flags) to the `sigXa-endpoint`. The endpoint serializes it into an IEEE 1722 ACF_CAN_BRIEF message, wraps it for the selected transport (UDP/IP or raw AVTP L2), and transmits it to the TC4 DRE, which decapsulates it and places the CAN frame on the target physical CAN bus.

### 2. Actors

| Actor | Type | Role in this use case |
|---|---|---|
| Application producer | Primary | On-SOC software that emits CAN frames to be sent on the vehicle bus |
| `sigXa-endpoint` | System under design | Serializes and transmits ACF_CAN_BRIEF frames from the SOC |
| AURIX TC4 DRE | External system | Receives ACF_CAN_BRIEF over Ethernet and emits the CAN frame on the target bus |

### 3. Goal & Value

- **Goal**: A CAN frame produced on the SOC is delivered onto the correct physical CAN bus via the DRE with bounded latency.
- **Trigger**: A producer calls the endpoint's transmit API with a CAN frame and a target logical CAN channel / stream.
- **Frequency / Rate**: Up to the configured TX rate per stream (project-defined; subject to DRE-side rate shaping).

### 4. Preconditions

- The Ethernet link between the SOC and the TC4 DRE is up.
- The transmit transport mode (UDP or AVTP L2) and destination (IP/port or MAC, optional VLAN) are configured.
- The stream map associates the logical CAN channel with a `stream_id` and `can_bus_id` recognized by the DRE routing table.
- A per-stream outgoing `sequence_num` counter is initialized.

### 5. Postconditions

- **Success (guarantee)**: A well-formed ACF_CAN_BRIEF frame is transmitted on the selected transport; the per-stream TX counter and `sequence_num` are incremented.
- **Failure (minimal guarantee)**: The frame is not transmitted (or is dropped), the TX-error counter is incremented, the producer is notified per the API contract, and the endpoint remains operational.

---

### 6. Main Success Flow

| Step | Actor | Action |
|---|---|---|
| 1 | Application producer | Submits a CAN frame (CAN-ID, DLC, data, `eff/rtr/fdf/brs/esi`) and target logical channel to the endpoint's TX API. |
| 2 | `sigXa-endpoint` | Resolves the channel → `stream_id` / `can_bus_id` via the stream map and validates the request (DLC vs payload length, flag consistency). |
| 3 | `sigXa-endpoint` | Builds the ACF_CAN_BRIEF message: CAN frame descriptor + payload + `pad` for 4-byte alignment. |
| 4 | `sigXa-endpoint` | Builds the AVTP common header: `subtype`, `version`, `sv=1`, `sequence_num` (incremented), `stream_id`, `acf_format=0x03`, `acf_msg_length`; sets `tv`/timestamp if required. |
| 5 | `sigXa-endpoint` | Wraps for the selected transport — UDP/IP datagram to the configured endpoint, or raw AVTP L2 frame (EtherType `0x22F0`, optional VLAN tag) — and sends it. |
| 6 | AURIX TC4 DRE | Receives, validates and decapsulates the frame, then emits the CAN/CAN FD frame on the target physical bus. |

### 7. Alternate Flows

| ID | Branch point | Condition | Steps |
|---|---|---|---|
| A1 | Step 4 | Timestamping requested and gPTP locked | Set `tv=1` and insert the 32-bit timestamp; otherwise leave `tv=0`. |
| A2 | Step 1 | `rtr=1` requested (remote frame) | Build descriptor with requested `dlc` and empty payload. |
| A3 | Step 5 | Multicast/broadcast destination configured | Send to the configured multicast IP group (UDP) or multicast MAC (L2). |
| A4 | Step 5 | VLAN/PCP priority configured (AVTP L2) | Insert 802.1Q tag with the configured PCP for the traffic class. |

### 8. Exception Flows

| ID | Branch point | Condition | Handling |
|---|---|---|---|
| E1 | Step 2 | Unknown channel / no stream mapping | Reject the request, return error to producer, increment config-error counter. |
| E2 | Step 2 | Invalid request (DLC/payload mismatch, illegal flag combination) | Reject, return error, increment validation-error counter (`REQ-FUN-004`). |
| E3 | Step 5 | Socket/interface send failure or TX queue full | Drop or back-pressure per policy, increment TX-drop counter, notify producer (`REQ-ERR-001, REQ-ERR-002`). |
| E4 | Step 5 | DRE link down | Buffer per policy or fail fast; enter degraded state and attempt reconnect (`REQ-SAF-002`, `REQ-REL-001`). |

---

### 9. Data / Interface Details

| Item | Value |
|---|---|
| Transport | **UDP** over IPv4/IPv6 (configurable destination/port) **or** **raw AVTP L2** (IEEE 802.3, EtherType `0x22F0`, optional VLAN 802.1Q PCP) |
| Encapsulation | `IEEE 1722-2016 AVTP — ACF_CAN_BRIEF (acf_format = 0x03)` |
| Key fields produced | `stream_id`, `sequence_num`, optional `avtp_timestamp`/`tv`, `can_bus_id`, `eff/rtr/fdf/brs/esi`, `dlc`, `can_identifier`, payload, `pad` |
| Config inputs | transport mode, destination (IP/port or MAC), VLAN/PCP, stream map (channel → Stream ID / CAN bus) |
| Outputs | transmitted Ethernet frame to DRE, TX/TX-error/drop counters, log & diagnostic events |

### 10. Non-Functional Constraints

| Aspect | Constraint |
|---|---|
| Latency | Producer submit → frame on wire ≤ target (project-defined). |
| Throughput | Sustain configured per-stream TX rate without internal loss. |
| Safety | Deterministic behavior on link loss / queue full; no unbounded buffering. |
| Security | Support SecOC tagging and MACsec on egress where required; never log full payloads/keys at default level. |
| Resource budget | Bounded TX buffers; project-defined RAM/CPU budget. |

---

### 11. Acceptance Criteria _(AI-fillable)_

- Given a valid CAN frame submitted for transmit over **UDP**, when sent, then a well-formed ACF_CAN_BRIEF frame is observed on the wire with matching CAN-ID/DLC/data and the TX counter increments by 1.
- Given the same submission over **AVTP L2**, when sent, then the on-wire ACF_CAN_BRIEF payload is byte-identical to the UDP case (transport-independent serialization).
- Given consecutive transmits on one stream, when sent, then `sequence_num` increments and wraps correctly per stream.
- Given an unknown target channel, when submitted, then the request is rejected with an error and no frame is transmitted.
- Given a CAN FD frame up to 64 B, when transmitted, then the DRE emits a CAN FD frame with the correct `fdf/brs/esi` and payload on the target bus.

### 12. Open Questions / Assumptions

- TBD: back-pressure vs drop policy when the TX queue is full.
- TBD: whether TX timestamping (`tv=1`) is required and under what conditions.
- Assumption: DRE routing tables already map the transmitted `stream_id`/`can_bus_id` to the intended physical bus.

### 13. Traceability

| Use Case Step | Requirement ID(s) | Test Case ID(s) |
|---|---|---|
| `6 / steps 2–5` | `REQ-FUN-003` | `TBD-TC-…` |
| `6 / step 2` | `REQ-FUN-004` | `TBD-TC-…` |
| `8 / E3, E4` | `REQ-ERR-001, REQ-ERR-002, REQ-SAF-002` | `TBD-TC-…` |
| `7 / A1` | `REQ-IF-004` | `TBD-TC-…` |

---

## UC-003: Run the endpoint portably on Linux and QNX

| Field | Value |
|---|---|
| Use case ID | `UC-003` |
| Name | Portable SOC deployment on Linux and QNX |
| Version | `0.1.0` |
| Status | `Draft` |
| Author(s) | signalXautomotive team (AI-drafted) |
| Created / Updated | `2026-06-06` / `2026-06-06` |
| Priority | `Must` |
| Related requirements | `REQ-OS-002, REQ-OS-003, REQ-LANG-001, REQ-IF-003, REQ-TOOL-001` |

---

### 1. Summary _(AI-fillable)_

The same `sigXa-endpoint` source base is built and deployed on two SOC operating systems — **Linux** and **QNX** — and provides identical CAN receive (UC-001) and transmit (UC-002) behavior over both UDP and AVTP L2 transports. OS- and transport-specific details (socket APIs, raw L2 access, timing) are isolated behind an abstraction layer so application consumers are unaffected by the host OS.

### 2. Actors

| Actor | Type | Role in this use case |
|---|---|---|
| Integrator | Primary | Builds and deploys the endpoint on a Linux or QNX SOC |
| `sigXa-endpoint` | System under design | Provides OS-independent CAN-over-Ethernet RX/TX on the SOC |
| Application consumer / producer | Secondary | Uses the same API regardless of host OS |

### 3. Goal & Value

- **Goal**: One portable endpoint runs unchanged (at the API/behavior level) on both Linux and QNX SOCs.
- **Trigger**: The integrator builds and starts the endpoint on a target SOC running Linux or QNX.
- **Frequency / Rate**: Per deployment / integration cycle.

### 4. Preconditions

- A supported toolchain is available for each target OS.
- The OS provides the required networking primitives: UDP sockets and raw Layer-2 frame access (e.g., `AF_PACKET` on Linux; the equivalent raw/packet interface on QNX), plus optional gPTP support.
- Endpoint configuration (transport mode, sockets/interfaces, stream map) is supplied via the common configuration interface.

### 5. Postconditions

- **Success (guarantee)**: The endpoint starts on the target OS and performs UC-001 and UC-002 with identical, transport-independent behavior; platform differences are confined to the abstraction layer.
- **Failure (minimal guarantee)**: If a required OS facility is unavailable, the endpoint fails to start with a clear diagnostic and does not partially run.

---

### 6. Main Success Flow

| Step | Actor | Action |
|---|---|---|
| 1 | Integrator | Builds the endpoint for the target OS using the common build system. |
| 2 | Integrator | Provides configuration (transport mode, listen/destination, stream map) and starts the endpoint. |
| 3 | `sigXa-endpoint` | Selects the OS-specific transport backend (UDP and/or AVTP L2) via the abstraction layer. |
| 4 | `sigXa-endpoint` | Opens sockets/interfaces and begins RX (UC-001) and/or TX (UC-002) processing. |
| 5 | Application consumer / producer | Interacts through the same OS-independent API and observes identical CAN behavior. |

### 7. Alternate Flows

| ID | Branch point | Condition | Steps |
|---|---|---|---|
| A1 | Step 3 | Only UDP transport configured | Skip raw L2 backend initialization; run UDP-only. |
| A2 | Step 3 | Only AVTP L2 transport configured | Skip UDP backend initialization; run L2-only. |
| A3 | Step 4 | Both transports configured | Initialize both backends concurrently. |

### 8. Exception Flows

| ID | Branch point | Condition | Handling |
|---|---|---|---|
| E1 | Step 4 | Raw L2 access not permitted (e.g., insufficient privileges/capabilities) | Fail to start with a clear error, or fall back to UDP only if policy allows; log the cause. |
| E2 | Step 4 | Configured interface/port unavailable | Fail to start with a clear diagnostic; do not partially initialize. |
| E3 | Step 1 | Toolchain/feature unsupported on target OS | Build fails with an explicit, actionable message. |

---

### 9. Data / Interface Details

| Item | Value |
|---|---|
| Transport | UDP/IP and/or raw AVTP L2 — selectable per deployment, behavior identical across OS |
| Encapsulation | `IEEE 1722-2016 AVTP — ACF_CAN_BRIEF (acf_format = 0x03)` |
| OS targets | Linux and QNX (versions to be fixed in requirements) |
| Config inputs | common configuration file/format (transport mode, sockets/interfaces, stream map, filters) |
| Outputs | identical consumer/producer API, RX/TX counters, logs/diagnostics |

### 10. Non-Functional Constraints

| Aspect | Constraint |
|---|---|
| Portability | OS-specific code confined to a thin abstraction layer; no OS-specific behavior leaks to the application API. |
| Latency / Throughput | Targets of UC-001/UC-002 met on each supported OS. |
| Safety | Deterministic startup/failure behavior on each OS. |
| Security | Least-privilege networking (e.g., scoped capabilities for raw L2). |
| Resource budget | Per-OS RAM/ROM/CPU budgets (project-defined). |

---

### 11. Acceptance Criteria _(AI-fillable)_

- Given the same source base, when built and run on Linux and on QNX, then UC-001 and UC-002 acceptance tests pass identically on both.
- Given UDP-only configuration, when started, then the endpoint runs without requiring raw L2 access.
- Given AVTP L2 configuration without sufficient privileges, when started, then the endpoint reports a clear error (and falls back to UDP only if policy permits) rather than crashing.
- Given identical input frames, when decoded on Linux vs QNX, then the delivered CAN data is byte-identical (no OS-dependent decoding differences).

### 12. Open Questions / Assumptions

- TBD: exact supported Linux kernel and QNX versions.
- TBD: raw L2 access mechanism and privilege model on QNX.
- Assumption: both target OSes provide UDP sockets and a raw/packet L2 interface plus optional gPTP.

### 13. Traceability

| Use Case Step | Requirement ID(s) | Test Case ID(s) |
|---|---|---|
| `6 / steps 1, 3` | `REQ-OS-002, REQ-OS-003, REQ-TOOL-001` | `TBD-TC-…` |
| `6 / steps 4–5` | `REQ-OS-003, REQ-IF-003` | `TBD-TC-…` |
| `8 / E1, E2` | `REQ-ERR-001` | `TBD-TC-…` |
