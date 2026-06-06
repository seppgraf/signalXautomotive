# IEEE 1722 AVTP Control Frame (ACF) — ACF_CAN_BRIEF: Detailed Overview
### AURIX TC4 DRE Support Analysis
> Part of the **signalXautomotive** Gateway Infrastructure — CAN to Ethernet Transport Layer

---

## 1. Standard Context

IEEE 1722-2016 defines the **Audio Video Transport Protocol (AVTP)** — a Layer 2 transport standard for time-sensitive data over Ethernet (AVB/TSN). Its **AVTP Control Frame (ACF)** sub-specification extends this to carry non-A/V control data, most importantly CAN traffic.

Two ACF formats are defined for CAN bridging:

| ACF Format Code | Name | Description |
|---|---|---|
| `0x02` | **ACF_CAN** | Full format — includes optional per-message 64-bit hardware timestamp (`mtv`/`message_timestamp` fields) |
| `0x03` | **ACF_CAN_BRIEF** | **Compact format** — omits the per-message timestamp; lower per-frame overhead, preferred for gateway applications |

The AURIX TC4 DRE hardware implements **`ACF_CAN_BRIEF` (`0x03`)** — the compact variant.

---

## 2. Complete PDU Bit Map

An ACF_CAN_BRIEF frame is transmitted inside a standard Ethernet/UDP/IP frame. The full stack from Ethernet to CAN payload:

```
┌─────────────────────────────────────────────────────────────────────────┐
│  L2: Ethernet Frame                                                      │
│  [ Dst MAC | Src MAC | EtherType (0x0800/0x86DD) ]                       │
├─────────────────────────────────────────────────────────────────────────┤
│  L3: IP Header (IPv4 / IPv6)                                             │
├─────────────────────────────────────────────────────────────────────────┤
│  L4: UDP Header (Dst Port: typically 17220 / configurable)               │
├─────────────────────────────────────────────────────────────────────────┤
│  AVTP Common Header  (IEEE 1722-2016, Clause 4)                          │
├─────────────────────────────────────────────────────────────────────────┤
│  ACF_CAN_BRIEF PDU  (IEEE 1722-2016, Clause 13 / Annex D)               │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### 2a. AVTP Common Header (IEEE 1722-2016, Clause 4)

The DRE wraps ACF messages in an AVTP header. Automotive CAN bridging typically uses **NTSCF** (Non-Time-Synchronized Control Frame) or **TSCF** (Time-Synchronized Control Frame):

| Octet(s) | Bits | Field | Size | Description |
|---|---|---|---|---|
| 0 | 7:0 | `subtype` | 8 | AVTP subtype identifier (e.g. `0x82` for TSCF) |
| 1 | 7 | `sv` | 1 | **Stream ID Valid** — if `1`, `stream_id` is meaningful |
| 1 | 6:4 | `version` | 3 | Protocol version = `0b000` |
| 1 | 3 | `mr` | 1 | **Media Reset** — sequence discontinuity flag |
| 1 | 2 | `r` | 1 | Reserved |
| 1 | 1 | `gv` | 1 | **Gateway Valid** — gateway-specific data present |
| 1 | 0 | `tv` | 1 | **AVTP Timestamp Valid** — `avtp_timestamp` field is valid |
| 2 | 7:0 | `sequence_num` | 8 | **Sequence number** — wrapping 8-bit counter per stream |
| 3 | 7 | `tu` | 1 | **Timestamp Uncertain** — timestamp may be imprecise |
| 3 | 6:0 | `reserved` | 7 | Reserved, set to `0` |
| 4–11 | 63:0 | `stream_id` | 64 | **Stream identifier** — globally unique 64-bit ID for this CAN channel |
| 12–15 | 31:0 | `avtp_timestamp` | 32 | **AVTP Presentation Timestamp** — gPTP-derived time (valid if `tv=1`) |
| 16 | 7:0 | `format` | 8 | ACF format code = `0x02` or `0x03` |
| 17–19 | 23:0 | `format_specific` | 24 | Format-specific data / reserved |
| 20–21 | 15:0 | `acf_msg_length` | 16 | Length of ACF payload in **quadlets** (4-byte units) |

---

### 2b. ACF_CAN_BRIEF Message Header (IEEE 1722-2016, Clause 13 / Table D.1)

Immediately following the AVTP common header:

| Octet(s) | Bits | Field | Size | Description |
|---|---|---|---|---|
| 0 | 7:0 | `acf_format` | 8 | `0x03` = ACF_CAN_BRIEF |
| 1–2 | 15:7 | `acf_msg_length` | 9 | Length of this ACF message in 4-octet words |
| 2 | 6:5 | `pad` | 2 | **Padding byte count** — trailing pad bytes for 4-byte alignment (0–3) |
| 2 | 4 | *(omitted)* | — | `mtv` bit is **NOT present** in ACF_CAN_BRIEF (only in ACF_CAN `0x02`) |
| 2 | 3 | `tv` | 1 | **Timestamp Valid** — if `1`, optional 32-bit `timestamp` field is present |
| 2 | 2:1 | `rsvd` | 2 | Reserved |
| 2–3 | 4:0 | `can_bus_id` | 5 | **Source CAN Bus Identifier** — identifies the physical CAN bus |
| *(if tv=1)* | 31:0 | `timestamp` | 32 | **Per-message 32-bit timestamp** — present only if `tv=1` |

---

### 2c. ACF_CAN_BRIEF CAN Frame Descriptor

| Bits | Field | Size | Description |
|---|---|---|---|
| 31 | `eff` | 1 | **Extended Frame Format** — `0` = 11-bit ID, `1` = 29-bit ID |
| 30 | `rtr` | 1 | **Remote Transmission Request** — `0` = data frame, `1` = remote frame |
| 29 | `fdf` | 1 | **FD Format** — `0` = classic CAN, `1` = CAN FD |
| 28 | `brs` | 1 | **Bit Rate Switch** (CAN FD only) — `1` = data phase uses higher bit rate |
| 27 | `esi` | 1 | **Error State Indicator** (CAN FD only) — `0` = error active, `1` = error passive |
| 26:22 | `rsvd` | 5 | Reserved |
| 21:18 | `dlc` | 4 | **Data Length Code** — `0x0`–`0xF` (CAN FD maps to 0–64 bytes) |
| 17:0 | `can_identifier` | 29 or 11 | **CAN Message Identifier** (MSB-aligned; 11 bits used if `eff=0`) |
| — | `payload` | 0–64 B | **CAN data payload** (byte count per DLC decode) |
| — | `padding` | 0–3 B | Zero-padding bytes to reach 4-byte alignment (`pad` field indicates count) |

> **Key distinction — ACF_CAN vs ACF_CAN_BRIEF:**
> - `ACF_CAN` (`0x02`) includes an optional 64-bit `message_timestamp` field per CAN frame, controlled by the `mtv` (Message Timestamp Valid) bit.
> - `ACF_CAN_BRIEF` (`0x03`) **removes the `mtv` bit and the 64-bit `message_timestamp` entirely** — this is the defining difference of "BRIEF". It retains the optional 32-bit `tv`/`timestamp` field.

---

## 3. Field-by-Field TC4 DRE Support Matrix

| Field | Spec (IEEE 1722-2016) | TC4 DRE Support | Notes |
|---|---|---|---|
| `subtype` | 8-bit AVTP subtype | ✅ **HW** | Set to correct value by DRE automatically |
| `sv` (Stream ID Valid) | 1-bit flag | ✅ **HW** | DRE sets `sv=1` when `stream_id` is configured in routing table |
| `version` | 3-bit, = `0` | ✅ **HW** | Hardcoded to `0b000` |
| `mr` (Media Reset) | 1-bit discontinuity flag | ✅ **HW** | Set on stream restart / routing table reload |
| `gv` (Gateway Valid) | 1-bit | ✅ **HW** | Set when gateway-specific data is valid |
| `tv` (AVTP Timestamp Valid) | 1-bit | ✅ **HW** | DRE sets `tv=1` when gPTP lock is acquired |
| `sequence_num` | 8-bit wrapping counter | ✅ **HW** | Incremented per-packet by DRE; per-stream counter |
| `tu` (Timestamp Uncertain) | 1-bit | ✅ **HW** | Reflects gPTP synchronization quality; set if clock is unlocked |
| `stream_id` | 64-bit | ✅ **HW** | Configured per routing table entry (`ACF_STREAM_ID`); maps CAN-ID → stream |
| `avtp_timestamp` | 32-bit gPTP timestamp | ✅ **HW** | DRE inserts hardware-captured gPTP timestamp (IEEE 802.1AS) |
| `acf_format` | 8-bit = `0x03` | ✅ **HW** | Fixed to `0x03` (ACF_CAN_BRIEF) by DRE hardware |
| `acf_msg_length` | 9-bit quadlet count | ✅ **HW** | Computed automatically based on DLC and optional fields |
| `pad` | 2-bit padding count | ✅ **HW** | Computed from payload length; DRE appends zero-bytes |
| `tv` (ACF-level Timestamp Valid) | 1-bit | ✅ **HW** | DRE can insert per-ACF-message 32-bit timestamp |
| `can_bus_id` | 5-bit | ✅ **HW** | Source CAN bus index from routing table (`CAN_BUS_ID` field) |
| `timestamp` (32-bit ACF-level) | Optional 32-bit | ✅ **HW** | Captured at CAN frame reception by DRE timer hardware |
| **`mtv` (Message Timestamp Valid)** | **1-bit — in ACF_CAN only** | ⚫ **N/A** | **NOT present in ACF_CAN_BRIEF (`0x03`) — by format definition, not a TC4 gap** |
| **`message_timestamp` (64-bit)** | **64-bit — in ACF_CAN only** | ⚫ **N/A** | **NOT present in ACF_CAN_BRIEF (`0x03`) — by format definition** |
| `eff` (Extended Frame Format) | 1-bit | ✅ **HW** | Extracted from CAN frame reception; reflects 11-bit vs 29-bit ID |
| `rtr` (Remote Transmission Request) | 1-bit | ✅ **HW** | Extracted directly from CAN frame |
| `fdf` (FD Format flag) | 1-bit | ✅ **HW** | `1` for CAN FD frames; DRE supports CAN FD natively |
| `brs` (Bit Rate Switch) | 1-bit (CAN FD) | ✅ **HW** | Extracted from CAN FD frame |
| `esi` (Error State Indicator) | 1-bit (CAN FD) | ✅ **HW** | Extracted from CAN FD frame |
| `dlc` (Data Length Code) | 4-bit | ✅ **HW** | Extracted from CAN frame; maps to 0–64 bytes for CAN FD |
| `can_identifier` | 11 or 29 bits | ✅ **HW** | Full CAN ID passed through; mapped via routing table |
| `payload` | 0–64 bytes | ✅ **HW (zero-copy)** | CAN data bytes DMA'd directly — no CPU, no signal decomposition |
| `padding` | 0–3 bytes | ✅ **HW** | DRE appends padding automatically to ensure 4-byte alignment |
| VLAN / PCP tagging (802.1Q) | Not in 1722 itself | ✅ **HW** | DRE inserts VLAN tag per routing table `PRIORITY` field |
| SecOC MAC tag | Not in 1722 | ✅ **HW** | Optional; HSM appends freshness counter + CMAC |
| MACsec (802.1AE) encryption | Not in 1722 | ✅ **HW** | Optional frame-level AES-GCM via MACsec engine |

---

## 4. Key Design Findings: ACF_CAN_BRIEF vs. ACF_CAN

The selection of **ACF_CAN_BRIEF** over **ACF_CAN** by the TC4 DRE is a deliberate hardware design decision, not a capability gap:

| Concern | ACF_CAN (`0x02`) | ACF_CAN_BRIEF (`0x03`) used by TC4 DRE |
|---|---|---|
| Per-message 64-bit timestamp (`mtv` + `message_timestamp`) | ✅ Present | ⚫ **Omitted by format definition** |
| Per-message 32-bit timestamp (`tv` + `timestamp`) | ✅ Optional | ✅ Optional |
| AVTP-level 32-bit gPTP timestamp | ✅ Present | ✅ Present |
| Frame overhead per CAN message | Higher (8 extra bytes for 64-bit ts) | **Lower — better for high-frequency CAN** |
| DRE hardware suitability | More complex parsing | **Optimized for zero-copy HW path** |

**Implication**: Applications needing sub-microsecond per-message CAN timestamps must either:
1. Use the AVTP-level 32-bit `avtp_timestamp` (gPTP-locked, ~1 µs resolution) which TC4 DRE does support, or
2. Switch to `ACF_CAN` (`0x02`) format — but this requires verifying whether the TC4 DRE supports generating that format in hardware (current documentation only confirms `ACF_CAN_BRIEF`)

---

## 5. Review Against Specifications & Known Limitations

### ✅ Confirmed Compliant

- The TC4 DRE's `ACF_CAN_BRIEF` output correctly implements IEEE 1722-2016 Clause 13 (ACF) with `acf_format = 0x03`
- gPTP-derived `avtp_timestamp` is IEEE 802.1AS-compliant
- `stream_id` allocation (64-bit) matches IEEE 1722-2016 §4.4
- CAN FD payload sizes (up to 64 bytes) match ISO 11898-1:2015
- 4-byte alignment via `pad` field is correct per spec
- `sequence_num` wrapping is per-stream, matching §4.3

### ⚠️ Design Trade-offs / Constraints

| Item | Detail |
|---|---|
| **No 64-bit per-message timestamp** | ACF_CAN_BRIEF format intentionally excludes `mtv`/`message_timestamp`. Use AVTP-level timestamp instead. |
| **`tv` / 32-bit timestamp availability** | The optional ACF-level 32-bit `timestamp` field is present only when DRE captures a CAN RX timestamp; availability depends on gPTP lock status (`tu` flag) |
| **Timestamp uncertainty during cold-start** | `tu=1` (Timestamp Uncertain) will be set by DRE until gPTP synchronization is established — receivers must handle this gracefully |
| **No signal-level decomposition** | DRE is a PDU-level gateway; no factor/offset/byte-order CAN signal transformation — signal parsing must happen in SW above the DRE layer |
| **SOME/IP is NOT produced by DRE** | The DRE outputs raw IEEE 1722 ACF frames only. SOME/IP service wrapping requires a software transformer on the TC4 CPU cores |
| **Routing table capacity** | The number of simultaneous CAN-ID → stream mappings is finite (hardware routing table rows); overflows trigger CPU interrupt, not HW drop |
| **RTR frames** | RTR frame support should be verified per silicon revision; remote frames carry no payload and `dlc` represents requested length |

### 📋 Specification Cross-Reference

| Standard | Relevant Clause | Topic |
|---|---|---|
| **IEEE 1722-2016** | Clause 4 | AVTP Common Header (subtype, sv, version, tv, sequence_num, stream_id, avtp_timestamp) |
| **IEEE 1722-2016** | Clause 13 / Annex D | ACF format, ACF_CAN, ACF_CAN_BRIEF PDU structure |
| **IEEE 802.1AS-2020** | Full | gPTP — source of `avtp_timestamp` |
| **IEEE 802.1Q** | Clause 9 | VLAN tagging (PCP priority mapping) |
| **IEEE 802.1AE** | Full | MACsec frame-level encryption |
| **ISO 11898-1:2015** | Clause 10 | CAN FD frame format (fdf, brs, esi, DLC coding for >8 bytes) |
| **Infineon AURIX TC4xx UM** | DRE chapter | Routing table structure, filter rules, timestamp capture registers |
| **AUTOSAR PRS SOME/IP** | §5–6 | SOME/IP header (SW layer only, not DRE) |

---

## 6. Summary

The AURIX TC4 DRE provides **full hardware support for all fields defined in IEEE 1722-2016 ACF_CAN_BRIEF (`acf_format = 0x03`)**. The absence of the 64-bit `message_timestamp`/`mtv` fields is **not a TC4 limitation** — it is correct behavior because those fields are exclusive to `ACF_CAN` (`0x02`) and are structurally absent from the BRIEF format by specification. All fields that belong to ACF_CAN_BRIEF are handled in hardware with zero CPU involvement for the normal routing path. The AVTP-level gPTP timestamp (`avtp_timestamp`, valid when `tv=1` and `tu=0`) provides IEEE 802.1AS-synchronized timing with ~1 µs resolution for applications that need it.

---

## 7. References

- [IEEE 1722-2016 – AVTP (Audio Video Transport Protocol, including ACF)](https://standards.ieee.org/ieee/1722/6408/)
- [Infineon AURIX TC4xx Product Family](https://www.infineon.com/cms/en/product/microcontroller/32-bit-tricore-microcontroller/aurix-family/aurix-tc4xx-family/)
- [Infineon AURIX TC4x DRE Training Document](https://www.infineon.com/row/public/documents/10/56/infineon-aurix-tc4x-data-routing-engine-v1.0.pdf-training-en.pdf)
- [AURIX TC4xx Documentation – Data Routing Engine (DRE)](https://documentation.infineon.com/aurixtc4xx/docs/car1553877122639)
- [IEEE 802.1AS – Timing and Synchronization (gPTP)](https://1.ieee802.org/tsn/802-1as/)
- [IEEE 802.1AE MACsec Standard](https://1.ieee802.org/security/802-1ae/)
- [ISO 11898-1:2015 – CAN FD Frame Format](https://www.iso.org/standard/63648.html)
- [AUTOSAR Classic Platform – COM Stack Specification](https://www.autosar.org/standards/classic-platform)
