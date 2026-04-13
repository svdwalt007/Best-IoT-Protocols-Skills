# PAN & Short-Range Protocols

**Scope**: Personal Area Network (PAN) and short-range wireless protocols operating within 0-100m range. Covers Bluetooth (Classic/LE/Mesh), Zigbee, Z-Wave, Thread, IEEE 802.15.4, UWB, industrial wireless (WirelessHART, ISA100.11a), DECT, and NFC/RFID. Cross-references [smart-home-consumer.md](smart-home-consumer.md) for Matter integration, [industrial-ot.md](industrial-ot.md) for WirelessHART/ISA100 in process automation, and [automotive-transport.md](automotive-transport.md) for UWB Digital Key.

## Table of Contents
1. Bluetooth Low Energy (BLE 4.0-5.4)
2. Bluetooth Classic (BR/EDR)
3. Bluetooth Mesh v1.0/1.1
4. Zigbee (IEEE 802.15.4 + ZHA/ZLL/PRO)
5. Z-Wave (500/700/800 Series)
6. Thread / OpenThread
7. IEEE 802.15.4 (MAC/PHY Layer)
8. WirelessHART (IEC 62591)
9. ISA100.11a (IEC 62734)
10. Ultra-Wideband (UWB) - IEEE 802.15.4a/4z
11. DECT NR+ / DECT ULE
12. NFC (ISO/IEC 14443, ISO/IEC 18092)
13. EnOcean (ISO/IEC 14543-3-10)

---

## 1. Bluetooth Low Energy (BLE 4.0-5.4)

**Reference**: Bluetooth Core Specification 5.4 (2023), Bluetooth SIG GATT Specifications

**Protocol Stack**:
```
┌─────────────────────────────────────────┐
│ Application (GATT Services/Profiles)    │
├─────────────────────────────────────────┤
│ GATT (Generic Attribute Profile)        │
│   └─ Service → Characteristic → Desc   │
├─────────────────────────────────────────┤
│ ATT (Attribute Protocol)                 │
│   └─ Read/Write/Notify/Indicate         │
├─────────────────────────────────────────┤
│ L2CAP (Logical Link Control & Adapt)    │
│   ├─ LE Credit-Based Flow Control       │
│   └─ CoC (Connection-oriented Channels) │
├─────────────────────────────────────────┤
│ HCI (Host Controller Interface)         │
├─────────────────────────────────────────┤
│ Link Layer (LL) - Advertising, Scanning,│
│   Connection, Encryption (AES-CCM)      │
├─────────────────────────────────────────┤
│ PHY (2.4 GHz ISM band, 40 channels)     │
│   ├─ 1M PHY (1 Mbps, legacy)            │
│   ├─ 2M PHY (2 Mbps, BT 5.0+)           │
│   └─ Coded PHY (125/500 kbps long range)│
└─────────────────────────────────────────┘
```

### GAP (Generic Access Profile)

**Roles**:
```
Central (master)          Peripheral (slave)
  ├─ Initiates connection   ├─ Advertises presence
  ├─ Scans for devices      ├─ Accepts connections
  └─ Multiple connections   └─ Single connection (or multi in BT 5.0+)

Observer                  Broadcaster
  ├─ Scans passively        ├─ Advertises only
  └─ No connections         └─ No connections (e.g., beacons)
```

**Advertising Packet Types** (Legacy):
```
ADV_IND          - Connectable and scannable undirected advertising
ADV_DIRECT_IND   - Connectable directed advertising (fast reconnect)
ADV_NONCONN_IND  - Non-connectable undirected (beacons, Eddystone, iBeacon)
ADV_SCAN_IND     - Scannable undirected advertising
SCAN_REQ/SCAN_RSP - Scan request/response (31 bytes payload)
```

**Extended Advertising** (BT 5.0+):
```
AdvDataInfo (ADI) field for data continuity
ChainedPDU for payloads >255 bytes (up to 1650 bytes)
Periodic Advertising: Synchronized broadcasts (e.g., audio sync in Auracast)
```

**Advertising Channels**: 37, 38, 39 (2402 MHz, 2426 MHz, 2480 MHz - minimize interference with Wi-Fi)

**Advertising Data Format** (AD Type examples):
```
0x01 - Flags (LE Limited/General Discoverable, BR/EDR support)
0x02/0x03 - 16-bit UUIDs (incomplete/complete)
0x06/0x07 - 128-bit UUIDs (incomplete/complete)
0x09 - Complete Local Name
0x16 - Service Data (UUID + data, e.g., Eddystone URL)
0xFF - Manufacturer Specific Data (Company ID + payload)
```

**iBeacon Format** (Apple):
```
0x4C 0x00 0x02 0x15 [UUID 16B] [Major 2B] [Minor 2B] [TX Power 1B]
Example: Proximity detection in retail (UUID = store chain, Major = location, Minor = zone)
```

**Eddystone Format** (Google):
```
Frame Types:
  UID: Namespace (10B) + Instance (6B)
  URL: Compressed URL (e.g., https://goo.gl/abc → 0x03 goo.gl/abc)
  TLM: Battery voltage, temperature, advertising count
  EID: Ephemeral ID (rotating for privacy)
```

### GATT (Generic Attribute Profile)

**Hierarchy**:
```
Service (UUID 0x1800 - Generic Access, 0x180F - Battery Service)
  └─ Characteristic (UUID 0x2A19 - Battery Level)
      ├─ Value (1 byte: 0-100%)
      ├─ Properties: Read, Write, Notify, Indicate
      ├─ Client Characteristic Config Descriptor (CCCD) - enable notifications
      └─ Characteristic User Description (optional text)
```

**GATT Operations** (via ATT):
```
Read:    ATT_READ_REQ → ATT_READ_RSP
Write:   ATT_WRITE_REQ → ATT_WRITE_RSP (acknowledged)
Write Command: ATT_WRITE_CMD (no response, faster)
Notify:  ATT_HANDLE_VALUE_NTF (no ACK from client)
Indicate: ATT_HANDLE_VALUE_IND → ATT_HANDLE_VALUE_CFM (acknowledged)
```

**Common GATT Services**:
```
0x1800 - Generic Access (Device Name, Appearance)
0x1801 - Generic Attribute (Service Changed indication)
0x180A - Device Information (Manufacturer, Model, Serial, Firmware Rev)
0x180D - Heart Rate Service (Measurement, Body Sensor Location)
0x180F - Battery Service (Battery Level %)
0x181A - Environmental Sensing (Temperature, Humidity, Pressure)
0x181C - User Data (Age, Gender, Weight, Height)
```

**Heart Rate Service Example**:
```
Service: 0x180D (Heart Rate)
  ├─ Characteristic: 0x2A37 (Heart Rate Measurement)
  │   ├─ Properties: Notify
  │   ├─ Value: [Flags 1B] [HR 1-2B] [Energy 0-2B] [RR-Interval 0-nB]
  │   └─ CCCD: 0x2902 (write 0x01 to enable notifications)
  └─ Characteristic: 0x2A38 (Body Sensor Location)
      ├─ Properties: Read
      └─ Value: 0x01 (Chest), 0x02 (Wrist), 0x03 (Finger), etc.
```

### SMP (Security Manager Protocol)

**Pairing Methods**:
```
Just Works:        No MITM protection (auto-accept)
Passkey Entry:     6-digit PIN (MITM protection)
Numeric Comparison: Both devices show 6-digit code, user confirms match
Out-of-Band (OOB): NFC tap or QR code exchange
```

**Pairing Flow (LE Secure Connections, BT 4.2+)**:
```
Initiator (Central)              Responder (Peripheral)
      │                                  │
      ├─ 1. Pairing Request ────────────>│
      │    (IO Capability, MITM, SC)     │
      │                                  │
      │<─ 2. Pairing Response ───────────┤
      │    (IO Capability, MITM, SC)     │
      │                                  │
      ├─ 3. Public Key Exchange (P-256) ─>│
      │<─────────────────────────────────┤
      │                                  │
      ├─ 4. Authentication (Passkey/NC)──>│
      │    ECDH(PubKeyA, PrivKeyB) → DHKey
      │                                  │
      ├─ 5. DHKey Check ────────────────>│
      │<─────────────────────────────────┤
      │    (f6 CMAC using DHKey)         │
      │                                  │
      ├─ 6. LTK Generation (f5 function) │
      │    LTK derived from DHKey        │
      │                                  │
      ├─ 7. Encryption Start ────────────>│
      │    LL_ENC_REQ with LTK           │
      │<─ LL_ENC_RSP ────────────────────┤
      │                                  │
      └─ Encrypted AES-CCM link established
```

**Legacy Pairing (BT 4.0-4.1)**: Uses STK (Short-Term Key) derived from TK (Temporary Key) via Confirm/Rand exchange. Vulnerable to passive eavesdropping (fixed in BT 4.2 LE Secure Connections with ECDH).

**Privacy Features**:
- **Resolvable Private Address (RPA)**: Changes every 15 minutes, resolved using IRK (Identity Resolving Key)
- **Non-Resolvable Private Address (NRPA)**: Random, cannot be tracked

### BLE 5.x Features

**BLE 5.0 (2016)**:
```
2M PHY:         2 Mbps data rate (2x throughput, same range as 1M)
Coded PHY:      125/500 kbps (4x range: ~200m line-of-sight outdoors)
Extended Advertising: 255 bytes → 1650 bytes payload
Advertising Sets: Concurrent advertising on multiple sets
```

**BLE 5.1 (2019)**:
```
Direction Finding:
  ├─ AoA (Angle of Arrival): Receiver uses antenna array
  │   Use case: Indoor positioning (1m accuracy)
  └─ AoD (Angle of Departure): Transmitter uses antenna array
      Use case: Asset tracking (beacons with AoD, phones calculate position)

GATT Caching: Client caches service/characteristic layout (reduces rediscovery overhead)
```

**BLE 5.2 (2020)**:
```
LE Power Control:      Adaptive TX power (improve battery + coexistence)
Enhanced ATT (EATT):   Multiple parallel ATT transactions over L2CAP CoC
Isochronous Channels:  Time-synchronized audio streams (foundation for LE Audio)
```

**BLE 5.3 (2021)**:
```
Periodic Advertising Enhancement: Subevents for improved sync
Connection Subrating:              Reduce connection event frequency (power saving)
Channel Classification:            Enhanced interference mitigation
```

**BLE 5.4 (2023)**:
```
Advertising Coding Selection:      Prefer S=2 or S=8 coded PHY per use case
Encrypted Advertising Data:        Privacy for periodic advertising
Periodic Advertising with Responses (PAwR): Two-way sync (ESL - Electronic Shelf Labels)
```

### LE Audio & LC3

**Reference**: Bluetooth LE Audio Specification 1.0 (2022)

**Key Technologies**:
```
LC3 (Low Complexity Communications Codec):
  ├─ 7.5ms or 10ms frame duration
  ├─ Bitrates: 16-320 kbps
  └─ 25% better quality than SBC at same bitrate

Isochronous Channels (ISO):
  ├─ CIS (Connected Isochronous Stream): Unicast audio
  └─ BIS (Broadcast Isochronous Stream): Multicast audio (Auracast)

Auracast Broadcast Audio:
  ├─ Public broadcast (airport announcements, museums)
  ├─ Personal audio sharing (watch movie together)
  └─ Assistive listening (hearing aids in theaters)
```

**Multi-Stream Audio**:
```
Stereo earbuds: Each earbud is a separate CIS endpoint
  Phone (Central) ─┬─ CIS 1 → Left Earbud
                   └─ CIS 2 → Right Earbud
(Synchronized via CIG - Connected Isochronous Group)
```

---

## 2. Bluetooth Classic (BR/EDR)

**Reference**: Bluetooth Core Specification 5.4, Profiles (A2DP, HFP, SPP, etc.)

**PHY Layer**:
```
Frequency Hopping Spread Spectrum (FHSS):
  ├─ 79 channels (2402-2480 MHz, 1 MHz spacing)
  ├─ Hop rate: 1600 hops/sec
  └─ Adaptive Frequency Hopping (AFH) - avoid Wi-Fi interference

Modulation:
  ├─ Basic Rate (BR): GFSK, 1 Mbps
  └─ Enhanced Data Rate (EDR): π/4-DQPSK (2 Mbps), 8-DPSK (3 Mbps)
```

**Baseband (Link Controller)**:
```
Piconet: Master (1) + Slaves (up to 7 active)
Scatternet: Device participates in multiple piconets
Time slots: 625 μs (TDD - Time Division Duplex)
Packet types: DM1/3/5 (medium rate), DH1/3/5 (high rate), AUX1
```

**Pairing (Secure Simple Pairing - SSP)**:
```
1. IO Capability exchange
2. Public Key exchange (P-192 ECDH in legacy, P-256 in Secure Connections)
3. Authentication stage (Numeric Comparison, Passkey Entry, Just Works, OOB)
4. Link Key generation (Combination Key K_AB)
5. Mutual authentication with SAFER+ / AES-CCM encryption
```

**Profiles**:
```
A2DP (Advanced Audio Distribution):  High-quality audio streaming (SBC, AAC, aptX, LDAC)
AVRCP (Audio/Video Remote Control):  Play/Pause/Volume control
HFP (Hands-Free Profile):            Phone calls in cars (mSBC codec, wideband speech)
HID (Human Interface Device):        Keyboards, mice, game controllers
SPP (Serial Port Profile):           RFCOMM over L2CAP (UART emulation)
OPP (Object Push Profile):           File transfer (vCard, vCalendar)
PBAP (Phone Book Access Profile):    Contact sync
```

**A2DP Codecs**:
```
SBC (Subband Coding):      Mandatory, 328 kbps max, ~220 kbps typical
AAC (Advanced Audio Coding): 256 kbps, better quality than SBC
aptX / aptX HD:            Qualcomm proprietary, 352/576 kbps, low latency
aptX Adaptive:             279-420 kbps variable bitrate
LDAC (Sony):               330/660/990 kbps (Hi-Res Audio)
```

---

## 3. Bluetooth Mesh v1.0 / v1.1

**Reference**: Bluetooth Mesh Profile 1.0.1 (2019), Mesh Model Specification 1.1 (2021)

**Architecture**:
```
Provisioner (smartphone app)
  ↓ Provisioning (add device to network)
Unprovisioned Device Beacon (UUID)
  ↓ Provisioning Protocol (ECDH + OOB auth)
Node (provisioned device with unicast address)
  ├─ Elements (addressable entities within a node)
  ├─ Models (Generic OnOff, Lightness, Sensor, etc.)
  └─ Publish/Subscribe (many-to-many messaging)
```

**Message Addressing**:
```
Unicast Address:   0x0001-0x7FFF (specific node element)
Group Address:     0xC000-0xFEFF (multiple nodes subscribe)
Virtual Address:   0x8000-0xBFFF (UUID-based group, collision-resistant)
Unassigned:        0x0000 (not yet provisioned)
```

**Managed Flooding (Relay)**:
```
Node A publishes message to Group 0xC001
  → Neighbors relay (TTL=5)
    → Next hop relays (TTL=4)
      → Message propagates across mesh
  ├─ Message Cache prevents duplicate processing (SRC + SEQ)
  └─ Heartbeat messages monitor network health
```

**Friendship (Low Power Node Support)**:
```
Low Power Node (LPN)                Friend Node
      │                                  │
      ├─ 1. Friend Request ──────────────>│
      │    (RxWindow, Poll Timeout)       │
      │                                  │
      │<─ 2. Friend Offer ────────────────┤
      │    (Cache size, RSSI)             │
      │                                  │
      ├─ 3. Friend Poll (wake up) ───────>│
      │                                  │
      │<─ 4. Cached messages ─────────────┤
      │    (accumulated since last poll)  │
      │                                  │
      └─ LPN sleeps (Friend buffers msgs) │
```

**GATT Proxy**:
```
Non-mesh BLE device (smartphone)
  │
  └─ GATT connection to Proxy Node
      │
      └─ Mesh Network (relayed via Proxy Node ADV_IND / SCAN_RSP)
```

**Security Layers**:
```
Network Layer:    NetKey (encrypts SRC, DST, TTL) - AES-CCM
                  ObfuscatedNetworkPDU prevents tracking
Transport Layer:  AppKey (encrypts application payload)
                  DevKey (device-specific, for config messages)
Device Key:       Unique per device (provisioner ↔ node communication)
```

**Mesh Models**:
```
Generic OnOff:           Binary state (light on/off)
Generic Level:           16-bit signed integer (-32768 to 32767)
Lightness:               Perceived lightness (0-65535)
CTL (Color Temperature): Tunable white (Mired scale: 153-555, 6500K-1800K)
HSL (Hue Saturation):    Color control (Hue 0-360°, Saturation 0-100%)
Sensor:                  Environmental data (temperature, occupancy, lux)
Scene:                   Store/recall lighting states (16 scenes/node)
Scheduler:               Time-based automation (daily events)
```

**Bluetooth Mesh 1.1 Enhancements (2021)**:
```
Certificate-based Provisioning: X.509 certs for enterprise security
Directed Forwarding:            Optimized routing (reduce flooding overhead)
Subnet Bridge:                  Connect multiple subnets (e.g., building floors)
Remote Provisioning:            Provision nodes without BLE proximity (via mesh)
Private Beacons:                Randomized beacon data (privacy enhancement)
```

---

## 4. Zigbee (IEEE 802.15.4 + ZHA/ZLL/PRO)

**Reference**: Zigbee PRO 2023 (R23), Zigbee Cluster Library 8 (ZCL 8)

**Architecture**:
```
IEEE 802.15.4 MAC/PHY (2.4 GHz, channels 11-26, DSSS)
  ↓
Zigbee Network Layer (NWK)
  ├─ Routing (AODV-based mesh)
  ├─ Security (AES-128 CCM*)
  └─ Address assignment (16-bit short address)
  ↓
Zigbee Application Support (APS)
  ├─ Binding (indirect addressing)
  ├─ Group addressing
  └─ End-to-end security
  ↓
Zigbee Cluster Library (ZCL)
  └─ Clusters (On/Off, Level, Color, Thermostat, etc.)
```

**Device Types**:
```
Coordinator (ZC):   Forms network, assigns addresses, Trust Center (1 per network)
Router (ZR):        Routes packets, extends range, mains-powered
End Device (ZED):   Leaf node, battery-powered, sleeps (parent = ZR or ZC)
  ├─ Sleepy End Device (SED): Polls parent for messages (duty cycle <1%)
  └─ Rx-on-when-idle: Always awake (rare in battery devices)
```

**Network Formation**:
```
1. Coordinator scans channels 11-26 for best channel (lowest interference)
2. Selects 16-bit PAN ID (0x0001-0xFFFE)
3. Assigns itself address 0x0000
4. Broadcasts Beacon (PAN ID, Extended PAN ID 64-bit, Join Permit)
5. Routers/End Devices send Association Request
6. Coordinator assigns 16-bit short address (stochastic addressing or tree)
7. Device receives Network Key (encrypted with install code-derived link key)
```

**Routing (AODV - Ad hoc On-Demand Distance Vector)**:
```
Route Discovery:
  Source sends Route Request (RREQ) broadcast → neighbors forward
  Destination sends Route Reply (RREP) unicast back to source
  Route stored in routing table (Next Hop, Cost)

Route Maintenance:
  Link failure detected → Route Error (RERR) sent to source
  Source initiates new RREQ
```

**Security**:
```
Trust Center Link Key (TCLK):
  ├─ Install Code method: AES-MMO hash of 6-20 byte install code (QR/NFC)
  │   → Derived preconfigured link key (unique per device)
  └─ Default Global Link Key: 0x5A6967426565416C6C69616E636530390909...
      (deprecated due to security risk, disabled in Zigbee 3.0 by default)

Network Key (NWK Key):
  ├─ 128-bit AES key distributed by Trust Center
  ├─ Encrypts all network-layer traffic (NWK frames)
  └─ Periodic key rotation (optional)

APS Link Key:
  ├─ End-to-end encryption between two devices (optional)
  └─ Used for sensitive data (e.g., door lock PIN codes)
```

**Zigbee Cluster Library (ZCL)**:
```
Cluster Structure:
  ├─ Attributes (state data, e.g., OnOff attribute in On/Off cluster)
  ├─ Commands (actions, e.g., Toggle command)
  └─ Reporting (periodic or on-change attribute updates)

Common Clusters:
  0x0000 - Basic (ZCL Version, Manufacturer, Model, Power Source)
  0x0003 - Identify (blink LED for device identification)
  0x0006 - On/Off (binary switch)
  0x0008 - Level Control (dimmer 0-255)
  0x0300 - Color Control (Hue, Saturation, XY, Color Temp)
  0x0201 - Thermostat (Local Temp, Setpoint, Mode)
  0x0400 - Illuminance Measurement (lux)
  0x0402 - Temperature Measurement (°C × 100)
  0x0500 - IAS Zone (motion, door/window sensor, alarm)
```

**Binding**:
```
Example: Bind light switch (source) to light bulb (destination)
  Switch (Endpoint 1, On/Off Client)
    → Binding Table entry:
      Cluster: 0x0006 (On/Off)
      DstAddr: 0xABCD (bulb's 16-bit address) or GroupID
      DstEndpoint: 1
  When switch pressed → On/Off command sent to bound device(s)
```

**Zigbee Application Profiles**:
```
Zigbee Home Automation (ZHA):        Legacy, deprecated
Zigbee Light Link (ZLL):             Touchlink commissioning (deprecated)
Zigbee 3.0:                          Unified spec (2016+), mandatory BDB commissioning
Zigbee Green Power (GP):             Energy harvesting devices (no battery)
  └─ GP Proxy/Sink: Translates GP frames to Zigbee
```

**BDB (Base Device Behavior) Commissioning**:
```
Network Steering:       Join existing network (scan, authenticate)
Network Formation:      Coordinator creates new network
Finding & Binding:      Automated binding without user mapping
Touchlink:              Proximity-based commissioning (legacy from ZLL)
```

---

## 5. Z-Wave (500 / 700 / 800 Series)

**Reference**: ITU-T G.9959 (PHY/MAC), Z-Wave Alliance Specification

**Frequencies**:
```
Region-specific sub-1 GHz ISM bands:
  EU: 868.4 MHz
  US: 908.4 MHz
  AU/NZ: 921.4 MHz
  JP: 922-926 MHz (multiple channels)
```

**Chip Generations**:
```
500 Series (ZW0500):     40 kbps, mesh routing, S2 security
700 Series (ZGM130):     100 kbps, SmartStart, Long Range (up to 1 mile LoS)
800 Series (ZG28):       100 kbps, Z-Wave Long Range (ZWLR), lower power
```

**Network Topology**:
```
Controller (Primary/Secondary):  Initiates commands, inclusion/exclusion
Routing Slave:                   Mains-powered, routes packets
Listening Slave:                 Always awake, does not route
Sleeping Slave:                  Battery-powered, wakes on FLiRS or WakeUp
FLiRS (Frequently Listening):    Wakes every ~250ms for low-latency (door locks)
```

**Routing**:
```
Source Routing (Explorer Frames):
  Controller sends Explorer Frame (broadcast flood)
  Each hop appends its Node ID to route list
  Destination receives multiple routes, selects shortest
  Controller stores route in routing table

Route Optimization:
  Periodic background route updates (Network Health Monitoring)
  Automatic rerouting on link failure (fallback routes)
```

**Security**:
```
S0 (Security 0): Legacy AES-128 encryption (single network key, deprecated)

S2 (Security 2):
  ├─ S2 Unauthenticated: Network key only (lowest security)
  ├─ S2 Authenticated:   DSK (Device Specific Key) verified via QR/PIN
  └─ S2 Access Control:  Separate keys for different device classes (doors, HVAC)

Key Exchange:
  1. Device sends Node Info Frame with S2 support
  2. Controller initiates Key Exchange (ECDH Curve25519)
  3. User verifies first 5 digits of DSK (QR code or manual entry)
  4. Controller grants security keys (S2 Authenticated, S2 Access, S0)
  5. Device stores keys in secure memory

Nonce-based Encryption:
  Each message uses unique nonce (Sender Nonce + Receiver Nonce)
  Prevents replay attacks
```

**SmartStart**:
```
1. Scan DSK QR code into controller app (device powered off)
2. Provision entry stored in controller's SmartStart list
3. Power on device → broadcasts Node Info Frame
4. Controller auto-includes device using pre-shared DSK
5. S2 Authenticated security applied automatically
(Simplifies installation: scan → power → done)
```

**Z-Wave Long Range (800 Series)**:
```
PHY: OQPSK modulation, 100 kbps
Range: Up to 1 mile (1.6 km) line-of-sight
Coexistence: Operates alongside traditional Z-Wave (dual-PHY)
Use case: Outdoor sensors, gate controllers, agricultural IoT
```

**Command Classes** (Application Layer):
```
0x20 - Basic (generic on/off, deprecated in favor of specific classes)
0x25 - Switch Binary (on/off switch)
0x26 - Switch Multilevel (dimmer 0-99%)
0x32 - Meter (energy consumption kWh, power W, voltage V, current A)
0x43 - Thermostat Setpoint (heating/cooling setpoint)
0x62 - Door Lock (lock/unlock, user code management)
0x70 - Configuration (device-specific parameters)
0x71 - Notification (motion, tamper, door/window open, smoke, CO)
0x84 - Wake Up (sleeping device wake interval and notification)
0x8E - Multi Channel (control multiple endpoints in single device)
0x98 - Security (S0 encapsulation)
0x9F - Security 2 (S2 encapsulation)
```

**Z-Wave Plus v2** (2017):
- S2 security mandatory
- SmartStart support
- Improved battery life (dynamic power management)
- Network-wide inclusion (no proximity required)

---

## 6. Thread / OpenThread

**Reference**: Thread 1.3.0 Specification (Thread Group), RFC 8931 (IPv6 over Thread)

**Architecture**:
```
Thread Network (802.15.4 mesh)
  ↓
6LoWPAN (IPv6 compression)
  ↓
Thread Network Layer
  ├─ Leader election (single Leader per partition)
  ├─ Router promotion/demotion
  └─ Addressing (RLOC16, ML-EID)
  ↓
Thread Transport (UDP)
  ↓
Application (CoAP, Matter)
```

**Device Roles**:
```
Leader:             Partition coordinator, assigns Router IDs, maintains network data
Router:             Routes packets, maintains child table (max 511 children)
Router-Eligible End Device (REED): Can become Router if needed
End Device (ED):    Sleepy, polls parent Router for messages
Border Router:      Connects Thread network to Wi-Fi/Ethernet (IPv6 gateway)
Commissioner:       Authenticates new devices (DTLS-based joining)
```

**Network Formation**:
```
1. Device scans 802.15.4 channels (11-26 @ 2.4 GHz)
2. If no network found → Becomes Leader, selects PAN ID, Channel, Network Key
3. Leader assigns itself Router ID 0
4. Other devices attach as End Devices → promoted to Routers as needed
   (Max 32 Routers per partition, dynamic based on topology)
```

**Addressing**:
```
RLOC16 (Routing Locator):
  ├─ 16-bit address: [Router ID 6 bits][Child ID 9 bits]
  └─ Example: 0x0401 → Router 1, Child 1

ML-EID (Mesh-Local Endpoint Identifier):
  ├─ 64-bit IID derived from EUI-64 (stable across network changes)
  └─ Prefix: fd00::/64 (Mesh-Local Prefix unique per network)

Link-Local Address: fe80::IID (IEEE 802.15.4 MAC-derived)

Global Unicast Address (GUA):
  ├─ Assigned by Border Router (RA - Router Advertisement)
  └─ Example: 2001:db8::1234:5678
```

**Routing Protocol (MLE - Mesh Link Establishment)**:
```
MLE Advertisement:      Leader/Routers advertise network parameters
MLE Parent Request:     End Device discovers parent Routers
MLE Parent Response:    Router responds with link quality
MLE Child ID Request:   End Device attaches to selected parent
MLE Child ID Response:  Router assigns RLOC16
```

**Commissioning**:
```
1. Commissioner creates PSKc (Pre-Shared Key for Commissioner) from passphrase
2. Joiner sends DTLS Client Hello with Joiner ID (EUI-64 or text)
3. Commissioner responds with DTLS Server Hello (PSKc-based auth)
4. Commissioner sends Network Key (DTLS-encrypted)
5. Joiner stores credentials → reboots → attaches to network as End Device
```

**Thread 1.2 Features (2019)**:
```
Backbone Link:          5 GHz 802.11 or Ethernet backbone for multi-hop reduction
Domain Unicast Addressing: Seamless roaming across Border Routers
Commercial Extensions: Centralized commissioning, network diagnostics
```

**Thread 1.3 Features (2021)**:
```
Link Metrics:           RSSI, LQI reporting for adaptive routing
Enhanced Frame Pending: Faster sleepy end device polling
Coordinated Sampled Listening (CSL): Ultra-low power synchronized wake-up
```

**OpenThread**:
- Open-source Thread stack by Google (adopted by Matter)
- Certified Thread 1.3.0 implementation
- Platforms: nRF52840, EFR32MG, CC2652, Qorvo QPG6105

Cross-reference: [smart-home-consumer.md](smart-home-consumer.md) section 1 (Matter uses Thread as transport).

---

## 7. IEEE 802.15.4 (MAC/PHY Layer)

**Reference**: IEEE 802.15.4-2020

**PHY Bands**:
```
868 MHz (Europe):     1 channel, 20/100/250 kbps (BPSK/ASK/O-QPSK)
915 MHz (Americas):   10 channels, 40/250 kbps (BPSK/O-QPSK)
2.4 GHz (Global):     16 channels (11-26), 250 kbps (O-QPSK)
  └─ Channel spacing: 5 MHz (2405 + 5×(ch-11) MHz)
```

**Frame Types**:
```
Beacon:            Synchronization (coordinator broadcasts PAN ID, superframe spec)
Data:              Payload up to 127 bytes (MAC header ~25 bytes → ~102B payload)
Acknowledgment:    5-byte ACK for confirmed delivery
MAC Command:       Association, disassociation, data request, orphan scan
```

**Addressing**:
```
Short Address:     16-bit (0x0000-0xFFFD), 0xFFFE = no short addr, 0xFFFF = broadcast
Extended Address:  64-bit (EUI-64), globally unique
PAN ID:            16-bit network identifier (0x0000-0xFFFD)
```

**CSMA/CA (Carrier Sense Multiple Access with Collision Avoidance)**:
```
1. Random backoff (2^BE - 1 slots, BE = Backoff Exponent starts at 3)
2. CCA (Clear Channel Assessment): Listen for 128 μs
3. If channel busy → increment BE, retry (max macMaxCSMABackoffs = 4)
4. If channel clear → transmit
5. Wait for ACK (macAckWaitDuration = 54 symbol periods @ 2.4 GHz)
```

**Security (AES-128 CCM\*)**:
```
Security Levels:
  0 - None
  1 - MIC-32 (32-bit MIC, no encryption)
  2 - MIC-64 / MIC-128
  4 - ENC (encryption only, no integrity)
  5 - ENC-MIC-32 / ENC-MIC-64 / ENC-MIC-128 (authenticated encryption)

Key Management:
  ├─ Key ID Mode 0: Implicit (key known from context)
  ├─ Key ID Mode 1: Key Index (1 byte)
  └─ Key ID Mode 2/3: Key Source + Index

Frame Counter:
  ├─ 32-bit monotonic counter (prevents replay)
  └─ Device maintains per-key frame counter
```

**Superframe Structure** (Beacon-enabled mode):
```
Beacon Interval (BI) = BaseSuperframeDuration × 2^BO (BO = Beacon Order 0-15)
Superframe Duration (SD) = BaseSuperframeDuration × 2^SO (SO = Superframe Order 0-BO)

┌─────────────────────────────────────────────────────┐
│ Beacon │ CAP (CSMA/CA) │ CFP (GTS slots) │ Inactive │
└─────────────────────────────────────────────────────┘
          ← Superframe →                    ← Sleep   →

CAP (Contention Access Period):  CSMA/CA for all devices
CFP (Contention-Free Period):    GTS (Guaranteed Time Slots) for latency-sensitive
Inactive:                         Devices sleep (battery saving)
```

**Association Flow**:
```
Device                           Coordinator
  │                                  │
  ├─ 1. Beacon Request (broadcast) ──>│
  │                                  │
  │<─ 2. Beacon Response ─────────────┤
  │    (PAN ID, Coordinator Addr)    │
  │                                  │
  ├─ 3. Association Request ─────────>│
  │    (Device Capability)           │
  │                                  │
  │<─ 4. ACK ─────────────────────────┤
  │                                  │
  ├─ 5. Data Request (poll for resp)─>│
  │                                  │
  │<─ 6. Association Response ────────┤
  │    (Short Address 0x0001-0xFFFD) │
  │    or (0xFFFF if failed)         │
```

---

## 8. WirelessHART (IEC 62591)

**Reference**: IEC 62591:2016, HART 7.x Specification

**Architecture**:
```
Field Devices (sensors, actuators)
  ↓ WirelessHART (IEEE 802.15.4 PHY, 2.4 GHz, TSCH MAC)
Gateway (converts to wired HART or Ethernet)
  ↓
Network Manager (centralized routing, scheduling)
  ↓
Plant Asset Management / DCS
```

**TSCH (Time-Slotted Channel Hopping)**:
```
Time Slot: 10 ms (fixed)
  ├─ TX: 1st half (5 ms)
  └─ RX ACK: 2nd half (5 ms)

Channel Hopping:
  ├─ 15 channels (11-25, avoiding 26 due to overlap with Wi-Fi)
  ├─ ASN (Absolute Slot Number): Global time reference
  └─ Channel = f(ASN, slot offset) via hash function
      (Frequency diversity → mitigate interference)
```

**Mesh Routing**:
```
Graph Routing:
  Network Manager computes directed graphs
  ├─ Uplink graph (field device → gateway)
  └─ Downlink graph (gateway → field device)
  Each device has routing table entry: [Next Hop, Graph ID]

Source Routing:
  Gateway specifies full path in message header
  Used for diagnostics and commissioning
```

**Security**:
```
Symmetric Keys:
  ├─ Join Key (pre-shared, device-specific)
  ├─ Network Key (distributed after join)
  └─ Session Keys (per-link, derived from Network Key + nonce)

MIC (Message Integrity Code):
  ├─ AES-128-CCM with 4-byte MIC
  └─ Protects against tampering

Nonce:
  ├─ Source Address + ASN + frame counter
  └─ Ensures unique IV per message
```

**Power Management**:
```
Dedicated Slots:     Device assigned specific slots in schedule (power efficient)
Shared Slots:        Contention-based (used for retries, low-priority traffic)
Sleep Scheduling:    Device sleeps between assigned slots (99% duty cycle reduction)
```

**Use Case**: Process automation (oil/gas refineries, chemical plants). Requires precise timing and reliability. See [industrial-ot.md](industrial-ot.md).

---

## 9. ISA100.11a (IEC 62734)

**Reference**: IEC 62734:2014, ISA100.11a-2011

**Similarities to WirelessHART**:
- IEEE 802.15.4 PHY (2.4 GHz)
- TSCH MAC (10 ms slots, channel hopping)
- AES-128-CCM security

**Differences**:
```
Tunneling:        ISA100 supports IPv6 (6LoWPAN)
Routing:          IETF RPL (Routing Protocol for LLNs) instead of graph routing
QoS:              4 priority levels (network control, real-time, normal, low)
Backbone:         Native IP backbone (vs HART's proprietary gateway)
```

**Device Roles**:
```
Field Device:         Sensor/actuator
Backbone Router:      Connects to plant backbone (Ethernet/Wi-Fi)
System Manager:       Network configuration, key distribution
Security Manager:     Key lifecycle, certificate management
Gateway:              Protocol translation (e.g., Modbus to ISA100)
```

**6LoWPAN Integration**:
```
IPv6 Packet (1280 bytes MTU)
  ↓ 6LoWPAN compression (IPHC, NHC)
IEEE 802.15.4 Frame (127 bytes)
  └─ Header compression: IPv6 40B → 2-6B
  └─ UDP 8B → 4B
```

**Contract-based Communication**:
```
Contract:     Service agreement between devices (bandwidth, latency, reliability)
  ├─ Period:  Minimum interval between messages (100 ms - 60 s)
  ├─ Deadline: Max latency (10 ms - 10 s)
  └─ Phase:   Slot offset within superframe

Example:
  Pressure sensor publishes every 1s, deadline 500ms → Contract ID 42
  System Manager allocates dedicated slots at ASN % 100 == 0
```

**Use Case**: Factory automation, building automation, smart grid. More flexible than WirelessHART due to IPv6 support.

---

## 10. Ultra-Wideband (UWB) - IEEE 802.15.4a/4z

**Reference**: IEEE 802.15.4-2020 (4a/4z amendments), CCC Digital Key 3.0

**PHY Layer**:
```
Frequency Bands:
  ├─ Sub-GHz: 250-750 MHz (low band), 3.1-4.8 GHz (high band, rare)
  └─ 6-9 GHz: Most common (Channel 5: 6.5 GHz, Channel 9: 8 GHz)

Modulation: Impulse Radio (IR-UWB)
  ├─ 500 MHz or 1 GHz bandwidth
  └─ Sub-nanosecond pulses → cm-level ranging accuracy

Data Rate: 110 kbps, 850 kbps, 6.8 Mbps
```

**Ranging (Time-of-Flight)**:
```
Two-Way Ranging (TWR):

Initiator                                   Responder
    │                                           │
    ├─ 1. Poll (T1) ─────────────────────────> │
    │                                           │
    │ <─ 2. Response (T2, T3) ──────────────────┤
    │                                           │
    ├─ 3. Final (T4, T5, T6) ─────────────────> │
    │                                           │
    └─ Calculate distance:
       Round Trip Time (RTT) = [(T4-T1) - (T3-T2) + (T6-T5) - (T3-T2)] / 2
       Distance = RTT × c (speed of light) / 2

Accuracy: 10-30 cm (typical indoor), sub-10 cm in ideal conditions
```

**IEEE 802.15.4z (2020) - Enhanced Ranging**:
```
Scrambled Timestamp Sequence (STS):
  ├─ Cryptographically secure pseudorandom sequence
  ├─ Prevents relay attacks (ghost peak injection)
  └─ Derived from session key (AES-128)

HRP UWB PHY (High-Rate Pulse):
  ├─ 27 Mbps peak data rate
  └─ Improved coexistence with narrowband systems
```

**Angle of Arrival (AoA)**:
```
Antenna Array (Receiver):
  ├─ 2+ antennas with known spacing (e.g., λ/2 apart)
  ├─ Measure phase difference between antennas
  └─ Calculate azimuth/elevation angle

Use case: Indoor positioning (1° angular accuracy)
  Phone with UWB → AoA anchor points → cm-level localization
```

**CCC Digital Key 3.0** (Car Connectivity Consortium):
```
UWB-based Passive Keyless Entry (PKE):
  1. Car advertises BLE (discovery)
  2. Phone responds via BLE (authentication starts)
  3. Transition to UWB ranging session
  4. Car measures distance (STS-secured ranging prevents relay)
  5. If distance <1.5m → Unlock (or <0.5m → Start engine)

Security:
  ├─ Applet in phone's Secure Element (eSE)
  ├─ SPAKE2+ pairing (owner enrolls phone to car)
  └─ STS keys rotated per session (ephemeral)
```

Cross-reference: [automotive-transport.md](automotive-transport.md) section 5 (CCC Digital Key).

---

## 11. DECT NR+ / DECT ULE

**Reference**: ETSI TS 103 636 (DECT-2020 NR), ETSI TS 102 939 (DECT ULE)

**DECT ULE (Ultra Low Energy)**:
```
Frequency: 1880-1900 MHz (Europe), 1920-1930 MHz (US, unlicensed)
PHY: GFSK, 1.152 Mbps
Range: 50-300m (outdoor)
Topology: Star (base station + sensors)
Power: <1 mW TX (10+ years battery life)
```

**Use Case**: Smart home sensors (door/window, motion), EU market (DECT spectrum reservation).

**DECT-2020 NR (New Radio) / DECT NR+**:
```
Mesh topology:          Self-healing, multi-hop
Data Rate:              Up to 3 Mbps
Latency:                <10 ms (time-slotted)
Security:               AES-128-CCM
IoT Profile:            Low-power nodes, coordinator-based sync
```

**DECT vs Thread/Zigbee**:
- **Advantage**: Licensed/reserved spectrum (less interference than 2.4 GHz)
- **Disadvantage**: Regional (Europe-centric), limited ecosystem vs Zigbee/Thread

Cross-reference: [smart-home-consumer.md](smart-home-consumer.md) section 6 (DECT NR+ in Matter over IP).

---

## 12. NFC (ISO/IEC 14443, ISO/IEC 18092)

**Reference**: ISO/IEC 14443 (Proximity Cards), ISO/IEC 18092 (NFCIP-1), NFC Forum

**Frequencies**: 13.56 MHz (ISM band)

**Operating Modes**:
```
Reader/Writer Mode:      Phone reads NFC tag (NDEF data)
Peer-to-Peer (P2P):      Two phones exchange data (Android Beam, file sharing)
Card Emulation:          Phone emulates contactless card (payments, transit)
  └─ HCE (Host Card Emulation): Software-based, no Secure Element required
```

**NFC Tag Types**:
```
Type 1 (Topaz):          96-2048 bytes, read/write, Innovision/Broadcom
Type 2 (MIFARE Ultralight): 48-2048 bytes, most common for consumer tags
Type 3 (FeliCa):         Variable, Sony, used in Japan (Suica transit cards)
Type 4 (DESFire):        4-32 KB, ISO/IEC 7816 compliant, high security
Type 5 (ISO 15693):      Vicinity cards, up to 1m range
```

**NDEF (NFC Data Exchange Format)**:
```
NDEF Message:
  └─ NDEF Record 1
      ├─ TNF (Type Name Format): URI, MIME, External
      ├─ Type: "text/plain", "application/vnd.wfa.wsc" (Wi-Fi config)
      ├─ ID: Optional identifier
      └─ Payload: "https://example.com" or {SSID, password}
```

**NFC Pairing (Wi-Fi Easy Connect, Bluetooth OOB)**:
```
Tag contains:
  ├─ Bluetooth MAC address
  ├─ Out-of-Band (OOB) data (public key hash)
  └─ Pairing passkey

Phone taps tag → Auto-pairs Bluetooth (no PIN entry)
```

**Use Cases**:
- **Smart Posters**: Tap to open URL (museum exhibits, marketing)
- **Access Control**: MIFARE DESFire badges (offices, hotels)
- **Payments**: EMV contactless (Visa payWave, Mastercard PayPass)
- **IoT Provisioning**: Tap-to-provision (Zigbee install codes, Matter setup codes)

---

## 13. EnOcean (ISO/IEC 14543-3-10)

**Reference**: ISO/IEC 14543-3-10, EnOcean Equipment Profiles (EEP)

**Energy Harvesting**:
```
Sources:
  ├─ Kinetic: Mechanical switch press (piezoe electric generator, 120 μWs/press)
  ├─ Solar: Photovoltaic cells (10-20 μW indoor light)
  └─ Thermal: Peltier element (temperature differential, 10-50 μW)
```

**PHY**:
```
Frequency: 868 MHz (EU), 902 MHz (US), 928 MHz (JP)
Modulation: ASK
Data Rate: 125 kbps
Packet: 14 bytes (1B sync, 4B header, 4B data, 1B CRC, 4B optional)
Range: 30m indoor, 300m outdoor
```

**Telegram Structure**:
```
Preamble | Sync | Header | Data | CRC
  └─ RORG (Radio Org): 1BS (1 byte), 4BS (4 bytes), RPS (rocker switch)
  └─ Data: Sensor value or switch state
  └─ Sender ID: 32-bit unique identifier (no pairing needed)
```

**EnOcean Equipment Profiles (EEP)**:
```
A5-02-05:   Temperature sensor (0-40°C, 0.25°C resolution)
A5-04-01:   Humidity + Temperature
A5-07-01:   Occupancy sensor (PIR + illuminance)
D2-01-12:   Electronic switch (on/off actuator)
F6-02-01:   Rocker switch (2-button)
```

**Gateway Integration**:
```
EnOcean devices → EnOcean USB Gateway (TCM 310)
                   ↓
                 Controller (Home Assistant, openHAB)
                   └─ MQTT bridge to cloud
```

**Use Case**: Battery-free wireless switches, hotel room automation, HVAC sensors. Niche market due to proprietary nature but unique energy harvesting advantage.

---

## Key Specification References

| Protocol       | Primary Spec                        | Publisher       | Year |
|----------------|-------------------------------------|-----------------|------|
| BLE 5.4        | Bluetooth Core Spec 5.4             | Bluetooth SIG   | 2023 |
| Bluetooth Mesh | Mesh Profile 1.1                    | Bluetooth SIG   | 2021 |
| Zigbee PRO     | Zigbee PRO 2023 (R23)               | CSA             | 2023 |
| Z-Wave         | Z-Wave Specification                | Z-Wave Alliance | 2023 |
| Thread 1.3     | Thread Specification 1.3.0          | Thread Group    | 2021 |
| IEEE 802.15.4  | IEEE 802.15.4-2020                  | IEEE            | 2020 |
| WirelessHART   | IEC 62591:2016                      | IEC             | 2016 |
| ISA100.11a     | IEC 62734:2014                      | IEC             | 2014 |
| UWB            | IEEE 802.15.4z-2020                 | IEEE            | 2020 |
| DECT NR+       | ETSI TS 103 636                     | ETSI            | 2022 |
| NFC            | ISO/IEC 14443, 18092                | ISO/IEC         | 2016 |
| EnOcean        | ISO/IEC 14543-3-10                  | ISO/IEC         | 2012 |

---

## Related Files
- **[smart-home-consumer.md](smart-home-consumer.md)**: Matter 1.x (uses Thread/Wi-Fi), BLE commissioning, DECT NR+
- **[industrial-ot.md](industrial-ot.md)**: WirelessHART, ISA100.11a in process automation, 6TiSCH
- **[automotive-transport.md](automotive-transport.md)**: CCC Digital Key 3.0 (UWB + NFC), BLE TPMS
- **[healthcare-iot.md](healthcare-iot.md)**: BLE GATT health profiles (Heart Rate, Glucose, Pulse Ox)
- **[tsn-deterministic.md](tsn-deterministic.md)**: IEEE 802.15.4 TSCH integration with TSN
