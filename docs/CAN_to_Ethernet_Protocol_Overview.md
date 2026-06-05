# CAN → Ethernet Protocol Overview
### Infineon AURIX TC4 — Data Routing Engine (DRE)
> Part of the **signalXautomotive** Gateway Infrastructure — CAN to Ethernet Transport Layer

---

## 1. Introduction

This document describes the CAN / CAN FD to Automotive Ethernet **frame transport protocol** implemented entirely in hardware by the **Infineon AURIX TC4** microcontroller family via its integrated **Data Routing Engine (DRE)**. All conversion between CAN and Ethernet is performed exclusively by the DRE hardware — no software conversion is involved in the transport path described here.

The AURIX TC4 serves as a **zonal or domain gateway ECU** — bridging legacy CAN networks (body, chassis, powertrain) with the high-bandwidth Ethernet backbone used by modern ADAS, central compute, and cloud-connected architectures.

> **Scope:** This document covers only the CAN → Ethernet frame transport layer (HW DRE path). Signal-to-Service (S2S) normalization and service exposure are addressed in a separate document.

---

## 2. Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                        AURIX TC4 Gateway ECU                         │
│                                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                           │
│  │ CAN FD 0 │  │ CAN FD 1 │  │ CAN FD n │   ← Physical CAN Buses   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                           │
│       │              │              │                                 │
│  ─────┴──────────────┴──────────────┴─────────────────────────────── │
│                     Message Buffer (HW FIFO)                         │
│  ─────────────────────────────────────────────────────────────────── │
│                                                                      │
│          ┌───────────────────────────────────────┐                  │
│          │     DRE  (Data Routing Engine)         │                  │
│          │  ┌──────────┐   ┌────────────────────┐│                  │
│          │  │ Filter / │   │  Frame Encapsulation││                  │
│          │  │ Route    │──▶│  (CAN PDU →        ││                  │
│          │  │ Tables   │   │   SOME/IP / UDP)    ││                  │
│          │  └──────────┘   └────────────────────┘│                  │
│          │          ▲              │               │                  │
│          │          │      ┌───────▼──────────┐   │                  │
│          │          │      │  Security / Auth  │   │                  │
│          │          │      │  (SecOC / MACsec) │   │                  │
│          │          │      └───────────────────┘   │                  │
│          └───────────────────────┬────────────────┘                  │
│                                  │                                   │
│          ┌───────────────────────▼───────────────┐                  │
│          │       Ethernet MAC (100/1000BASE-T1)   │                  │
│          └───────────────────────────────────────┘                  │
│                                  │                                   │
└──────────────────────────────────┼──────────────────────────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │   Automotive Ethernet Spine  │
                    │  (100BASE-T1 / 1000BASE-T1)  │
                    └──────────────────────────────┘
```

---

## 3. CAN → Ethernet Frame Transport Flow

```
Step 1 ─ CAN Frame Reception
  ┌────────────────────────────────────────────────────────────────┐
  │  Physical CAN Bus (e.g., 500 kbit/s or 2/5/8 Mbit/s CAN FD)  │
  │  → CAN FD Module receives raw frame:                          │
  │     [ SOF | ID (11/29bit) | DLC | Data (0–64B) | CRC | EOF ] │
  └────────────────────────────────────────────────────────────────┘
                          │
                          ▼
Step 2 ─ Buffering & Filtering (Hardware FIFO)
  ┌────────────────────────────────────────────────────────────────┐
  │  DRE applies configurable filter/routing tables:              │
  │  • CAN-ID whitelist / blacklist                               │
  │  • CAN-ID → Ethernet destination mapping                      │
  │  • Rate limiting / debounce rules                             │
  └────────────────────────────────────────────────────────────────┘
                          │
                          ▼
Step 3 ─ Frame Payload Passthrough (HW — zero-copy)
  ┌────────────────────────────────────────────────────────────────┐
  │  The raw CAN frame payload (0–64 bytes) is passed through     │
  │  as-is to the Ethernet encapsulation stage.                   │
  │  • No signal-level decomposition                              │
  │  • No byte-order conversion or factor/offset scaling          │
  │  • DRE operates at PDU/frame level; CPU is not involved       │
  └────────────────────────────────────────────────────────────────┘
                          │
                          ▼
Step 4 ─ Ethernet Encapsulation (HW — DRE)
  ┌────────────────────────────────────────────────────────────────┐
  │  The CAN frame payload is encapsulated by the DRE HW into:   │
  │                                                               │
  │  SOME/IP Header (16 bytes):                                   │
  │  [ Service ID | Method ID | Length | Client ID |             │
  │    Session ID | Version | Msg Type | Return Code ]           │
  │                                                               │
  │  CAN-ID → SOME/IP Service/Method ID mapping is resolved      │
  │  entirely via the DRE HW routing table (no SW intervention).  │
  │                                                               │
  │  Wrapped in:  UDP/IP → Ethernet Frame (IEEE 802.3)            │
  └────────────────────────────────────────────────────────────────┘
                          │
                          ▼
Step 5 ─ Security Processing (optional, SecOC / MACsec — HW)
  ┌────────────────────────────────────────────────────────────────┐
  │  • SecOC: appends Freshness Value + MAC tag to payload        │
  │  • MACsec: frame-level encryption at Ethernet layer           │
  └────────────────────────────────────────────────────────────────┘
                          │
                          ▼
Step 6 ─ Ethernet Frame Transmission (100/1000BASE-T1)
  ┌────────────────────────────────────────────────────────────────┐
  │  [ Preamble | Dst MAC | Src MAC | EtherType |                 │
  │    IP Header | UDP Header | SOME/IP Header | Payload | FCS ]  │
  └────────────────────────────────────────────────────────────────┘
```

---

## 4. Reverse Path: Ethernet → CAN

```
Ethernet Frame arrives at MAC
          │
          ▼
SOME/IP / UDP Parsing (DRE HW)
  • Extract SOME/IP Service ID → map to CAN-ID via HW routing table
  • Extract raw payload bytes (no signal-level deserialization)
          │
          ▼
CAN Frame Assembly (DRE HW)
  • Insert raw payload bytes into CAN data field
  • Set CAN-ID and DLC per routing table entry
  • No endianness conversion or factor/offset scaling at this stage
          │
          ▼
CAN FD Module Transmission onto target CAN bus
```

---

## 5. Protocol Stack

```
┌──────────────────────────────────────────────┐
│      Transport / Network Layer                │
│   UDP · TCP · IP (IPv4 / IPv6)               │
├──────────────────────────────────────────────┤
│         Ethernet Data Link Layer              │
│   IEEE 802.3  /  VLAN (802.1Q)  /  AVB       │
├──────────────────────────────────────────────┤
│         Physical Layer                        │
│   100BASE-T1  /  1000BASE-T1  (BroadR-Reach) │
└──────────────────────────────────────────────┘
```

> **Note:** Application-layer software components (CAN Gateway SWC, AUTOSAR PDU Router, SOME/IP-SD) are **not** part of the HW transport path described in this document and are out of scope here.

---

## 6. Key Protocols & Standards

| Protocol | Layer | Role in Gateway |
|---|---|---|
| **CAN / CAN FD** | Physical + Data Link | Source bus (up to 64 B payload at 8 Mbit/s) |
| **SOME/IP** | Application | CAN PDU transport encapsulation over Ethernet |
| **DoIP** | Application | Diagnostic messages (UDS) over Ethernet *(diagnostic path only, separate from frame routing)* |
| **UDP / TCP** | Transport | SOME/IP transport; TCP for reliable channels |
| **IPv4 / IPv6** | Network | Addressing and routing |
| **VLAN (802.1Q)** | Data Link | Traffic segmentation / QoS |
| **100BASE-T1** | Physical | Single-pair automotive Ethernet (100 Mbit/s) |
| **1000BASE-T1** | Physical | Single-pair automotive Ethernet (1 Gbit/s) |
| **SecOC** | Security | CAN/Ethernet message authentication |
| **MACsec (802.1AE)** | Security | Ethernet frame-level encryption |

---

## 7. DRE — Data Routing Engine Details (AURIX TC4)

The DRE is a **hardware-accelerated routing engine** integrated into the AURIX TC4:

- **Zero-copy routing**: Frames are moved directly from CAN FIFO → ETH TX buffer without CPU intervention.
- **Parallel routing rules**: Supports hundreds of simultaneous routing table entries.
- **Rate shaping**: Can throttle high-frequency CAN frames before Ethernet forwarding.
- **Timestamping**: Hardware timestamps on both CAN and Ethernet side for latency measurement and time-synchronization (IEEE 802.1AS / gPTP).
- **Interrupt offloading**: DRE triggers CPU only on exceptions (filter miss, overflow), minimizing jitter.

### DRE Routing Table Entry Structure

| Field | Size | Description |
|---|---|---|
| `CAN_BUS_ID` | 4 bit | Source CAN bus index |
| `CAN_MSG_ID` | 29 bit | CAN identifier (11 or 29 bit) |
| `ETH_DST_MAC` | 48 bit | Target MAC address |
| `ETH_DST_IP` | 32/128 bit | Target IP address (v4/v6) |
| `ETH_DST_PORT` | 16 bit | UDP destination port |
| `SOMEIP_SVC_ID` | 16 bit | SOME/IP Service ID |
| `SOMEIP_MTH_ID` | 16 bit | SOME/IP Method/Event ID |
| `PRIORITY` | 3 bit | VLAN PCP / QoS class |
| `FLAGS` | 8 bit | SecOC, rate-limit, direction bits |

---

## 8. Timing & Latency Budget

| Stage | Typical Latency |
|---|---|
| CAN frame reception (HW) | ~1 µs |
| DRE filter + route lookup | ~2–5 µs |
| PDU transformation (HW) | ~3–8 µs |
| Ethernet frame assembly | ~2–4 µs |
| 100BASE-T1 TX | ~10 µs (per frame) |
| **Total CAN → Ethernet** | **~20–30 µs** |

> For safety-critical signals (ASIL-D), the full path latency must be validated against the worst-case reaction time of the receiving node.

---

## 9. Security Considerations

```
CAN Side                          Ethernet Side
─────────────────────────────     ─────────────────────────────
SecOC:                            MACsec (802.1AE):
 • Freshness counter              • Frame-level AES-GCM encryption
 • CMAC-based authentication      • Per-port key negotiation (MKA)
 • Prevents frame injection
                    ┌────────────────┐
                    │  Firewall /    │
                    │  IDS (DRE HW) │
                    └────────────────┘
                    • ID-based whitelist
                    • Anomaly rate detection
                    • Secure boot (HSM on TC4)
```

---

## 10. System Boundary

The AURIX TC4 DRE delivers raw CAN frame payloads, encapsulated in SOME/IP/UDP/IP, onto the Automotive Ethernet backbone. **The scope of this document ends at the Ethernet output of the TC4 gateway.**

Signal-to-Service (S2S) normalization, service discovery, and exposure to application consumers (ADAS, Cloud, OTA, Digital Twin) are addressed in a separate document.

---

## 11. References

- [Infineon AURIX TC4xx Product Family](https://www.infineon.com/cms/en/product/microcontroller/32-bit-tricore-microcontroller/aurix-family/aurix-tc4xx-family/)
- [AUTOSAR Classic Platform – COM Stack Specification](https://www.autosar.org/standards/classic-platform)
- [SOME/IP Protocol Specification (AUTOSAR_PRS_SOMEIPProtocol)](https://www.autosar.org/fileadmin/user_upload/standards/foundation/1-3/AUTOSAR_PRS_SOMEIPProtocol.pdf)
- [IEEE 802.1AE MACsec Standard](https://1.ieee802.org/security/802-1ae/)
- [IEEE 802.1AS – Timing and Synchronization (gPTP)](https://1.ieee802.org/tsn/802-1as/)
- [Open Alliance TC10 – Automotive Ethernet Wake-up](https://opensig.org/automotive-ethernet-specifications/)
