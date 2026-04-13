# Automotive & Transport IoT Protocols

> **Scope:** This reference covers automotive and electric vehicle protocols — from vehicle
> diagnostics (OBD-II, UDS), software architecture (AUTOSAR Classic/Adaptive), EV charging
> (ISO 15118, OCPP), digital keys (CCC Digital Key 3.0), to cybersecurity regulations
> (UNECE WP.29 R155/R156). For V2X protocols (IEEE 802.11p, C-V2X, NR-V2X), see
> [cellular-wan.md](cellular-wan.md).

---

## Table of Contents

1. [Vehicle Diagnostics](#1-vehicle-diagnostics)
   - [OBD-II — On-Board Diagnostics](#obd-ii--on-board-diagnostics)
   - [UDS — Unified Diagnostic Services (ISO 14229)](#uds--unified-diagnostic-services-iso-14229)
   - [SAE J1939 — Heavy Vehicle CAN Protocol](#sae-j1939--heavy-vehicle-can-protocol)
2. [Vehicle Software Architecture](#2-vehicle-software-architecture)
   - [AUTOSAR Classic](#autosar-classic)
   - [AUTOSAR Adaptive](#autosar-adaptive)
3. [Digital Keys & Connectivity](#3-digital-keys--connectivity)
   - [CCC Digital Key 3.0](#ccc-digital-key-30)
4. [EV Charging Protocols](#4-ev-charging-protocols)
   - [ISO 15118 — EV Charging Communication](#iso-15118--ev-charging-communication)
   - [OCPP — Open Charge Point Protocol](#ocpp--open-charge-point-protocol)
   - [OCPI — Open Charge Point Interface](#ocpi--open-charge-point-interface)
5. [Cybersecurity & Regulatory](#5-cybersecurity--regulatory)
   - [UNECE WP.29 R155 — Cybersecurity Management](#unece-wp29-r155--cybersecurity-management)
   - [UNECE WP.29 R156 — Software Update Management](#unece-wp29-r156--software-update-management)

---

## 1. Vehicle Diagnostics

### OBD-II — On-Board Diagnostics

**OBD-II** is the standardized vehicle diagnostic protocol mandated in the US (1996+), EU (2001+ petrol, 2004+ diesel), and globally.

**Physical Interface:**
- **J1962 Connector**: 16-pin Data Link Connector (DLC) under dashboard
- Pin assignments:
  - Pin 4, 5: Chassis ground
  - Pin 16: Battery power (12V)
  - Pin 6, 14: CAN High/Low (ISO 15765-4, most common since 2008)
  - Pin 7: K-Line (ISO 9141-2 / ISO 14230-4 KWP2000, legacy)
  - Pin 2, 10: J1850 PWM/VPW (GM/Ford, pre-2008)

**Protocols:**
- **ISO 15765-4 (CAN)**: 250 kbps or 500 kbps, dominant since 2008
- **ISO 9141-2**: K-Line, 10.4 kbps (legacy Asian vehicles)
- **ISO 14230-4 (KWP2000)**: K-Line, keyword protocol (legacy European)
- **SAE J1850 PWM/VPV**: 41.6 kbps / 10.4 kbps (Ford/GM, legacy US)

**OBD-II Modes:**

| Mode | Service | Description | Example PIDs |
|---|---|---|---|
| Mode 01 | Show current data | Real-time sensor values | 0x0C (Engine RPM), 0x0D (Vehicle Speed), 0x05 (Coolant Temp), 0x0F (Intake Air Temp), 0x2F (Fuel Level) |
| Mode 02 | Show freeze frame data | Snapshot when DTC was set | Same PIDs as Mode 01 |
| Mode 03 | Show stored DTCs | Emission-related trouble codes | Returns DTC list (e.g., P0171, P0300) |
| Mode 04 | Clear DTCs | Clear trouble codes and freeze frame | (no PID) |
| Mode 05 | Oxygen sensor monitoring | O2 sensor test results | (varies by vehicle) |
| Mode 06 | On-board test results | Non-continuously monitored systems | (varies) |
| Mode 07 | Show pending DTCs | DTCs detected current/last driving cycle | (varies) |
| Mode 08 | Control on-board systems | Bi-directional control (test mode) | (varies) |
| Mode 09 | Request vehicle info | VIN, calibration ID, ECU name | 0x02 (VIN), 0x0A (ECU Name) |
| Mode 0A | Permanent DTCs | Emissions DTCs (cannot be cleared manually) | (varies) |

**PID (Parameter ID) Structure:**
- Request: `[Mode] [PID]` (e.g., `01 0C` = request Engine RPM)
- Response: `[Mode + 0x40] [PID] [Data Bytes]` (e.g., `41 0C 1A F8` = 6904 RPM)
- Data decoding: Formula varies per PID (e.g., RPM = ((A × 256) + B) / 4)

**DTC (Diagnostic Trouble Code) Format:**
- **5 characters**: Letter + 4 digits (e.g., `P0300`)
- **First character** (system):
  - P = Powertrain
  - C = Chassis
  - B = Body
  - U = Network/Communication
- **Second digit** (code type):
  - 0 = SAE-defined (generic)
  - 1 = Manufacturer-specific
  - 2/3 = SAE-reserved / manufacturer-specific
- **Third digit** (subsystem):
  - 0 = Entire system
  - 1 = Fuel/Air
  - 2 = Fuel/Air (injector circuit)
  - 3 = Ignition
  - 4 = Emission control
  - 5 = Speed/Idle
  - 6 = Computer/Output
  - 7/8 = Transmission
- **Last 2 digits**: Specific fault code

**Common DTCs:**
- `P0300`: Random/multiple cylinder misfire
- `P0171`: System too lean (Bank 1)
- `P0420`: Catalyst efficiency below threshold (Bank 1)
- `P0455`: EVAP large leak detected

**OBD-II vs Extended Diagnostics:**
- OBD-II: Standardized, emission-related only, limited access
- **UDS (ISO 14229)**: Manufacturer-extended diagnostics, full ECU access, requires authentication

---

### UDS — Unified Diagnostic Services (ISO 14229)

**UDS** is the comprehensive diagnostic protocol for modern vehicles, supporting FOTA, ECU programming, and advanced diagnostics.

**Specifications:**
- **ISO 14229-1**: Service specification (diagnostic services)
- **ISO 14229-2**: Session layer services
- **ISO 14229-3**: Unified Diagnostic Services on CAN (UDSonCAN)
- **ISO 14229-4**: UDS on FlexRay
- **ISO 14229-5**: UDS on IP (Diagnostic over IP — DoIP, Ethernet)
- **ISO 14229-6**: UDS on K-Line
- **ISO 14229-7**: UDS on LIN

**Transport Layer:**
- **ISO 15765-2 (CAN)**: 500 kbps, segmented messages (up to 4095 bytes)
- **ISO 13400 (DoIP)**: Diagnostics over IP, Ethernet-based (100 Mbps / 1 Gbps)
  - UDP port 13400 (vehicle discovery)
  - TCP port 13400 (diagnostic communication)
  - Replaces K-Line for modern vehicles with Ethernet backbones

**UDS Service Structure:**
- Request: `[SID] [Sub-function] [Data...]`
- Positive Response: `[SID + 0x40] [Sub-function] [Data...]`
- Negative Response: `0x7F [SID] [NRC]` (NRC = Negative Response Code)

**Core UDS Services:**

| SID (hex) | Service Name | Function | Use Case |
|---|---|---|---|
| **0x10** | DiagnosticSessionControl | Switch diagnostic session (default, programming, extended) | Enter programming mode for FOTA |
| **0x11** | ECUReset | Reset ECU (hard, soft, key off/on) | Apply firmware after flash |
| **0x14** | ClearDiagnosticInformation | Clear DTCs | Dealer service after repair |
| **0x19** | ReadDTCInformation | Read DTCs by status, snapshot, extended data | Detailed diagnostics |
| **0x22** | ReadDataByIdentifier | Read data via DID (Data Identifier) | Read VIN, software version, sensor values |
| **0x23** | ReadMemoryByAddress | Read memory at physical address | Advanced debugging |
| **0x27** | **SecurityAccess** | **Seed-key authentication for protected services** | **Unlock ECU for programming/calibration** |
| **0x28** | CommunicationControl | Enable/disable message transmission | Suppress CAN traffic during programming |
| **0x2E** | WriteDataByIdentifier | Write configuration data via DID | Calibration, VIN programming |
| **0x31** | **RoutineControl** | Start/stop/request results of diagnostic routines | Execute self-tests, erase flash memory |
| **0x34** | **RequestDownload** | **Initiate firmware download to ECU** | **FOTA step 1: Prepare for flash** |
| **0x35** | RequestUpload | Upload data from ECU to tester | Backup calibration data |
| **0x36** | **TransferData** | **Transfer firmware blocks** | **FOTA step 2: Send firmware chunks** |
| **0x37** | **RequestTransferExit** | **Complete transfer** | **FOTA step 3: Finalize and verify** |
| **0x3E** | TesterPresent | Keep session alive | Prevent timeout during long operations |
| **0x85** | ControlDTCSetting | Enable/disable DTC storage | Prevent false DTCs during calibration |

**UDS Session Types (Service 0x10):**
- **0x01: Default Session** — Standard diagnostics (OBD-II equivalent)
- **0x02: Programming Session** — ECU reprogramming (FOTA)
- **0x03: Extended Diagnostic Session** — Advanced diagnostics, calibration
- **0x04–0x7F**: Manufacturer-specific sessions

**Security Access (Service 0x27) — Seed-Key Authentication:**

```
Diagnostic Tester                   ECU
    │                                │
    ├─ 1. Request Seed (0x27 0x01) ──>
    │                                │
    │  <── 2. Seed (4-byte random)   │
    │      (0x67 0x01 [seed bytes])  │
    │                                │
    ├─ 3. Send Key (0x27 0x02)    ──>
    │      (key = algorithm(seed))   │
    │                                │
    │  <── 4. Positive Response      │
    │      (0x67 0x02)               │
    │      OR Negative (0x7F 0x27 0x35 = Invalid Key)
    │                                │
    └─ 5. Protected services unlocked│
```

- **Seed**: Random challenge generated by ECU
- **Key**: Response computed by tester using proprietary algorithm (HSM-protected, OEM-specific)
- **Security levels**: Multiple levels (e.g., 0x01/0x02 for calibration, 0x11/0x12 for programming)
- **Lockout**: After 3–5 failed attempts, ECU locks for time period or requires power cycle

**FOTA (Firmware Over-The-Air) Flow via UDS:**

```
1. DiagnosticSessionControl (0x10 0x02) → Enter Programming Session
2. SecurityAccess (0x27 0x01/0x02)      → Authenticate
3. RoutineControl (0x31 0x01 0xFF00)    → Erase flash memory
4. RequestDownload (0x34)               → Prepare for firmware download
5. TransferData (0x36) × N              → Send firmware in blocks (e.g., 4 KB each)
6. RequestTransferExit (0x37)           → Finalize transfer
7. RoutineControl (0x31 0x01 0x0202)    → Verify integrity (CRC/signature check)
8. ECUReset (0x11 0x01)                 → Hard reset to boot new firmware
9. DiagnosticSessionControl (0x10 0x01) → Return to default session
```

**Negative Response Codes (NRC):**
- `0x11`: Service not supported
- `0x12`: Sub-function not supported
- `0x13`: Incorrect message length
- `0x22`: Conditions not correct (e.g., vehicle not in Park)
- `0x24`: Request sequence error (e.g., TransferData before RequestDownload)
- `0x31`: Request out of range
- `0x33`: Security access denied
- `0x35`: Invalid key (SecurityAccess)
- `0x36`: Exceeded number of attempts (SecurityAccess lockout)
- `0x78`: Response pending (ECU needs more time)

---

### SAE J1939 — Heavy Vehicle CAN Protocol

**SAE J1939** is the CAN-based protocol for heavy-duty vehicles (trucks, buses, construction, agriculture).

**Physical Layer:**
- **CAN 2.0B** (29-bit extended identifier)
- **250 kbps** or **500 kbps**
- **Two-wire differential**: CAN High (yellow), CAN Low (green)
- **120Ω termination resistors** at both ends of bus

**Message Structure (29-bit Identifier):**
```
[Priority (3 bits)] [Reserved (1)] [Data Page (1)] [PDU Format (8)] [PDU Specific (8)] [Source Address (8)]
```

- **Priority**: 0 (highest) to 7 (lowest)
- **PGN (Parameter Group Number)**: 18-bit identifier (Reserved + Data Page + PDU Format + PDU Specific)
- **Source Address**: ECU address (0–253)

**Common PGNs:**
- `0xF004` (61444): Electronic Engine Controller 1 (EEC1) — Engine speed, torque
- `0xFEF1` (65265): Cruise Control/Vehicle Speed (CCVS)
- `0xFECA` (65226): Diagnostic Message 1 (DM1) — Active DTCs
- `0xFEF5` (65269): Ambient Conditions — Barometric pressure, temperature
- `0x00EA00` (59904): Request PGN (used to request specific data)

**Transport Protocol (J1939-21):**
- **BAM (Broadcast Announce Message)**: Broadcast multi-packet messages
- **CMDT (Connection Mode Data Transfer)**: Peer-to-peer multi-packet transfer
- Supports messages up to 1785 bytes (split into 8-byte CAN frames)

**Diagnostics (J1939-73):**
- **DM1** (Active DTCs): Continuously broadcast active faults
- **DM2** (Previously Active DTCs): History of resolved faults
- **DM3** (Clear DTCs): Request to clear diagnostic info
- **SPN (Suspect Parameter Number)**: Identifies faulty parameter (e.g., SPN 110 = Engine Coolant Temperature)
- **FMI (Failure Mode Identifier)**: Type of failure (0–31, e.g., 3 = voltage above normal, 4 = voltage below normal)

---

## 2. Vehicle Software Architecture

### AUTOSAR Classic

**AUTOSAR (AUTomotive Open System ARchitecture)** is the standardized software architecture for automotive ECUs.

**AUTOSAR Classic** (original, for microcontroller-based ECUs with static scheduling).

**Layered Architecture:**

```
┌─────────────────────────────────────────────────┐
│  Application Layer (SW-C = Software Components) │
│  ────────────────────────────────────────────── │
│  RTE (Runtime Environment) — API abstraction    │  ← Virtual Function Bus (VFB)
├─────────────────────────────────────────────────┤
│  BSW (Basic Software)                           │
│  ┌───────────────────────────────────────────┐  │
│  │ Services Layer                            │  │
│  │  - ECU State Manager, Memory, Diagnostics│  │
│  │  - COM (Communication), DCMM (Diagnostic)│  │
│  │  - NvM (Non-volatile Memory), WdgM        │  │
│  ├───────────────────────────────────────────┤  │
│  │ ECU Abstraction Layer                     │  │
│  │  - CAN Interface, LIN Interface, FlexRay │  │
│  ├───────────────────────────────────────────┤  │
│  │ MCAL (Microcontroller Abstraction Layer) │  │
│  │  - CAN Driver, SPI Driver, ADC Driver    │  │
│  └───────────────────────────────────────────┘  │
├─────────────────────────────────────────────────┤
│  Hardware (Microcontroller + Peripherals)       │
└─────────────────────────────────────────────────┘
```

**RTE (Runtime Environment):**
- Virtual Function Bus (VFB) abstraction
- Allows SW-Cs to communicate without knowing physical ECU mapping
- Sender/Receiver communication (signals) and Client/Server (operations)

**Key BSW Modules:**
- **COM**: Communication manager (CAN/LIN/FlexRay signal packing/unpacking)
- **COMM**: Communication manager (network startup, sleep, wake-up)
- **DCMM**: Diagnostic Communication Manager (UDS service handling)
- **UCM**: Update and Configuration Manager (FOTA coordination, Adaptive R20-11+)
- **CanIf**: CAN Interface (abstraction over CAN driver)
- **NvM**: Non-volatile Memory Manager (EEPROM/Flash data persistence)
- **WdgM**: Watchdog Manager (monitors task execution, resets on hang)
- **Det**: Default Error Tracer (development error reporting)

**AUTOSAR Classic Releases:**
- **R4.0** (2011): First production-grade release
- **R4.3** (2016): Ethernet support (TCP/IP, SOME/IP), partial updates
- **R4.4** (2018): UCM (Update and Configuration Manager), enhanced security (SecOC)
- **R20-11** (2020): Harmonized with Adaptive AUTOSAR

**Communication Stacks:**
- **CAN**: ISO 11898, CanIf + CanTp (ISO 15765-2 transport)
- **LIN**: ISO 17987, LinIf + LinTp
- **FlexRay**: ISO 17458, high-speed deterministic (10 Mbps)
- **Ethernet**: IEEE 802.3, TCP/UDP/IP, SOME/IP

---

### AUTOSAR Adaptive

**AUTOSAR Adaptive** (for high-performance ECUs with POSIX OS, dynamic architecture).

**Target Use Cases:**
- **ADAS (Advanced Driver Assistance Systems)**: Camera/radar/lidar fusion
- **Autonomous Driving**: Real-time sensor processing, AI inference
- **Connectivity Gateways**: V2X, cloud connectivity, OTA updates
- **Infotainment**: HMI, navigation, multimedia
- **SDV (Software-Defined Vehicles)**: Over-the-air feature updates, app platforms

**Architecture Differences from Classic:**

| Feature | AUTOSAR Classic | AUTOSAR Adaptive |
|---|---|---|
| OS | Proprietary RTOS (static) | POSIX-compliant (Linux, QNX, etc.) |
| Scheduling | Static task scheduling | Dynamic process scheduling |
| Communication | Signal-based (COM) | Service-oriented (SOME/IP, DDS) |
| Memory | Fixed memory partitions | Dynamic memory allocation |
| Update | Full ECU flash (FOTA) | Application-level updates (OTA apps) |
| Hardware | Microcontrollers (32–200 MHz) | Microprocessors (GHz-class, multi-core) |
| Use Case | Body control, powertrain | ADAS, autonomous, infotainment |

**AUTOSAR Adaptive Platform Services (ara::):**

```
┌──────────────────────────────────────────────────┐
│  Adaptive Applications                           │
│  (Functional Clusters as processes)              │
├──────────────────────────────────────────────────┤
│  ARA (AUTOSAR Runtime for Adaptive Applications) │
│  ┌────────────────────────────────────────────┐  │
│  │ ara::com   — Service-oriented communication│  │
│  │ ara::exec  — Execution management          │  │
│  │ ara::log   — Logging & tracing             │  │
│  │ ara::per   — Persistency (key-value store) │  │
│  │ ara::phm   — Platform Health Management    │  │
│  │ ara::sm    — State Management              │  │
│  │ ara::ucm   — Update & Configuration Mgmt   │  │
│  │ ara::nm    — Network Management            │  │
│  │ ara::crypto— Cryptographic services        │  │
│  │ ara::iam   — Identity & Access Management  │  │
│  └────────────────────────────────────────────┘  │
├──────────────────────────────────────────────────┤
│  POSIX OS (Linux, QNX, etc.)                     │
└──────────────────────────────────────────────────┘
```

**ara::com — Service-Oriented Communication:**
- Based on **SOME/IP** (Scalable service-Oriented MiddlewarE over IP)
  - Service discovery via UDP multicast
  - Method calls (request/response) via TCP
  - Events/fields (pub/sub) via UDP
- Alternative: **DDS (Data Distribution Service)** for high-throughput sensor data
- Language bindings: C++14/17
- IDL (Interface Definition Language): ARXML or Franca IDL

**ara::ucm — Update & Configuration Manager:**
- **OTA updates** for Adaptive applications
- **Differential updates**: Binary diff to reduce download size
- **Rollback**: Automatic revert on failed update
- **A/B partitioning**: Update inactive partition, switch on success

**SOME/IP Protocol:**
- **Service ID**: 16-bit identifier
- **Method ID / Event ID**: 16-bit identifier
- **Client ID**: 16-bit identifier
- **Session ID**: 16-bit counter
- **Message Types**: REQUEST, RESPONSE, NOTIFICATION (event), ERROR
- **Serialization**: TLV (Type-Length-Value) or fixed-length structs

**AUTOSAR Adaptive Releases:**
- **R17-03** (2017): Initial release
- **R18-03** (2018): Security (ara::crypto, ara::iam)
- **R19-03** (2019): Safety (ara::phm), deterministic Ethernet (TSN)
- **R20-11** (2020): Harmonization with Classic, enhanced UCM
- **R21-11** (2021): Functional Safety (ISO 26262), SOTIF (Safety of the Intended Functionality)

**SDV (Software-Defined Vehicle) Architecture:**
- **Central Compute Platform**: High-performance SoC (e.g., NVIDIA Drive, Qualcomm Snapdragon Ride)
- **Zone Controllers**: Regional ECUs for sensors/actuators (Classic AUTOSAR)
- **Ethernet Backbone**: 100 Mbps / 1 Gbps / 10 Gbps (TSN for deterministic traffic)
- **OTA Platform**: Cloud-based update management (AWS IoT, Azure IoT, proprietary)

---

## 3. Digital Keys & Connectivity

### CCC Digital Key 3.0

**CCC (Car Connectivity Consortium) Digital Key** enables smartphones to unlock/start vehicles.

**Specifications:**
- **Digital Key Release 1.0** (2018): NFC-based (ISO 14443, ISO 18092)
- **Digital Key Release 2.0** (2020): BLE (Bluetooth Low Energy) + NFC
- **Digital Key Release 3.0** (2021): **UWB (Ultra-Wideband) + NFC** — precise ranging, relay attack prevention

**Phase Architecture:**
- **Phase 1**: Pairing (NFC tap or QR code scan)
- **Phase 2**: Standard unlock (NFC tap on door handle)
- **Phase 3**: Passive entry (UWB proximity detection, no tap required)

**Digital Key 3.0 — Phase 2 (NFC):**

```
Smartphone (with Digital Key)           Vehicle (NFC reader in door handle)
    │                                        │
    ├─ 1. User approaches vehicle            │
    │    (NFC field activated)               │
    │                                        │
    ├─ 2. NFC tap on door handle         ──> │
    │                                        │
    │  <── 3. Vehicle sends challenge       │
    │      (ISO 7816 APDU command)          │
    │                                        │
    ├─ 4. Phone signs challenge with     ──> │
    │      Digital Key private key          │
    │      (ECDSA P-256)                    │
    │                                        │
    │  <── 5. Vehicle verifies signature    │
    │      (against enrolled public key)    │
    │                                        │
    └─ 6. Door unlocks                       │
```

- **NFC Protocol**: ISO 14443-4 (Type A/B) + ISO 18092 (P2P mode)
- **Applet**: Secure Element (SE) or Host Card Emulation (HCE)
- **Authentication**: ECDSA with P-256 curve
- **Key Storage**: Secure Enclave (Apple), StrongBox Keymaster (Android)

**Digital Key 3.0 — Phase 3 (UWB Passive Entry):**

**UWB Ranging (IEEE 802.15.4z):**
- **Frequency**: 6.5–8 GHz (UWB Channel 9)
- **Ranging Accuracy**: ±10 cm
- **Two-Way Ranging (TWR)**:
  - Phone sends UWB pulse → Vehicle responds → RTT measured
  - Distance = (RTT × speed of light) / 2
  - **Angle of Arrival (AoA)**: Vehicle antenna array detects direction

**Passive Entry Flow:**

```
Smartphone (in pocket)                  Vehicle (UWB anchors in vehicle)
    │                                        │
    ├─ 1. BLE advertisement (background)    │
    │     (Device identifier)            ──> │
    │                                        │
    │  <── 2. BLE: Wake UWB               │
    │                                        │
    ├─ 3. UWB ranging session initiated  <──>│
    │     Two-Way Ranging (TWR)             │
    │     Angle of Arrival (AoA)            │
    │                                        │
    │  <── 4. Distance + Angle computed     │
    │      (e.g., 1.5 m, 45° left of driver door)
    │                                        │
    ├─ 5. Mutual authentication          <──>│
    │     (Challenge-response, ECDSA)       │
    │                                        │
    │  <── 6. If within zone (< 2 m) +      │
    │      authenticated → unlock doors     │
    │                                        │
    └─ 7. User approaches, door unlocks     │
       (no tap required)                    │
```

**Security Features:**
- **Relay Attack Prevention**: UWB Time-of-Flight (ToF) measurement prevents relay (attacker cannot spoof distance)
- **Mutual Authentication**: Phone and vehicle both verify each other's credentials
- **Context Awareness**: Vehicle unlocks only if phone is outside vehicle (prevents trunk entrapment attacks)
- **Owner Transfer**: Transfer Digital Key to another user via secure channel (iMessage, SMS with OTP)

**Platform Integration:**
- **Apple Wallet (Car Keys)**: iOS 13.6+, iPhone XS and later (NFC), iPhone 11 and later (UWB)
- **Google Wallet (Digital Car Key)**: Android 12+, Pixel 6 and later (UWB)
- **Supported OEMs**: BMW, Hyundai/Genesis, Kia, Audi, Mercedes-Benz, Polestar, BYD

**CCC Specifications:**
- **Digital Key Release 3.0 (2021)**: UWB ranging, relay attack prevention
- **Digital Key Release 3.1 (2023)**: Enhanced privacy, multiple device support
- **IEEE 802.15.4z**: UWB PHY/MAC for secure ranging

---

## 4. EV Charging Protocols

### ISO 15118 — EV Charging Communication

**ISO 15118** defines communication between EV (Electric Vehicle) and EVSE (Electric Vehicle Supply Equipment / charging station).

**Specifications:**
- **ISO 15118-1**: General requirements
- **ISO 15118-2**: Network and application protocol (AC/DC, HomePlug GreenPHY PLC)
- **ISO 15118-3**: Physical and data link layer (PLC)
- **ISO 15118-4**: Network conformance testing
- **ISO 15118-20**: **2nd generation** (AC, DC, bidirectional V2G, wireless WPT, Automated Connection Device)

**ISO 15118-2 (1st Generation):**

**Physical Layer:**
- **PLC (Power Line Communication)**: HomePlug GreenPHY over Control Pilot (CP) line
- **Frequency**: 1.8–30 MHz OFDM
- **Data Rate**: Up to 10 Mbps
- **CP Line**: Existing SAE J1772 / IEC 61851 pilot signal (PWM duty cycle for charging current)

**Protocol Stack:**
- **Physical**: HomePlug GreenPHY
- **Data Link**: IEEE 802.3 (Ethernet)
- **Network**: IPv6 (SLAAC for address assignment)
- **Transport**: TCP
- **Application**: EXI-encoded XML messages (Efficient XML Interchange)

**Charging Session Flow (ISO 15118-2, Plug & Charge):**

```
EV                                      EVSE (Charging Station)
    │                                        │
    ├─ 1. Plug connected (CP line active)   │
    │                                        │
    ├─ 2. SLAC (Signal Level Attenuation <──>
    │     Characterization) — PLC matching  │
    │     HomePlug AV coordination          │
    │                                        │
    ├─ 3. SDP (SECC Discovery Protocol)  ──>│
    │     (UDP multicast, discover SECC IP) │
    │                                        │
    │  <── 4. SECC IP address + port        │
    │                                        │
    ├─ 5. TLS handshake (optional)       <──>│
    │     (Contract Certificate-based auth) │
    │                                        │
    ├─ 6. SessionSetup                   ──>│
    │     (EV requests session)             │
    │                                        │
    │  <── 7. SessionSetupResponse          │
    │      (EVSE session ID)                │
    │                                        │
    ├─ 8. ServiceDiscovery               ──>│
    │     (EV queries supported services)   │
    │                                        │
    │  <── 9. ServiceDiscoveryResponse      │
    │      (AC/DC, payment options)         │
    │                                        │
    ├─ 10. PaymentServiceSelection       ──>│
    │      (Contract = Plug & Charge cert)  │
    │                                        │
    │  <── 11. Certificate installation /   │
    │       verification                    │
    │                                        │
    ├─ 12. Authorization                 ──>│
    │      (EV sends contract cert chain)   │
    │                                        │
    │  <── 13. AuthorizationResponse        │
    │      (EVSE validates cert via backend)│
    │                                        │
    ├─ 14. ChargeParameterDiscovery      ──>│
    │      (EV: max current, voltage, SoC)  │
    │                                        │
    │  <── 15. ChargeParameterDiscoveryRes  │
    │      (EVSE: available power)          │
    │                                        │
    ├─ 16. PowerDelivery (Start)         ──>│
    │                                        │
    │  <── 17. Contactors close, charging   │
    │                                        │
    ├─ 18. CurrentDemand / ChargingStatus<──>│
    │      (Loop: EV requests current)      │
    │                                        │
    ├─ 19. PowerDelivery (Stop)          ──>│
    │                                        │
    │  <── 20. Contactors open              │
    │                                        │
    ├─ 21. SessionStop                   ──>│
    │                                        │
    └─ 22. Unplug                            │
```

**Plug & Charge (PnC):**
- **Contract Certificate**: X.509 certificate stored in EV, signed by OEM CA
- **Certificate Chain**: Leaf Cert (EV) → Sub-CA 1/2 → OEM Root CA → V2G Root CA
- **Backend Validation**: EVSE queries backend (via OCPP or proprietary) to authorize certificate
- **No user interaction**: No RFID card, no app, no payment terminal needed
- **Roaming**: Contract certificates enable cross-network charging (Hubject, Gireve)

**ISO 15118-20 (2nd Generation, 2022):**

**Enhancements over -2:**
- **Bidirectional V2G (Vehicle-to-Grid)**: Discharge EV battery to grid
- **Wireless Power Transfer (WPT)**: Inductive charging support
- **Automated Connection Device (ACD)**: Robotic plug connection
- **Enhanced security**: TLS 1.3, improved certificate management
- **Simplified messages**: Streamlined protocol state machine

**V2G (Vehicle-to-Grid) Power Flow:**
- **Charging (Grid-to-Vehicle)**: EVSE → EV battery
- **Discharging (Vehicle-to-Grid)**: EV battery → EVSE → Grid
- **Use Cases**:
  - Grid stabilization (frequency regulation)
  - Peak shaving (discharge during high demand)
  - Renewable energy buffering (store solar/wind excess)

**V2G Schedule:**
```
ScheduledPowerProfile:
  ├─ Time slot 1 (00:00–06:00): Charge at 7 kW
  ├─ Time slot 2 (06:00–09:00): Discharge at -3 kW (to grid)
  ├─ Time slot 3 (09:00–18:00): Idle
  └─ Time slot 4 (18:00–24:00): Charge at 11 kW
```

**Wireless Power Transfer (WPT) per ISO 15118-20:**
- **Communication**: Same ISO 15118 protocol over Wi-Fi (not PLC)
- **Power Transfer**: Inductive coupling (IEC 61980-1, SAE J2954)
- **Alignment**: Fine positioning via AoA/UWB or visual markers
- **Power Levels**: 3.7 kW (AC Level 1), 7.7 kW (AC Level 2), 11 kW (AC Level 3)

---

### OCPP — Open Charge Point Protocol

**OCPP (Open Charge Point Protocol)** is the communication protocol between EV charging stations and central management systems.

**Specifications:**
- **OCPP 1.6** (2015): JSON over WebSocket (most widely deployed)
- **OCPP 2.0.1** (2020): Enhanced device model, smart charging, security

**Architecture:**

```
Charge Point (EVSE)                Central System (CSMS)
    │                                   │
    ├─ WebSocket (wss://)           <───>
    │  JSON-RPC 2.0 messages            │
    │                                   │
    ├─ OCPP Messages:                   │
    │  BootNotification                 │
    │  Heartbeat                        │
    │  StartTransaction                 │
    │  MeterValues                      │
    │  StopTransaction                  │
    │  StatusNotification               │
    │                                   │
    │  RemoteStartTransaction       <───┤
    │  RemoteStopTransaction        <───┤
    │  TriggerMessage               <───┤
    │  GetConfiguration              <───┤
    │  ChangeConfiguration          <───┤
    └───────────────────────────────────┘
```

**OCPP 1.6 — Core Profile:**

**Charge Point → Central System (CP-initiated):**
- **BootNotification**: Charge point startup (sends model, serial, firmware version)
- **Heartbeat**: Periodic keep-alive (default 900s interval)
- **Authorize**: RFID tag authorization request
- **StartTransaction**: Charging session started (sends RFID tag, connector ID, meter start value, timestamp)
- **StopTransaction**: Charging session ended (sends meter stop value, reason, transaction data)
- **MeterValues**: Periodic energy meter readings (kWh, kW, voltage, current, SoC)
- **StatusNotification**: Connector status change (Available, Occupied, Charging, Faulted, Unavailable)
- **DataTransfer**: Vendor-specific data exchange

**Central System → Charge Point (CS-initiated):**
- **RemoteStartTransaction**: Initiate charging (e.g., via mobile app)
- **RemoteStopTransaction**: Stop charging remotely
- **UnlockConnector**: Unlock cable (emergency release)
- **Reset**: Reboot charge point (soft/hard)
- **GetConfiguration**: Read configuration keys
- **ChangeConfiguration**: Update configuration (e.g., HeartbeatInterval, MeterValueSampleInterval)
- **TriggerMessage**: Request immediate message (e.g., StatusNotification, MeterValues)
- **UpdateFirmware**: OTA firmware update (sends FTP URL + retrieve time)

**OCPP 1.6 — Smart Charging Profile:**
- **ChargingProfile**: Time-based power limit schedule
- **SetChargingProfile**: Central system sends charging schedule to CP
- **ClearChargingProfile**: Remove charging schedule
- **GetCompositeSchedule**: Query effective charging schedule

**Example ChargingProfile:**
```json
{
  "chargingProfileId": 1,
  "stackLevel": 0,
  "chargingProfilePurpose": "TxDefaultProfile",
  "chargingProfileKind": "Absolute",
  "chargingSchedule": {
    "chargingRateUnit": "W",
    "chargingSchedulePeriod": [
      {"startPeriod": 0, "limit": 7400},      // 0–1h: 7.4 kW
      {"startPeriod": 3600, "limit": 11000},  // 1–3h: 11 kW
      {"startPeriod": 10800, "limit": 3700}   // 3h+: 3.7 kW
    ]
  }
}
```

**OCPP 2.0.1 Enhancements:**

**Device Model:**
- **Components**: Modular representation (Connector, EVSE, ChargingStation, Controller)
- **Variables**: Configurable attributes per component (e.g., Connector.AvailabilityState)
- **Characteristics**: Metadata (unit, min/max, data type, mutability)
- **GetBaseReport / GetVariables / SetVariables**: Replace 1.6's GetConfiguration/ChangeConfiguration

**Security Profile:**
- **TLS 1.2 minimum** (TLS 1.3 recommended)
- **Client certificate authentication**: Charge point presents X.509 cert to CSMS
- **Certificate management**: InstallCertificate, DeleteCertificate, GetInstalledCertificateIds
- **Firmware signing**: Signed firmware images (prevents tampering)

**ISO 15118 Integration:**
- **Iso15118CertificateRequest**: Charge point requests contract certificate installation
- **Get15118EVCertificate**: CSMS retrieves EV contract certificate from backend
- Enables Plug & Charge (PnC) via OCPP backend integration

**Transaction Events:**
- **TransactionEventRequest**: Replaces separate StartTransaction/StopTransaction/MeterValues
- **Event-based model**: Reports Started, Updated, Ended events with meter values

**Smart Charging v2.0:**
- **ChargingNeeds**: EV communicates desired SoC, departure time (via ISO 15118)
- **ChargingProfile v2.0**: Enhanced schedules with cost optimization
- **NotifyEVChargingNeeds**: Charge point forwards EV charging needs to CSMS

**OCPP 1.6 vs 2.0.1 Comparison:**

| Feature | OCPP 1.6 | OCPP 2.0.1 |
|---|---|---|
| Configuration | Key-value pairs | Device model (components/variables) |
| Security | Optional TLS | Mandatory TLS + client certs |
| ISO 15118 | Limited support | Full Plug & Charge integration |
| Transaction | 3 messages (Start/Stop/MeterValues) | TransactionEvent (unified) |
| Firmware | FTP download | Signed firmware, integrity checks |
| Display | GetLocalListVersion (basic) | SetDisplayMessage (rich content) |
| Monitoring | Predefined sampled values | Custom monitoring (alarms, thresholds) |

---

### OCPI — Open Charge Point Interface

**OCPI (Open Charge Point Interface)** is the roaming protocol between EV charging networks (CPO ↔ eMSP).

**Use Case:**
- **CPO (Charge Point Operator)**: Owns/operates charging stations
- **eMSP (e-Mobility Service Provider)**: Provides charging access to EV drivers (via app/RFID)
- **Roaming**: eMSP customers can charge at CPO stations; CPO sends CDR (Charge Detail Record) to eMSP for billing

**OCPI Architecture:**
```
eMSP (Driver's App)         CPO (Charging Network)
    │                            │
    ├─ 1. Location Info      <───┤  (GET /locations)
    │    (Station availability)  │
    │                            │
    ├─ 2. Start Session      ────>  (POST /sessions)
    │    (Authorization token)   │
    │                            │
    │  <── 3. Session Started    │
    │                            │
    ├─ 4. Session Updates    <───┤  (PATCH /sessions/{id})
    │    (Energy delivered)      │
    │                            │
    ├─ 5. Stop Session       ────>  (PUT /sessions/{id})
    │                            │
    │  <── 6. CDR (Charge Detail Record)
    │      (kWh, cost, timestamps)
```

**OCPI Modules:**
- **Locations**: Charging station metadata (GPS, connectors, pricing)
- **Sessions**: Active charging sessions
- **CDRs**: Charge Detail Records (billing data)
- **Tariffs**: Pricing structures (per kWh, per minute, flat fee)
- **Tokens**: Authorization tokens (RFID, app-based)
- **Commands**: Remote start/stop

**OCPI Version:**
- **OCPI 2.2.1** (2021): Most widely adopted
- **OCPI 3.0** (in development): Enhanced tariffs, smart charging integration

---

## 5. Cybersecurity & Regulatory

### UNECE WP.29 R155 — Cybersecurity Management

**UNECE WP.29 Regulation No. 155** mandates **Cyber Security Management Systems (CSMS)** for vehicles sold in EU, Japan, South Korea (effective July 2024 for new models, July 2026 for all vehicles).

**Scope:**
- Type approval requirement for M (passenger) and N (commercial) vehicles
- Applies to OEMs (vehicle manufacturers)
- Covers entire vehicle lifecycle: design, production, post-production

**CSMS Requirements:**

1. **Organizational Processes:**
   - Assign cybersecurity roles and responsibilities
   - Document cybersecurity policies and procedures
   - Continuous improvement process

2. **Risk Assessment & Management:**
   - **TARA (Threat Analysis and Risk Assessment)**: Identify attack vectors, evaluate risk
   - Risk treatment: Mitigate, accept, transfer, or avoid
   - Document risk decisions

3. **Secure Development:**
   - Security-by-design principles
   - Secure coding standards
   - Vulnerability testing (penetration testing, fuzzing)

4. **Monitoring & Response:**
   - **Vulnerability monitoring**: Subscribe to CVE feeds, coordinate with suppliers
   - **Incident response**: Detect, analyze, contain, remediate cybersecurity incidents
   - **Vulnerability disclosure**: Report vulnerabilities to authorities within defined timelines

5. **Supply Chain Management:**
   - Ensure suppliers follow cybersecurity requirements
   - Contractual obligations for vulnerability disclosure

**TARA (Threat Analysis and Risk Assessment):**

**Threat Categories (Annex 5, Part A):**
- **Backend server attacks**: Compromise OEM cloud, CSMS, OTA server
- **Communication channel attacks**: Man-in-the-middle, replay, eavesdropping (cellular, Wi-Fi, V2X)
- **Updating procedure attacks**: Malicious firmware injection, downgrade attacks
- **Vehicle external connectivity**: Exploit cellular modem, Wi-Fi, Bluetooth
- **Data/code attacks**: Buffer overflow, SQL injection, code execution
- **Physical attacks**: Access to ECU, CAN bus sniffing, OBD-II port exploitation

**Attack Feasibility Rating (ISO 21434):**
- **Elapsed Time**: < 1 day (High) to > 1 month (Low)
- **Specialist Expertise**: Layman to Expert
- **Knowledge of Item**: Public to Confidential
- **Window of Opportunity**: Unlimited to Difficult to identify
- **Equipment**: Standard to Bespoke

**Risk Level = Impact × Attack Feasibility**

**Mitigation Examples:**
- **Secure Boot**: Prevent execution of unsigned firmware
- **Code Signing**: Verify integrity of firmware updates
- **Encrypted Communication**: TLS for backend, DTLS for V2X
- **Intrusion Detection**: CAN bus monitoring (e.g., detect anomalous message rates)
- **Firewall**: Gateway ECU filters messages between vehicle domains

**Type Approval Process:**
- OEM submits CSMS documentation to **Type Approval Authority (TAA)**
- TAA audits CSMS compliance
- If compliant, TAA issues **Certificate of Compliance**
- Certificate required for vehicle sales in regulated markets

---

### UNECE WP.29 R156 — Software Update Management

**UNECE WP.29 Regulation No. 156** mandates **Software Update Management Systems (SUMS)** for vehicles (effective July 2024 for new models, July 2026 for all vehicles).

**Scope:**
- Covers **OTA (Over-The-Air)** and **in-shop** software updates
- Requires logging, validation, and rollback capabilities

**SUMS Requirements:**

1. **Update Process Integrity:**
   - **Secure update delivery**: Encrypted, authenticated firmware packages
   - **Validation**: Verify signature before installation
   - **Version control**: Track installed software versions per ECU

2. **Pre-Update Validation:**
   - Verify vehicle is in safe state (parked, ignition off)
   - Check battery charge level (prevent brick during update)
   - Verify update compatibility with current software

3. **Update Installation:**
   - **Atomic updates**: All-or-nothing (prevent partial installation)
   - **Rollback**: Automatic revert if update fails or vehicle becomes non-functional
   - **A/B partitioning**: Update inactive partition, switch on success

4. **Update Logging:**
   - **Audit trail**: Record all update attempts (timestamp, result, version)
   - **Tamper-proof logs**: Logs stored in secure memory (cannot be erased by attacker)
   - **Retention**: Logs retained for regulatory inspection

5. **User Notification:**
   - Inform driver of pending update
   - Allow deferral (up to X days, configurable)
   - Mandatory updates for safety/security must be applied within deadline

**OTA Update Flow (R156-compliant):**

```
1. OTA Server sends update notification → Vehicle TCU (Telematics Control Unit)
2. TCU checks update eligibility:
   - Vehicle parked, ignition off
   - Battery > 50%
   - Compatible hardware/software
3. Download firmware package (encrypted)
4. Verify signature (OEM certificate)
5. Log: "Update download completed"
6. Install to inactive partition (A/B update)
7. Verify installation (integrity check)
8. Log: "Update installed, pending activation"
9. Reboot → activate new partition
10. Post-update validation (ECU self-test)
11. If validation fails → rollback to previous partition
12. If validation succeeds → log: "Update activated successfully"
13. Report success to OTA server
```

**Logging Requirements:**
- **Unique update ID**: Trace update package
- **Timestamp**: ISO 8601 format
- **ECU identifier**: Which ECU was updated
- **Software version**: Before and after
- **Result**: Success, failure, rollback
- **Failure reason**: If failed, detailed error code

**Rollback Scenarios:**
- **Boot failure**: ECU fails to start after update → automatic rollback
- **Functional failure**: ECU starts but fails self-test → rollback
- **Safety-critical failure**: Vehicle systems non-operational → rollback + alert

**Type Approval:**
- OEM submits SUMS documentation to TAA
- TAA verifies:
  - Update process integrity (signature verification, rollback)
  - Logging capabilities
  - User notification procedures
- Certificate of Compliance issued

---

## Cross-References

- **V2X Protocols (IEEE 802.11p, C-V2X, NR-V2X)**: See [cellular-wan.md](cellular-wan.md) § V2X Protocols
- **CAN, LIN, FlexRay**: Physical layer protocols (out of scope; focus here is application/diagnostic layer)
- **UWB (IEEE 802.15.4z)**: See [pan-short-range.md](pan-short-range.md) for UWB ranging details
- **NFC (ISO 14443, ISO 18092)**: See [pan-short-range.md](pan-short-range.md) for NFC physical/protocol layers

---

## Key Specification References

| Technology | Primary Specification | Notes |
|---|---|---|
| OBD-II | SAE J1962, ISO 15765-4 | J1962 connector pinout, CAN transport |
| UDS | ISO 14229-1 through -7 | Diagnostic services |
| DoIP | ISO 13400 | Diagnostics over IP (Ethernet) |
| SAE J1939 | SAE J1939-71, J1939-73 | Heavy vehicle CAN, diagnostics |
| AUTOSAR Classic | AUTOSAR R4.4, R20-11 | Layered architecture, RTE, BSW |
| AUTOSAR Adaptive | AUTOSAR R21-11 | POSIX, ara:: services, SOME/IP |
| SOME/IP | AUTOSAR SOME/IP spec | Service-oriented middleware |
| CCC Digital Key 3.0 | CCC Digital Key Release 3.0/3.1 | NFC + UWB passive entry |
| ISO 15118-2 | ISO 15118-2:2014 | AC/DC charging, PLC, Plug & Charge |
| ISO 15118-20 | ISO 15118-20:2022 | V2G, WPT, ACD |
| OCPP 1.6 | OCPP 1.6 JSON (OCA) | WebSocket, JSON-RPC |
| OCPP 2.0.1 | OCPP 2.0.1 Edition 2 (OCA) | Device model, security, ISO 15118 |
| OCPI | OCPI 2.2.1 | Roaming protocol (CPO ↔ eMSP) |
| UNECE WP.29 R155 | UN Regulation No. 155 | CSMS, TARA, type approval |
| UNECE WP.29 R156 | UN Regulation No. 156 | SUMS, OTA logging, rollback |

---

*Cross-references:*
- *V2X protocols: `references/cellular-wan.md`*
- *UWB ranging: `references/pan-short-range.md`*
- *NFC protocols: `references/pan-short-range.md`*
