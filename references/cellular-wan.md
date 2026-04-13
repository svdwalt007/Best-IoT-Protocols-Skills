# Cellular WAN Protocols for IoT

> **Scope:** This reference covers cellular radio access and core network protocols
> relevant to IoT device design, from legacy 2G/GPRS through 5G NR RedCap/eRedCap,
> NTN satellite, V2X, and the 3GPP network exposure APIs that enable IoT platform
> integration. For 3GPP release structure, NAS/RRC internals, and 5G core architecture,
> defer to the sibling [3GPP skill](https://github.com/lugasia/3gpp-skill).

---

## Table of Contents

1. [UE Category Reference — IoT Technology Ladder](#1-ue-category-reference--iot-technology-ladder)
2. [2G / GPRS / EDGE](#2-2g--gprs--edge)
3. [3G / HSPA / WCDMA](#3-3g--hspa--wcdma)
4. [4G LTE — Full Category Coverage](#4-4g-lte--full-category-coverage)
   - [LTE-M (eMTC) — Cat-M1 / Cat-M2](#lte-m-emtc--cat-m1--cat-m2)
   - [NB-IoT — Cat-NB1 / Cat-NB2](#nb-iot--cat-nb1--cat-nb2)
   - [NB-IoT Deployment Modes](#nb-iot-deployment-modes)
   - [Coverage Enhancement (CE) Modes](#coverage-enhancement-ce-modes)
5. [Power Saving Mechanisms — All Generations](#5-power-saving-mechanisms--all-generations)
6. [EC-GSM-IoT](#6-ec-gsm-iot)
7. [5G NR — Full IoT Coverage](#7-5g-nr--full-iot-coverage)
   - [5G CIoT Optimisations](#5g-ciot-optimisations)
   - [RedCap — 3GPP Rel-17](#redcap--3gpp-rel-17)
   - [eRedCap — 3GPP Rel-18](#eredcap--3gpp-rel-18)
   - [5G TSN Integration](#5g-tsn-integration)
   - [5G LAN-Type Service](#5g-lan-type-service)
   - [Private Networks (SNPN / PNI-NPN)](#private-networks-snpn--pni-npn)
8. [3GPP SCEF / NEF — IoT Exposure APIs](#8-3gpp-scef--nef--iot-exposure-apis)
9. [V2X Protocols](#9-v2x-protocols)
10. [Satellite IoT / NTN](#10-satellite-iot--ntn)

---

## 1. UE Category Reference — IoT Technology Ladder

Understanding the full 3GPP UE category taxonomy is essential for technology selection.
The categories below represent the complete IoT-relevant ladder, from narrowband LPWA
up to 5G mid-tier. Broadband categories (Cat-6, Cat-9, Cat-16, etc.) are out of scope.

| Category | 3GPP Release | DL Peak | UL Peak | BW | Antennas | PSM/eDRX | Handover | Voice | IoT Role |
|---|---|---|---|---|---|---|---|---|---|
| Cat-0 | Rel-12 | 1 Mbps | 1 Mbps | 20 MHz | 1 Rx | ✅ | ✅ | ❌ | Deprecated; rarely deployed |
| **Cat-1** | Rel-8 | 10 Mbps | 5 Mbps | 20 MHz | **2 Rx** | ✅ | ✅ | ✅ | Legacy IoT/M2M; wide global coverage |
| **Cat-1 bis** | Rel-13 | 10 Mbps | 5 Mbps | 20 MHz | **1 Rx** | ✅ | ✅ | ✅ | Single-antenna Cat-1; 2G/3G migration target; active market |
| **Cat-M1** (LTE-M) | Rel-13 | 1 Mbps | 1 Mbps | 1.4 MHz | 1 Rx | ✅ | ✅ | ✅ | LPWA with mobility + voice; MCL 155.7 dB |
| **Cat-M2** | Rel-14 | 4 Mbps | 7 Mbps | 5 MHz | 1 Rx | ✅ | ✅ | ✅ | Enhanced LTE-M; higher throughput |
| **Cat-NB1** | Rel-13 | 27 kbps | 66 kbps | 200 kHz | 1 Rx | ✅ | ❌ | ❌ | NB-IoT baseline; MCL 164 dB |
| **Cat-NB2** | Rel-14 | 127 kbps | 159 kbps | 200 kHz | 1 Rx | ✅ | ❌ | ❌ | Enhanced NB-IoT; multicast, positioning, multi-carrier |
| Cat-4 | Rel-8 | 150 Mbps | 50 Mbps | 20 MHz | 2 Rx | ❌ | ✅ | ✅ | IoT gateways/routers; intermediate tier |
| **RedCap** | Rel-17 (5G NR) | ≤226 Mbps | ≤120 Mbps | 20 MHz FR1 | 1–2 Rx | ✅ | ✅ | ✅ | 5G mid-tier; replaces Cat-3/4 in IoT |
| **eRedCap** | Rel-18 (5G NR) | ≤10 Mbps | ≤10 Mbps | ≤20 MHz FR1 | 1 Rx | ✅ | ✅ | ✅ | 5G replacement for Cat-1/Cat-1 bis; 2026+ |

**Technology selection heuristic:**
- Ultra-low power, stationary, small data → NB-IoT (Cat-NB1/NB2)
- Low power, mobile, voice needed → LTE-M (Cat-M1/M2)
- Moderate bandwidth, single antenna, global LTE coverage → Cat-1 bis
- Mid-tier IoT on 5G SA networks → RedCap
- Future-proof low-cost 5G, Cat-1 bis replacement → eRedCap (2026+)

---

## 2. 2G / GPRS / EDGE

> ⚠️ **Sunset Warning:** 2G networks are being shut down progressively. AT&T (US, 2017),
> T-Mobile (US, 2020), Vodafone (UK, 2024), many European operators through 2025–2026.
> Do NOT design new IoT products with 2G as the primary or sole connectivity. GSM 900 MHz
> spectrum is being refarmed for NB-IoT guard-band and standalone deployments.

- **GSM data layer:** Circuit-switched data (CSD) at 9.6 kbps; largely obsolete
- **GPRS (Rel-97/98):** Packet-switched overlay on GSM; multi-slot classes 1–33; up to 171 kbps theoretical (Class 10: 48 kbps practical)
- **EDGE (Rel-4):** Enhanced GPRS; 8-PSK modulation; up to 384 kbps; still widely used for legacy M2M
- **EGPRS-2 (EDGE Evolution):** Up to 1.3 Mbps; rarely deployed
- **EC-GSM-IoT** (see §6) is a distinct NB technology layered on GSM spectrum

---

## 3. 3G / HSPA / WCDMA

> ⚠️ **Sunset Warning:** 3G WCDMA/HSPA networks are shut down or in process of shutdown
> in most major markets: T-Mobile US (2022), AT&T US (2022), Verizon US (2022), most EU
> operators by 2025. Spectrum refarmed for LTE and 5G. New IoT designs must not depend
> on 3G connectivity.

- **WCDMA (UMTS):** 5 MHz FDD; up to 384 kbps DPCH; circuit and packet modes
- **HSDPA (Rel-5):** Up to 14.4 Mbps DL
- **HSUPA (Rel-6):** Up to 5.76 Mbps UL
- **HSPA+ (Rel-7/8):** Up to 42 Mbps DL; MIMO; widely deployed
- **IoT relevance today:** Legacy 3G-only modules in deployed assets being replaced; use LTE Cat-1 bis for replacements

---

## 4. 4G LTE — Full Category Coverage

LTE (3GPP Rel-8, 2008) remains the dominant cellular IoT platform through at least 2030.
Network operators have committed to LTE maintenance through the mid-2030s in most markets.

**Core LTE architecture references:**
- eNB (eNodeB) — LTE base station; TS 36.300
- EPC (Evolved Packet Core): MME, S-GW, P-GW, HSS
- EPS bearer: default + dedicated; QCI-based QoS

### LTE Cat-1 and Cat-1 bis

**Cat-1 (Rel-8):**
- First LTE category designed for IoT; specified 2008 but gained IoT traction ~2014–2016
- Requires 2 Rx antennas — prevents compact device design; this was the design friction
- Available on virtually every LTE network globally; excellent roaming
- Still widely used in legacy deployments; do not recommend for new designs

**Cat-1 bis (Rel-13, 2016):**
- Single Rx antenna variant of Cat-1; same throughput (10/5 Mbps), same latency (<100ms)
- Resolves antenna ambiguity in original Cat-1 spec
- Same infrastructure, same frequencies — no network upgrade required
- Supports PSM and eDRX (added with Rel-12/13 features)
- Active market: major module vendors (Quectel, Telit, Fibocom, Sequans) have production parts
- **Primary 2G/3G migration target for mid-bandwidth IoT** (asset tracking, fleet, metering)
- eRedCap (Rel-18, 5G) is the long-term replacement; Cat-1 bis remains dominant through ~2028+

### LTE-M (eMTC) — Cat-M1 / Cat-M2

**Cat-M1 (eMTC, Rel-13):**
- Maximum channel BW: 1.4 MHz (operates within a 20 MHz LTE carrier via narrow-band scheduling)
- Coverage enhancement: up to 15.5 dB → MCL 155.7 dB; modes CE-A (≤32 repetitions) and CE-B (≤2048 repetitions)
- PSM + eDRX standard; battery lifetime measured in years
- **Full mobility**: seamless handover between cells — key differentiator vs NB-IoT
- **VoLTE support**: voice capability; used in medical alert devices, security panels, wearables
- Half-duplex FDD (HD-FDD) option to reduce modem complexity
- MPDCCH (Machine-type Physical Downlink Control Channel) for signalling efficiency
- 3GPP TS 36.331, 36.211, 36.212, 36.213; deployment spec TS 36.101

**Cat-M2 (Rel-14):**
- Extended BW to 5 MHz; up to 4 Mbps DL / 7 Mbps UL
- Maintains PSM/eDRX; adds TDD support
- Higher throughput for firmware OTA, video clips, audio recording applications

### NB-IoT — Cat-NB1 / Cat-NB2

**Cat-NB1 (Rel-13, 2016):**
- 200 kHz single narrowband channel (one LTE resource block)
- Half-duplex; no handover (stationary sensors only)
- MCL 164 dB (20 dB coverage improvement over GPRS)
- 3GPP TS 36.331 (RRC), 36.211/212/213 (PHY layer)

**Cat-NB2 (Rel-14, 2017):**
- Higher throughput: 127 kbps DL / 159 kbps UL
- **Multi-carrier mode**: up to 15 non-anchor carriers; supports 1 million devices/km²
- **Positioning**: OTDOA (Observed Time Difference Of Arrival) + E-CID (Enhanced Cell-ID) — location without GPS
- **14 dBm power class**: enables smaller batteries and ultra-compact form factors
- **Multicast (SC-PTM)**: broadcast firmware updates to groups of devices simultaneously — major FOTA enabler
- New FDD frequency bands added: 11, 21, 25, 31, 70

### NB-IoT Deployment Modes

Three radio deployment modes; operator and device must both support the chosen mode:

| Mode | Description | Spectrum | Notes |
|---|---|---|---|
| **In-band** | Uses LTE resource blocks within existing LTE carrier | Within LTE carrier | Most common; no additional spectrum needed |
| **Guard-band** | Uses the guard band of an LTE carrier | LTE guard band | No LTE PRB consumed; slightly different interference profile |
| **Standalone** | Dedicated 200 kHz spectrum | Can reuse GSM 200 kHz | GSM refarming path; enables NB-IoT where LTE unavailable |

Device must declare supported deployment modes in UE capabilities (TS 36.306).

### Coverage Enhancement (CE) Modes

Coverage extension via repeated transmissions; both LTE-M and NB-IoT use CE modes:

| Mode | Technology | Max Repetitions | CE Gain | MCL | Use Case |
|---|---|---|---|---|---|
| CE Mode A | LTE-M | 32 (DL+UL) | ~15.5 dB | 155.7 dB | Normal enhanced coverage |
| CE Mode B | LTE-M | 2048 (DL), 2048 (UL) | ~25 dB | ~164 dB | Deep indoor, basement, underground |
| CE Mode A | NB-IoT | 128 | ~15 dB | 144 dB | Standard in-building |
| CE Mode B | NB-IoT | 2048 | ~20 dB | 164 dB | Deep coverage; high latency trade-off |

**Trade-off:** Higher repetition count → better coverage, lower throughput, longer time-on-air, higher battery drain per message. CE mode is negotiated per-device per-cell dynamically.

---

## 5. Power Saving Mechanisms — All Generations

> These mechanisms are critical for IoT device firmware design and directly interact
> with LwM2M Queue Mode, registration lifetime, and DTLS session persistence.

### PSM — Power Saving Mode (Rel-12)

Device enters a deep sleep state, unreachable from network, for a negotiated period.

- **T3412 extended timer** (periodic TAU timer): how long the device may sleep between check-ins; configurable up to 310 hours
- **T3324 timer** (active timer): how long the device stays reachable after signalling before entering PSM
- Device context preserved at MME/AMF (no re-attach needed on wake)
- Network queues MT (mobile-terminated) traffic during PSM; delivers on wake
- **LwM2M impact:** Queue Mode (`b=UQ`) is the application-layer counterpart; DTLS session + CID must survive PSM sleep period (NVM persistence required)

### eDRX — Extended Discontinuous Reception (Rel-13)

Device wakes periodically to check for paging; shorter sleep cycles than PSM but device remains reachable.

| eDRX Cycle | Applicable to | Range | Notes |
|---|---|---|---|
| Short eDRX | LTE-M | 5.12 s – 2621.44 s | Page monitoring in paging hyperframes |
| Short eDRX | NB-IoT | 20.48 s – 2621.44 s | Wider cycles due to lower data rates |
| Extended eDRX (Rel-18) | RedCap, eRedCap | Extended cycles in RRC_INACTIVE | New capability beyond LTE |

### C-DRX — Connected-Mode DRX (Rel-8)

- Device sleeps during ON/OFF cycles while RRC_CONNECTED
- Cycle: 10–2560 ms; reduces active power while maintaining a data connection
- Configured by network via RRC signalling (RRCReconfiguration)

### WUS — Wake-Up Signal (Rel-15)

- Pre-paging signal sent before the main paging occasion
- Device wakes briefly to decode WUS; only fully wakes if WUS indicates pending page
- Reduces false wake-ups; saves 50–70% of paging monitoring power
- Supported in LTE-M and NB-IoT; specified in TS 36.211 (Physical signals)

### RRC_INACTIVE State (5G NR, Rel-15)

- Third RRC state between RRC_CONNECTED and RRC_IDLE
- UE context preserved at gNB (AS context suspended); fast resume (<10 ms) vs IDLE
- Dramatically reduces signalling overhead for frequent small-data transmissions
- Supported by RedCap and eRedCap
- 3GPP TS 38.331 §5.3.13

### Small Data Transmission (SDT, Rel-17)

- Enables uplink small data to be sent from RRC_INACTIVE without full RRC resume
- Two methods: RA-SDT (random access based) and CG-SDT (configured grant based)
- Critical for infrequent sensor reports — avoids full connection establishment overhead
- 3GPP TS 38.331, 38.321

### Timer Interaction with LwM2M Queue Mode

```
PSM Timeline:
  T3324 (Active Time: 20s)     T3412-ext (PSM: 4h)
  ◄──────────────────►◄────────────────────────────────►
  Device reachable    |         Device in PSM sleep
                      │
                      ├─ Server MUST buffer ops during PSM
                      ├─ Registration Lifetime > T3412-ext
                      └─ DTLS session + CID persisted to NVM
```

---

## 6. EC-GSM-IoT

> ⚠️ **Deployment Status: No commercial deployments as of 2026.** EC-GSM-IoT has been
> standardised but not operationally deployed by any major MNO. Treat as a legacy path
> for academic reference only; do NOT select for new designs.

- **3GPP Release 13** (2016); specified in TS 45.820, TS 45.002
- Extended Coverage GSM for IoT; 33 dB MCL improvement over standard GSM (MCL 164 dB)
- Operates on existing GSM 900 MHz spectrum; reuses GSM base station hardware with software upgrade
- Half-duplex; PSM and eDRX supported
- Competes with NB-IoT and LTE-M on the same spectrum refarming path; NB-IoT/LTE-M have won commercially

---

## 7. 5G NR — Full IoT Coverage

5G NR (New Radio, 3GPP Rel-15+) encompasses a broad range of IoT capabilities extending well beyond the eMBB/URLLC original focus.

**Frequency ranges:**
- **FR1 (sub-6 GHz):** 410 MHz – 7125 MHz; primary range for IoT (coverage, penetration)
- **FR2 (mmWave):** 24.25 GHz – 52.6 GHz; high capacity, short range; RedCap supports FR2 at 100 MHz BW

**5G NR IoT-relevant specs:** TS 38.300 (architecture), TS 38.304 (idle mode), TS 38.331 (RRC), TS 38.306 (UE capability)

### 5G CIoT Optimisations

5G-specific cellular IoT optimisations (3GPP Rel-16+, TS 23.501 §5.31):

| Feature | 5G Equivalent | Notes |
|---|---|---|
| PSM | 5GC PSM via AMF (vs EPC MME) | Same concept; different timers/procedures |
| eDRX | 5G eDRX | Similar to LTE; extended in Rel-17/18 for RedCap |
| Control Plane CIoT optimisation | CP optimisation | Data in NAS signalling; no user plane bearer |
| User Plane CIoT optimisation | UP optimisation | Lightweight suspended bearer |
| NIDD | 5G NIDD via NEF | Non-IP Data Delivery; no IP stack on device |
| Small Data Transmission | SDT (Rel-17) | From RRC_INACTIVE; major battery win |
| RRC_INACTIVE | 5G NR native | Context preserved; fast resume |

**Control Plane vs User Plane CIoT:**
```
Control Plane (CP):   Device → NAS → AMF → NEF/AS
                      No PDU session; data in signalling; lowest overhead

User Plane (UP):      Device → gNB → UPF → DN
                      Lightweight PDU session; suspended between data bursts
```

### RedCap — 3GPP Rel-17

**NR Reduced Capability (TS 38.306, 38.101-1, 38.211–38.215)**

RedCap fills the mid-tier connectivity gap between LTE Cat-3/4 and full 5G NR eMBB.
It requires 5G SA (Standalone) architecture — no EPC fallback.

**Key simplifications vs standard NR:**

| Parameter | Full NR (FR1) | RedCap (FR1) |
|---|---|---|
| Max DL bandwidth | 100 MHz | **20 MHz** |
| Max DL MIMO layers | 4–8 | **2** |
| Min Rx antennas | 2 | **1 or 2** |
| Carrier aggregation | ✅ | **❌** |
| Dual connectivity | ✅ | **❌** |
| Max UL modulation | 256-QAM | **64-QAM mandatory** |
| Half-duplex FDD | Optional | **Optional** |
| Peak DL rate | ~1 Gbps+ | **≤226 Mbps** |
| Peak UL rate | ~500 Mbps | **≤120 Mbps** |

**Power saving features:**
- eDRX in RRC_IDLE and RRC_INACTIVE (optional; network configures extended cycles)
- RRM measurement relaxation for neighbour cells in idle/inactive
- Wake-Up Signal (WUS) support
- Small Data Transmission (SDT) from RRC_INACTIVE

**Target use cases and data rate requirements:**

| Use Case | DL Req | UL Req | Latency | Battery |
|---|---|---|---|---|
| Wearables (smartwatch, AR glass) | ≤150 Mbps | ≤50 Mbps | <100 ms | Multi-day |
| Industrial wireless sensors | ≤2 Mbps | ≤2 Mbps | <100 ms | Years |
| Video surveillance cameras | ≤25 Mbps | ≤5 Mbps | Best effort | Powered |

**Deployment status (Q1 2026):** 30 operators across 21 countries investing in RedCap;
commercial launches by China Mobile, China Telecom, China Unicom, STC (Kuwait),
T-Mobile US. Module ecosystem: Qualcomm, MediaTek, ASR, Sequans chipsets;
Fibocom, Quectel, Telit Cinterion, MeiG Smart modules. GCF certification underway.

### eRedCap — 3GPP Rel-18 (5G Advanced)

**Enhanced RedCap; part of 5G Advanced (TS 38.306 Rel-18)**

eRedCap is a **distinct device type** (not an evolution of RedCap); created by further
capping the peak rate to 10 Mbps — matching LTE Cat-1/Cat-1 bis performance on a 5G SA network.

**Key differences vs RedCap:**

| Parameter | RedCap (Rel-17) | eRedCap (Rel-18) |
|---|---|---|
| Peak DL rate | ≤226 Mbps | **≤10 Mbps** |
| Peak UL rate | ≤120 Mbps | **≤10 Mbps** |
| FR2 (mmWave) support | ✅ | **❌ (FR1 only)** |
| Single Rx antenna | Optional | **Expected baseline** |
| Carrier aggregation | ❌ | ❌ |
| eDRX in RRC_INACTIVE | Optional | **Enhanced (longer cycles)** |
| Target replacement | LTE Cat-3/4 | **LTE Cat-1/Cat-1 bis** |

**Market timeline:** Commercial module availability expected 2026+; first major deployments 2027.

**Rel-19 roadmap:** Wake-Up Receiver (WUR) — dedicated low-power radio listens for
wake-up signal while main radio sleeps; potential 10× additional power reduction.

### 5G TSN Integration

**3GPP Rel-16; TS 23.501 §5.27, TS 23.502, TR 22.804**

5G NR acting as an IEEE 802.1Q-compliant TSN bridge — enables replacement of
wired deterministic Ethernet with wireless 5G in factories and industrial environments.

- **5G system as TSN bridge:** UE + 5GS + UPF form a logical TSN bridge per IEEE 802.1Qcc
- **gPTP distribution:** IEEE 802.1AS time synchronisation propagated over 5G air interface; 5GC distributes grand master clock reference
- **Scheduled traffic (802.1Qbv):** Time-aware shaper implemented at DS-TT (Device-Side TSN Translator, in UE) and NW-TT (Network-Side TSN Translator, in UPF)
- **Bounded latency:** Sub-5 ms deterministic latency possible for factory-floor applications
- **URLLC + TSN:** Complements URLLC (Ultra-Reliable Low Latency Communications) Rel-16 enhancements
- Spec refs: TS 23.501, TS 23.502, TR 22.804 (industrial IoT requirements)

### 5G LAN-Type Service

**3GPP Rel-16; TS 23.501 §5.29**

Device-to-device communication via 5GC without traversing the public internet.

- Devices in the same 5G LAN Virtual Network (VN) group communicate directly via UPF
- No cloud round-trip for factory floor D2D communication
- Group management via 5G LAN service API; NEF exposure
- Multicast within 5G LAN group
- Used in: robotic arm coordination, AGV (Automated Guided Vehicle) control, machine-to-machine on factory floor

### Private Networks (SNPN / PNI-NPN)

**3GPP Rel-16/17; TS 23.501 §5.30, TS 22.261**

| Type | Description | Core Network | Spectrum |
|---|---|---|---|
| **SNPN** (Stand-alone Non-Public Network) | Independent of any PLMN; own 5GC | Private | Licensed (CBRS, sXGP, mmWave) |
| **PNI-NPN** (Public Network Integrated NPN) | Slice within a PLMN; shared RAN possible | Shared or private | Operator's licensed spectrum |

- SNPN: full enterprise control; used in automotive factories, ports, mining
- PNI-NPN: simpler deployment; depends on MNO; used in campus/hospital IoT
- RedCap devices are particularly well-suited for private 5G deployments

---

## 8. 3GPP SCEF / NEF — IoT Exposure APIs

The Service Capability Exposure Function (SCEF, EPC/Rel-13) and Network Exposure Function (NEF, 5GC/Rel-15+) provide IoT platform integration APIs. NEF is the 5G successor to SCEF.

### API Surface

| API | SCEF (4G) | NEF (5G) | Description |
|---|---|---|---|
| Monitoring Events | T8 (TS 29.122) | Nnef_EventExposure | Subscribe to device state events |
| Device Triggering | T8 | Nnef_Trigger | MT trigger to wake sleeping device |
| NIDD | T8 | Nnef_NIDD | Non-IP Data Delivery; small data without IP stack |
| Background Data Transfer | T8 | Nnef_BDT | Negotiate upload time windows for bulk IoT |
| Resource Management | T8 | Nnef_ResourceManagement | QoS resource requests |

### Monitoring Events

Subscribe via SCEF/NEF to receive notifications when:
- **Loss of connectivity**: device unreachable (PSM entry, SIM removal, power loss)
- **UE reachability**: device wakes from PSM / becomes reachable
- **Location reporting**: cell-level or more granular (requires location services)
- **Roaming status**: device enters/leaves home network
- **PDN connectivity status**: data session established/released
- **Communication failure**: repeated delivery failures

### Device Triggering

- MT-SMS based mechanism to signal a sleeping device to wake and initiate communication
- Platform sends trigger via T8/Nnef → SCEF/NEF → MME/AMF → device
- Useful for PSM devices where server has no direct way to initiate contact
- 3GPP TS 23.682, TS 29.122

### NIDD — Non-IP Data Delivery

```
Device (no IP stack)  →  NAS/SCEF  →  NEF  →  AS/Platform
                         (T8 API)
```
- Device transmits raw bytes; no IPv4/IPv6 required on device
- Maximum payload: ~1600 bytes per NAS message (network-dependent)
- Ideal for ultra-constrained devices; eliminates TCP/IP stack footprint
- 3GPP TS 23.682, 24.250

---

## 9. V2X Protocols

### IEEE 802.11p / WAVE / DSRC

> ⚠️ **US Status:** FCC reallocated the upper 30 MHz of the 5.9 GHz DSRC band to
> Wi-Fi 6E in 2020; only 30 MHz (5.850–5.895 GHz) retained for ITS. 802.11p/DSRC
> deployments in North America are effectively abandoned in favour of C-V2X.
> **EU/ETSI ITS-G5 (ETSI EN 302 663)** continues use of 802.11p in Europe.

- IEEE 802.11p: PHY/MAC for 5.9 GHz vehicular DSRC; 10 MHz channel; OFDM
- **WAVE stack** (US): IEEE 1609.1/2/3/4 — WSMP protocol, security (1609.2), multi-channel (1609.4)
- **ETSI ITS-G5** (EU): equivalent DSRC stack; GeoNetworking (EN 302 636-4); BTP (EN 302 636-5)
- Latency: <5 ms (no infrastructure required — direct V2V/V2I)

### C-V2X — Cellular V2X (3GPP Rel-14+)

**The primary V2X path for new deployments:**

| Interface | Description | Range | Infrastructure |
|---|---|---|---|
| **PC5 (sidelink)** | Direct D2D; no network | ≤500 m | None required |
| **Uu (network)** | Via cellular base station | Network range | LTE/5G required |

**LTE-V2X (Rel-14):** PC5 Mode 4 (autonomous resource selection for no-infrastructure); Uu Mode 3/4; 10 MHz @ 5.9 GHz
**NR-V2X (Rel-16):**
- PC5 Mode 1 (network scheduled) and Mode 2 (autonomous) for NR sidelink
- Enhanced reliability: HARQ, CBR/CR congestion control
- New use cases: sensor sharing (Collective Perception), remote driving, extended sensors (3GPP TS 22.186)
- Higher throughput and lower latency vs LTE-V2X

### SAE Application Layer

| Standard | Description |
|---|---|
| **SAE J2735** | V2X message set dictionary: BSM (Basic Safety Message), SPaT (Signal Phase and Timing), MAP (intersection geometry), TIM (Traveller Information) |
| **SAE J2945/1** | OBE (On-Board Equipment) performance requirements for V2V safety |
| **SAE J3161** | Pedestrian safety message (PSM) |
| **ETSI EN 302 637-2** | CAM (Cooperative Awareness Message) — European equivalent of BSM |
| **ETSI EN 302 637-3** | DENM (Decentralised Environmental Notification Message) — hazard alerts |

---

## 10. Satellite IoT / NTN

### Orbit Classifications

| Orbit | Altitude | Latency | Doppler | Examples |
|---|---|---|---|---|
| LEO | 300–1200 km | 10–40 ms | High (must compensate) | Starlink, AST SpaceMobile, Orbcomm |
| MEO | 8000–20000 km | 100–300 ms | Moderate | O3b/SES |
| GEO | 35786 km | ~600 ms | Minimal | Inmarsat, Viasat |

### Legacy Satellite IoT

- **Iridium SBD (Short Burst Data):** 9600 bps; 340-byte MO/MT payloads; global pole-to-pole LEO coverage; ideal for remote asset tracking; AT command interface (ISU) on devices
- **Inmarsat BGAN M2M:** GEO; IP-based; higher throughput (up to 384 kbps); maritime/aviation
- **Globalstar SPOT/Simplex:** Simplex uplink only (no downlink ACK); asset tracking
- **Orbcomm:** LEO; store-and-forward; industrial fleet management

### 3GPP NTN — Non-Terrestrial Networks

**3GPP Rel-17 (study + normative work); TS 38.821, TS 36.763**

NTN integrates satellite connectivity directly into the 3GPP standard stack:

**NR NTN (Rel-17, TS 38.821):**
- Standard 5G NR air interface adapted for satellite propagation
- Key challenges addressed: large timing advance (TA) up to 600 ms (GEO), Doppler shift (LEO up to ±24 ppm at 2 GHz), long RTT
- Solutions: extended TA range in RRC; UE-based Doppler pre-compensation using GNSS ephemeris; HARQ process adaptation
- Supports FR1 bands; transparent (bent-pipe) and regenerative satellite payloads
- Enables global IoT without terrestrial coverage gaps

**NB-IoT/eMTC over NTN (Rel-17, TS 36.763):**
- NB-IoT and LTE-M adapted for LEO/GEO satellite operation
- Extended timing advance; eDRX adaptation for longer orbital periods
- Targets: remote agricultural sensors, maritime containers, pipeline monitoring

**Rel-18 eNTN:**
- Coverage enhancement for NB-IoT/LTE-M over NTN
- Service continuity between terrestrial and NTN (seamless handoff)
- Reduced UE complexity for satellite-only devices
- IoT-specific enhancements: longer sleep cycles matched to satellite visibility windows

### Direct-to-Device Satellite (Non-3GPP)

Distinct from NTN — leverages existing device LTE/NR modems without modification:

| Service | Provider | Technology | Capability |
|---|---|---|---|
| Direct-to-Cell | SpaceX Starlink | NR (modified) | SMS, data; standard LTE-M handsets |
| BlueBird | AST SpaceMobile | NR | Voice + data on unmodified smartphones |
| Cell of the Future | Lynk | LTE | SMS to standard phones |

**IoT relevance:** These services can extend NB-IoT/LTE-M coverage to genuinely global
reach without custom satellite modems — significant for asset tracking in remote areas.

---

## Key Specification References

| Technology | Primary 3GPP TS | Notes |
|---|---|---|
| LTE Cat-1 bis | TS 36.306, 36.331 | UE capability, RRC |
| LTE-M (Cat-M1/M2) | TS 36.300, 36.211–36.214 | Architecture + PHY |
| NB-IoT (Cat-NB1/NB2) | TS 36.300, 36.211–36.214 | Same PHY layer docs |
| PSM / eDRX | TS 23.682, 24.008 | Core network procedures |
| SDT | TS 38.331, 38.321 | NR RRC + MAC |
| RedCap | TS 38.306, 38.331, 38.101-1 | UE cap, RRC, RF |
| eRedCap | TS 38.306 (Rel-18) | UE capability extensions |
| SCEF / T8 API | TS 29.122, 23.682 | API definition |
| NEF / Nnef | TS 23.501, 29.522 | 5GC exposure |
| 5G TSN | TS 23.501 §5.27, TR 22.804 | Architecture + requirements |
| SNPN / PNI-NPN | TS 23.501 §5.30, TS 22.261 | Non-public networks |
| NTN | TS 38.821, 36.763 | Satellite integration |
| NR-V2X | TS 23.285, 38.885, 22.186 | Architecture + requirements |

---

*Cross-references:*
- *LPWAN (LoRaWAN, Sigfox, SCHC): `references/lpwan.md`*
- *3GPP release structure, NAS, 5GC core architecture: [3GPP skill](https://github.com/lugasia/3gpp-skill)*
- *LwM2M Queue Mode + PSM/CID interaction: [LwM2M skill](https://github.com/svdwalt007/Best-LwM2M-Agentic-Skills)*
