# Application/Messaging Protocols — Complete Reference

## Overview
Application-layer messaging protocols define how IoT devices exchange data with servers, brokers, and cloud platforms. This reference covers the major IoT messaging protocols from lightweight constrained-device protocols (CoAP, MQTT-SN) through enterprise messaging (AMQP, DDS) with full RFC coverage.

---

## MQTT
### Overview
**MQTT (Message Queuing Telemetry Transport)** — publish-subscribe messaging protocol over TCP/IP. Originally developed by IBM/Arcom for satellite telemetry; now an OASIS standard. The dominant IoT messaging protocol.
**Versions:** MQTT v3.1 (legacy), **MQTT v3.1.1** (current/widely deployed), **MQTT v5.0** (current, feature-rich)
### Architecture
```
Publisher → [Topic] → Broker → [Topic filter] → Subscriber
                                        ↓
                                        Retained message store
                                        Will message store
```
### QoS Levels
| QoS | Guarantee | Message Exchange |
|-----|-----------|-----------------|
| 0 | At most once (fire-and-forget) | PUBLISH only |
| 1 | At least once | PUBLISH → PUBACK |
| 2 | Exactly once | PUBLISH → PUBREC → PUBREL → PUBCOMP |
### MQTT v3.1.1 Key Features
- **Retained messages:** Broker stores last message per topic; new subscribers get immediate value
- **Last Will and Testament (LWT):** Message published by broker on unexpected client disconnect
- **Clean session:** `cleanSession=true` → no persistent session state; `false` → durable subscriptions
- **Keep-alive:** Heartbeat interval; broker disconnects on timeout (1.5× keep-alive)
- **Topic wildcards:** `+` (single level), `#` (multi-level): `sensors/+/temperature`, `sensors/#`
### MQTT v5.0 Additions
| Feature | Description |
|---------|-------------|
| Shared subscriptions | `$share/group/topic` — load-balanced consumer groups |
| Message expiry | TTL on PUBLISH packets |
| Request/Response | Correlation Data + Response Topic for request-reply |
| User properties | Key-value metadata on any packet |
| Reason codes | Granular error codes on all ACK packets |
| Subscription options | No-local, retain-as-published, retain-handling |
| Topic aliases | Integer aliases for long topic names |
| Flow control | Receive Maximum for backpressure |
| Server disconnect | Server can send DISCONNECT with reason code |
| Enhanced authentication | SASL-like multi-step auth via AUTH packet |
### MQTT-SN (MQTT for Sensor Networks)
Protocol variant for non-TCP constrained devices (Zigbee, BLE, UDP serial):
- **Transport:** UDP, ZigBee, BLE, serial; no TCP required
- **Topic IDs:** Pre-registered integer IDs replace string topic names (saves bytes)
- **DTLS support:** Optional DTLS security layer for UDP transport
- **Gateway:** MQTT-SN Gateway translates to MQTT broker
- **Client sleep:** Supports sleeping client pattern for battery devices
### MQTT Sparkplug B
MQTT application layer specification for industrial SCADA:
- **Namespace:** `spBv1.0/{group_id}/{message_type}/{edge_node_id}/{device_id}`
- **Message types:** NBIRTH/NDEATH (node), DBIRTH/DDEATH (device), NDATA/DDATA (telemetry), NCMD/DCMD (commands)
- **Encoding:** Protocol Buffers (protobuf) for efficient binary payload
- **State management:** Birth certificates establish initial metric state; Death certificates on disconnect
- **Primary Host:** Single designated consumer that manages Edge of Network node state
- **Sparkplug 3.0:** Now under Eclipse Foundation governance
- **Use cases:** SCADA historian integration, OT/IT convergence via MQTT broker

---

## CoAP (Constrained Application Protocol)
### Overview
**RFC 7252** — RESTful protocol for constrained devices, designed to work over UDP with low overhead. Maps to HTTP semantics (GET/POST/PUT/DELETE) but binary-encoded and UDP-native.
### Message Types
| Type | Code | Description |
|------|------|-------------|
| CON | Confirmable | Reliable; recipient must ACK |
| NON | Non-confirmable | Best-effort; no ACK |
| ACK | Acknowledgement | Response to CON |
| RST | Reset | Error/not interested |
### Response Codes (similar to HTTP)
- `2.xx` Success: 2.01 Created, 2.02 Deleted, 2.03 Valid, 2.04 Changed, 2.05 Content
- `4.xx` Client Error: 4.00 Bad Request, 4.01 Unauthorized, 4.04 Not Found, 4.05 Method Not Allowed
- `5.xx` Server Error: 5.00 Internal Server Error, 5.03 Service Unavailable
### CoAP Message Format
```
    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |Ver| T | TKL | Code | Message ID |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   | Token (if any, TKL bytes) ...
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   | Options (if any) ...
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |1 1 1 1 1 1 1 1| Payload (if any) ...
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

   Ver=01, T=Type (00=CON,01=NON,10=ACK,11=RST), TKL=Token Length
```
### CoAP Observe (RFC 7641)
Server-push notification mechanism — fundamental to LwM2M Information Reporting:
```
Client → Server: GET coap://server/temperature (Observe: 0) [Register]
Server → Client: 2.05 Content "23.5" (Observe: 1) [Initial response]
Server → Client: 2.05 Content "24.1" (Observe: 2) [Notification on change]
Server → Client: 2.05 Content "23.8" (Observe: 3) [Notification on change]
Client → Server: GET coap://server/temperature (Observe: 1) [Deregister]
```
- **Freshness:** Max-Age option controls when observation is considered stale
- **Token:** Identifies the observation; must be unique per client-server pair
- **Replay detection:** Observe sequence numbers detect out-of-order delivery
### CoAP over TCP/TLS/WebSocket (RFC 8323)
Removes UDP dependency; critical for LwM2M v1.1+ on NB-IoT:
- **TCP binding:** CoAP over TCP; persistent connections; no CON/ACK needed (TCP handles reliability)
- **TLS binding:** CoAP over TLS 1.2/1.3; `coaps+tcp://` URI scheme
- **WebSocket binding:** CoAP over WebSocket; `coap+ws://` and `coaps+ws://`
- **Why it matters:** NB-IoT NAT traversal; DTLS CID mitigates but TCP eliminates the problem
- **Message framing:** Length-prefixed (vs UDP datagram-based); CSM (Capabilities and Settings Message)
### CoAP Extensions
| RFC | Name | Purpose |
|-----|------|---------|
| RFC 7641 | Observe | Server-push notifications |
| RFC 7959 | Blockwise Transfer | Large payload fragmentation |
| RFC 8323 | CoAP over TCP/TLS/WS | Reliable transport binding |
| RFC 8613 | OSCORE | Object Security (E2E encryption) |
| RFC 8974 | Extended Tokens | Stateless client support; >8 byte tokens |
| RFC 9175 | Echo, Request-Tag, Token Processing | Replay protection; request freshness verification |
| RFC 8768 | Hop-Limit | TTL for CoAP proxies |
| RFC 8516 | Too Many Requests | Rate limiting response code (4.29) |
| RFC 9177 | Q-Block | LPWAN-optimised blockwise transfer |
| RFC 9203 | Group OSCORE | OSCORE for multicast groups |
### CoAP Group Communication / Multicast (RFC 7390)
- **Address:** IPv6 multicast (`ff0x::fd` site-local CoAP); IPv4 `224.0.1.187`
- **Transport:** Non-confirmable only (CON multicast not allowed)
- **Security:** Group OSCORE (RFC 9203) for authenticated multicast
- **Use case:** One-to-many commands (e.g., turn off all lights in a zone)
### CoAP Pub/Sub (draft-ietf-core-coap-pubsub)
Broker-mediated publish-subscribe for CoAP:
- **Broker:** Acts as resource directory for topics
- **Publish:** PUT to `coap://broker/ps/{topic}`
- **Subscribe:** GET with Observe to `coap://broker/ps/{topic}`
- **Difference from MQTT:** CoAP native; no separate protocol bridge needed
### CoRE Link Format (RFC 6690) and Resource Directory (RFC 9176)
Resource discovery for CoAP networks:
- **Link Format (RFC 6690):** `GET /.well-known/core` returns device resource list
- **Resource Directory (RFC 9176):** Central registry where devices register their resources
- **Registration:** `POST coap://rd.example.com/rd?ep=device1&lt=3600 <sensor/temp>;rt="temperature"`
- **Lookup:** `GET coap://rd.example.com/rd/lookup/res?rt=temperature` — find all temperature sensors
- **Used by:** LwM2M (object/resource registration), Matter, CoAP deployments

---

## AMQP (Advanced Message Queuing Protocol)
### Overview
**AMQP v1.0** (ISO/IEC 19464) — enterprise message queuing protocol. Binary, connection-oriented, broker-based. Used in Azure Service Bus, RabbitMQ, Apache ActiveMQ, Eclipse Hono northbound.
### AMQP v1.0 Key Concepts
- **Connection:** TCP/TLS connection between two AMQP containers
- **Session:** Bidirectional channel within a connection (flow control)
- **Link:** Unidirectional message channel (source → target); attached to session
- **Node:** Named entity within a container (queue, topic, exchange)
- **Transfer:** Message delivery; three settlement modes (at-most-once, at-least-once, exactly-once)
### vs MQTT
| Feature | MQTT | AMQP v1.0 |
|---------|------|-----------|
| Topology | Pub/Sub only | Pub/Sub + P2P + Request/Reply |
| Overhead | Very low (2-byte minimum) | Higher (binary framing) |
| Routing | Topic-based only | Flexible (exchange bindings) |
| Flow control | Basic | Advanced (credit-based per link) |
| Transactions | No | Yes |
| Constrained devices | Yes (MQTT-SN) | No |
| Enterprise features | Limited | Full |

---

## HTTP/2 and HTTP/3
### HTTP/2 for IoT
- **Multiplexing:** Multiple streams over single TCP connection (vs HTTP/1.1 head-of-line blocking)
- **Header compression:** HPACK reduces overhead
- **Server Push:** Server proactively sends resources
- **Binary framing:** More efficient than HTTP/1.1 text
- **IoT use:** REST APIs on gateways; not suitable for constrained devices
### HTTP/3 (QUIC-based)
- **Transport:** QUIC (RFC 9000) over UDP — built-in TLS 1.3, 0-RTT
- **Advantages:** No head-of-line blocking; faster connection establishment; connection migration
- **IoT implications:** QUIC connection migration beneficial for mobile IoT devices; 0-RTT reduces reconnect latency
- **Constraint:** Still heavyweight for MCU-class devices

---

## WebSocket (RFC 6455)
Full-duplex communication over HTTP upgrade:
- **Handshake:** HTTP/1.1 GET with `Upgrade: websocket`
- **Frames:** Binary and text frames; ping/pong keepalive
- **IoT use:** MQTT over WebSocket (browsers, firewall-friendly); CoAP over WebSocket (RFC 8323); OCPP 1.6/2.0.1; Sparkplug B over MQTT/WS

---

## DDS (Data Distribution Service)
**OMG DDS** standard for real-time publish-subscribe in robotics and industrial IoT:
- **Domain:** Logical partition for participants
- **Topic:** Named, typed data channel (strongly typed — unlike MQTT)
- **DataWriter / DataReader:** Publishers and subscribers per topic
- **QoS policies:** 22 configurable policies (reliability, durability, deadline, liveliness, etc.)
- **Transport:** UDP multicast (default), TCP, shared memory
- **Security:** DDS Security spec (OMG DDS-SECURITY v1.1) — authentication, access control, encryption
- **Use cases:** Autonomous vehicles (ROS 2), industrial robotics, defence C2, medical devices
- **Implementations:** RTI Connext, FastDDS (eProsima), Eclipse Cyclone DDS, OpenDDS

---

## XMPP IoT Extensions
Extensible Messaging and Presence Protocol with IoT profiles:
- **XEP-0323:** IoT Sensor Data — reading sensor values
- **XEP-0325:** IoT Control — writing actuator values
- **XEP-0347:** IoT Discovery — device registry and discovery
- **Strengths:** Federation, existing identity infrastructure (JID), presence
- **Weaknesses:** XML overhead; less suitable for constrained devices
- **Use cases:** Smart building BMS, legacy industrial integration