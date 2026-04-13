# Smart Home & Consumer IoT Protocols

> **Scope:** This reference covers consumer IoT protocols for smart home, voice assistants,
> and residential automation — from Matter (CSA), HomeKit (Apple), Google Home, Amazon
> platforms (Sidewalk, Alexa), to DECT ULE/NR+. For underlying wireless PAN protocols
> (BLE, Thread, Zigbee, Wi-Fi), see [pan-short-range.md](pan-short-range.md).

---

## Table of Contents

1. [Matter — CSA Connectivity Standard](#1-matter--csa-connectivity-standard)
   - [Matter Architecture & Fabric Model](#matter-architecture--fabric-model)
   - [PASE Commissioning (Setup)](#pase-commissioning-setup)
   - [CASE Operational Handshake](#case-operational-handshake)
   - [Matter Clusters & Device Types](#matter-clusters--device-types)
   - [Matter 1.3 Additions](#matter-13-additions)
   - [Controller & Bridge Model](#controller--bridge-model)
2. [HomeKit — Apple HAP](#2-homekit--apple-hap)
3. [Google Home](#3-google-home)
4. [Amazon Ecosystem](#4-amazon-ecosystem)
   - [Amazon Sidewalk](#amazon-sidewalk)
   - [Alexa Connect Kit (ACK)](#alexa-connect-kit-ack)
5. [DECT — Digital Enhanced Cordless Telecommunications](#5-dect--digital-enhanced-cordless-telecommunications)

---

## 1. Matter — CSA Connectivity Standard

**Matter** (Connectivity Standards Alliance, formerly Project CHIP — Connected Home over IP) is the unified IP-based application-layer protocol for smart home devices. It enables cross-platform interoperability across Apple HomeKit, Google Home, Amazon Alexa, Samsung SmartThings, and others.

### Key Specifications
- **CSA Matter Specification 1.0** (Oct 2022), **1.1** (May 2023), **1.2** (Oct 2023), **1.3** (May 2024)
- **Matter SDK**: Open-source reference implementation (C++)
- **Matter.js**: JavaScript/TypeScript implementation for Node.js
- **Test harness**: TH (Test Harness) for certification

### Matter Architecture & Fabric Model

**Network topology:**
```
Controller (Hub/Phone)
    │
    ├─ Device 1 (Matter Light) ──> Thread Border Router ──> Thread network
    ├─ Device 2 (Matter Lock)  ──> Wi-Fi
    └─ Device 3 (Matter Sensor)──> Ethernet bridge
```

**Fabric Model:**
- **Fabric** = a secure grouping of devices controlled by the same root of trust (controller)
- Each fabric has a **Root CA (RCAC)**, **Intermediate CA (ICAC)**, and per-device **Node Operational Certificates (NOC)**
- Devices can participate in **multiple fabrics** (e.g., Apple Home + Google Home simultaneously)
- Each fabric assigns a unique **NodeID** to each device
- Fabrics are isolated — devices cannot see or communicate across fabrics directly

**Endpoints & Clusters:**
- Device = collection of **Endpoints** (logical device units)
  - Example: Dual-gang switch = Endpoint 0 (Root Node) + Endpoint 1 (Switch 1) + Endpoint 2 (Switch 2)
- Each endpoint exposes **Clusters** (functional units) with **Attributes, Commands, Events**
  - Example: Light endpoint exposes On/Off cluster + Level cluster + Color Control cluster

**Transport Layers:**
- **BLE** (Bluetooth Low Energy): Used for commissioning only (not operational)
- **Thread**: IEEE 802.15.4-based mesh network; low power, reliable
- **Wi-Fi**: IEEE 802.11 (2.4 GHz + 5 GHz); high bandwidth
- **Ethernet**: Wired; for bridges and infrastructure

**Application Protocol:**
- **UDP (port 5540)** for multicast (discovery, group commands) and unicast
- **TCP (port 5540)** for reliable unicast streams (e.g., large data transfers)
- **MRP (Matter Reliable Protocol)** provides retransmission over UDP
- **Secure Channel**: TLS-like encryption using AES-128-CCM

---

### PASE Commissioning (Setup)

**PASE = Password-Authenticated Session Establishment** (commissioning phase)

**Goal:** Establish a secure session between the commissioner (controller) and commissionee (new device) using a shared password derived from a setup code.

**Setup Code formats:**
1. **Manual Pairing Code**: 11-digit decimal code (e.g., `34970112332`)
   - Digits 1-3: Discriminator (12-bit, range 0–4095)
   - Digits 4-11: 27-bit passcode with check digit
2. **QR Code**: Encoded as `MT:Y3.14%ABC123...` with Base-38 encoding
   - Contains: discriminator, passcode, vendor ID, product ID, commissioning flow hint

**PASE Protocol Flow (SPAKE2+):**

SPAKE2+ is a password-authenticated key exchange (PAKE) that prevents offline dictionary attacks.

```
Commissioner                            Commissionee
    │                                        │
    ├─ 1. BLE advertisement scan             │
    │  (Service UUID: 0xFFF6)                │
    │  (discriminator in advertising data)   │
    │                                        │
    ├─ 2. PASE Handshake (SPAKE2+ over BLE) ──>
    │      pbkdf2(passcode, salt, iterations=1000)
    │      → verifier (w0, w1, L)            │
    │                                        │
    │  <── pA (ephemeral public key A)       │
    ├──> pB (ephemeral public key B)         │
    │                                        │
    │  Both derive shared secret K           │
    │  Both derive session keys (I2R, R2I)   │
    │                                        │
    ├─ 3. Encrypted channel established      │
    │     (AES-128-CCM)                      │
    │                                        │
    ├─ 4. Commissioning commands             │
    │     (Network credentials: Thread/Wi-Fi)│
    │     (Fabric association)               │
    │     (NOC request)                      │
    │                                        │
    └─ 5. Device joins operational network   │
```

**SPAKE2+ Details:**
- **Password**: Passcode from QR or manual code
- **Salt**: Device-specific (usually derived from discriminator + vendor ID)
- **Verifier**: `(w0, w1, L)` stored on device; `w0`, `w1` prevent offline attacks
- **pA, pB**: Ephemeral Elliptic Curve Diffie-Hellman (ECDH) public keys over P-256 curve
- **Shared secret K**: Derived from SPAKE2+ protocol; never transmitted
- **Session keys**: I2R (Initiator-to-Responder) and R2I (Responder-to-Initiator) for AES-128-CCM

**Discriminator purpose:**
- 12-bit value (0–4095) uniquely identifies the device during BLE scan
- Allows commissioner to filter devices during commissioning in crowded environments
- Combined with passcode for SPAKE2+ key derivation

**Commissioning steps:**
1. User scans QR code or enters manual pairing code into commissioner app
2. Commissioner scans BLE advertisements for matching discriminator
3. PASE handshake establishes encrypted session
4. Commissioner sends network credentials (Thread Extended PAN ID + Network Key OR Wi-Fi SSID + PSK)
5. Commissioner requests device generate Certificate Signing Request (CSR)
6. Commissioner signs CSR with ICAC, returns NOC (Node Operational Certificate)
7. Device transitions to operational mode, disconnects BLE, joins Thread/Wi-Fi network
8. Commissioner discovers device via mDNS on operational network

---

### CASE Operational Handshake

**CASE = Certificate-Authenticated Session Establishment** (operational phase)

**Goal:** Establish a secure operational session between controller and commissioned device using mutual certificate authentication (mTLS-like).

**Certificate Hierarchy (per Fabric):**
```
RCAC (Root CA Certificate) — self-signed, fabric root of trust
    │
    ├─ ICAC (Intermediate CA Certificate) — signed by RCAC
    │     │
    │     ├─ NOC₁ (Node Operational Certificate) — Device 1
    │     ├─ NOC₂ (Node Operational Certificate) — Device 2
    │     └─ NOC₃ (Node Operational Certificate) — Controller
```

**CASE Protocol Flow:**

```
Controller                              Device
    │                                     │
    ├─ 1. Sigma1 (Session Init Request) ──>
    │      initiatorRandom (32 bytes)     │
    │      initiatorSessionId             │
    │      destinationId (Fabric ID)      │
    │      initiatorEphPubKey (ECDH)      │
    │      initiatorResumptionMIC (optional)
    │                                     │
    │  <── 2. Sigma2 (Session Response)   │
    │      responderRandom (32 bytes)     │
    │      responderSessionId             │
    │      responderEphPubKey (ECDH)      │
    │      encrypted2 {                   │
    │         responderNOC                │
    │         responderICAC (optional)    │
    │         Signature_PrivKey(NOC)      │
    │         responderResumptionMIC      │
    │      }                              │
    │                                     │
    ├─ 3. Sigma3 (Session Confirm)   ──> │
    │      encrypted3 {                   │
    │         initiatorNOC                │
    │         initiatorICAC (optional)    │
    │         Signature_PrivKey(NOC)      │
    │      }                              │
    │                                     │
    ├─ 4. Session established             │
    │     (I2R & R2I session keys derived)│
    │     (AES-128-CCM encryption)        │
    └─────────────────────────────────────┘
```

**Sigma1/2/3 Handshake Details:**
- **Sigma1**: Controller initiates handshake with random nonce, session ID, fabric ID, ephemeral public key
- **Sigma2**: Device responds with its ephemeral key, encrypts its NOC (+ ICAC if present), signs with private key
- **Sigma3**: Controller sends its NOC (+ ICAC if present), signed with private key
- Both parties derive shared secret via ECDH (P-256 curve)
- Both parties verify peer certificates chain to RCAC
- Session keys derived using HKDF-SHA256

**Session Resumption:**
- Matter supports **session resumption** to avoid full CASE handshake on reconnection
- `initiatorResumptionMIC` / `responderResumptionMIC` are Message Integrity Codes (MIC) proving possession of previous session key
- Resumption reduces latency from ~500 ms to <50 ms

**Fabric Isolation:**
- Each fabric has distinct RCAC; devices cannot decrypt/verify commands from other fabrics
- Multi-fabric devices store multiple (RCAC, NOC) pairs in non-volatile memory
- Controller addressing: `<FabricIndex>:<NodeID>`

---

### Matter Clusters & Device Types

**Cluster = functional unit** exposing Attributes (state), Commands (actions), Events (notifications).

**Common Clusters (Matter 1.x):**

| Cluster ID | Cluster Name | Function | Key Attributes | Key Commands |
|---|---|---|---|---|
| 0x0003 | Identify | Device identification (flashing LED) | IdentifyTime | Identify, TriggerEffect |
| 0x0004 | Groups | Group addressing (multicast) | NameSupport | AddGroup, ViewGroup, RemoveGroup |
| 0x0005 | Scenes | Scene storage/recall | SceneCount, CurrentScene | AddScene, ViewScene, RecallScene |
| 0x0006 | **On/Off** | Binary on/off control | OnOff (bool) | On, Off, Toggle |
| 0x0008 | **Level Control** | Dimming/brightness | CurrentLevel (0–254) | MoveToLevel, Move, Step, Stop |
| 0x0300 | **Color Control** | Color temperature + RGB/HSV | ColorMode, ColorTemperature, CurrentHue, CurrentSaturation | MoveToHue, MoveToSaturation, MoveToColor, MoveToColorTemperature |
| 0x001D | Descriptor | Endpoint composition | DeviceTypeList, ServerList, PartsList | (none) |
| 0x0028 | Basic Information | Device metadata | VendorName, ProductName, SerialNumber, SoftwareVersion | (none) |
| 0x002F | Power Source | Battery/mains status | Status, BatPercentRemaining | (none) |
| 0x0030 | **General Commissioning** | Commissioning primitives | Breadcrumb, RegulatoryConfig | ArmFailSafe, SetRegulatoryConfig, CommissioningComplete |
| 0x0031 | **Network Commissioning** | Network credential management | Networks, ConnectMaxTimeSeconds | ScanNetworks, AddOrUpdateWiFiNetwork, AddOrUpdateThreadNetwork, RemoveNetwork, ConnectNetwork |
| 0x003C | **Administrator Commissioning** | Fabric management | WindowStatus, AdminFabricIndex | OpenCommissioningWindow, OpenBasicCommissioningWindow, RevokeCommissioning |
| 0x003E | **Operational Credentials** | NOC/Fabric lifecycle | NOCs, Fabrics, CurrentFabricIndex | AttestationRequest, CertificateChainRequest, CSRRequest, AddNOC, UpdateNOC, RemoveFabric |
| 0x0101 | **Door Lock** | Lock control | LockState, LockType, ActuatorEnabled | LockDoor, UnlockDoor, SetCredential |
| 0x0102 | **Window Covering** | Blinds/shades | CurrentPositionLiftPercent100ths, TargetPositionLiftPercent100ths | UpOrOpen, DownOrClose, GoToLiftPercentage |
| 0x0201 | **Thermostat** | HVAC control | LocalTemperature, OccupiedCoolingSetpoint, OccupiedHeatingSetpoint, SystemMode | SetpointRaiseLower |
| 0x0202 | Fan Control | Fan speed | FanMode, PercentSetting | (none, attribute writes) |
| 0x0400 | Illuminance Measurement | Light sensor | MeasuredValue (lux) | (none) |
| 0x0402 | Temperature Measurement | Temperature sensor | MeasuredValue (°C × 100) | (none) |
| 0x0405 | Relative Humidity Measurement | Humidity sensor | MeasuredValue (% × 100) | (none) |
| 0x0406 | **Occupancy Sensing** | Motion/presence | Occupancy (bitmap), OccupancySensorType | (none) |

**Device Types (Matter 1.x):**

Device Type = set of required + optional clusters on a specific endpoint.

| Device Type ID | Device Type | Required Clusters | Optional Clusters |
|---|---|---|---|
| 0x0100 | On/Off Light | Identify, On/Off | Level Control, Groups, Scenes |
| 0x0101 | Dimmable Light | Identify, On/Off, Level Control | Groups, Scenes |
| 0x010C | Color Temperature Light | Identify, On/Off, Level Control, Color Control (temp mode) | Groups, Scenes |
| 0x010D | Extended Color Light | Identify, On/Off, Level Control, Color Control (full) | Groups, Scenes |
| 0x0302 | Temperature Sensor | Identify, Temperature Measurement | (none) |
| 0x0307 | Occupancy Sensor | Identify, Occupancy Sensing | (none) |
| 0x010A | Door Lock | Identify, Door Lock | (none) |
| 0x0301 | Thermostat | Identify, Thermostat | Fan Control |
| 0x0202 | Window Covering | Identify, Window Covering | (none) |
| 0x0103 | On/Off Plug-in Unit | Identify, On/Off | Level Control |
| 0x010B | Contact Sensor | Identify, Boolean State | (none) |
| 0x002B | Bridged Node | Bridged Device Basic Information | (varies) |

**Composition (Endpoint Model):**
- **Endpoint 0**: Root node; always present; exposes Descriptor, Basic Information, Operational Credentials, Administrator Commissioning, Network Commissioning, General Commissioning
- **Endpoint 1+**: Application endpoints; each represents a functional device (light, sensor, lock, etc.)

---

### Matter 1.3 Additions

**Matter 1.3** (May 2024) added:

**New Device Types:**
- **EV Charger (0x050C)**: Energy EVSE cluster, Electrical Power/Energy Measurement clusters
- **Water Heater (0x0078)**: Water Heater Management cluster, Water Heater Mode cluster
- **Microwave Oven (0x0079)**: Microwave Oven Control cluster, Operational State cluster

**New Clusters:**
- **Energy EVSE (0x0099)**: EV charging station control — State, SupplyState, ChargingEnabledUntil, MinimumChargeCurrent, MaximumChargeCurrent; commands: Disable, EnableCharging, StartDiagnostics
- **Water Heater Management (0x0094)**: Boost, EcoMode, operating mode
- **Microwave Oven Control (0x005F)**: CookTime, PowerSetting, MinPower, MaxPower; commands: SetCookingParameters

**Enhanced Multi-Admin:**
- Improved fabric management for better multi-controller coordination
- AdminFabricIndex attribute tracking which fabric performed last action

**Energy Management Profile:**
- Device Energy Management (DEM) cluster enhancements
- Power/Energy Measurement clusters for monitoring consumption

---

### Controller & Bridge Model

**Controller:**
- Device that commissions and controls Matter devices
- Examples: Apple HomePod, Google Nest Hub, Amazon Echo, Samsung SmartThings Hub, smartphone app
- Implements full Matter stack (PASE, CASE, cluster clients)
- Manages Fabrics (RCAC, ICAC, NOC issuance)

**Bridge:**
- Device Type 0x002B (Bridged Node)
- Translates non-Matter protocols to Matter
- Example: Zigbee/Z-Wave devices exposed as Matter Bridged Nodes
- Each bridged device appears as a separate endpoint
- Bridge itself is Endpoint 0; bridged devices are Endpoint 1+

**Multi-Admin / Multi-Fabric:**
- **Multi-Admin**: Single fabric, multiple controllers (primary + secondary controllers)
- **Multi-Fabric**: Device participates in multiple fabrics simultaneously (Apple + Google ecosystems)
- Conflicts resolved by device (e.g., if Apple turns light on, Google sees updated state via attribute reporting)

**Matter SDK & Libraries:**
- **Matter SDK (C++)**: Official CSA reference implementation; supports Linux, macOS, Android, ESP32, nRF Connect SDK
- **Matter.js**: JavaScript/Node.js implementation; useful for bridges and custom controllers
- **Home Assistant Matter integration**: Python-based Matter controller
- **Apple HomeKit Accessory Protocol (HAP)**: HomeKit controllers support Matter via HAP-to-Matter bridge in iOS 16+

---

## 2. HomeKit — Apple HAP

**HomeKit Accessory Protocol (HAP)** is Apple's proprietary smart home protocol predating Matter.

**Key Specs:**
- **HAP Specification** (Apple MFi, NDA-restricted; publicly reverse-engineered)
- Transport: HAP-IP (TCP/IP with HTTP-like messages) or HAP-BLE (Bluetooth Low Energy GATT)
- Security: Ed25519 (Curve25519 variant) for pairing; ChaCha20-Poly1305 for session encryption

### HAP-IP (HAP over IP)

**Transport:**
- **mDNS/DNS-SD** for discovery (service type `_hap._tcp.`)
- **HTTP/1.1-like protocol** over TCP port 80 (unencrypted) or custom port (encrypted)
- **TLS equivalent**: ChaCha20-Poly1305 AEAD encryption after SRP pairing

**Pairing (SRP — Secure Remote Password):**

```
iOS Controller                    Accessory
    │                                │
    ├─ 1. Start Pairing (M1)     ──> │
    │                                │
    │  <── M2 (SRP public key, salt) │
    │                                │
    ├─ 2. M3 (SRP proof A)       ──> │
    │                                │
    │  <── M4 (SRP proof B, encrypted)
    │                                │
    ├─ 3. M5 (iOS identity)      ──> │
    │      encrypted { iOSDeviceX,   │
    │        iOSDeviceLTPK, signature}
    │                                │
    │  <── M6 (Accessory identity)   │
    │      encrypted { AccessoryX,   │
    │        AccessoryLTPK, signature}
    │                                │
    └─ Paired. LTPK exchanged.       │
```

- **Setup Code**: 8-digit code (e.g., `123-45-678`) printed on accessory or displayed on screen
- **SRP**: Username = "Pair-Setup", Password = setup code
- **LTPK (Long-Term Public Key)**: Ed25519 public key stored on both sides
- **Encrypted sessions**: Established with ChaCha20-Poly1305 after LTPK exchange

**Accessory Categories:**
- Other (1), Bridge (2), Fan (3), Garage Door Opener (4), Lightbulb (5), Door Lock (6), Outlet (7), Switch (8), Thermostat (9), Sensor (10), Security System (11), Door (12), Window (13), Window Covering (14), Programmable Switch (15), Range Extender (16), IP Camera (17), Video Doorbell (18), Air Purifier (19), Heater (20), Air Conditioner (21), Humidifier (22), Dehumidifier (23), Sprinkler (28), Faucet (29), Shower System (30)

**Services & Characteristics:**
- **Service**: Functional unit (e.g., Lightbulb, Lock Mechanism, Thermostat, Motion Sensor)
- **Characteristic**: Attribute within a service (e.g., On, Brightness, Hue, Current Temperature)
- UUIDs in Base UUID format: `00000XXX-0000-1000-8000-0026BB765291`

**Common Services:**
- Lightbulb (0x43), Switch (0x49), Outlet (0x47), Thermostat (0x4A), Lock Mechanism (0x45), Garage Door Opener (0x41), Motion Sensor (0x85), Contact Sensor (0x80), Temperature Sensor (0x8A), Humidity Sensor (0x82)

**HomeKit Secure Video (HSV):**
- End-to-end encrypted video recording to iCloud
- HomeKit-enabled cameras analyze video on-device using Apple Neural Engine (ANE)
- Recordings encrypted with keys only the user has; Apple cannot decrypt

**Matter & HomeKit:**
- iOS 16+ supports Matter devices via HomeKit framework
- Matter devices appear in Home app as native accessories
- HomeKit controllers can commission Matter devices (Apple acts as Matter Fabric root)

---

### HAP-BLE (HAP over Bluetooth Low Energy)

**GATT Profile:**
- **Service UUID**: `00000055-0000-1000-8000-0026BB765291` (HAP Service)
- **Characteristics**: Pairing Setup, Pairing Verify, HAP Characteristic (dynamic)
- Used for battery-powered accessories (locks, sensors)

**Pairing flow similar to HAP-IP** but over BLE GATT instead of TCP.

---

## 3. Google Home

**Google Home** (formerly Google Nest) uses a cloud-centric model with **local fulfillment** for low-latency control.

### Architecture

**Components:**
- **Google Home app**: iOS/Android controller
- **Google Assistant**: Voice control
- **Google Home Graph**: Cloud-based device registry and state management
- **Local Home SDK**: Runs JavaScript on Google Home devices (Nest Hub, Chromecast) for local control

**Device Integration Methods:**
1. **Works with Google Home (Cloud-to-Cloud)**: OAuth2-based integration; all commands routed through cloud
2. **Matter**: Direct IP control via Matter protocol (local, no cloud dependency)
3. **Local Home SDK**: Hybrid model — discovery/authentication via cloud, control commands via local UDP/TCP

### Local Fulfillment

**Local Home SDK** (JavaScript runtime on Google Home Hub/Nest devices):

```
User: "Hey Google, turn on the light"
   │
Google Assistant (cloud) ──> Local Home SDK (on Nest Hub)
                                   │
                                   └──> Matter/TCP/UDP ──> Light (local network)
```

- Commands execute locally (<100 ms) after initial cloud NLU (Natural Language Understanding)
- Fallback to cloud if local execution fails
- Developer provides **app.js** with device discovery and command handlers

**SCAN protocol** (Google proprietary):
- Used by Chromecast, Google Home Mini for discovery and control
- mDNS + SSDP hybrid
- Control via HTTP POST to device IP

---

## 4. Amazon Ecosystem

### Amazon Sidewalk

**Amazon Sidewalk** is a shared low-bandwidth, long-range wireless network using customer-owned Amazon devices (Echo, Ring) as bridges.

**Frequency & Modulation:**
- **900 MHz ISM band**: LoRa modulation (CSS — Chirp Spread Spectrum); range up to 1 mile
- **2.4 GHz**: FSK (Frequency Shift Keying); range ~100 ft
- **Bluetooth Low Energy (BLE)**: Short-range (30 ft), used for device setup

**Network Architecture:**

```
Sidewalk Endpoint (IoT device)
    │
    ├─ LoRa 900 MHz / FSK 2.4 GHz / BLE
    │
Sidewalk Bridge (Echo, Ring doorbell, etc.)
    │
    ├─ Wi-Fi / Ethernet
    │
Amazon Sidewalk Network Server (cloud)
    │
Endpoint Application Server
```

**Coverage Model:**
- Amazon devices (Echo, Ring) act as **Sidewalk Bridges** for free (opt-in)
- Bridges share **up to 500 MB/month** of home internet bandwidth
- Creates a **community mesh network** across neighborhoods
- Use cases: Tile trackers, pet trackers, smart lighting, sensors

**Security:**
- **Three layers of encryption**: Link layer (Sidewalk protocol), network layer (AES-128), application layer (customer-defined)
- **Sidewalk Bridge never sees endpoint data** (end-to-end encryption between endpoint and application server)
- **Capability-based access control**: Time-limited, scope-limited tokens

**AWS IoT Core for Amazon Sidewalk:**
- Developers register endpoints, define device profiles
- Messages routed to AWS IoT Core (MQTT/HTTP)
- Device shadow, rules engine integration

**LoRa Modulation (Sidewalk CSS):**
- **Spreading Factor (SF)**: 7–12 (higher SF = longer range, lower data rate)
- **Bandwidth**: 125 kHz, 250 kHz, 500 kHz
- **Data Rate**: 0.3–50 kbps (adaptive)
- **Message size**: Up to 256 bytes per message

---

### Alexa Connect Kit (ACK)

**Alexa Connect Kit** simplifies Wi-Fi connectivity and Alexa integration for IoT devices.

**ACK Module:**
- **ACK1** (deprecated): Nordic nRF52840 SoC + Espressif ESP32 Wi-Fi
- **ACK2** (current): Integrated module with Amazon-customized firmware
- Handles: Wi-Fi provisioning, Alexa authentication, OTA firmware updates, TLS encryption

**Device Provisioning:**
1. User opens Alexa app, selects "Add Device"
2. Device broadcasts BLE advertisement with ACK service UUID
3. Alexa app connects via BLE, sends Wi-Fi credentials
4. Device connects to Wi-Fi, authenticates with Amazon cloud (OAuth2 + device certificate)
5. Device appears in Alexa app; user can assign to room, group

**Alexa Smart Home API:**
- **Capability model**: Devices expose capabilities (PowerController, BrightnessController, ColorController, LockController, ThermostatController)
- **Directives**: Alexa sends directives (TurnOn, SetBrightness, SetTargetTemperature) to device cloud via AWS Lambda
- **State Reporting**: Devices report state changes proactively to Alexa Event Gateway

**Voice Control:**
```
User: "Alexa, turn on the bedroom light"
    │
Alexa Voice Service (AVS) ──> Alexa Smart Home Skill (AWS Lambda)
    │                              │
    │                              └──> Device Cloud Endpoint (HTTPS)
    │                                       │
    └───────────────────────────────────────┴──> Device (via MQTT/ACK)
```

---

## 5. DECT — Digital Enhanced Cordless Telecommunications

### DECT ULE (Ultra Low Energy)

**DECT ULE** is a low-power variant of DECT optimized for IoT (medical alert, home automation, sensors).

**Frequency:**
- **1.88–1.9 GHz** (Europe), unlicensed
- **1.92–1.93 GHz** (US), unlicensed

**Specs:**
- **ETSI EN 300 175** (DECT), **ETSI TS 102 939** (DECT ULE)
- **Modulation**: GFSK (Gaussian Frequency Shift Keying)
- **Channel bandwidth**: 1.728 MHz
- **Data rate**: Up to 1.152 Mbps (DECT), 100 kbps typical (ULE)
- **Range**: Up to 300 m (outdoor), 50 m (indoor)

**Use Cases:**
- Medical alert pendants (emergency call buttons)
- Door/window sensors
- Smoke detectors
- Smart meter readouts

**DECT ULE vs Zigbee/BLE:**
- **Advantage**: Dedicated 1.9 GHz spectrum (no 2.4 GHz congestion)
- **Disadvantage**: Limited ecosystem; proprietary vendor implementations

---

### DECT NR+ (ETSI DECT-2020)

**DECT NR+ (New Radio Plus)** is the next-generation DECT standard for private wireless IoT and industrial networks.

**Frequency:**
- **1.88–1.93 GHz** (Europe), unlicensed
- **1.92–1.93 GHz** (US), unlicensed

**Specs:**
- **ETSI TS 103 636** series (DECT-2020 NR, NR+)
- Ratified **2020**; chipsets available **2023+**
- **Modulation**: OFDM (Orthogonal Frequency Division Multiplexing), similar to LTE
- **Bandwidth**: 1.728 MHz per carrier; up to 4 aggregated carriers
- **Data rate**: Up to 10 Mbps (PHY), ~2 Mbps (application)
- **Latency**: <10 ms (time-synchronized); <100 ms (best effort)
- **Range**: Up to 500 m (outdoor), 100 m (indoor)

**Use Cases:**
- **Industrial IoT**: Factory floor sensors, AGV control, predictive maintenance
- **Smart buildings**: HVAC, lighting, access control
- **Private campus networks**: Hospitals, warehouses, airports
- **Replacement for proprietary 1.9 GHz systems** (WirelessHART alternatives)

**DECT NR+ vs Cellular (5G/LTE-M/NB-IoT):**
| Feature | DECT NR+ | 5G/LTE-M |
|---|---|---|
| Spectrum | 1.88–1.93 GHz unlicensed | Licensed (operator-controlled) |
| Coverage | 500 m | km-scale |
| Latency | <10 ms (synchronized) | 10–100 ms |
| Cost | No subscription fees | Monthly/annual SIM fees |
| Deployment | Private on-premises | Operator infrastructure |
| Security | AES-128, local root of trust | 3GPP AKA, operator-managed |

**DECT NR+ vs Wi-Fi:**
| Feature | DECT NR+ | Wi-Fi 6/6E |
|---|---|---|
| Spectrum | 1.9 GHz (no interference) | 2.4/5/6 GHz (congested) |
| Roaming | Seamless handover | 802.11k/v/w required |
| Determinism | Time-synchronized slots | Best-effort (unless TSN) |
| Power consumption | Optimized for battery | Higher (except Wi-Fi HaLow) |

**DECT NR+ Ecosystem (2024–2026):**
- **Chipsets**: Dialog (Renesas), DSP Group
- **Protocol Stack**: Open-source implementations emerging (RFQuack project, experimental)
- **Deployment**: Early trials in European industrial sites, German smart building pilots

**Standardization Roadmap:**
- DECT-2020 NR (Release 1, 2020): Base specification
- DECT NR+ (Release 2, 2023): Enhanced features (mesh, positioning, security)
- Release 3 (planned 2026): IoT-specific profiles, interoperability with 5G

---

## Cross-References

- **Bluetooth Low Energy (BLE)**: See [pan-short-range.md](pan-short-range.md) for GAP/GATT/ATT/SMP details
- **Thread**: See [pan-short-range.md](pan-short-range.md) for IEEE 802.15.4, border router, Commissioner/Joiner
- **Zigbee**: See [pan-short-range.md](pan-short-range.md) for ZHA/PRO profiles, ZCL clusters
- **Wi-Fi**: See [local-network-ip.md](local-network-ip.md) for 802.11 variants, WPA2/WPA3, roaming
- **MQTT**: See [application-messaging.md](application-messaging.md) for MQTT v3.1.1/v5.0 details

---

## Key Specification References

| Technology | Primary Specification | Notes |
|---|---|---|
| Matter | CSA Matter 1.0/1.1/1.2/1.3 | Available at CSA website (members) |
| Matter SDK | GitHub: project-chip/connectedhomeip | Open-source C++ implementation |
| Matter.js | GitHub: project-chip/matter.js | JavaScript/Node.js implementation |
| HomeKit HAP | Apple MFi Program (NDA) | Reverse-engineered: HAP-NodeJS project |
| Google Home | Actions on Google Console | Cloud integration docs |
| Local Home SDK | Google Local Home SDK docs | JavaScript runtime for local control |
| Amazon Sidewalk | AWS IoT Core for Sidewalk docs | Developer portal |
| Alexa Connect Kit | ACK Developer Guide | Amazon internal docs |
| DECT ULE | ETSI TS 102 939 | DECT ULE profile |
| DECT NR+ | ETSI TS 103 636 series | DECT-2020 NR, NR+ |

---

*Cross-references:*
- *PAN protocols (BLE, Thread, Zigbee): `references/pan-short-range.md`*
- *Local network (Wi-Fi, mDNS, IPv6): `references/local-network-ip.md`*
- *Application messaging (MQTT, CoAP): `references/application-messaging.md`*
