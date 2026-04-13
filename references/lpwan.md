# LPWAN Protocols — Complete Reference

## Overview
LPWAN (Low-Power Wide Area Network) protocols enable battery-powered IoT devices to transmit small data payloads over long distances. This reference covers all major LPWAN technologies including their physical layers, MAC protocols, network architecture, application layer considerations, and deployment contexts.

---

## LoRaWAN
### Overview
LoRaWAN is a MAC-layer protocol defined by the LoRa Alliance (TS001) operating over the LoRa chirp spread-spectrum PHY. Widely deployed globally in smart city, utility metering, and industrial asset tracking applications.
**Specification versions:** LoRaWAN v1.0.x (widely deployed), v1.1 (current, adds roaming and improved security)
### Device Classes
| Class | Downlink Window | Power | Use Case |
|-------|----------------|-------|----------|
| Class A | 2 receive windows after uplink only | Lowest | Battery sensors, default |
| Class B | Scheduled beacon-synchronized slots | Medium | Actuators needing regular downlink |
| Class C | Nearly continuous receive | Highest | Mains-powered devices, actuators |
### Activation Methods
**OTAA (Over-the-Air Activation) — Recommended:**
```
Device → Network: JoinRequest (DevEUI, AppEUI/JoinEUI, DevNonce)
Network → Device: JoinAccept (AppNonce, NetID, DevAddr, DLSettings, RxDelay, CFList)
# Session keys derived: NwkSKey, AppSKey (v1.0) or NwkSEncKey, FNwkSIntKey, SNwkSIntKey, AppSKey (v1.1)
```
**ABP (Activation By Personalization) — Legacy:**
- DevAddr, session keys pre-provisioned; no join procedure
- Risk: frame counter reset on device reset; security compromise
- Avoid for new deployments
### MAC Layer
- **Frame Counter (FCnt):** 16-bit (v1.0) or 32-bit (v1.1); prevents replay attacks
- **ADR (Adaptive Data Rate):** Network-controlled SF/bandwidth/TX power optimisation
- **Data Rates:** DR0 (SF12, 250bps) to DR5 (SF7, 5.5kbps) — region-dependent
- **Regional parameters:** EU868, US915, AU915, AS923, CN779, KR920, IN865, RU864
### Network Architecture
```
Device → Gateway (LoRa RF) → Network Server (NS) → Application Server (AS)
                                                                 ↓
                                                               Join Server (JS) [v1.1]
```
### LoRaWAN FUOTA (TS004 / RFC 9011)
Firmware Update Over The Air for LoRaWAN:
- **RFC 9011:** Fragmented Data Block Transfer (FOTA transport layer)
- **TS004:** LoRa Alliance FUOTA specification
- **Multicast groups:** Class B/C multicast for fleet-wide updates
- **SUIT manifest (RFC 9124):** Authenticated firmware manifests; pairs with LoRaWAN FUOTA
- Typical flow: unicast fragmentation parameters → multicast firmware download → local CRC verify → reboot
### LoRaWAN Roaming (TS010)
Three roaming types:
1. **Passive Roaming:** Visited NS forwards traffic to Home NS transparently; device unaware
2. **Handover Roaming (v1.1):** Device moves between network coverage areas; session transfers
3. **LoRa Alliance Packet Broker:** Third-party service enabling inter-network roaming; widely used in EU

---

## NB-IoT (Narrowband IoT)
### Overview
3GPP Rel-13 standard for narrowband cellular IoT operating in licensed spectrum (in-band LTE, guard band, or standalone). Optimised for deep coverage, massive device density, and ultra-low power.
**Maximum Coupling Loss (MCL):** 164 dB (vs 140 dB for GPRS) — ~20 dB improvement for underground/building penetration
### Key Power-Saving Features
**PSM (Power Saving Mode):**
- Device enters deep sleep after data exchange; radio off
- Periodic TAU (Tracking Area Update) timer: T3412 — configurable from minutes to days
- Active Timer: T3324 — window after TAU where device is reachable
- Device-initiated: sends data, starts Active Timer, then sleeps until next TAU
```
Timeline: [Data TX] → [Active Timer T3324] → [PSM Sleep until T3412] → [TAU] → repeat
```
**eDRX (Extended Discontinuous Reception):**
- Device "wakes" on known schedule within a Paging Hyper Frame
- Less aggressive than PSM; allows periodic downlink paging
- eDRX cycle: 20.48s to 2621.44s (configurable)
- Used when low-latency downlink needed but always-on is power-prohibitive
### Data Delivery Methods
**User Plane (UP) CIoT Optimisation:**
- Data via evolved S1-U interface; standard bearer
- Lower latency; suitable for moderate data rates
**Control Plane (CP) CIoT Optimisation:**
- **DoNAS (Data over Non-Access Stratum):** Small data embedded in NAS signalling messages
- No data radio bearer needed; lower overhead for tiny payloads
- Recommended for <100 byte infrequent messages
**NIDD (Non-IP Data Delivery):**
- Non-IP PDU sessions via SCEF (Rel-13) / NEF (5G)
- Bypasses IP stack entirely; no IP addressing on device
- Ideal for devices with no IP stack (simple sensors)
- SCEF API: device sends raw bytes → SCEF → Application Server via T8 interface (RESTful)
**NIDD via SMS (3GPP TS 23.040):**
- MO-SMS and MT-SMS for uplink/downlink data delivery
- Alternative to SCEF-based NIDD for legacy network compatibility
- Used where SCEF not deployed; higher latency; limited payload
### 3GPP SCEF/NEF Architecture
```
Device → eNB → MME → SCEF (Rel-13) → Application Server
                                                           ↓
                                                           T8 REST API (NIDD, Device Triggering, Monitoring Events)
```
- **SCEF (Rel-13):** Service Capability Exposure Function
- **NEF (5G Rel-15+):** Network Exposure Function (successor; adds OAuth 2.0, richer APIs)
- **Device Triggering (TS 23.682):** Wakes PSM-sleeping device via network SMS trigger or T8 API

---

## LTE-M / Cat-M1
### Overview
3GPP Rel-13 (eMTC). Higher throughput than NB-IoT; supports mobility, VoLTE, full handover. 1.4 MHz bandwidth (vs NB-IoT's 200 kHz).
| Feature | NB-IoT | LTE-M |
|---------|--------|-------|
| Bandwidth | 200 kHz | 1.4 MHz |
| Peak DL | 250 kbps | 1 Mbps |
| Mobility | Limited/none | Full handover |
| VoLTE | No | Yes |
| PSM | Yes | Yes |
| eDRX | Yes | Yes |
| MCL | 164 dB | 155.7 dB |
| Use case | Static sensors | Mobile trackers, voice |

---

## Sigfox
Ultra-narrow band (UNB) proprietary LPWAN. 192 kHz channelised; 100 Hz signal bandwidth.
- **Uplink:** 140 messages/day max; 12-byte payload; 3 transmission diversity
- **Downlink:** 4 messages/day max; 8-byte payload; triggered by uplink
- **Coverage:** ~1000 km² per base station in rural; proprietary network
- **IP Licensing:** Sigfox acquired by Unabiz 2022; commercial network subscription required; open implementation restricted
- **Device types:** 0 (uplink only), 1 (uplink + downlink)

---

## Wi-SUN (IEEE 802.15.4g / FAN)
**Wi-SUN FAN (Field Area Network)** profile for smart grid AMI:
- **PHY:** IEEE 802.15.4g — 2.4 GHz, 868/915 MHz, 920 MHz (Japan); OFDM and FSK modulations
- **MAC:** IEEE 802.15.4e TSCH
- **Network:** IPv6 + 6LoWPAN + RPL (RFC 6550) mesh routing
- **Security:** 802.1X/EAP-TLS with PKI certificates; AES-128 CCM* link layer
- **Application:** DLMS/COSEM, IEEE 2030.5/SEP 2.0
- **Deployments:** Major utility AMI globally; Japan NTT Docomo, UK, USA utilities
- **Wi-SUN FAN 1.1:** Enhanced frequency hopping, improved interoperability

---

## MIOTY (ETSI TS 103 357)
Multi-tone ALOHA LPWAN for industrial IoT:
- Developed by Fraunhofer IIS; standardised by ETSI in 2021
- **Modulation:** Telegram Splitting (TS-UNB) — splits message across 6 sub-telegrams on different frequencies
- **Robustness:** High immunity to interference; reliable in industrial RF environments
- **Range:** Up to 40 km line-of-sight; excellent penetration
- **Data rate:** 0.8–100 kbps depending on mode
- **Adopted by Wize Alliance** for European utility metering alongside 169 MHz
- **Base stations:** Compatible with LoRaWAN gateway infrastructure in some implementations

---

## D7AP — DASH7 Alliance Protocol (ISO/IEC 18000-7)
Sub-GHz bi-directional LPWAN:
- **Standard:** ISO/IEC 18000-7; operating frequencies: 433, 868, 915 MHz
- **Characteristics:** Bursty, bi-directional, session-oriented (vs LoRaWAN's message-oriented)
- **Topology:** Star, mesh, peer-to-peer
- **Data rate:** 9.6 – 167 kbps
- **File-based data model:** Devices expose "files" (configuration, sensor data)
- **Use cases:** Military/defence logistics, supply chain track-and-trace, smart buildings
- **Latency:** Sub-second wake-up and response — better than LoRaWAN for bi-directional
- **Security:** AES-128 at application and link layer

---

## EC-GSM-IoT (3GPP Rel-13)
Extended Coverage GSM for IoT operating on legacy 2G networks:
- **MCL:** 164 dB (same as NB-IoT) via coverage extension (CE) modes A and B
- **Data rate:** 70-240 kbps (EGPRS-based)
- **SIM:** Standard SIM; fits existing M2M SIM deployments
- **Migration path:** For operators with large 2G installed base; bridge to NB-IoT
- **Availability:** Declining as 2G sunset accelerates in many markets

---

## SCHC (Static Context Header Compression)
**RFC 8724** — Header compression and fragmentation for LPWAN:
| Feature | Detail |
|---------|--------|
| Standard | RFC 8724 (SCHC framework) |
| Purpose | IPv6/UDP/CoAP header compression for LPWAN |
| Compression | Context-based; shared static context eliminates redundant header fields |
| Fragmentation | Splits IPv6 packets across multiple LPWAN frames (< IPv6 1280B MTU) |
| Modes | No-Ack, Ack-on-Error, Ack-Always |
**Transport-specific profiles:**
- **RFC 9011:** SCHC over LoRaWAN — fragmentation/compression rules for LoRa frames
- **RFC 9363:** SCHC over NB-IoT — adaptation for NB-IoT frame structure
**Compression example (CoAP over UDP over IPv6):**
```
Before SCHC: IPv6 (40B) + UDP (8B) + CoAP (4-20B) = 52-68 bytes overhead
After SCHC: 2-5 bytes (context-compressed) + payload
```

---

## Wireless M-Bus / OMS (EN 13757-4)

**Purpose**: The dominant European smart metering RF protocol for utility meters (water, gas, heat, electricity). Deployed on 100M+ meters across EU. The three-layer stack (wM-Bus PHY → OMS application layer → DLMS/COSEM data model) provides standardized wireless meter reading with vendor interoperability.

**Standards Bodies**:
- **CEN TC294**: European Committee for Standardization (EN 13757 series)
- **OMS-Group**: Open Metering System alliance (www.oms-group.org) — 100+ members (meter manufacturers, utilities)
- **DLMS UA**: DLMS User Association (IEC 62056-5-3, COSEM object model)

### Three-Layer Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│             APPLICATION LAYER (DLMS/COSEM)                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ OBIS Codes (Object Identification System)               │   │
│  │   - 1.8.0: Cumulative energy import (kWh)               │   │
│  │   - 6.8.0: Cumulative heat energy (MWh)                 │   │
│  │   - 13.7.0: Instantaneous power factor                  │   │
│  │ COSEM Interface Classes:                                │   │
│  │   - Data Class (read-only values)                       │   │
│  │   - Register Class (metered values with scaler/unit)    │   │
│  │   - Profile Generic Class (load profile, 15-min intervals)│ │
│  └──────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│                 OMS LAYER (EN 13757-3)                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ OMS Specification v4.0.4 (2020)                         │   │
│  │ DIF/VIF Encoding:                                       │   │
│  │   - DIF (Data Information Field): data type, storage    │   │
│  │   - VIF (Value Information Field): unit, multiplier     │   │
│  │ OMS Security Modes:                                     │   │
│  │   - Mode 5: AES-128-CBC encrypted, MAC                  │   │
│  │   - Mode 7: AES-128-CTR encrypted, CMAC                 │   │
│  └──────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│              WIRELESS M-BUS LAYER (EN 13757-4)                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ Physical Layer: 868 MHz (EU), 169 MHz (Wize), 433 MHz   │   │
│  │ Modulation: 2-FSK, 4-GFSK (mode-dependent)              │   │
│  │ Transmission Modes: T1/T2, S1/S2, R2, C1/C2, N         │   │
│  │ Frame Structure: L-field, C-field, M-field, A-field, CI │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Wireless M-Bus (wM-Bus) — Physical & Data Link (EN 13757-4)

**Frequency Bands and Transmission Modes**:

| Mode | Frequency (MHz) | Modulation | Data Rate | Direction | Tx Interval | Use Case | Power |
|------|-----------------|------------|-----------|-----------|-------------|----------|-------|
| **T1** | 868.95 | 2-FSK | 100 kbps | Uplink only | 2-3 min | Walk-by/drive-by meter reading (handheld) | 10 mW (10 dBm) |
| **T2** | 868.95 | 2-FSK | 100 kbps | Bidirectional | On-demand | Two-way meter reading (AMR collector) | 10 mW |
| **S1** | 868.3 | 2-FSK | 32.768 kbps | Uplink only | Hourly | Stationary fixed network (low traffic) | 25 mW (14 dBm) |
| **S2** | 868.3 | 2-FSK | 32.768 kbps | Bidirectional | On-demand | Bidirectional stationary network | 25 mW |
| **R2** | 868.33 | 2-FSK | 32.768 kbps | Receive-only | — | Concentrator/collector listening mode | N/A (RX) |
| **C1** | 869.525 | 4-GFSK | 100 kbps | Uplink only | 2 min | Compact high-throughput (higher density) | 500 mW (27 dBm) |
| **C2** | 869.525 | 4-GFSK | 100 kbps | Bidirectional | On-demand | Compact bidirectional | 500 mW |
| **N** | 169.4 | DBPSK/4-GFSK | 2.4-19.2 kbps | Bidirectional | 15 min - 1 day | Long-range Wize (underground gas/water) | 500 mW |

**Modulation Details**:
```
2-FSK (Frequency Shift Keying):
  - Binary modulation: 0 = f_center - Δf, 1 = f_center + Δf
  - Frequency deviation: ±25 kHz (T-mode), ±50 kHz (S-mode)
  - Simple demodulation → lower power consumption

4-GFSK (Gaussian Frequency Shift Keying):
  - Four-level modulation → 2 bits per symbol → higher data rate
  - Gaussian filter reduces spectral sidelobes
  - Used in C-mode (compact, high-density urban areas)
```

**Frame Structure** (EN 13757-4):

```
┌────────────────────────────────────────────────────────────────┐
│                    wM-Bus Frame (OSI Layer 2)                  │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────┬──────┬──────┬──────┬──────┬─────────┬──────┬──────┐ │
│  │ L    │ C    │ M    │ A    │ CI   │ Payload │ CRC  │ RSSI │ │
│  │ (1B) │ (1B) │ (2B) │ (6B) │ (1B) │ (var)   │ (2B) │      │ │
│  └──────┴──────┴──────┴──────┴──────┴─────────┴──────┴──────┘ │
│                                                                │
├────────────────────────────────────────────────────────────────┤
│ Field Definitions:                                             │
│                                                                │
│ L-field (Length): Number of bytes following L-field            │
│   - Example: 0x2F = 47 bytes total frame                      │
│                                                                │
│ C-field (Control): Frame type and direction                   │
│   - 0x44: SND-NR (send, no reply) — typical meter uplink      │
│   - 0x46: SND-IR (send, installation request) — meter install │
│   - 0x5B: RSP-UD (response, user data) — meter response       │
│   - 0x7A: ACC-NR (access, no reply) — meter access request    │
│                                                                │
│ M-field (Manufacturer): 2 bytes, manufacturer code             │
│   - 0x2D44: "DME" (Diehl Metering)                            │
│   - 0x4C4F: "LOO" (Landis+Gyr)                                │
│   - 0x4B4D: "KAM" (Kamstrup)                                  │
│   - Full list: EN 13757-3 Annex C                             │
│                                                                │
│ A-field (Address): 6 bytes, device unique identifier           │
│   - Bytes 0-3: Meter serial number (BCD encoded)              │
│   - Byte 4: Version (0x01 = v1, 0x02 = v2, etc.)              │
│   - Byte 5: Medium type (see table below)                     │
│                                                                │
│ CI-field (Control Information): Application layer type        │
│   - 0x72: Variable length data (short header)                 │
│   - 0x78: Variable length data (long header, encryption)      │
│   - 0x7A: Long header, AES-128 encrypted                      │
│   - 0x8C: Extended Link Layer II (ELL II) — OMS encryption    │
│                                                                │
│ Payload: OMS data records (DIF/VIF encoded)                   │
│                                                                │
│ CRC: 16-bit CRC-CCITT for frame integrity                     │
└────────────────────────────────────────────────────────────────┘
```

**Medium Type Codes** (A-field byte 5):

| Code | Medium | Examples |
|------|--------|----------|
| 0x02 | Electricity | Smart electric meters |
| 0x03 | Gas | Natural gas meters |
| 0x04 | Heat | District heating meters, heat cost allocators |
| 0x06 | Warm Water | Hot water meters (30-90°C) |
| 0x07 | Water | Cold water meters |
| 0x0A | Heat Cost Allocator | Radiator usage metering (not actual heat) |
| 0x0D | Heat/Cooling Load Meter | Combined heating and cooling |
| 0x15 | Hot Water | >90°C |
| 0x16 | Cold Water | <30°C |
| 0x37 | Gas (Mode 2) | Two-way communication gas meters |

### OMS (Open Metering System) — Application Layer

**OMS Specification**: v4.0.4 (2020), available at www.oms-group.org

**DIF/VIF Encoding** (Data Information Field / Value Information Field):

```
OMS Data Record Structure:
  DIF (1-2 bytes) + VIF (1-2 bytes) + Data (variable length)

DIF (Data Information Field):
  Bit 7: Extension bit (1 = DIFE follows)
  Bit 6-4: Function field
    - 000: Instantaneous value
    - 001: Maximum value
    - 010: Minimum value
    - 011: Value during error state
  Bit 3-2: Storage number (0-3 for tariff registers)
  Bit 1-0: Data field length and encoding
    - 00: No data
    - 01: 8-bit integer
    - 10: 16-bit integer
    - 11: 24-bit integer
    - 100: 32-bit integer
    - 110: 48-bit integer
    - 111: 64-bit integer
    - BCD variants: Add 0x04 to length code

Example DIF: 0x04 = 32-bit integer, instantaneous, storage 0

VIF (Value Information Field):
  Defines unit and multiplier for the data value

  Primary VIFs (EN 13757-3 Table 18):
    0x00-0x07: Energy (Wh), 10^(n-3) to 10^(n+4)
    0x08-0x0F: Energy (J), 10^(n) to 10^(n+7)
    0x10-0x17: Volume (m³), 10^(n-6) to 10^(n+1)
    0x18-0x1F: Mass (kg), 10^(n-3) to 10^(n+4)
    0x28-0x2F: Power (W), 10^(n-3) to 10^(n+4)
    0x38-0x3F: Volume flow (m³/h), 10^(n-6) to 10^(n+1)
    0x58-0x5B: Temperature (°C), 10^(n-3) to 10^n
    0xFD: VIF extension (VIFE follows)
    0xFF: Manufacturer-specific VIF

Example VIF: 0x13 = Volume in liters (10^0 m³ = 1000 liters)

Combined DIF+VIF+Data Example:
  Hex: 04 13 12 34 56 78
  DIF: 0x04 = 32-bit integer
  VIF: 0x13 = Volume in liters
  Data: 0x78563412 (little-endian) = 2,018,915,346 liters = 2,018,915 m³
```

**Real-World OMS Telegram Example** (Water Meter):

```
Raw Frame (hex):
  44 2D44 14152637 01 07 7A 6A000000 046D250B9C26 0413F70E0000 ...

Decoded:
  C-field: 0x44 = SND-NR (meter transmission)
  M-field: 0x2D44 = "DME" (Diehl Metering)
  A-field:
    Serial: 37261514 (BCD)
    Version: 0x01
    Medium: 0x07 (Water)
  CI-field: 0x7A = Long header with encryption

  Encrypted Payload (after decryption with AES-128 key):
    046D 250B9C26   → DIF 0x04 (32-bit), VIF 0x6D (timestamp), Data: 2026-12-25 11:37
    0413 F70E0000   → DIF 0x04 (32-bit), VIF 0x13 (volume liters), Data: 3,831 m³
    ...additional data records...

Interpretation:
  Water meter serial 37261514 transmitted reading on 2026-12-25 at 11:37
  Cumulative water consumption: 3,831 m³
```

### OMS Security Modes

**OMS Security Profile Evolution**:

| Mode | Encryption | MAC/CMAC | Key Length | Use Case | OMS Version |
|------|------------|----------|------------|----------|-------------|
| **Mode 5** | AES-128-CBC | MAC (8 bytes) | 128-bit | Standard security (water, gas, heat) | OMS v3.0+ |
| **Mode 7** | AES-128-CTR | CMAC (4 bytes) | 128-bit | Advanced security (electricity, bidirectional) | OMS v4.0+ |
| **Mode 13** | AES-128-GCM | GCM tag (12 bytes) | 128-bit | Future-proof (OMS v5 planned) | Draft |

**Mode 5 (AES-128-CBC with MAC)**:

```
Encryption Flow:
  1. Plaintext data records (DIF/VIF/Data)
  2. Padding: PKCS#7 (pad to 16-byte boundary)
  3. IV (Initialization Vector): Constructed from M-field + A-field + access number
     IV = M-field (2B) || A-field (6B) || Access Number (8B)
  4. AES-128-CBC encryption (plaintext + IV + key → ciphertext)
  5. MAC calculation: First 8 bytes of AES-CBC-MAC over (M || A || CI || plaintext)
  6. Transmitted: L || C || M || A || CI=0x7A || Access Number (8B) || Ciphertext || MAC (8B) || CRC

Key Derivation:
  - Meter-specific key derived from master key + meter serial number
  - K_meter = AES-CMAC(K_master, M-field || A-field)
  - Master key provisioned at meter manufacturing or commissioning

Decryption at Collector:
  - Lookup meter key from database (indexed by M-field + A-field)
  - Reconstruct IV from M-field + A-field + Access Number
  - Decrypt ciphertext with AES-128-CBC
  - Verify MAC (reject if mismatch → tampering detected)
```

**Mode 7 (AES-128-CTR with CMAC)** — Improved Performance:

```
Advantages over Mode 5:
  - CTR mode allows parallelization (faster decryption)
  - No padding required (exact plaintext length preserved)
  - CMAC stronger than CBC-MAC (no padding oracle vulnerability)

Encryption Flow:
  1. Nonce construction: M-field (2B) || A-field (6B) || CI (1B) || Access Number (5B)
  2. Counter mode encryption: Ciphertext = Plaintext XOR AES-CTR(Nonce, Counter, Key)
  3. CMAC: Truncated to 4 bytes (over M || A || CI || Ciphertext)
  4. Transmitted: L || C || M || A || CI=0x8C || Access Number (5B) || Ciphertext || CMAC (4B) || CRC
```

**Key Management**:

```
Factory Key Provisioning:
  - Each meter assigned unique encryption key at manufacturing
  - Key stored in secure memory (non-extractable)
  - Key ID or meter serial recorded in Head-End System (HES) database

Key Distribution to Collectors:
  - Secure channel from HES to field collectors (TLS, VPN)
  - Collector stores keys in encrypted local database (SQLite/PostgreSQL)
  - Key rotation: OMS allows over-the-air key update via downlink command

Typical Deployment:
  - Utility owns master keys (stored in HSM — Hardware Security Module)
  - Per-meter keys derived on-demand or pre-generated
  - Key escrow: Backup keys stored for meter replacement scenarios
```

### DLMS/COSEM Integration (IEC 62056)

**Three-Layer Stack in Practice**:

```
1. Physical Layer: wM-Bus (868 MHz, 2-FSK, Mode T1)
   └─ Frame transmission every 2 minutes

2. Application Layer: OMS (DIF/VIF encoding)
   └─ Data records map to DLMS OBIS codes

3. Data Model: COSEM (IEC 62056-6-2)
   └─ Objects represent meter registers

Example Mapping (Water Meter):

wM-Bus Frame (encrypted Mode 5):
  CI=0x7A → Encrypted data follows

OMS Data Record (after decryption):
  DIF: 0x04 (32-bit integer)
  VIF: 0x13 (Volume in liters)
  Data: 0x00001234 = 4,660 liters

DLMS COSEM Object:
  OBIS Code: 8.0.1.0.0.255 (Volume register 1)
  Class: Register (class ID 3)
  Attributes:
    - Logical Name: 8.0.1.0.0.255
    - Value: 4.660 (m³)
    - Scaler/Unit: scaler=-3, unit=m³

Utility HES (Head-End System):
  - Receives wM-Bus telegram from collector
  - Decrypts with meter key
  - Parses OMS DIF/VIF records
  - Updates DLMS device object in database
  - Billing system queries COSEM objects for monthly consumption
```

### Real-World Deployments

**Scale**: 100M+ wM-Bus/OMS meters deployed across Europe (2024 estimate)

**Country-Specific Deployments**:

```
Germany:
  - 25M+ water meters (Diehl, Zenner, Sensus)
  - District heating (Kamstrup, Techem) — 10M+ heat cost allocators
  - Gas meters (Itron, Elster) — wM-Bus Mode N (169 MHz Wize) for underground

France:
  - Veolia: 5M+ water meters (wM-Bus + Wize 169 MHz)
  - Suez: 3M+ water meters (OMS Mode 5)
  - Enedis (electricity): Limited wM-Bus (primarily G3-PLC/Linky)

Spain:
  - Gas Natural Fenosa: 2M+ gas meters (wM-Bus Mode T1)
  - Water utilities: Barcelona, Madrid (Kamstrup, Sensus)

Italy:
  - Hera: 1.5M+ water meters (wM-Bus OMS)
  - A2A: Combined heat/cooling (wM-Bus Mode S1)

Scandinavia:
  - District heating dominant use case (80% market share)
  - Kamstrup MultiCal (heat meter) — wM-Bus Mode C1 (high data rate)
  - Norway: Remote mountain cabin metering (Mode N, 169 MHz long-range)
```

**Meter Manufacturers**:

| Vendor | Product Line | wM-Bus Mode | Medium | OMS Version |
|--------|--------------|-------------|--------|-------------|
| Kamstrup | MultiCal, FlowIQ | C1, T1 | Heat, Water | OMS v4.0 (Mode 7) |
| Diehl Metering | Hydrus, IZAR | T1, N | Water, Heat | OMS v4.0 |
| Landis+Gyr | W350, Ultraheat | T1, S1 | Water, Heat | OMS v3.0 (Mode 5) |
| Sensus | iPERL, HRI | T1 | Water, Heat | OMS v4.0 |
| Itron | Cyble, Actaris | T1, N (gas) | Water, Gas | OMS v4.0 |
| Zenner | Minomess, Zelsius | T1, S1 | Water, Heat | OMS v3.0 |
| Elster (Honeywell) | BK-G4, BK-G6 | N | Gas | OMS v4.0 |

**Collector/Gateway Architectures**:

```
Walk-by/Drive-by Reading (Mode T1):
  Handheld collector (Bluetooth + wM-Bus radio)
  → Meter reader walks/drives past meters
  → Meters transmit every 2 min → Collector captures telegrams
  → Data uploaded to HES via cellular/Wi-Fi

Fixed Network (Mode S1/C1):
  Stationary collector (pole-mounted, building-mounted)
  → 500m-2km coverage radius (urban)
  → Receives hourly transmissions from meters
  → Backhaul: 4G/LTE, Ethernet, LoRaWAN gateway
  → HES polls collectors for meter data

Hybrid AMR/AMI:
  - AMR (Automatic Meter Reading): wM-Bus for data collection
  - AMI (Advanced Metering Infrastructure): DLMS/COSEM head-end
  - Gateway translates wM-Bus → DLMS Client/Server protocol
```

### wM-Bus vs NB-IoT vs LoRaWAN (Utility Metering Comparison)

| Feature | wM-Bus (Mode T1) | NB-IoT | LoRaWAN |
|---------|------------------|--------|---------|
| Frequency | 868 MHz (unlicensed) | Licensed LTE bands | 868 MHz (unlicensed) |
| Range | 500m - 2km (urban) | 10-15 km | 2-10 km |
| Data Rate | 100 kbps | 20-250 kbps | 0.3-50 kbps |
| Latency | <1s (uplink) | 1-10s | 1-10s |
| Battery Life | 10-15 years | 10+ years | 10+ years |
| Infrastructure | Proprietary collectors | Carrier network | LoRaWAN gateways |
| Cost/Meter | Low (€5-15 module) | Medium (€10-30 module + SIM) | Low (€5-10 module) |
| EU Deployment | 100M+ meters | 5M+ meters (growing) | 10M+ meters |
| Pros | Mature, OMS standard, no OPEX | No gateway needed, MNO support | Long range, low power |
| Cons | Requires collector infrastructure | SIM cost, carrier dependency | Non-standardized application layer |

**Migration Path**: Legacy wM-Bus → Hybrid (wM-Bus + NB-IoT) → Full cellular (NB-IoT/LTE-M) for new deployments

Cross-reference: [industrial-ot.md](industrial-ot.md) for DLMS/COSEM protocol detail; [cellular-wan.md](cellular-wan.md) for NB-IoT AMI deployments

---

## Wize Alliance (169 MHz LPWAN)
Designed specifically for water, gas, and heat meter reading in dense urban environments:
- **Frequency:** 169.4–169.8 MHz (Europe, licensed band)
- **Modulation:** 4-GFSK, DBPSK
- **Range:** ~5 km urban; 40 km rural (superior urban penetration vs 868 MHz)
- **Building penetration:** Excellent — 169 MHz penetrates concrete/underground infrastructure
- **Data rate:** 2.4 kbps (low) to 19.2 kbps
- **Protocol version:** Wize v1.3
- **Application layer:** Compatible with wM-Bus/OMS data records; M-Bus application objects
- **Deployments:** French/Spanish/Italian water distribution networks; Suez, Veolia, Saur
- **Based on:** Wavenis technology (Coronis Systems) evolved for utility metering

---

## Satellite IoT / 3GPP NTN
### Overview
Satellite-based IoT connectivity for global coverage beyond terrestrial networks.
**Non-Terrestrial Networks (3GPP NTN):**
- **Rel-17:** NB-IoT and LTE-M device-to-LEO/GEO satellite (direct-to-satellite, no gateway)
- **Rel-18:** Enhanced NTN; better handover, regulatory alignment
- **Challenge:** Satellite propagation delay (LEO: ~20ms, GEO: ~600ms) requires timing advance adaptation
- **Key modification:** Extended timing advance (TA) compensation in NB-IoT/LTE-M MAC for satellite propagation
**Commercial Satellite IoT:**
| Provider | Technology | Coverage | Notes |
|----------|-----------|----------|-------|
| Iridium SBD | L-band 1.6 GHz | Global (66 LEO) | Short Burst Data; 340 byte payload |
| Inmarsat BGAN M2M | L-band | Near-global (GEO) | Higher throughput; BGAN terminal |
| Starlink IoT | Ka/Ku-band | Growing LEO | Primarily broadband; IoT via gateway |
| Kinéis | VHF (137-138 MHz) | LEO, global | Low-power, asset tracking |
| Orbcomm | VHF + L-band | LEO, global | M2M focused |
| Astrocast | L-band 1.6 GHz | LEO | NB-IoT compatible |

---

## RedCap / eRedCap (3GPP Rel-17/18)
**Reduced Capability (RedCap)** — new device class between Cat-M and full LTE:
| Parameter | NB-IoT | LTE-M | RedCap (Rel-17) | Full LTE |
|-----------|--------|-------|-----------------|----------|
| Bandwidth | 200 kHz | 1.4 MHz | 20 MHz | 100 MHz |
| Rx antennas | 1 | 1 | 2 | 2-4 |
| Peak DL | 250 kbps | 1 Mbps | 150 Mbps | 1 Gbps |
| Target | Ultra-low cost | Low power mobile | Wearables, industrial | Full capability |
| Use case | Sensors | Trackers | Smart watches, cameras | Smartphones |
**eRedCap (Rel-18):** Further cost reduction; 5 MHz bandwidth option; single Rx antenna variant

---

## EC-GSM-IoT, Weightless, RPMA
**Weightless:**
- Weightless-N: 169 MHz, uplink-only, Nwave proprietary
- Weightless-P: 868/915 MHz, bidirectional, 12.5 kHz channels
- Weightless-W: TV whitespace, limited deployment
**Ingenu RPMA:**
- Random Phase Multiple Access; 2.4 GHz ISM band
- High capacity; primarily North American deployment
- Proprietary network model; limited global expansion