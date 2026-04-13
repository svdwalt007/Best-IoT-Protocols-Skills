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

### SIM OTA (GSMA TS.02 / TS.03)
Legacy OTA management of SIM card applications (Java Card applets) — predates SGP.32:
- **GSMA TS.02:** Security requirements for OTA via SMS Bearer
- **GSMA TS.03:** Requirements for OTA via BIP (Bearer Independent Protocol)
- **SMS Bearer OTA:** Commands tunnelled in GSM SMS; SIM responds via SMS; SCP80 security
- **BIP (Bearer Independent Protocol):** ETSI TS 102 127, 3GPP TS 31.111; commands over GPRS/LTE data bearer; SCP81 security; higher throughput than SMS OTA
- **Commands:** OPEN CHANNEL, SEND DATA, RECEIVE DATA, CLOSE CHANNEL (proactive SIM commands)
- **Key management:** Key Set Identifier (KSI) + OTA keys (TAR — Toolkit Application Reference identifies target applet)
- **Status:** Still widely deployed for M2M SIM fleet management on legacy devices; being superseded by SGP.32

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

### EU Radio Equipment Directive Article 3.3 (RED)
Mandatory cybersecurity requirements for radio devices in EU (from August 2025):
- **Article 3.3(d):** Network protection — devices must not harm the network
- **Article 3.3(e):** Privacy safeguards — personal data protection
- **Article 3.3(f):** Fraud prevention — consumer protection features
- **Standards:** ETSI EN 303 645, IEC 62443 may be used for compliance

### EU Battery Passport (EU 2023/1542)
From 2027, mandatory digital battery passport for EV, LMT, and industrial batteries >2kWh:
- **Data requirements:** Chemistry, capacity, SoH, SoC, cycle count, supply chain, recycled content
- **Access:** QR code → digital passport accessible to consumers, recyclers, regulators
- **LwM2M opportunity:** Device-side LwM2M Objects for battery lifecycle reporting (under development)