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
**The dominant European smart metering RF protocol for water, gas, heat, and electricity meters.**
### Standard
- **EN 13757-4:** Wireless M-Bus (wM-Bus) — physical and data link layer
- **OMS (Open Metering System):** Application profile over wM-Bus; managed by OMS-Group
### Frequency and Transmission Modes
| Mode | Frequency | Direction | Interval | Use Case |
|------|-----------|-----------|----------|----------|
| T1 | 868.95 MHz | Uplink only | 2-3 min | Standard meter walk-by/drive-by reading |
| T2 | 868.95 MHz | Bidirectional | — | Bidirectional meter reading |
| S1 | 868.3 MHz | Uplink only | Hourly | High-security, low-frequency stationary |
| S2 | 868.3 MHz | Bidirectional | — | Bidirectional stationary |
| R2 | 868.33 MHz | Receive-only | — | Collector/concentrator mode |
| C1 | 869.525 MHz | Uplink only | 2 min | Compact mode, higher throughput |
| C2 | 869.525 MHz | Bidirectional | — | Compact bidirectional |
| N | 169.4 MHz | Bidirectional | — | Long range for gas/water (Wize) |
### Protocol Stack
```
Application Layer: DLMS/COSEM (IEC 62056) or OMS application objects
Data Link Layer: wM-Bus frame (CI field, M-field, A-field, DIF/VIF data records)
Physical Layer: EN 13757-4 (868 MHz or 169 MHz)
```
### OMS Data Records
Data encoded as DIF (Data Information Field) + VIF (Value Information Field) + Data:
- **DIF:** Data type (integer, BCD, real), storage number, tariff, function field
- **VIF:** Unit and multiplier (energy kWh, volume m³, flow rate m³/h, temperature °C)
- **OBIS codes:** Standardised meter object addressing (cross-reference DLMS/COSEM)
### Security
- **AES-128 CBC** encryption for meter data (OMS Security Profile A/B/C)
- **Message Authentication Code (MAC):** Integrity protection
- Key management: Initial keys programmed at manufacturing; OMS key exchange protocol
### Deployments
Widely deployed across EU for:
- **Gas meters:** Elster, Itron, Landis+Gyr, Honeywell
- **Water meters:** Sensus, Kamstrup, Landis+Gyr (W350/W370), Diehl, Zenner
- **Heat meters:** Kamstrup, Techem, Brunata
- **Electricity meters:** Itron, Landis+Gyr (alongside DLMS/COSEM primary)

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