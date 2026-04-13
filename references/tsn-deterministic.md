# TSN & Deterministic Networking

**Scope**: Time-Sensitive Networking (TSN) for Ethernet-based industrial automation, IEEE 802.1 standards suite, IEEE 1588 Precision Time Protocol (PTP), IETF DetNet, and 6TiSCH for wireless deterministic mesh. Covers synchronization, scheduling, traffic shaping, redundancy, and configuration mechanisms for real-time industrial IoT. Cross-references [industrial-ot.md](industrial-ot.md) for OPC UA over TSN, [pan-short-range.md](pan-short-range.md) for WirelessHART/ISA100 TSCH, and [edge-gateway.md](edge-gateway.md) for OPC UA PubSub over TSN.

## Table of Contents
1. IEEE 802.1AS (Time Synchronization)
2. IEEE 802.1Qbv (Time-Aware Shaper)
3. IEEE 802.1Qav (Credit-Based Shaper)
4. IEEE 802.1CB (Frame Replication & Elimination)
5. IEEE 802.1Qcc (Stream Reservation Protocol)
6. IEEE 1588 PTP (Precision Time Protocol)
7. IETF DetNet (Deterministic Networking)
8. 6TiSCH (IPv6 over TSCH)

---

## 1. IEEE 802.1AS (Time Synchronization)

**Reference**: IEEE 802.1AS-2020 (gPTP - generalized Precision Time Protocol)

**Purpose**: Distribute common time base across Ethernet network with ±1 μs precision (factory floor synchronization for coordinated motion control, sensor fusion).

**Architecture**:
```
Grandmaster Clock (GM)
  ├─ Timestamp: TAI (International Atomic Time)
  ├─ Clock Source: GPS, GNSS, atomic oscillator
  └─ Clock Class: 248 (default for automotive), 6 (GPS-locked)
       ↓ Sync + Follow_Up messages
Bridge (Boundary Clock or Transparent Clock)
  ├─ Boundary Clock: Terminates & regenerates PTP
  ├─ Transparent Clock: Forwards PTP, adds residence time correction
  └─ Hardware timestamping (PHY-level, nanosecond accuracy)
       ↓
End Station (Ordinary Clock)
  ├─ Slave to upstream Grandmaster
  └─ Local clock disciplined via PI/PID controller
```

**Message Flow** (Peer-to-Peer Delay Mechanism):
```
Master Port (Bridge)                   Slave Port (End Station)
      │                                        │
      ├─ 1. Sync (t1) ───────────────────────>│
      │    (timestamp when frame leaves PHY)  │
      │                                        │
      ├─ 2. Follow_Up ───────────────────────>│
      │    (carries t1 value)                  │
      │                                        │
      │<─ 3. Pdelay_Req (t3) ──────────────────┤
      │                                        │
      ├─ 4. Pdelay_Resp (t4, t3) ────────────>│
      │    (t4 = RX timestamp of Pdelay_Req)  │
      │                                        │
      ├─ 5. Pdelay_Resp_Follow_Up (t4) ──────>│
      │                                        │
      └─ Slave calculates:
         Path Delay = [(t4 - t3) + (t2 - t1)] / 2
         Offset = t2 - t1 - Path Delay
         Disciplined Clock = Local Time + Offset
```

**Time Domains**:
```
gPTP Domain 0:     Audio/Video Bridging (AVB) applications
gPTP Domain 1-127: Custom domains (e.g., factory floor automation)
```

**Best Master Clock Algorithm (BMCA)**:
```
Priority 1 (0-255, lower wins)
  ↓ If tie
Clock Class (6=GPS, 248=default)
  ↓ If tie
Clock Accuracy (nanosecond drift spec)
  ↓ If tie
Priority 2 (0-255, lower wins)
  ↓ If tie
Clock Identity (MAC-based, lowest wins)
```

**Use Case**: Factory motion control (robot arm + CNC coordination), automotive Ethernet (camera/LIDAR timestamping), pro audio (Dante, AVB).

---

## 2. IEEE 802.1Qbv (Time-Aware Shaper)

**Reference**: IEEE 802.1Qbv-2015 (Enhancements for Scheduled Traffic)

**Purpose**: Time-slot-based traffic scheduling (TDMA for Ethernet). Guarantees deterministic latency by opening/closing gates for priority queues at precise times.

**Gate Control List (GCL)**:
```
Switch Port (8 priority queues)
┌────────────────────────────────────────────────────┐
│ Queue 7 (Highest)  ─┐                              │
│ Queue 6            ─┤                              │
│ Queue 5            ─┤                              │
│ Queue 4            ─┤  Gates (Open/Closed)        │
│ Queue 3            ─┤                              │
│ Queue 2            ─┤                              │
│ Queue 1            ─┤                              │
│ Queue 0 (Lowest)   ─┘                              │
└────────────────────────────────────────────────────┘
         ↓ Transmission Selection Algorithm (TSA)
     Ethernet PHY

Gate Control List (per port):
  ├─ Entry 0: [Gate States: ooCCCCCC] Duration: 200μs
  │   (Queue 7,6 open; Queue 5-0 closed → time-critical traffic)
  ├─ Entry 1: [Gate States: CCOOOOOO] Duration: 800μs
  │   (Queue 5-0 open → best-effort traffic)
  └─ Cycle Time: 1ms (repeats)
```

**Configuration Example**:
```
Base Time: TAI 2023-12-01T10:00:00.000000000Z
Cycle Time: 1,000,000 ns (1 ms)

GCL Entry 0:
  Gate States: 0xC0 (11000000 binary → Q7,Q6 open)
  Time Interval: 200,000 ns
  Operation: SetGates

GCL Entry 1:
  Gate States: 0x3F (00111111 binary → Q5-Q0 open)
  Time Interval: 800,000 ns
  Operation: SetGates
```

**Guard Band**:
```
Time-critical slot (200μs)
  ├─ Transmission time: 150μs
  └─ Guard band: 50μs (prevent best-effort frame from overlapping)
     (Any frame starting at t=149μs that takes >51μs will be held until next cycle)
```

**Use Case**: Industrial Ethernet (Profinet IRT-like determinism), automotive Ethernet (ADAS sensor data priority), 5G fronthaul (eCPRI time-slicing).

---

## 3. IEEE 802.1Qav (Credit-Based Shaper)

**Reference**: IEEE 802.1Qav-2009 (Forwarding and Queuing Enhancements for Time-Sensitive Streams)

**Purpose**: Audio/Video Bridging (AVB) traffic shaping. Prevents bandwidth starvation of time-sensitive streams while allowing best-effort traffic.

**Credit Algorithm**:
```
Stream Reservation (SR) Class A/B:
  Class A: Max latency 2ms (7 hops), bandwidth reservation
  Class B: Max latency 50ms (7 hops)

Credit-Based Shaper:
  ├─ idleSlope: Credit accumulation rate when queue is idle
  ├─ sendSlope: Credit drain rate when transmitting (-idleSlope × (Port BW - Reserved BW) / Reserved BW)
  └─ Credit bounds: [hiCredit, loCredit]

Example (1 Gbps port, 100 Mbps reserved for Class A):
  idleSlope = 100 Mbps
  sendSlope = -900 Mbps
  Frame can only transmit when Credit ≥ 0
```

**State Machine**:
```
State: IDLE
  Credit += idleSlope × Δt
  If (Credit > hiCredit): Credit = hiCredit
  If (Queue not empty & Credit ≥ 0): State = TRANSMIT

State: TRANSMIT
  Credit += sendSlope × Δt
  Transmit frame
  If (Credit < 0 or Queue empty): State = IDLE
```

**Use Case**: Professional audio/video (Dante, AVB/TSN cameras), automotive Ethernet (camera streaming + best-effort diagnostics).

---

## 4. IEEE 802.1CB (Frame Replication & Elimination)

**Reference**: IEEE 802.1CB-2017 (Frame Replication and Elimination for Reliability)

**Purpose**: Achieve 99.9999% reliability (Six Sigma) by sending duplicate frames over redundant paths and eliminating duplicates at receiver.

**FRER (Frame Replication and Elimination for Reliability)**:
```
Source (Talker)
  ├─ Stream ID: {MAC src, VLAN, seq num}
  └─ Replicates frame → sends copy A & copy B
       ↓
  Path A (primary)       Path B (backup)
  Switch 1               Switch 2
       ↓                      ↓
       └────── Merge ─────────┘
                 ↓
           Destination (Listener)
             ├─ Sequence Number Recovery: Detects duplicate seq num
             └─ Eliminates duplicate, forwards first-received frame
```

**Sequence Number Tagging** (R-TAG):
```
Ethernet Frame:
  [Dest MAC][Src MAC][R-TAG][VLAN][EtherType][Payload][FCS]

R-TAG (6 bytes):
  ├─ TPID: 0xF1C1 (Redundancy Tag Protocol ID)
  ├─ Sequence Number: 16-bit (wraps at 65535)
  └─ Reserved fields
```

**Use Case**: Industrial control (safety-critical PLC loops), railway signaling (ETCS), substation automation (IEC 61850).

---

## 5. IEEE 802.1Qcc (Stream Reservation Protocol)

**Reference**: IEEE 802.1Qcc-2018 (Stream Reservation Protocol Enhancements and Performance Improvements)

**Purpose**: Centralized Network Configuration (CNC) or distributed User/Network Interface (UNI) for TSN stream setup.

**Configuration Models**:
```
Model A (Fully Centralized):
  CNC (Centralized Network Controller)
    ├─ Computes GCL for all switches (802.1Qbv schedules)
    ├─ Allocates bandwidth, paths, priorities
    └─ Uses NETCONF/YANG to configure switches
       (draft-ietf-detnet-yang-XX)

Model B (Centralized User, Distributed Network):
  CUC (Centralized User Controller)
    ├─ Requests stream reservation from network
    └─ Network autonomously allocates resources (MSRP/SRP)

Model C (Fully Distributed):
  End stations use SRP (Stream Reservation Protocol)
    ├─ MSRP (Multiple Stream Registration Protocol)
    ├─ MVRP (VLAN Registration Protocol)
    └─ MRP (Multiple Registration Protocol)
```

**SRP Message Flow** (Model C):
```
Talker                      Bridge                      Listener
  │                            │                            │
  ├─ 1. Talker Advertise ─────>│                            │
  │    (Stream ID, VLAN, BW)   │                            │
  │                            │                            │
  │                            ├─ 2. Talker Advertise ─────>│
  │                            │                            │
  │                            │<─ 3. Listener Ready ────────┤
  │                            │                            │
  │<─ 4. Listener Ready ───────┤                            │
  │    (Reservation confirmed) │                            │
  │                            │                            │
  ├─ 5. Stream starts ──────────────────────────────────────>│
```

**YANG Model** (Centralized Configuration):
```yang
module ieee802-dot1q-tsn-config-uni {
  container streams {
    list user-to-network-requirements {
      key "stream-id";
      leaf stream-id { type stream-id-type; }
      container traffic-specification {
        leaf max-latency { type uint32; units "nanoseconds"; }
        leaf max-frame-size { type uint16; }
        leaf interval { type uint32; units "nanoseconds"; }
      }
      list end-station-interfaces {
        leaf mac-address { type ieee:mac-address; }
        leaf interface-name { type if:interface-ref; }
      }
    }
  }
}
```

**Use Case**: Factory TSN configuration (centralized scheduler for 100+ devices), automotive E/E architecture (zonal TSN orchestration).

---

## 6. IEEE 1588 PTP (Precision Time Protocol)

**Reference**: IEEE 1588-2019 (PTPv2.1)

**Profiles**:
```
Default Profile:         General industrial use
Power Profile:           IEC/IEEE 61850-9-3 (substation automation)
Telecom Profile:         ITU-T G.8265.1 (mobile backhaul)
Automotive Profile:      IEEE 802.1AS-automotive (in-vehicle networks)
White Rabbit:            Sub-nanosecond sync (CERN particle accelerators)
```

**Clock Types**:
```
Ordinary Clock (OC):     Single port, endpoint (master or slave)
Boundary Clock (BC):     Multiple ports, acts as slave on one port, master on others
Transparent Clock (TC):  Forwards PTP, corrects for residence/path delay
  ├─ E2E TC: End-to-End delay correction
  └─ P2P TC: Peer-to-Peer delay correction (802.1AS uses P2P)
```

**PTPv2 Message Types**:
```
Event Messages (timestamped):
  ├─ Sync:              Carries master's time reference
  ├─ Delay_Req:         Slave requests delay measurement
  ├─ Pdelay_Req/Resp:   Peer-to-peer delay measurement
  └─ Follow_Up:         Carries precise Sync egress timestamp

General Messages (not timestamped):
  ├─ Announce:          BMCA info (clock quality, priority)
  ├─ Management:        Config, status queries
  └─ Signaling:         Unicast negotiation
```

**E2E Delay Mechanism**:
```
Master                                 Slave
  │                                      │
  ├─ Sync (t1) ───────────────────────> │ (receive at t2)
  │                                      │
  ├─ Follow_Up (t1 value) ─────────────>│
  │                                      │
  │ <─ Delay_Req (t3) ────────────────── ┤
  │  (receive at t4)                     │
  │                                      │
  ├─ Delay_Resp (t4 value) ────────────>│
  │                                      │
  └─ Slave calculates:
     Mean Path Delay = [(t2 - t1) + (t4 - t3)] / 2
     Offset from Master = (t2 - t1) - Mean Path Delay
```

**Hardware Timestamping**:
```
Software Timestamping:   ±1 ms precision (kernel scheduling jitter)
Hardware Timestamping:   ±10 ns precision (PHY-level timestamp)
  └─ Timestamp taken at SFD (Start of Frame Delimiter) on wire
```

**Use Case**: 5G RAN synchronization (phase alignment), power grid protection relays (IEC 61850), financial trading (SMPTE 2059 A/V sync).

---

## 7. IETF DetNet (Deterministic Networking)

**Reference**: RFC 8655 (DetNet Architecture), RFC 8938 (DetNet Data Plane)

**Purpose**: Extend TSN-like determinism over Layer 3 routed networks (WAN, IP/MPLS).

**Architecture**:
```
┌─────────────────────────────────────────────────────┐
│ DetNet Application (Industrial Control, A/V)       │
├─────────────────────────────────────────────────────┤
│ DetNet Service Layer                                │
│   ├─ Service Protection (PREOF - replication)       │
│   └─ Ordering, Duplicate Elimination                │
├─────────────────────────────────────────────────────┤
│ DetNet Forwarding Layer                             │
│   ├─ Resource Allocation (bandwidth reservation)    │
│   ├─ Explicit Routes (RSVP-TE, Segment Routing)     │
│   └─ Traffic Engineering                            │
├─────────────────────────────────────────────────────┤
│ Data Plane (IP or MPLS)                             │
│   ├─ IP: IPv6 with DetNet options                   │
│   └─ MPLS: Label-switched paths (LSP)               │
└─────────────────────────────────────────────────────┘
```

**DetNet Flow Requirements**:
```
Bounded Latency:         Max end-to-end delay (e.g., <10 ms)
Low Packet Loss:         <10^-6 (one in a million)
Bounded Jitter:          Packet delay variation <1 ms
In-Order Delivery:       Sequence number restoration
```

**PREOF (Packet Replication, Elimination, Ordering)**:
```
Source                            Destination
  │                                    │
  ├─ Packet (seq=1) replicated        │
  │    Copy A → Path 1 → Router A ──┐ │
  │    Copy B → Path 2 → Router B ──┼─┴─> Merge Point
  │                                 │      ├─ Detect duplicate (seq=1)
  │                                 │      └─ Eliminate, forward first copy
  │                                 └───────> (Path B arrives 5ms late, dropped)
```

**DetNet over MPLS**:
```
MPLS Label Stack:
  [S-Label][F-Label][Payload]

S-Label (Service Label):   DetNet flow identifier
F-Label (Forwarding Label): MPLS LSP for traffic engineering
```

**DetNet over IPv6**:
```
IPv6 Header:
  [Src][Dst][Flow Label][Extension Headers]

Extension Header (Hop-by-Hop):
  DetNet Option:
    ├─ Sequence Number (duplicate elimination)
    └─ Flow ID (resource reservation lookup)
```

**Use Case**: Industrial WAN (remote factory control over MPLS), 5G URLLC (ultra-reliable low-latency), critical infrastructure (power grid SCADA over IP).

---

## 8. 6TiSCH (IPv6 over TSCH)

**Reference**: RFC 8180 (Minimal 6TiSCH Configuration), RFC 9030 (6TiSCH Architecture)

**Purpose**: Bring IP connectivity and TSN-like determinism to IEEE 802.15.4 wireless mesh (industrial IoT, smart cities).

**Stack**:
```
┌────────────────────────────────────────────┐
│ CoAP / UDP                                 │
├────────────────────────────────────────────┤
│ IPv6                                       │
├────────────────────────────────────────────┤
│ 6LoWPAN (RFC 4944, 6282)                   │
│   ├─ IPHC header compression              │
│   └─ Fragmentation (802.15.4 127B limit)  │
├────────────────────────────────────────────┤
│ 6top Protocol (RFC 8480)                   │
│   ├─ Add/Delete timeslots                 │
│   └─ Negotiates schedule with neighbors   │
├────────────────────────────────────────────┤
│ IEEE 802.15.4 TSCH MAC                     │
│   ├─ Time slots (10 ms default)           │
│   ├─ Channel hopping (16 channels @ 2.4GHz│
│   └─ ASN (Absolute Slot Number) time ref  │
├────────────────────────────────────────────┤
│ IEEE 802.15.4 PHY (O-QPSK, 250 kbps)      │
└────────────────────────────────────────────┘
```

**TSCH Schedule**:
```
Slotframe (100 slots × 10 ms = 1 second repeat)

Slot 0:  [Ch 15] Node A → Coordinator (TX)
Slot 1:  [Ch 20] Node B → Coordinator (TX)
Slot 2:  [Ch 11] Node A → Node C (RX)
...
Slot 99: [Ch 25] Shared slot (CSMA/CA)

Channel Hopping:
  Channel = f(ASN, slot_offset, channel_offset)
  Example: ASN=12345, slot_offset=0, ch_offset=2
    → Channel = (12345 + 0 + 2) mod 16 = Channel 11
```

**6top Protocol** (RFC 8480):
```
Node A (requester)                 Node B (responder)
  │                                      │
  ├─ 1. ADD Request ────────────────────>│
  │    (num_cells=3, cell_options=TX)    │
  │                                      │
  │ <─ 2. ADD Response ───────────────────┤
  │    (cell_list: [(10, 5), (20, 7)])   │
  │    (slot, channel offset pairs)      │
  │                                      │
  └─ Cells added to TSCH schedule
```

**Minimal Configuration** (RFC 8180):
```
1. Node joins network via Enhanced Beacon (EB)
   ├─ EB advertises: PANID, Join Metric, slotframe size
   └─ Secured with IEEE 802.15.4 AES-128-CCM*

2. Node configures link-local IPv6 address
   fe80::IID (IID derived from EUI-64)

3. Uses minimal shared slot (slot 0, all channels)
   ├─ CSMA/CA contention (for bootstrap)
   └─ Once joined, negotiates dedicated cells via 6top
```

**RPL over TSCH** (RFC 6550):
```
DODAG (Destination-Oriented DAG):
  Root (Border Router)
    ├─ Rank 1: Node A
    │   └─ Rank 2: Node C
    └─ Rank 1: Node B
        └─ Rank 2: Node D

DIO (DODAG Information Object):
  Broadcasts routing metric (ETX - Expected Transmission Count)
  Node selects preferred parent (lowest ETX to root)

DAO (Destination Advertisement Object):
  Upward routes advertised to root
  Root builds downward routing table
```

**Use Case**: Smart factory (wireless sensors + deterministic latency), smart agriculture (soil sensor mesh), smart city (streetlight control).

---

## Key Specification References

| Standard        | Title                                        | Publisher | Year |
|-----------------|----------------------------------------------|-----------|------|
| IEEE 802.1AS    | Timing and Synchronization (gPTP)            | IEEE      | 2020 |
| IEEE 802.1Qbv   | Enhancements for Scheduled Traffic           | IEEE      | 2015 |
| IEEE 802.1Qav   | Forwarding/Queuing for Time-Sensitive Streams| IEEE      | 2009 |
| IEEE 802.1CB    | Frame Replication and Elimination            | IEEE      | 2017 |
| IEEE 802.1Qcc   | Stream Reservation Protocol (SRP)            | IEEE      | 2018 |
| IEEE 1588-2019  | Precision Time Protocol v2.1                 | IEEE      | 2019 |
| RFC 8655        | DetNet Architecture                          | IETF      | 2019 |
| RFC 8938        | DetNet Data Plane Framework                  | IETF      | 2020 |
| RFC 8180        | Minimal 6TiSCH Configuration                 | IETF      | 2017 |
| RFC 9030        | 6TiSCH Architecture                          | IETF      | 2021 |

---

## Related Files
- **[industrial-ot.md](industrial-ot.md)**: OPC UA over TSN, Profinet IRT, EtherCAT real-time performance
- **[pan-short-range.md](pan-short-range.md)**: WirelessHART TSCH, ISA100.11a TSCH, IEEE 802.15.4 MAC
- **[edge-gateway.md](edge-gateway.md)**: OPC UA PubSub with UADP over TSN Ethernet
- **[local-network-ip.md](local-network-ip.md)**: 6LoWPAN header compression, RPL routing
- **[cellular-wan.md](cellular-wan.md)**: 5G URLLC integration with TSN (3GPP Rel-16)
