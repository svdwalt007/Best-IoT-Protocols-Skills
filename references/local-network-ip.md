# Local Network & IP Protocols

**Scope**: IP-based local area networking for IoT, including Wi-Fi (802.11), Ethernet (802.3), IPv6 transition mechanisms, header compression (6LoWPAN), routing (RPL), service discovery (mDNS/DNS-SD), and modern transport (QUIC). Cross-references [pan-short-range.md](pan-short-range.md) for Thread's use of 6LoWPAN, [tsn-deterministic.md](tsn-deterministic.md) for Ethernet TSN, [smart-home-consumer.md](smart-home-consumer.md) for Matter's mDNS discovery, and [security-provisioning.md](security-provisioning.md) for 802.1X/EAP authentication.

## Table of Contents
1. Wi-Fi (IEEE 802.11)
2. Ethernet (IEEE 802.3)
3. 6LoWPAN (IPv6 over Low-Power Wireless)
4. RPL (Routing Protocol for Low-Power and Lossy Networks)
5. IPv4 / IPv6 Fundamentals
6. IPv6 Transition Mechanisms
7. mDNS / DNS-SD (Service Discovery)
8. QUIC (HTTP/3 Transport)

---

## 1. Wi-Fi (IEEE 802.11)

**Reference**: IEEE 802.11-2020 (consolidated standard)

**Generations & PHY Specs**:
```
Wi-Fi 4 (802.11n, 2009):
  ├─ Frequency: 2.4 GHz, 5 GHz
  ├─ Max data rate: 600 Mbps (4×4 MIMO, 40 MHz channel)
  ├─ Modulation: OFDM (BPSK, QPSK, 16-QAM, 64-QAM)
  └─ Features: MIMO, frame aggregation (A-MPDU)

Wi-Fi 5 (802.11ac, 2013):
  ├─ Frequency: 5 GHz only
  ├─ Max data rate: 6.9 Gbps (8×8 MU-MIMO, 160 MHz, 256-QAM)
  ├─ MU-MIMO: Simultaneous transmission to 4 clients
  └─ Beamforming: TxBF for improved range

Wi-Fi 6 (802.11ax, 2019):
  ├─ Frequency: 2.4 GHz, 5 GHz
  ├─ Max data rate: 9.6 Gbps (8×8 MIMO, 160 MHz, 1024-QAM)
  ├─ OFDMA: Divides channel into Resource Units (RU) for concurrent users
  ├─ TWT (Target Wake Time): Power saving (battery devices sleep, wake on schedule)
  ├─ BSS Coloring: Spatial reuse (reduce co-channel interference)
  └─ 8×8 UL/DL MU-MIMO

Wi-Fi 6E (802.11ax, 2020):
  ├─ Adds 6 GHz band (5.925-7.125 GHz)
  ├─ 14 additional 80 MHz channels, 7× 160 MHz channels
  └─ No legacy devices → cleaner spectrum

Wi-Fi 7 (802.11be, 2024):
  ├─ Max data rate: 46 Gbps (16×16 MIMO, 320 MHz, 4096-QAM)
  ├─ MLO (Multi-Link Operation): Aggregate 2.4/5/6 GHz simultaneously
  ├─ 16×16 MU-MIMO, 512 compressed block ACK
  └─ Preamble puncturing (skip interfered 20 MHz sub-channels)
```

**Wi-Fi HaLow (IEEE 802.11ah, 2017)**:
```
Purpose: Long-range IoT (sub-1 GHz)
Frequency: 900 MHz ISM band (US: 902-928 MHz, EU: 863-868 MHz)
Range: 1 km (indoor), 1-10 km (outdoor, LoS)
Data rate: 150 kbps - 347 Mbps
Modulation: OFDM (MCS0-MCS10)
MAC: TWT, RAW (Restricted Access Window) for collision avoidance
Use case: Smart agriculture, industrial IoT, smart meters
```

**Target Wake Time (TWT)**:
```
AP                                 STA (IoT device)
│                                        │
├─ 1. TWT Setup Request ────────────────>│
│    (wake_interval=60s, wake_duration=10ms)
│                                        │
│<─ 2. TWT Setup Response ───────────────┤
│    (Accepted)                          │
│                                        │
├─ 3. Device sleeps for 60s              │
│                                        │
├─ 4. Wake at scheduled time ────────────>│
│    (data buffered at AP delivered)     │
│                                        │
└─ 5. Sleep again                        │

Power savings: 99% reduction in active time (10ms wake / 60s interval)
```

**Wi-Fi Aware (NAN - Neighbor Awareness Networking)**:
```
Purpose: Peer-to-peer discovery without AP
Discovery Window (DW): Every 512 TU (524 ms)
Service Discovery:
  ├─ Publish: Advertise service ("FileShare", "Printer")
  └─ Subscribe: Query for service
Ranging: Measure distance to peer (FTM - Fine Timing Measurement)
Use case: Proximity-based services (Find My Device, Wi-Fi Direct)
```

**Wi-Fi Security**:
```
WEP (deprecated):        RC4, 40/104-bit key, broken (IV reuse attack)
WPA (deprecated):        TKIP, temporal keys, broken (Beck-Tews attack)
WPA2 (802.11i, 2004):
  ├─ Personal (PSK): Pre-Shared Key (PBKDF2 from passphrase)
  ├─ Enterprise (802.1X): RADIUS + EAP-TLS/PEAP/EAP-TTLS
  ├─ Encryption: CCMP (AES-128-CCM)
  └─ 4-way handshake: PTK derivation, GTK distribution

WPA3 (2018):
  ├─ Personal: SAE (Simultaneous Authentication of Equals)
  │   └─ Dragonfly key exchange (resistant to offline dictionary)
  ├─ Enterprise: 192-bit mode (GCMP-256, HMAC-SHA-384)
  ├─ Forward secrecy (no KRACK vulnerability)
  └─ Opportunistic Wireless Encryption (OWE) for open networks
```

---

## 2. Ethernet (IEEE 802.3)

**Reference**: IEEE 802.3-2022

**Physical Layer Standards**:
```
10BASE-T:      10 Mbps, Cat 3, 100m, RJ45 (legacy)
100BASE-TX:    100 Mbps (Fast Ethernet), Cat 5, 100m
1000BASE-T:    1 Gbps (Gigabit), Cat 5e/6, 100m, 4-pair
2.5GBASE-T:    2.5 Gbps, Cat 5e/6, 100m (Wi-Fi 6 backhaul)
5GBASE-T:      5 Gbps, Cat 6, 100m
10GBASE-T:     10 Gbps, Cat 6a/7, 100m
10GBASE-SX:    10 Gbps, multimode fiber, 400m (850nm)
10GBASE-LR:    10 Gbps, single-mode fiber, 10km (1310nm)
```

**802.1Q VLANs**:
```
Ethernet Frame:
  [Dest MAC][Src MAC][802.1Q Tag][EtherType][Payload][FCS]

802.1Q Tag (4 bytes):
  ├─ TPID: 0x8100 (Tag Protocol ID)
  ├─ PCP: 3 bits (Priority Code Point, 0-7 for QoS)
  ├─ DEI: 1 bit (Drop Eligible Indicator)
  └─ VID: 12 bits (VLAN ID, 1-4094)

Example: VLAN 100 for IoT devices, VLAN 200 for management
  → Logical network segmentation without physical separation
```

**802.1X (Port-Based NAC)**:
```
Supplicant (IoT device)       Authenticator (Switch)      Authentication Server (RADIUS)
        │                            │                            │
        ├─ 1. EAPOL Start ──────────>│                            │
        │                            │                            │
        │<─ 2. EAP Request Identity ─┤                            │
        │                            │                            │
        ├─ 3. EAP Response (ID) ────>│                            │
        │                            ├─ RADIUS Access-Request ───>│
        │                            │   (encapsulates EAP)       │
        │                            │                            │
        │<─ 4. EAP Request (TLS) ─────────────────────────────────┤
        │    (via switch relay)      │                            │
        │                            │                            │
        ├─ 5. EAP-TLS Handshake ─────────────────────────────────>│
        │    (Client cert + Server cert mutual auth)              │
        │                            │                            │
        │                            │<─ RADIUS Access-Accept ─────┤
        │                            │   (VLAN assignment, ACL)   │
        │<─ 6. EAP Success ───────────┤                            │
        │                            │                            │
        └─ Port authorized (traffic allowed)                      │

EAP Methods:
  ├─ EAP-TLS:  X.509 certificate mutual auth (most secure)
  ├─ EAP-PEAP: Protected EAP (TLS tunnel + password)
  ├─ EAP-TTLS: Tunneled TLS (flexible inner auth)
  └─ EAP-PWD:  Password auth (no server cert, Dragonfly-like)
```

**Power over Ethernet (PoE)**:
```
802.3af (PoE, 2003):      15.4W (13W at device), 48V, Alt A/B
802.3at (PoE+, 2009):     30W (25.5W at device), 48V
802.3bt (PoE++, 2018):
  ├─ Type 3: 60W (51W at device), 4-pair
  └─ Type 4: 100W (71W at device), 4-pair

Use case: IP cameras, Wi-Fi APs, VoIP phones, IoT sensors
```

---

## 3. 6LoWPAN (IPv6 over Low-Power Wireless)

**Reference**: RFC 4944, RFC 6282 (IPHC), RFC 6775 (ND Optimization)

**Purpose**: Adapt IPv6 to IEEE 802.15.4 (127-byte MTU). Compresses IPv6 headers from 40 bytes to 2-6 bytes.

**Header Compression (IPHC - IPv6 Header Compression)**:
```
Standard IPv6 Header (40 bytes):
  [Version 4b][Traffic Class 8b][Flow Label 20b][Payload Length 16b]
  [Next Header 8b][Hop Limit 8b][Source Address 128b][Dest Address 128b]

6LoWPAN IPHC (2-6 bytes):
  [Dispatch 2b][IPHC encoding 13b][Inline fields variable]

Compression strategies:
  ├─ Version: Elided (always 6)
  ├─ Traffic Class / Flow Label: Elided if zero
  ├─ Payload Length: Elided (derived from 802.15.4 frame length)
  ├─ Next Header: Compressed via NHC (Next Header Compression)
  │   └─ UDP: 8 bytes → 1-4 bytes (port compression)
  ├─ Hop Limit: Elided if 64, or sent inline (1 byte)
  └─ Addresses: Compressed based on context
      ├─ Link-local: Derived from 802.15.4 64-bit MAC → 0 bytes
      ├─ Multicast: ff02::1 (all-nodes) → 1 byte
      └─ Global: Prefix shared with PAN → 8 bytes (IID only)

Example:
  Uncompressed: 40 (IPv6) + 8 (UDP) + payload = 48 bytes overhead
  Compressed:    2 (IPHC) + 1 (UDP NHC) + payload = 3 bytes overhead
  Savings: 93% overhead reduction
```

**Fragmentation** (RFC 4944):
```
IEEE 802.15.4 MTU: 127 bytes (minus ~25B MAC header = ~102B payload)
IPv6 minimum MTU: 1280 bytes
  → Fragmentation required for large packets

6LoWPAN Fragmentation Header:
  First Fragment:
    [Dispatch 0xC0][Datagram Size 11b][Datagram Tag 16b][Payload]
  Subsequent Fragments:
    [Dispatch 0xE0][Datagram Size 11b][Datagram Tag 16b][Offset 8b][Payload]

Example (send 300-byte IPv6 packet):
  Fragment 0: [0xC0 012C 1234][100B payload]
  Fragment 1: [0xE0 012C 1234 64][100B payload]
  Fragment 2: [0xE0 012C 1234 C8][100B payload]
```

**Mesh Addressing** (RFC 4944 §5):
```
Mesh Header (optional, for multi-hop):
  [Dispatch 0x80-0xC0][Hops Left 4b][Orig/Final Address 64/16-bit]

Example: Node A → Node B → Node C
  Node A sends:
    [Mesh Header: Hops=2, Orig=A, Final=C][IPv6 packet]
  Node B decrements Hops=1, forwards to C
```

---

## 4. RPL (Routing Protocol for Low-Power and Lossy Networks)

**Reference**: RFC 6550 (RPL), RFC 6719 (Multicast)

**Purpose**: Distance-vector routing for constrained IoT networks. Builds DODAG (Destination-Oriented Directed Acyclic Graph) rooted at border router.

**DODAG Construction**:
```
Root (Border Router, Rank=0)
  ├─ Node A (Rank=256)
  │   ├─ Node C (Rank=512)
  │   └─ Node D (Rank=512)
  └─ Node B (Rank=256)
      └─ Node E (Rank=512)

Rank: Topological distance from root (lower = closer)
  Calculated from ETX (Expected Transmission Count):
    ETX = Number of transmissions needed for successful delivery
    Rank_Node = Rank_Parent + ETX_to_Parent

Objective Function (OF):
  ├─ OF0 (RFC 6552): Minimize hop count
  └─ MRHOF (RFC 6719): Minimize ETX (reliability-based)
```

**Control Messages (ICMPv6)**:
```
DIO (DODAG Information Object):
  ├─ Sent by: Nodes with lower Rank (downward)
  ├─ Purpose: Advertise DODAG parameters (DODAG ID, Rank, OF)
  ├─ Trickle Timer: Exponential backoff (suppress redundant DIOs)
  └─ Multicast: ff02::1a (all-RPL-nodes)

DIS (DODAG Information Solicitation):
  ├─ Sent by: New node joining network
  ├─ Purpose: Request DIO from neighbors
  └─ Trigger: Immediate DIO transmission

DAO (Destination Advertisement Object):
  ├─ Sent by: Leaf nodes (upward to root)
  ├─ Purpose: Advertise reachable destinations (downward routes)
  ├─ Storing Mode: Each node stores routing table
  └─ Non-Storing Mode: Root stores all routes, source-routes packets

Example DAO:
  Node C sends DAO to Node A:
    "Prefix fc00::C/128 is reachable via me"
  Node A aggregates, sends DAO to Root:
    "Prefix fc00::A/64 (includes C, D) is reachable via me"
```

**RPL Modes**:
```
Storing Mode (MOP=1):
  ├─ Each node stores routing table (next-hop per destination)
  ├─ Downward traffic: Hop-by-hop routing
  └─ Memory: High (scales poorly for large networks)

Non-Storing Mode (MOP=0):
  ├─ Only root stores routing table
  ├─ Downward traffic: Source routing (IPv6 Routing Header)
  │   Example: Root → [A, C] → Node C
  └─ Memory: Low per node (root handles complexity)
```

**Use Case**: Thread networks (see [smart-home-consumer.md](smart-home-consumer.md)), 6TiSCH (see [tsn-deterministic.md](tsn-deterministic.md)), smart city sensor mesh.

---

## 5. IPv4 / IPv6 Fundamentals

**IPv4 Address Exhaustion** (IANA pool exhausted 2011):
```
Total IPv4: 4.3 billion addresses (2^32)
Reserved:   Private (10/8, 172.16/12, 192.168/16), Multicast (224/4)
Available:  ~3.7 billion (insufficient for IoT scale)
```

**IPv6 Addressing**:
```
Format: 128-bit address (8 groups of 16-bit hex)
  Example: 2001:0db8:85a3:0000:0000:8a2e:0370:7334
  Compressed: 2001:db8:85a3::8a2e:370:7334 (:: replaces consecutive zeros)

Address Types:
  ├─ Unicast:
  │   ├─ Global Unicast: 2000::/3 (routable internet)
  │   ├─ Link-Local: fe80::/10 (non-routable, local segment)
  │   └─ ULA (Unique Local): fc00::/7 (private, RFC 4193)
  ├─ Multicast: ff00::/8
  │   ├─ ff02::1 (all-nodes link-local)
  │   ├─ ff02::2 (all-routers link-local)
  │   └─ ff02::1:ff00:0/104 (solicited-node multicast for ND)
  └─ Anycast: Same as unicast (multiple interfaces, nearest responds)
```

**SLAAC (Stateless Address Autoconfiguration)**:
```
1. Node generates link-local address:
   fe80::<IID derived from EUI-64 or random>

2. Node performs DAD (Duplicate Address Detection):
   Sends Neighbor Solicitation to ff02::1:ff<IID suffix>
   If no response → address is unique

3. Node listens for Router Advertisement (RA):
   Router sends RA with prefix (e.g., 2001:db8::/64)

4. Node generates global address:
   Prefix (from RA) + IID → 2001:db8::IID

5. Node learns default gateway from RA (router's link-local address)
```

**DHCPv6**:
```
Stateful: Server assigns full address (like DHCPv4)
Stateless: Node uses SLAAC for address, DHCPv6 for DNS/NTP (RFC 8415)

Message Flow:
  1. Solicit (multicast to ff02::1:2, all-DHCP-servers)
  2. Advertise (server responds)
  3. Request (client selects server)
  4. Reply (server assigns address + options)
```

---

## 6. IPv6 Transition Mechanisms

**NAT64 / DNS64** (RFC 6146, RFC 6147):
```
Purpose: IPv6-only clients access IPv4 servers

DNS64 (Synthesize AAAA records):
  Client queries: example.com
  DNS64 resolver:
    1. Queries A record → 203.0.113.1 (IPv4)
    2. No AAAA record exists
    3. Synthesizes AAAA: 64:ff9b::cb00:7101 (prefix + IPv4)
       (64:ff9b::/96 is well-known NAT64 prefix)
    4. Returns AAAA to client

NAT64 (Translate IPv6 ↔ IPv4):
  Client sends: 2001:db8::1 → 64:ff9b::cb00:7101
  NAT64 gateway:
    1. Extracts IPv4: 203.0.113.1
    2. Allocates dynamic IPv4 + port for client
    3. Forwards to IPv4 server: 198.51.100.5:12345 → 203.0.113.1:80
    4. Translates response back to IPv6
```

**464XLAT** (RFC 6877):
```
Mobile operator IPv6-only core + IPv4 app compatibility

CLAT (Customer-side translator):
  ├─ On device (phone): Translates IPv4 app traffic → IPv6
  └─ Prefix: 192.0.0.0/29 (WKP - Well-Known Prefix for CLAT)

PLAT (Provider-side translator):
  ├─ NAT64 in operator core
  └─ Translates back to IPv4 for internet

Flow:
  App sends IPv4: 10.0.0.1 → 8.8.8.8
  CLAT: 10.0.0.1 → 64:ff9b::808:808 (synthesized)
  PLAT: 203.0.113.1 → 8.8.8.8 (translated)
```

**MAP-T / MAP-E** (RFC 7599, RFC 7597):
```
MAP-T (Translation, stateless):
  ├─ Deterministic 1:1 IPv4 ↔ IPv6 mapping
  └─ No NAT state in core (scales better)

MAP-E (Encapsulation):
  ├─ IPv4 packet encapsulated in IPv6
  └─ Similar to DS-Lite but with port-sharing

Example: ISP assigns 203.0.113.1/16 to customer
  Customer gets ports 2048-4095 (2048 ports)
  MAP rule: IPv6 prefix + IPv4 + port range
```

---

## 7. mDNS / DNS-SD (Service Discovery)

**Reference**: RFC 6762 (mDNS), RFC 6763 (DNS-SD)

**mDNS (Multicast DNS)**:
```
Purpose: Zero-config name resolution on local network (no DNS server)

Query:
  Host sends: "printer._tcp.local" query to 224.0.0.251:5353 (IPv4) or ff02::fb (IPv6)
  Responder(s) send answer directly (multicast or unicast)

Record Types:
  ├─ A / AAAA: IP address
  ├─ PTR: Service instance enumeration
  ├─ SRV: Service location (hostname + port)
  └─ TXT: Service metadata (key=value pairs)

Example (print.local → 192.168.1.100):
  Query:  print.local (A record)
  Answer: print.local 120 IN A 192.168.1.100
```

**DNS-SD (DNS Service Discovery)**:
```
Purpose: Discover services on network (printers, AirPlay, Matter devices)

Service Instance Name:
  <Instance>.<Service>.<Domain>
  Example: "Office Printer._ipp._tcp.local"

Browsing:
  1. Query PTR: _ipp._tcp.local
  2. Response: "Office Printer._ipp._tcp.local"
  3. Query SRV + TXT: Office Printer._ipp._tcp.local
  4. Response:
     SRV: printer.local:631
     TXT: txtvers=1 pdl=application/pdf

TXT Record Metadata:
  ├─ Matter device: "VP=65521+32768" (Vendor ID + Product ID)
  ├─ HomeKit: "c#=1 s#=1 ci=8 sf=0" (config number, setup flag, category)
  └─ AirPlay: "features=0x1 model=AppleTV"
```

**Use Case**: Matter commissioning (see [smart-home-consumer.md](smart-home-consumer.md)), HomeKit accessory discovery, Chromecast, Bonjour.

---

## 8. QUIC (HTTP/3 Transport)

**Reference**: RFC 9000 (QUIC), RFC 9114 (HTTP/3)

**Purpose**: Low-latency, reliable transport over UDP. Replaces TCP+TLS with single handshake. Resistant to head-of-line blocking.

**QUIC vs TCP+TLS**:
```
TCP+TLS 1.3 Handshake (1-RTT):
  1. Client → SYN
  2. Server → SYN-ACK
  3. Client → ACK + TLS ClientHello
  4. Server → TLS ServerHello + Certificate
  5. Client → TLS Finished
  6. Data transfer starts (2-RTT total)

QUIC Handshake (0-RTT on resume, 1-RTT new):
  1. Client → Initial packet (ClientHello + QUIC crypto)
  2. Server → Handshake packet (ServerHello + Certificate)
  3. Client → Handshake Complete + 0-RTT Data
  4. Data transfer (0-RTT if cached, 1-RTT if new)

Benefits:
  ├─ Faster connection (50% latency reduction)
  ├─ Connection migration (switch Wi-Fi → cellular seamlessly)
  └─ No head-of-line blocking (independent streams, lost packet affects only 1 stream)
```

**Stream Multiplexing**:
```
HTTP/2 over TCP:
  Stream 1: index.html (blocked if packet 5 lost)
  Stream 2: style.css  (blocked waiting for packet 5 retransmit)
  Stream 3: image.png  (blocked)
  → Head-of-line blocking (TCP retransmit stalls all streams)

HTTP/3 over QUIC:
  Stream 1: index.html (independent)
  Stream 2: style.css  (independent, continues even if Stream 1 packet lost)
  Stream 3: image.png  (independent)
  → No blocking (QUIC retransmit only affects lost stream)
```

**Connection Migration**:
```
Mobile device switches network (Wi-Fi → 4G):
  TCP: Connection drops, must re-establish (SYN/SYN-ACK)
  QUIC: Connection persists via Connection ID
    ├─ Connection ID: Random 64-bit identifier (not tied to IP)
    ├─ Client sends packet from new IP with same Connection ID
    └─ Server validates, continues session (0 downtime)
```

**Use Case**: Cloud IoT platforms (AWS IoT Core supports QUIC), video streaming (YouTube), web browsing (Chrome/Firefox HTTP/3).

---

## Key Specification References

| Standard        | Title                                    | Publisher | Year |
|-----------------|------------------------------------------|-----------|------|
| IEEE 802.11-2020| Wi-Fi (consolidated standard)            | IEEE      | 2020 |
| IEEE 802.11ax   | Wi-Fi 6 (High Efficiency WLAN)           | IEEE      | 2021 |
| IEEE 802.11ah   | Wi-Fi HaLow (Sub-1 GHz)                  | IEEE      | 2017 |
| IEEE 802.3-2022 | Ethernet (consolidated standard)         | IEEE      | 2022 |
| RFC 4944        | 6LoWPAN (Transmission of IPv6 over 802.15.4) | IETF  | 2007 |
| RFC 6282        | 6LoWPAN IPHC (Header Compression)        | IETF      | 2011 |
| RFC 6550        | RPL (Routing Protocol for LLNs)          | IETF      | 2012 |
| RFC 8415        | DHCPv6                                   | IETF      | 2018 |
| RFC 6146        | NAT64 (IPv6/IPv4 Translation)            | IETF      | 2011 |
| RFC 6762        | mDNS (Multicast DNS)                     | IETF      | 2013 |
| RFC 6763        | DNS-SD (Service Discovery)               | IETF      | 2013 |
| RFC 9000        | QUIC Transport Protocol                  | IETF      | 2021 |
| RFC 9114        | HTTP/3                                   | IETF      | 2022 |

---

## Related Files
- **[pan-short-range.md](pan-short-range.md)**: Thread (uses 6LoWPAN + RPL), IEEE 802.15.4 PHY/MAC
- **[tsn-deterministic.md](tsn-deterministic.md)**: IEEE 802.1 Ethernet TSN, 6TiSCH (6LoWPAN over TSCH)
- **[smart-home-consumer.md](smart-home-consumer.md)**: Matter (mDNS/DNS-SD discovery), Wi-Fi commissioning
- **[security-provisioning.md](security-provisioning.md)**: 802.1X/EAP, WPA3-Enterprise, TLS 1.3
- **[application-messaging.md](application-messaging.md)**: CoAP over 6LoWPAN, MQTT over QUIC
