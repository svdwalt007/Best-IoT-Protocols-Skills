# Security & Provisioning Protocols — Complete Reference

## Overview
IoT security and provisioning protocols span from initial device manufacturing through credential management, secure communication channels, firmware updates, and regulatory compliance. This reference covers the complete security stack for IoT deployments.

---

## Transport Security
### DTLS (Datagram Transport Layer Security)
DTLS provides TLS security over UDP — the primary security protocol for CoAP, LwM2M, and MQTT-SN.

**DTLS 1.2 (RFC 6347):**
- Based on TLS 1.2; adds datagram-specific: cookie exchange (DoS prevention), sequence numbers, retransmission, epoch
- Cipher suites: AES-128-CCM, AES-256-GCM, ChaCha20-Poly1305
- Authentication: PSK (Pre-Shared Key), RPK (Raw Public Key), x.509 certificates
- Handshake: Flight-based; ClientHello → ServerHello → Certificate → CertVerify → Finished

**DTLS 1.3 (RFC 9147):**
- Based on TLS 1.3; shorter handshake (1-RTT), stronger forward secrecy
- Eliminates: RSA key exchange, CBC cipher modes, MD5/SHA-1, renegotiation, compression
- AEAD only: AES-128-GCM, AES-256-GCM, ChaCha20-Poly1305
- Epoch-based key updates; Connection ID natively integrated

### DTLS Connection ID (RFC 9146)
**CRITICAL INTEROP NOTE:**
- **Extension type 54** = RFC 9146 (ratified March 2022) — the correct standard
- **Extension type 53** = `draft-ietf-tls-dtls-connection-id` (obsolete IETF draft)
- Mixing type 53 and type 54 implementations causes **silent session failures** — no error reported; DTLS simply cannot resume
- `tls12_cid` ContentType = **25** (decimal)
- Always verify in Wireshark: `dtls.handshake.extension.type == 54`

**Purpose:** Allows DTLS sessions to survive IP address and port changes:
```
Without CID: NAT rebinding → new IP:port → Server sends to old address → session broken
With CID: CID in record header → Server routes to session regardless of IP:port
```

**CID in Record Header:**
```
ContentType: 25 (tls12_cid)
Connection ID: [8 bytes negotiated during handshake]
Epoch: [2 bytes]
Sequence Number: [6 bytes]
Length: [2 bytes]
Encrypted Data: [...]
```

**NB-IoT context:** Devices wake from PSM → IP address may change → CID allows DTLS session resumption without full handshake (saves battery, avoids re-registration with LwM2M server)

### DTLS RRC — Return Routability Check (RFC 9853)
New RFC (2024) for DTLS 1.3 addressing address-migration security:
- **Problem:** CID allows server to follow address changes — but what if the change is spoofed?
- **Solution:** On address change, DTLS 1.3 peer sends PathChallenge; peer must respond with PathResponse
- **Flow:**
```
Server detects peer IP:port change
Server → Client: PathChallenge (random 8-byte token)
Client → Server: PathResponse (same token)
Server: validates response → confirms legitimate address change
Server: updates routing table to new IP:port
```
- **Without RRC:** Attacker could inject packets with spoofed source, redirect server responses
- **With RRC:** Attacker cannot respond to PathChallenge → attack fails

### TLS 1.3 (RFC 8446)
Key improvements over TLS 1.2 for IoT:
- **1-RTT handshake** (vs 2-RTT in TLS 1.2): Faster connection establishment
- **0-RTT resumption:** For reconnecting devices; send data in first flight (replay risk — evaluate for IoT)
- **Forward secrecy:** Mandatory ECDHE; compromised long-term key can't decrypt past sessions
- **Removed:** RSA key exchange, CBC mode, MD5/SHA-1, renegotiation, compression
- **Session tickets:** Resume sessions without full handshake
- **ALPN (Application-Layer Protocol Negotiation):** IoT use: `coap`, `mqtt`

---

## Application Security
### OSCORE (RFC 8613)
Object Security for CoAP — end-to-end application-layer security:
- **Independent of transport:** Security survives CoAP proxies, protocol translators
- **Based on COSE (CBOR Object Signing and Encryption)**
- **Algorithm:** AES-128-CCM-8 (default); AES-256-GCM optional
- **Context:** Shared security context (Master Secret, Salt, Sender/Recipient IDs)
- **Protected fields:** Method, URI, payload → encrypted; Proxy-Uri, Proxy-Scheme → unprotected
- **Sequence numbers:** Partial IV prevents replay attacks

**OSCORE context derivation:**
```
Common context: Master Secret, Master Salt, Algorithm (AES-128-CCM-8 default), KDF (HKDF-SHA256)
Sender context: Sender ID, Sender Key (derived), Sender IV, Sequence number
Recipient context: Recipient ID, Recipient Key (derived), Replay window
```

### Group OSCORE (RFC 9203)
OSCORE extended to multicast CoAP groups:
- **Group Manager:** Manages group membership, key distribution
- **Shared group key:** All members share group sending context
- **Individual context:** Each member has individual Sender Key for authentication
- **Counter Signature:** COSE_Sign1 added to Group OSCORE messages for source authentication
- **Use cases:** LwM2M multicast Bootstrap, smart lighting group commands, AMI meter groups

### EDHOC (RFC 9528)
Ephemeral Diffie-Hellman Over COSE — lightweight authenticated key exchange:
- **Purpose:** Establishes OSCORE security context with 3-message exchange (vs DTLS full handshake)
- **Designed for:** Constrained devices (NB-IoT, BLE, LoRaWAN)
- **Authentication methods:** Method 0 (SIGN_SIGN), Method 1 (SIGN_STAT), Method 2 (STAT_SIGN), Method 3 (STAT_STAT)
- **Credential types:** X.509 certificates, CWT (CBOR Web Token), RPK
- **Message sizes:** 39-74 bytes (vs DTLS handshake 1000+ bytes)

**EDHOC + OSCORE composition:**
```
Initiator → Responder: message_1 (METHOD, SUITES_I, G_X, C_I)
Initiator ← Responder: message_2 (G_Y, C_R, CIPHERTEXT_2)
Initiator → Responder: message_3 (CIPHERTEXT_3)
# OSCORE security context now established from EDHOC keys
# Use EDHOC connection identifier → OSCORE Sender/Recipient IDs
```
- **Flag:** EDHOC+OSCORE composition profiles still being finalised; recommend web search for current status

---

## Authorization Framework
### ACE (RFC 9200)
Authentication and Authorization for Constrained Environments — OAuth 2.0 for IoT:

**Roles:**
- **Authorization Server (AS):** Issues Access Tokens; verifies client credentials
- **Resource Server (RS):** Hosts protected resources (e.g., CoAP sensor server)
- **Client (C):** Device or app requesting resource access

**Flow:**
```
Client → AS: POST /token (grant_type, client_credentials, scope)
AS → Client: Access Token (CWT or JWT) + RS information
Client → RS: GET /resource + Access Token (in payload or header)
RS → AS: (optional) introspect token → POST /introspect
RS → Client: 2.05 Content (resource data)
```

**Profiles:**
- **DTLS Profile (RFC 9202):** Token in DTLS ClientHello extension; server-side verification
- **OSCORE Profile (RFC 9203):** Token exchange bootstraps OSCORE context
- **CoAP Transport Security Profile (RFC 9200 §3.3):** TLS/DTLS/OSCORE as transport

---

## COSE and CBOR Security
### COSE (RFC 9052/9053)
CBOR Object Signing and Encryption — CBOR equivalent of JOSE (JWS/JWE/JWK):

| COSE Structure | Purpose | JOSE Equivalent |
|---------------|---------|-----------------|
| COSE_Sign1 | Single-signer signature | JWS compact serialization |
| COSE_Sign | Multi-signer signature | JWS JSON serialization |
| COSE_Encrypt0 | Single-recipient encryption | JWE compact |
| COSE_Encrypt | Multi-recipient encryption | JWE JSON |
| COSE_Mac0 | Single-recipient MAC | — |
| COSE_Mac | Multi-recipient MAC | — |
| COSE_Key | Cryptographic key | JWK |

**Used in:** EDHOC (key material), ACE (access tokens as CWT), SUIT manifest (firmware signatures), LwM2M Object 25 (COSE sign object), Group OSCORE

---

## Firmware Update Security
### SUIT Manifest (RFC 9019/9124/9397)
IETF standard for authenticated, integrity-protected firmware update manifests:

**RFC 9019:** SUIT architecture overview
**RFC 9124:** SUIT manifest information model
**RFC 9397:** SUIT manifest CBOR format (the wire format)

**Manifest structure:**
```
SUIT Manifest:
  manifest-version: 1
  manifest-sequence-number: <monotonic counter, prevents rollback>
  common:
    dependencies: [<component references>]
    components: [<component IDs>]
    shared-sequence: <conditions and directives>
  install: <installation procedure>
  validate: <post-install verification>
  invoke: <execution procedure>
  candidate-verification: <pre-install digest check>
  # Outer wrapper (signed):
  authentication-wrapper: COSE_Sign1 / COSE_Mac0
```

**Key security features:**
- **COSE_Sign1 signature:** Authenticates manifest source
- **digest:** SHA-256 of firmware image; ensures integrity
- **sequence-number:** Monotonically increasing; prevents rollback to older firmware
- **Component IDs:** Multi-component update (OS + application independently)

**Transport pairings:**
- LwM2M Object 5 (Firmware Update) + SUIT manifest: cross-reference LwM2M skill
- LoRaWAN FUOTA (TS004) + RFC 9011 + SUIT: flag — profiling in progress
- TEEP (RFC 9397): SUIT for trusted execution environments

---

## eSIM / Remote SIM Provisioning
### SGP.32 (v1.2, June 2024) — IoT eSIM Standard
The current standard for IoT eSIM management, replacing SMS-based SGP.02 (M2M legacy).

**Key Innovation:** IP-based communication (CoAP/DTLS or HTTPS) instead of SMS — compatible with NB-IoT, LTE-M, LoRaWAN where SMS is unavailable or expensive.

**Architecture Components:**

| Component | Role | Implementation |
|-----------|------|----------------|
| eIM (eSIM IoT Remote Manager) | Fleet-level profile lifecycle management (download, enable, disable, delete) | Managed service (e.g., Thales, Kigen) |
| IPA (IoT Profile Assistant) | Device-side agent acting on eIM commands | IPAd or IPAe |
| SM-DP+ | Encrypted profile preparation and delivery | MNO / GSMA SGP.22 service |
| eUICC | Secure element storing/executing profiles | SIM vendor silicon |
| ESipa | Interface between eIM and IPA | HTTPS, CoAP/DTLS, or LwM2M |

**IPAd vs IPAe:**
- **IPAd (d = device):** Runs in device OS or application processor; OEM-implemented; communicates with eUICC via standard APDU interface; more flexible but requires OEM development
- **IPAe (e = eUICC):** Runs inside the eUICC secure element; silicon vendor-implemented (e.g., Thales MultiSIM); more secure but less flexible

**ESipa Transport Options:**
```
eIM ←→ IPAd: HTTPS/TLS or CoAP/DTLS (device OS carries the protocol)
eIM ←→ IPAe: LwM2M Object 3443 (ASN.1 pipe relayed via device OS)
```

**LwM2M Objects for SGP.32:**
*Object 504 (RSP — Remote SIM Provisioning):* 21 resources covering complete eSIM lifecycle:
- Resource 0: eSIM State
- Resource 1: Operator ICCID
- Resource 2: eIM URL
- Resource 4: RSP Update Execute (triggers profile operation)
- Resource 7: Event records list
- Resource 10: Current Operator profile

*Object 3443 (eSIM IoT):* Bidirectional ASN.1 message pipe between eIM and IPA (SGP.32 Annex B.1):
- Used when LwM2M is the transport for ESipa
- Encapsulates SGP.32 ASN.1 messages inside LwM2M Execute operations
- Enables fully LwM2M-managed eSIM lifecycle

**⚠️ Object 146 Misconception:** LwM2M Object 146 is NOT an eSIM object. The number 146 appears:
1. As a page number in the SGP.32 specification document
2. As GSMA's ISO OID namespace identifier
Never cite Object 146 as an eSIM LwM2M object.

### SIM OTA / BIP (Bearer Independent Protocol)
**Reference**: GSMA TS.02/TS.03 (SIM OTA), ETSI TS 102 127 (BIP), ETSI TS 102 225 (SCP80/81/02), ETSI TS 102 223 (CAT)

**Purpose**: Remote provisioning and management of SIM card applications (Java Card applets, file system updates) on deployed M2M SIMs — predates eSIM/SGP.32 but still deployed on millions of M2M devices

**Two Delivery Mechanisms**:

#### 1. SMS-PP OTA (SMS Point-to-Point)
**Reference**: ETSI TS 102 225 (SCP80), 3GPP TS 23.040/31.115

**Architecture**:
```
OTA Platform                   SMSC                    Device/SIM
┌──────────────┐             ┌──────┐               ┌────────────┐
│ OTA Server   │ SMS-MT      │ SMS  │   SMS-PP      │ SIM Card   │
│  (UICC-OTA)  ├────────────>│Center├──────────────>│ (JavaCard) │
│              │             └──────┘               │            │
│ Key Mgmt     │                                    │ Applet     │
│ (KIC/KID)    │             ┌──────┐               │ Processing │
│              │<────────────┤ SMS  │<──────────────┤ (CAT)      │
└──────────────┘ SMS-MO      └──────┘   Response    └────────────┘
```

**SMS-PP APDU Structure**:
```
SMS TPDU (TP-User-Data):
  ├─ SPI (Security Parameter Indicator): 2 bytes
  │  └─ Defines: Command type, cipher/MAC algorithm, key set
  ├─ TAR (Toolkit Application Reference): 3 bytes
  │  └─ Identifies target applet on SIM (e.g., 0xB00000 for Java Card applet)
  ├─ CNTR (Counter): 5 bytes
  │  └─ Anti-replay protection; incremented for each command
  ├─ PCNTR (Padding Counter): 1 byte
  │  └─ Length of padding to align to block size
  ├─ RC/CC/DS (Response/Ciphering/Digital Signature): Variable
  │  ├─ RC (Response Check): MAC over response (if requested)
  │  ├─ CC (Cryptographic Checksum): MAC over command (integrity)
  │  └─ DS (Digital Signature): Optional RSA signature
  └─ Secured Data: Encrypted command payload (DES/3DES/AES-128)

Security Levels (SPI):
  ├─ No security (0x00): Plaintext, no MAC (testing only)
  ├─ RC (Response Check): MAC on response
  ├─ CC (Command Check): MAC on command
  └─ RC+CC+Ciphering: Full security (encrypt+MAC command and response)
```

**SCP80 Key Hierarchy**:
```
Master Key (per SIM, provisioned at manufacturing)
  ├─ KIC (Key for Integrity and Ciphering)
  │  └─ Derives session keys for command encryption (DES/3DES/AES-128)
  └─ KID (Key for Integrity and Deciphering)
     └─ Derives session keys for MAC calculation (DES-MAC, AES-CMAC)

Key Derivation (SCP80):
  Session Key = KDF(Master Key, CNTR)
  - CNTR increments for each OTA session
  - Prevents key reuse; forward secrecy per session
```

**SMS-PP OTA Flow**:
```
1. OTA Server constructs secured APDU:
   - Plaintext command (e.g., INSTALL [for load] Java Card applet)
   - Encrypt with KIC session key (3DES-CBC or AES-128)
   - Calculate MAC with KID session key (DES-MAC-8 or AES-CMAC)
   - Assemble: SPI || TAR || CNTR || CC || Encrypted_Data

2. OTA Server → SMSC: SMS-SUBMIT (TP-User-Data = Secured APDU)

3. SMSC → Device: SMS-DELIVER (binary SMS, PID=0x7F for SIM data download)

4. Device receives SMS → routes to SIM via SMS-PP envelope command

5. SIM processes:
   - Verify TAR (is this command for me?)
   - Verify CNTR (is this fresher than last command? anti-replay)
   - Decrypt command data with KIC
   - Verify MAC with KID
   - Execute command (applet install, file update, APDU execution)

6. SIM generates response:
   - Response status (0x9000 success, 0x6xxx error)
   - Encrypt response with KIC
   - Calculate MAC with KID
   - Response APDU: SPI || TAR || CNTR || RC || Encrypted_Response

7. Device → SMSC → OTA Server: SMS-MO (response APDU)

8. OTA Server verifies response MAC, decrypts, confirms success
```

**Limitations**:
- **Throughput**: SMS limited to 140 bytes per message; large applets require fragmentation (100+ SMS for typical applet)
- **Latency**: SMS delivery not guaranteed real-time; minutes to hours
- **Cost**: SMS charges per message (expensive for large updates)
- **Coverage**: Requires SMS service (not available in some NB-IoT networks with NIDD-only)

#### 2. BIP OTA (Bearer Independent Protocol over GPRS/LTE)
**Reference**: ETSI TS 102 127 (BIP), 3GPP TS 31.111 (CAT — Card Application Toolkit), ETSI TS 102 225 (SCP81)

**Purpose**: High-throughput OTA over IP bearer (GPRS, LTE, NB-IoT) — faster and cheaper than SMS

**Architecture**:
```
OTA Platform                  TCP/IP Network           Device/SIM
┌──────────────┐             ┌──────────────┐        ┌────────────┐
│ OTA Server   │ TCP/IP      │ Packet Core  │ BIP    │ SIM Card   │
│  (BIP-OTA)   ├────────────>│ (GGSN/PGW)   ├───────>│ (JavaCard) │
│              │             └──────────────┘ Channel │            │
│ SCP81 Keys   │                                      │ Proactive  │
│ (KIC/KID)    │                                      │ SIM (CAT)  │
└──────────────┘                                      └────────────┘
```

**BIP Proactive Commands** (SIM → Device, via CAT):
```
1. OPEN CHANNEL (0x40)
   - SIM instructs device to open TCP/IP channel to OTA server
   - Parameters: Destination IP, Port, Bearer (GPRS/LTE), Buffer size
   - Device response: Channel ID (1-7, max 7 concurrent channels)

2. SEND DATA (0x43)
   - SIM sends data over open channel
   - Payload: Secured APDU (SCP81-encrypted command)
   - Device forwards data via TCP socket to OTA server

3. RECEIVE DATA (0x42)
   - SIM requests data from open channel
   - Device fetches data from TCP socket
   - Response: Secured APDU from OTA server (encrypted response)

4. GET CHANNEL STATUS (0x44)
   - SIM queries channel state (open, closed, bytes available)

5. CLOSE CHANNEL (0x41)
   - SIM closes TCP channel when session complete
```

**SCP81 Security** (BIP variant of SCP80):
```
Differences from SCP80:
  - Optimized for packet data (larger MTU, no SMS fragmentation)
  - Session-based: Single OPEN CHANNEL → multiple commands → CLOSE CHANNEL
  - Same key hierarchy: KIC/KID derived from Master Key
  - MAC calculation: AES-CMAC (SCP81) vs DES-MAC (SCP80)
  - Counter: Session counter (per BIP session) vs SMS counter

Advantages:
  - Throughput: 10-100x faster than SMS (full TCP bandwidth)
  - Cost: Data cost << SMS cost for large transfers
  - Reliability: TCP ensures in-order delivery, retransmission
```

**BIP OTA Flow**:
```
1. OTA Server triggers BIP session (via SMS or previous BIP session)

2. SIM issues OPEN CHANNEL proactive command:
   Device → SIM: TERMINAL RESPONSE (channel ID = 1)

3. SIM issues SEND DATA:
   - SIM constructs secured APDU (SCP81: SPI || TAR || CNTR || CC || Encrypted_Command)
   - Device sends data over TCP to OTA server

4. OTA Server processes command, responds via TCP

5. SIM issues RECEIVE DATA:
   - Device fetches response from TCP socket
   - SIM decrypts, verifies MAC, executes command

6. Repeat SEND DATA / RECEIVE DATA for multi-command session

7. SIM issues CLOSE CHANNEL when complete

8. OTA Server confirms all commands executed successfully
```

**BIP Use Cases**:
```
1. Java Card Applet Download (100-500 KB)
   - SMS OTA: 1000+ SMS messages → hours, expensive
   - BIP OTA: 10-30 seconds → single data session, cheap

2. SIM File System Update (EF_SMS, EF_MSISDN, phonebook)
   - Bulk updates to SIM file system
   - BIP allows atomic multi-file updates in single session

3. SIM Application Management (Install, Delete, Lock applets)
   - GP (GlobalPlatform) INSTALL/DELETE/LOCK commands
   - BIP enables full lifecycle management

4. OTA Key Rotation
   - Update KIC/KID keys remotely
   - SCP81 key diversification commands
```

**CAT (Card Application Toolkit) Integration**:
```
ETSI TS 102 223 — Proactive SIM interface:
  - SET UP CALL: SIM-initiated voice call (legacy M2M)
  - SEND SMS: SIM-initiated SMS (alerts, telemetry)
  - DISPLAY TEXT: SIM-controlled UI messages
  - OPEN CHANNEL: BIP TCP/IP (OTA, telemetry)
  - PROVIDE LOCAL INFORMATION: SIM requests device state (IMEI, location, battery)

CAT enables SIM to act as intelligent agent:
  - Autonomous network connection (no app needed)
  - Telemetry reporting (SIM reads device sensors → SEND DATA)
  - Remote management (OTA server controls SIM → controls device)
```

**SIM OTA vs eSIM (SGP.32)**:
```
Feature                | SIM OTA (SMS/BIP)         | eSIM (SGP.32)
-----------------------|---------------------------|---------------------------
Profile Management     | Applet updates only       | Full profile swap (MNO change)
Security               | SCP80/81 (DES/3DES/AES)  | TLS + eUICC certificates
Bootstrap              | Master key at manufacture | EID + RSP architecture
Use Case               | M2M fleet management      | Consumer + M2M (future)
Deployment             | 100M+ M2M SIMs (legacy)   | 1B+ consumer phones, growing M2M
Standard Body          | GSMA + ETSI               | GSMA (SGP.22 for M2M, SGP.32 for IoT)
```

**Status**: SIM OTA (SMS-PP and BIP) still widely deployed for legacy M2M device fleets (automotive, industrial IoT, smart meters manufactured pre-2020). New deployments favor eSIM (SGP.32) for multi-MNO support and future-proofing.

---

## Certificate Enrollment
### EST (RFC 7030)
Enrollment over Secure Transport — X.509 certificate lifecycle over HTTPS:
- **Simple enroll:** POST /est/simpleenroll — issue new certificate
- **Re-enroll:** POST /est/simplereenroll — renew expiring certificate
- **CA certs:** GET /est/cacerts — retrieve CA certificate chain
- **CSR attributes:** GET /est/csrattrs — server-specified CSR requirements
- **Security:** mTLS (client cert) or HTTP Basic Auth over TLS
- **IoT use:** Industrial PKI, LwM2M bootstrapping with certificates

### EST over CoAP (draft-ietf-ace-coap-est)
EST adapted for constrained environments:
- CoAP resources: `/.well-known/est/crts`, `/.well-known/est/sen`, `/.well-known/est/sren`
- DTLS transport (matches CoAP security)
- CBOR encoding of CSR and certificates
- Used in LwM2M EST bootstrap and TEEP environments

### ACME (RFC 8555)
Automatic Certificate Management Environment — automated PKI for domain validation:
- **Primary use:** Web PKI (Let's Encrypt); increasingly relevant for IoT gateway certificates
- **Challenges:** HTTP-01 (HTTP validation), DNS-01 (DNS TXT record), TLS-ALPN-01
- **Not designed for:** Constrained device certificates (EST is preferred for devices)

---

## AAA Protocols
### RADIUS (RFC 2865)
Remote Authentication Dial-In User Service — foundational AAA for network access:
- **Transport:** UDP (ports 1812 auth, 1813 accounting)
- **IoT relevance:**
  - 802.1X/EAP-TLS authentication for enterprise Wi-Fi IoT onboarding
  - NB-IoT/LTE-M device authentication via EAP-AKA (Kerberos-like SIM auth) carried over RADIUS to HSS
  - EAP methods: EAP-TLS (mutual cert auth, strongest), EAP-TTLS, EAP-PEAP, EAP-AKA (SIM-based)
- **Shared Secret:** RADIUS client-server shared secret (weakness: MD5 HMAC; prefer RadSec/TLS)

### Diameter (RFC 6733)
Evolved AAA for 4G/5G replacing RADIUS in core network:
- **Transport:** TCP or SCTP; TLS/DTLS mandatory
- **4G interfaces:** S6a (MME↔HSS device auth), Gx (PCRF policy), Gy (online charging)
- **5G:** NEF uses HTTP/2 REST APIs (SBI) rather than Diameter; legacy Diameter via interworking
- **IoT relevance:** MVNO/eSIM roaming authentication chains; device identity validation via HSS

---

## Hardware Root of Trust
### DICE (Device Identifier Composition Engine)
TCG DICE standard for hardware-rooted identity:
- **Layer 0 (UDS):** Unique Device Secret — burned in manufacturing
- **Layer 1 (CDI):** Compound Device Identifier = HMAC(UDS, measurement of firmware)
- **Measurement:** Hash of firmware code + configuration at boot time
- **Key derivation:** Each layer derives identity from layer below + software measurement
- **Output:** Hardware-bound identity that changes if firmware is tampered
- **Use:** Attestation, credential provisioning, secure boot

### TPM 2.0 (TCG Specification)
Trusted Platform Module — hardware security chip:
- **Endorsement Key (EK):** Factory-provisioned, non-extractable; proves device identity
- **Attestation Identity Key (AIK/AK):** Used for quotes and attestation
- **Platform Configuration Registers (PCRs):** Accumulate measurement hashes (TPM Extend)
- **Sealed storage:** Data sealed to specific PCR values — decryption only if platform state matches
- **IoT use:** Gateway attestation; enterprise IoT certificate binding; FIDO FDO integration

### FIDO Device Onboard (FDO)
Fast, secure IoT device onboarding without per-device configuration:
- **Manufacturing:** Device receives Ownership Voucher (OV) at factory
- **Transfer of Ownership:** OV chain signed to final operator
- **Onboarding:** Device → Rendezvous Server → Owner → receive credentials
- **Security:** ECDH key agreement; no pre-shared secrets in field
- **Use cases:** Smart factory commissioning; large-scale retail/enterprise IoT deployment

---

## Regulatory Security Context
### EU Cyber Resilience Act (EU CRA)
Mandatory for products with digital elements sold in EU (enforcement from 2027, vulnerability reporting from 2026):
- **Scope:** All IoT devices; hardware + software
- **Requirements:** Security by design, no default passwords, vulnerability handling, SBOM, disclosure within 24h of exploitation
- **Impact:** Affects all wireless IoT device manufacturers selling in EU

### ETSI EN 303 645 — Consumer IoT Security
**Reference**: ETSI EN 303 645 V2.1.1 (2020) — Cyber Security for Consumer Internet of Things: Baseline Requirements

**Status**: Foundational consumer IoT security standard; harmonised under EU RED Article 3.3; basis for UK PSTI Act 2023

**13 High-Level Security Provisions**:

1. **No universal default passwords** — Devices must not ship with default passwords that are universal across all devices; each device must have unique credentials or force password setup during commissioning

2. **Vulnerability disclosure policy** — Manufacturer must provide a public contact point (security@example.com) for vulnerability reporting; acknowledge reports within defined timeframe

3. **Software update mechanism** — Secure update mechanism must be implemented; updates signed and verified; users notified of critical security updates

4. **Secure credential storage** — Cryptographic credentials stored securely; hard-coded credentials prohibited; use hardware root of trust where applicable

5. **Secure communications** — Sensitive security parameters communicated securely (TLS/DTLS); state-of-the-art encryption

6. **Minimize attack surface** — Unused network services, ports, and functionality disabled by default; principle of least privilege

7. **Software integrity and authenticity** — Firmware and software updates verified before installation; secure boot where applicable

8. **Personal data protection** — Compliance with GDPR; data minimization; user consent for data collection

9. **Resilience to outages** — Devices remain functional when cloud backend is unavailable; local operation where feasible

10. **Examine system telemetry** — Security event logging; audit trails for forensic analysis; log protection

11. **Easy device deletion** — User-initiated factory reset; data wiping mechanism; clear instructions

12. **Installation and maintenance** — Secure installation guidance; no assumption of secure network environment

13. **Input data validation** — Validate all input (user, network, sensors); sanitize to prevent injection attacks

**Implementation Mapping**:
```
Provision 1 (No default passwords):
  ├─ Matter commissioning: SPAKE2+ setup code per device
  ├─ LwM2M Bootstrap: Unique PSK per endpoint
  └─ MQTT: TLS client certificates (unique per device)

Provision 3 (Software updates):
  ├─ LwM2M Object 5 (Firmware Update) with Package URI verification
  ├─ SUIT manifest (RFC 9019) for firmware integrity
  └─ DFU over BLE: signed firmware images

Provision 4 (Credential storage):
  ├─ TPM 2.0 for certificate storage
  ├─ Secure Element (ATECC608, SE050) for private keys
  └─ ARM TrustZone for key isolation

Provision 5 (Secure communications):
  ├─ DTLS 1.3 for CoAP/LwM2M
  ├─ TLS 1.3 for MQTT/HTTPS
  └─ OSCORE for end-to-end application security
```

**Certification**: Not mandatory certification program, but compliance required for UK market (PSTI Act) and EU market (RED Article 3.3 delegated act)

**Cross-reference**: See [lpwan.md](lpwan.md) for LwM2M security object implementation; [pan-short-range.md](pan-short-range.md) for Matter/BLE commissioning security

---

### UK PSTI Act 2023
**Reference**: UK Product Security and Telecommunications Infrastructure Act 2023 (Mandatory from April 2024)

**Scope**: Consumer "connectable products" (IoT devices that connect to internet or local network) sold in UK market

**Three Core Requirements**:

1. **Ban on universal default passwords** (Section 1)
   - No device may have a factory default password that is:
     - The same for all instances of the product
     - Common and publicly available (e.g., "admin", "password", "12345")
   - Enforcement: Product cannot be sold in UK without unique credentials or forced password change

2. **Vulnerability disclosure policy** (Section 2)
   - Manufacturer must publish a contact point for security researchers to report vulnerabilities
   - Contact must be easily discoverable (e.g., security.txt RFC 9116, CERT/CC disclosure)
   - Timeframe for acknowledgment: Reasonable response expected

3. **Minimum security update period** (Section 3)
   - Manufacturer must define and publish a "defined support period" for security updates
   - Support period communicated at point of sale
   - Minimum period not specified in Act (left to secondary legislation); industry best practice: 5 years for consumer products

**Penalties**:
- Civil: Up to £10 million or 4% of global turnover (whichever is higher)
- Criminal: Up to £20,000 fine for summary conviction; unlimited for indictment

**Enforcement**: Office for Product Safety and Standards (OPSS)

**Relationship to ETSI EN 303 645**:
```
PSTI Act Requirements → ETSI EN 303 645 Provisions
  ├─ Section 1 (No default passwords) → Provision 1
  ├─ Section 2 (Vulnerability disclosure) → Provision 2
  └─ Section 3 (Security updates) → Provision 3

ETSI EN 303 645 compliance demonstrates PSTI Act conformity
Additional 10 ETSI provisions (4-13) go beyond PSTI minimum requirements
```

**Impact for IoT Manufacturers**:
- Any consumer IoT device sold in UK must comply (smart home, wearables, toys)
- B2B IoT devices exempt (industrial sensors, building automation)
- Applies to devices sold online to UK consumers (cross-border enforcement)

---

### EU Radio Equipment Directive Article 3.3 (RED)
**Reference**: Delegated Act 2022/30/EU under Radio Equipment Directive 2014/53/EU (Enforcement from August 2025)

Mandatory cybersecurity requirements for radio devices in EU:
- **Article 3.3(d):** Network protection — devices must not harm the network (DoS protection, no default credentials)
- **Article 3.3(e):** Privacy safeguards — personal data protection (GDPR alignment, encryption of personal data)
- **Article 3.3(f):** Fraud prevention — consumer protection features (payment transaction security, caller ID authentication)

**Harmonised Standards**:
- ETSI EN 303 645 (consumer IoT baseline requirements)
- IEC 62443 (industrial IoT security)
- ETSI TS 103 701 (cybersecurity for cellular IoT)

**Compliance Demonstration**: Manufacturers must provide technical documentation showing conformity; CE marking requires RED Article 3.3 compliance alongside EMC/RF requirements

**Difference from PSTI Act**: RED applies to all radio devices (B2C and B2B); PSTI Act targets consumer products only

### EU Battery Passport (EU 2023/1542)
From 2027, mandatory digital battery passport for EV, LMT, and industrial batteries >2kWh:
- **Data requirements:** Chemistry, capacity, SoH, SoC, cycle count, supply chain, recycled content
- **Access:** QR code → digital passport accessible to consumers, recyclers, regulators
- **LwM2M opportunity:** Device-side LwM2M Objects for battery lifecycle reporting (under development)