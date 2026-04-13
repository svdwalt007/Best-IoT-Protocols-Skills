# Device Management Protocols

**Scope**: Remote configuration, firmware updates, monitoring, and lifecycle management of IoT devices and CPE (Customer Premises Equipment). Covers both legacy broadband gateway management (TR-069, TR-369) and modern cloud-native approaches (NETCONF/YANG, gRPC). Cross-references [lpwan.md](lpwan.md) for LwM2M, [cellular-wan.md](cellular-wan.md) for OMA DM, and [industrial-ot.md](industrial-ot.md) for SNMP in legacy PLCs.

## Table of Contents
1. TR-069 / CWMP (Broadband Forum)
2. TR-369 / USP (User Services Platform)
3. OMA DM v1.2/v2.0
4. NETCONF / YANG Stack
5. YANG Push (Telemetry Streaming)
6. SNMP v1/v2c/v3
7. gRPC for IoT Cloud Platforms
8. OMA GotAPI (Generic Open Terminal API)
9. uCIFI Smart City (LwM2M Extension)
10. oneM2M Interworking (LwM2M IPE)
11. OMA LwM2M Servers (Bootstrap, Registration, Management)

---

## 1. TR-069 / CWMP

**Reference**: BBF TR-069 Amendment 5 (2013), TR-098/135/181 data models

**Architecture**:
```
CPE (Customer Premises Equipment)      ACS (Auto-Configuration Server)
┌──────────────────────────────┐       ┌─────────────────────────────┐
│ TR-069 Agent                 │       │ Provisioning Engine         │
│   ├─ CWMP Session Handler    │ SOAP  │   ├─ Device Inventory      │
│   ├─ Data Model (TR-181)     │ HTTP  │   ├─ Firmware Repository   │
│   ├─ Parameter Database      │ TLS   │   └─ Policy Engine         │
│   └─ RPC Handler             │<─────>│                             │
│                              │       │ Northbound API              │
│ Connection Request Listener  │       │   └─ OSS/BSS Integration    │
│   └─ STUN/XMPP/HTTP:7547     │       │                             │
└──────────────────────────────┘       └─────────────────────────────┘
```

**CWMP RPC Methods**:
```
ACS → CPE (10 methods)                 CPE → ACS (7 methods)
├─ SetParameterValues                  ├─ Inform (periodic, boot, value change)
├─ GetParameterValues                  ├─ TransferComplete (firmware download)
├─ GetParameterNames                   ├─ RequestDownload (ACS-initiated DL)
├─ AddObject / DeleteObject            ├─ Kicked / RequestDownload
├─ Download (firmware/config)          └─ AutonomousTransferComplete
├─ Upload (logs/config)
├─ Reboot / FactoryReset
├─ ScheduleInform
└─ GetRPCMethods
```

**Data Model Hierarchy** (TR-181 Issue 2):
```
InternetGatewayDevice. (TR-098 legacy) → Device. (TR-181)
Device.
├─ DeviceInfo.
│  ├─ Manufacturer, ModelName, SerialNumber
│  ├─ SoftwareVersion, HardwareVersion
│  └─ ProvisioningCode (ACS-assigned grouping)
├─ ManagementServer.
│  ├─ URL (ACS endpoint)
│  ├─ Username / Password (HTTP Basic Auth)
│  ├─ PeriodicInformEnable / PeriodicInformInterval
│  ├─ ConnectionRequestURL (for ACS→CPE initiation)
│  └─ STUNEnable (NAT traversal)
├─ Time. (NTP: Server, Enable)
├─ IP. / PPP. / Ethernet. / WiFi. / MoCA.
├─ Firewall. / NAT. / Routing. / QoS.
├─ VoiceService. (TR-104: SIP profiles, codecs)
├─ STBService. (TR-135: IPTV multicast, CA)
└─ BulkData. (TR-157: HTTP/HTTPS/FTP bulk telemetry)
```

**Session Flow Example**:
```
1. CPE initiates HTTPS POST to ACS URL
   → Inform (EventCode: 2 PERIODIC, DeviceId, ParameterList)
2. ACS responds with InformResponse
3. ACS sends SetParameterValues (Device.ManagementServer.PeriodicInformInterval=3600)
4. CPE responds with SetParameterValuesResponse (Status=0 Success)
5. ACS sends empty SOAP envelope (signals session close)
6. CPE sends empty SOAP envelope (acknowledges)
```

**Connection Request** (ACS-initiated session):
```
ACS                                CPE (behind NAT)
│                                   │
├─ HTTP GET http://CPE_IP:7547      │ (if public IP)
│  Authorization: Digest ...        │
│                                   │
├─ STUN binding request (if NAT)    │
│  ↓                                │
│  STUN server relays to CPE        │
│                                   │
│                              <────┤ CPE opens HTTPS to ACS
│                              Inform│ (EventCode: 6 CONNECTION REQUEST)
```

**Use Cases**:
- DSL/cable modem provisioning (ISP mass deployment)
- Remote firmware upgrade (carrier-controlled CPE lifecycle)
- VoIP SIP profile injection (TR-104)
- IPTV service activation (TR-135 multicast IGMP proxy)

---

## 2. TR-369 / USP (User Services Platform)

**Reference**: BBF TR-369 Issue 1.3 (2023), TR-106 data model

**Motivation**: Replace SOAP/HTTP with modern encodings (protobuf) and transports (MQTT, WebSocket, STOMP). Support multi-controller federation, E2E encryption, and async operation tracking.

**Architecture**:
```
Agent (IoT Device, CPE)                Controller (Cloud Platform)
┌───────────────────────────┐          ┌────────────────────────────┐
│ USP Agent                 │          │ USP Controller             │
│   ├─ MTP Layer            │          │   ├─ MTP Client            │
│   │  ├─ MQTT 3.1.1/5.0    │ MQTTv5   │   │  (pub/sub to agent)    │
│   │  ├─ WebSocket         │ or WSS   │   │                        │
│   │  ├─ STOMP 1.2         │ or CoAP  │   ├─ Request Router        │
│   │  └─ CoAP              │          │   ├─ Subscription Manager  │
│   ├─ Record Layer         │<────────>│   └─ Firmware Orchestrator │
│   │  └─ E2ESession crypto │ protobuf │                            │
│   ├─ Message Layer        │          │ Northbound REST API        │
│   │  └─ USP Msg (proto)   │          │   └─ Multi-tenancy         │
│   └─ Data Model (TR-181)  │          │                            │
└───────────────────────────┘          └────────────────────────────┘
```

**USP Record Structure** (protobuf outer envelope):
```protobuf
message Record {
  string version = 1;           // "1.3"
  string to_id = 2;             // endpoint_id of recipient
  string from_id = 3;           // endpoint_id of sender
  PayloadSecurity payload_security = 4; // PLAINTEXT or TLS12
  bytes mac_signature = 5;      // HMAC-SHA256 if E2ESession
  bytes sender_cert = 6;        // X.509 if TLS
  oneof record_type {
    NoSessionContextRecord no_session_context = 7;
    SessionContextRecord session_context = 8;
  }
}
```

**USP Msg Structure** (protobuf inner payload):
```protobuf
message Msg {
  Header header = 1;
  Body body = 2;
}

message Header {
  string msg_id = 1;            // UUID for request correlation
  enum MsgType {
    ERROR = 0; GET = 1; GET_RESP = 2; NOTIFY = 3;
    SET = 4; SET_RESP = 5; OPERATE = 6; OPERATE_RESP = 7;
    ADD = 8; ADD_RESP = 9; DELETE = 10; DELETE_RESP = 11;
    GET_SUPPORTED_DM = 12; GET_INSTANCES = 14; NOTIFY_RESP = 15;
    GET_SUPPORTED_PROTOCOL = 16;
  }
  MsgType msg_type = 2;
}
```

**MTP Bindings**:

**MQTT 5.0 Binding**:
```
Topic Structure:
  usp/v1/{to_endpoint_id}/agent     (Controller → Agent)
  usp/v1/{to_endpoint_id}/controller (Agent → Controller)

MQTT PUBLISH properties:
  - ResponseTopic: usp/v1/{from_endpoint_id}/controller
  - ContentType: application/vnd.bbf.usp.msg
  - Retain: false (USP messages are ephemeral)

Example:
  Controller publishes Record to:
    usp/v1/os::01234-567890ABCDEF/agent
  Agent subscribes to:
    usp/v1/os::01234-+/agent (wildcard for multiple controllers)
```

**WebSocket Binding** (RFC 6455):
```
WSS handshake:
  GET /usp/v1/agent HTTP/1.1
  Upgrade: websocket
  Sec-WebSocket-Protocol: v1.usp

Framing:
  - Binary frames carry USP Record (protobuf serialized)
  - Ping/Pong for keep-alive (30s interval recommended)
```

**STOMP 1.2 Binding**:
```
CONNECT frame:
  accept-version: 1.2
  heart-beat: 10000,10000

SUBSCRIBE:
  destination: /usp/v1/agent/{endpoint_id}
  ack: auto

SEND:
  destination: /usp/v1/controller/{endpoint_id}
  content-type: application/vnd.bbf.usp.msg
```

**USP Operations**:
```
Get (read parameters)
  Request:  path = "Device.WiFi.Radio.*.Channel"
  Response: Device.WiFi.Radio.1.Channel = 6
            Device.WiFi.Radio.2.Channel = 36

Set (write parameters)
  Request:  Device.WiFi.SSID.1.Enable = true
            Device.WiFi.SSID.1.SSID = "MyNetwork"
  Response: oper_success with affected paths

Add (instantiate object)
  Request:  Device.WiFi.AccessPoint.{i}
            SSID = "Guest", Enable = true
  Response: Device.WiFi.AccessPoint.3 (instance number)

Delete (remove object)
  Request:  Device.WiFi.AccessPoint.3
  Response: oper_success

Operate (async command)
  Request:  Device.SelfTestDiagnostics() → command_key = "diag_20231201"
  Response: OperateResp with req_obj_path, command_key
  Notify:   Event OperationComplete (success/failure)

GetSupportedDM (schema introspection)
  Request:  path = "Device.LocalAgent."
  Response: Object/Parameter tree with access (read-only/read-write)
```

**Subscription & Notification**:
```
Controller sends Add to Device.LocalAgent.Subscription.{i}
  NotifType: ValueChange
  ReferenceList: Device.WiFi.Radio.1.Channel

Agent sends Notify when channel changes:
  Msg {
    header { msg_type: NOTIFY }
    body {
      notify {
        subscription_id: "sub-001"
        value_change {
          param_path: "Device.WiFi.Radio.1.Channel"
          param_value: "11"
        }
      }
    }
  }
```

**E2E Session Encryption** (optional):
```
1. Agent sends SessionContextRecord with session_id
2. Controller derives session keys via KDF(shared_secret, nonce)
3. Subsequent Records encrypted with AES-128-CBC
4. HMAC-SHA256 for integrity (mac_signature field)
```

**Migration from TR-069**:
- Controller can proxy CWMP RPC to USP (TR-069 Annex T)
- Data model remains TR-181 (Device. root)
- Benefits: async operations, multi-controller, modern transports

---

## 3. OMA DM v1.2 / v2.0

**Reference**: OMA-TS-DM-Protocol-V1_2, OMA-TS-DM_Protocol-V2_0

**SyncML XML Format** (v1.2):
```xml
<SyncML xmlns="SYNCML:SYNCML1.2">
  <SyncHdr>
    <VerDTD>1.2</VerDTD>
    <SessionID>1</SessionID>
    <MsgID>1</MsgID>
    <Target><LocURI>http://server.com/dm</LocURI></Target>
    <Source><LocURI>IMEI:123456789012345</LocURI></Source>
    <Cred><!-- MD5 digest --></Cred>
  </SyncHdr>
  <SyncBody>
    <Alert>
      <CmdID>1</CmdID>
      <Data>1200</Data> <!-- Client-initiated session -->
    </Alert>
    <Final/>
  </SyncBody>
</SyncML>
```

**Management Tree**:
```
. (root)
├─ DevInfo (device metadata)
│  ├─ DevID (IMEI)
│  ├─ Man (manufacturer)
│  ├─ Mod (model)
│  └─ SwV (software version)
├─ DevDetail
│  ├─ URI (./DevDetail/URI)
│  └─ Bearer (GSM, WCDMA, LTE)
└─ ./Vendor/AppSettings (OEM-specific nodes)
```

**Commands**:
- **Add**: Create node
- **Replace**: Update leaf value
- **Get**: Retrieve node data
- **Delete**: Remove node
- **Exec**: Execute command (e.g., firmware update)
- **Alert**: Generic event (1200 = session start, 1226 = firmware ready)

**Use Case**: Legacy cellular M2M (2G/3G modules). Largely replaced by LwM2M (see [lpwan.md](lpwan.md)).

---

## 4. NETCONF / YANG Stack

**References**:
- RFC 6241 (NETCONF 1.1)
- RFC 7950 (YANG 1.1)
- RFC 8040 (RESTCONF)
- RFC 8342 (NMDA - Network Management Datastore Architecture)

**NETCONF Architecture**:
```
NETCONF Client (Manager)               NETCONF Server (Agent)
┌──────────────────────────┐           ┌─────────────────────────────┐
│ Application Layer        │           │ YANG Data Models            │
│   ├─ Python ncclient     │           │   ├─ ietf-interfaces        │
│   ├─ Ansible netconf     │           │   ├─ openconfig-system      │
│   └─ OpenDaylight        │           │   └─ vendor-proprietary     │
├──────────────────────────┤           ├─────────────────────────────┤
│ NETCONF Operations       │ XML-RPC   │ Operations Layer            │
│   ├─ <get-config>        │ SSH port  │   ├─ Config validation      │
│   ├─ <edit-config>       │   830     │   ├─ Transaction mgmt       │
│   ├─ <copy-config>       │           │   └─ Candidate/Running/Start│
│   └─ <rpc>               │           ├─────────────────────────────┤
├──────────────────────────┤           │ Datastore (NMDA)            │
│ Transport: SSH 2.0       │<─────────>│   ├─ <running>              │
│   └─ netconf subsystem   │           │   ├─ <candidate>            │
└──────────────────────────┘           │   ├─ <startup>              │
                                       │   ├─ <intended> (RFC 8342)  │
                                       │   └─ <operational> (ro)     │
                                       └─────────────────────────────┘
```

**Session Initialization**:
```xml
<!-- Server sends capabilities -->
<hello xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <capabilities>
    <capability>urn:ietf:params:netconf:base:1.1</capability>
    <capability>urn:ietf:params:netconf:capability:writable-running:1.0</capability>
    <capability>urn:ietf:params:netconf:capability:candidate:1.0</capability>
    <capability>urn:ietf:params:netconf:capability:confirmed-commit:1.1</capability>
    <capability>urn:ietf:params:netconf:capability:xpath:1.0</capability>
    <capability>urn:ietf:params:netconf:capability:notification:1.0</capability>
    <capability>http://example.com/iot-device?module=acme-sensor&amp;revision=2023-01-15</capability>
  </capabilities>
  <session-id>42</session-id>
</hello>

<!-- Client responds with its capabilities -->
<hello xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <capabilities>
    <capability>urn:ietf:params:netconf:base:1.1</capability>
  </capabilities>
</hello>
```

**NETCONF Operations**:

**1. `<get-config>`** (read configuration datastore):
```xml
<rpc message-id="101" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <get-config>
    <source><running/></source>
    <filter type="subtree">
      <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces">
        <interface>
          <name>eth0</name>
        </interface>
      </interfaces>
    </filter>
  </get-config>
</rpc>

<rpc-reply message-id="101" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <data>
    <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces">
      <interface>
        <name>eth0</name>
        <type>ethernetCsmacd</type>
        <enabled>true</enabled>
        <ipv4>
          <address>
            <ip>192.168.1.100</ip>
            <prefix-length>24</prefix-length>
          </address>
        </ipv4>
      </interface>
    </interfaces>
  </data>
</rpc-reply>
```

**2. `<edit-config>`** (modify configuration):
```xml
<rpc message-id="102" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <edit-config>
    <target><candidate/></target>
    <default-operation>merge</default-operation>
    <config>
      <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces">
        <interface>
          <name>eth0</name>
          <enabled>false</enabled> <!-- Disable interface -->
        </interface>
      </interfaces>
    </config>
  </edit-config>
</rpc>

<rpc-reply message-id="102" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <ok/>
</rpc-reply>

<!-- Commit candidate to running -->
<rpc message-id="103" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <commit>
    <confirmed/>
    <confirm-timeout>120</confirm-timeout> <!-- Auto-rollback if not confirmed -->
  </commit>
</rpc>
```

**3. `<notification>`** (RFC 5277):
```xml
<!-- Subscription -->
<rpc message-id="104" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <create-subscription xmlns="urn:ietf:params:xml:ns:netconf:notification:1.0">
    <stream>NETCONF</stream>
  </create-subscription>
</rpc>

<!-- Server sends notifications -->
<notification xmlns="urn:ietf:params:xml:ns:netconf:notification:1.0">
  <eventTime>2023-12-01T10:30:00Z</eventTime>
  <netconf-config-change xmlns="urn:ietf:params:xml:ns:yang:ietf-netconf-notifications">
    <changed-by>
      <username>admin</username>
      <session-id>42</session-id>
    </changed-by>
    <datastore>running</datastore>
    <edit>
      <target>/interfaces/interface[name='eth0']/enabled</target>
      <operation>merge</operation>
    </edit>
  </netconf-config-change>
</notification>
```

**YANG 1.1 Module Structure**:
```yang
module acme-sensor {
  namespace "http://acme.com/ns/sensor";
  prefix "sensor";

  import ietf-inet-types { prefix inet; }

  revision 2023-01-15 { description "Initial release"; }

  container measurements {
    config false; // read-only operational data

    leaf temperature {
      type decimal64 { fraction-digits 2; }
      units "celsius";
      description "Ambient temperature";
    }

    leaf-list co2-ppm {
      type uint16 { range "400..5000"; }
      max-elements 10;
      description "CO2 concentration history";
    }
  }

  container settings {
    leaf sample-interval {
      type uint32 { range "1..3600"; }
      units "seconds";
      default 60;
      description "Measurement sampling interval";
    }

    leaf ntp-server {
      type inet:host;
      default "pool.ntp.org";
    }
  }

  rpc calibrate {
    input {
      leaf reference-temp {
        type decimal64 { fraction-digits 2; }
        mandatory true;
      }
    }
    output {
      leaf status {
        type enumeration {
          enum success;
          enum failed;
          enum in-progress;
        }
      }
    }
  }

  notification threshold-exceeded {
    leaf parameter {
      type enumeration { enum temperature; enum co2; }
    }
    leaf value { type decimal64 { fraction-digits 2; } }
    leaf threshold { type decimal64 { fraction-digits 2; } }
  }
}
```

**RESTCONF** (RFC 8040):
HTTP/REST mapping of NETCONF operations:
```http
GET /restconf/data/ietf-interfaces:interfaces/interface=eth0
Accept: application/yang-data+json

HTTP/1.1 200 OK
Content-Type: application/yang-data+json

{
  "ietf-interfaces:interface": {
    "name": "eth0",
    "type": "iana-if-type:ethernetCsmacd",
    "enabled": true,
    "ietf-ip:ipv4": {
      "address": [
        { "ip": "192.168.1.100", "prefix-length": 24 }
      ]
    }
  }
}

PATCH /restconf/data/ietf-interfaces:interfaces/interface=eth0
Content-Type: application/yang-data+json

{ "ietf-interfaces:interface": { "enabled": false } }
```

---

## 5. YANG Push (Telemetry Streaming)

**Reference**: RFC 8641 (Subscribed Notifications for YANG Datastores)

**Subscription Types**:
```
Configured Subscription (pre-provisioned on device)
  ├─ XPath filter: /interfaces/interface[enabled='true']/statistics
  ├─ Period: 10s (periodic push)
  └─ Transport: gRPC / UDP / HTTPS

Dynamic Subscription (on-demand via NETCONF/RESTCONF)
  ├─ Established via <establish-subscription> RPC
  ├─ On-change notification (only when data changes)
  └─ Dampening period: 5s (suppress high-frequency updates)
```

**On-Change Subscription Example**:
```xml
<!-- Client establishes subscription -->
<rpc message-id="201" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <establish-subscription xmlns="urn:ietf:params:xml:ns:yang:ietf-subscribed-notifications"
                          xmlns:yp="urn:ietf:params:xml:ns:yang:ietf-yang-push">
    <yp:datastore xmlns:ds="urn:ietf:params:xml:ns:yang:ietf-datastores">ds:operational</yp:datastore>
    <yp:datastore-xpath-filter xmlns:if="urn:ietf:params:xml:ns:yang:ietf-interfaces">
      /if:interfaces/if:interface/if:statistics
    </yp:datastore-xpath-filter>
    <yp:on-change>
      <yp:dampening-period>5000</yp:dampening-period> <!-- 5s in ms -->
    </yp:on-change>
  </establish-subscription>
</rpc>

<rpc-reply message-id="201" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <subscription-id xmlns="urn:ietf:params:xml:ns:yang:ietf-subscribed-notifications">1001</subscription-id>
</rpc-reply>

<!-- Server sends push-change-update notification -->
<notification xmlns="urn:ietf:params:xml:ns:netconf:notification:1.0">
  <eventTime>2023-12-01T11:00:05Z</eventTime>
  <push-change-update xmlns="urn:ietf:params:xml:ns:yang:ietf-yang-push">
    <subscription-id>1001</subscription-id>
    <datastore-contents>
      <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces">
        <interface>
          <name>eth0</name>
          <statistics>
            <in-octets>1234567890</in-octets>
            <out-octets>987654321</out-octets>
          </statistics>
        </interface>
      </interfaces>
    </datastore-contents>
  </push-change-update>
</notification>
```

**gRPC Transport** (gNMI - gRPC Network Management Interface):
```protobuf
service gNMI {
  rpc Subscribe(stream SubscribeRequest) returns (stream SubscribeResponse);
}

message SubscribeRequest {
  oneof request {
    SubscriptionList subscribe = 1;
    Poll poll = 3;
  }
}

message SubscriptionList {
  repeated Subscription subscription = 1;
  enum Mode { STREAM = 0; ONCE = 1; POLL = 2; }
  Mode mode = 2;

  message Subscription {
    Path path = 1; // e.g., /interfaces/interface[name=eth0]/state/counters
    enum Mode { TARGET_DEFINED = 0; ON_CHANGE = 1; SAMPLE = 2; }
    Mode mode = 2;
    uint64 sample_interval = 3; // nanoseconds
  }
}
```

---

## 6. SNMP v1/v2c/v3

**Reference**: RFC 3411-3418 (SNMPv3), RFC 1157 (SNMPv1)

**Protocol Stack**:
```
┌────────────────────────────────────────┐
│ SNMP Application (GetRequest, SetRequest, Trap) │
├────────────────────────────────────────┤
│ SNMPv3 Security (USM + VACM)           │
│   ├─ USM: User-based Security Model    │
│   │   └─ authPriv (MD5/SHA + DES/AES)  │
│   └─ VACM: View-based Access Control   │
├────────────────────────────────────────┤
│ Message Processing (v1/v2c/v3)         │
├────────────────────────────────────────┤
│ Transport: UDP/161 (agent) 162 (trap)  │
└────────────────────────────────────────┘
```

**SNMPv3 Message Format**:
```
msgGlobalData {
  msgID, msgMaxSize, msgFlags (authFlag, privFlag), msgSecurityModel
}
msgSecurityParameters (USM) {
  msgAuthoritativeEngineID, msgAuthoritativeEngineBoots,
  msgAuthoritativeEngineTime, msgUserName,
  msgAuthenticationParameters (HMAC-MD5/SHA), msgPrivacyParameters (DES/AES)
}
scopedPDU {
  contextEngineID, contextName,
  PDU (GetRequest, GetNextRequest, SetRequest, GetBulkRequest, InformRequest, SNMPv2-Trap)
}
```

**Common PDUs**:
```
GetRequest:      Retrieve single OID value
GetNextRequest:  Walk MIB tree (lexicographic successor)
GetBulkRequest:  Efficient batch retrieval (v2c/v3)
SetRequest:      Modify writable OID
Trap/InformRequest: Async notification (v2c adds acknowledgment)
```

**MIB-II (RFC 1213) Example**:
```
system.sysDescr.0         = "Cisco IOS 15.2"
system.sysObjectID.0      = 1.3.6.1.4.1.9.1.1 (Cisco enterprise OID)
system.sysUpTime.0        = 12345678 (hundredths of seconds)
system.sysContact.0       = "admin@example.com"

interfaces.ifNumber.0     = 4
interfaces.ifTable.ifEntry.1.ifDescr       = "FastEthernet0/0"
interfaces.ifTable.ifEntry.1.ifOperStatus  = 1 (up)
interfaces.ifTable.ifEntry.1.ifInOctets    = 1234567890
```

**SNMPv3 User Configuration**:
```
User: "admin"
  authProtocol: usmHMACSHAAuthProtocol
  authKey: sha1("MyAuthPassphrase")
  privProtocol: usmAesCfb128Protocol
  privKey: aes_key("MyPrivPassphrase")

VACM (View-based Access Control):
  securityName: admin
  securityModel: 3 (USM)
  securityLevel: authPriv
  read view: "internet" (subtree: 1.3.6.1)
  write view: "restricted" (subtree: 1.3.6.1.4.1.9999)
```

**Use Case**: Legacy industrial PLCs, network switches, UPS monitoring. See [industrial-ot.md](industrial-ot.md) for OT context.

---

## 7. gRPC for IoT Cloud Platforms

**Reference**: gRPC Core, protobuf3

**Advantages**:
- HTTP/2 multiplexing (multiple streams over single TCP connection)
- Bidirectional streaming (real-time telemetry + command injection)
- Strong typing (protobuf schema validation)
- Code generation (client stubs in 10+ languages)

**Device Management Service Example**:
```protobuf
syntax = "proto3";
package iot.device;

service DeviceManager {
  rpc GetDeviceConfig(DeviceConfigRequest) returns (DeviceConfig);
  rpc UpdateDeviceConfig(DeviceConfig) returns (UpdateResponse);
  rpc StreamTelemetry(TelemetryRequest) returns (stream TelemetryEvent);
  rpc SendCommand(stream Command) returns (stream CommandResponse); // bidirectional
}

message DeviceConfigRequest {
  string device_id = 1; // UUID or MAC
}

message DeviceConfig {
  string device_id = 1;
  map<string, string> parameters = 2; // key-value config
  string firmware_version = 3;
  google.protobuf.Timestamp last_updated = 4;
}

message TelemetryEvent {
  string device_id = 1;
  google.protobuf.Timestamp timestamp = 2;
  oneof payload {
    SensorData sensor = 3;
    SystemMetrics system = 4;
  }
}

message SensorData {
  float temperature = 1;
  float humidity = 2;
  uint32 co2_ppm = 3;
}

message Command {
  string device_id = 1;
  string command_id = 2; // correlation ID
  oneof action {
    Reboot reboot = 3;
    FirmwareUpdate firmware_update = 4;
    ConfigChange config_change = 5;
  }
}

message CommandResponse {
  string command_id = 1;
  enum Status { PENDING = 0; SUCCESS = 1; FAILED = 2; }
  Status status = 2;
  string error_message = 3;
}
```

**Bidirectional Streaming Example** (Python):
```python
import grpc
from iot.device import device_manager_pb2, device_manager_pb2_grpc

channel = grpc.secure_channel('iot.example.com:443',
                               grpc.ssl_channel_credentials())
stub = device_manager_pb2_grpc.DeviceManagerStub(channel)

# Bidirectional stream
def request_generator():
    while True:
        response = stub.SendCommand(request_generator())
        for cmd_response in response:
            print(f"Command {cmd_response.command_id}: {cmd_response.status}")

# Telemetry stream
for telemetry in stub.StreamTelemetry(device_manager_pb2.TelemetryRequest(device_id="dev001")):
    print(f"{telemetry.timestamp}: Temp={telemetry.sensor.temperature}°C")
```

**Authentication**:
- **TLS**: Mutual TLS (client cert + server cert)
- **Token**: OAuth 2.0 bearer token in metadata (`authorization: Bearer <JWT>`)
- **API Key**: Custom metadata (`x-api-key: <key>`)

---

## 8. OMA GotAPI (Generic Open Terminal API)

**Reference**: OMA-TS-REST_NetAPI_Common-V1_0

**Purpose**: RESTful API abstraction over device-specific protocols (SIM toolkit, Bluetooth, NFC).

**Example Endpoint**:
```http
GET /terminal/v1/bluetooth/devices
Authorization: Bearer <token>

Response:
{
  "devices": [
    { "address": "00:11:22:33:44:55", "name": "HeartRate Sensor", "rssi": -60 }
  ]
}

POST /terminal/v1/bluetooth/devices/00:11:22:33:44:55/connect
{ "profile": "gatt" }
```

**Use Case**: Mobile app interfacing with phone's hardware (mostly deprecated, superseded by Web Bluetooth API).

---

## 9. uCIFI Smart City (LwM2M Extension)

**Reference**: OMA LwM2M Object 10241 (Outdoor Lamp Controller)

**LwM2M Object 10241**:
```
Resource ID | Name                     | Type    | Access
------------|--------------------------|---------|-------
5850        | On/Off                   | Boolean | RW
5851        | Dimmer (%)               | Integer | RW
5706        | Colour (RGB hex)         | String  | RW
5852        | On Time (cumulative sec) | Integer | R
5820        | Power Factor             | Float   | R
5805        | Active Power (W)         | Float   | R
```

**Smart Streetlight Control**:
```
LwM2M Write /10241/0/5850 = true   (turn on)
LwM2M Write /10241/0/5851 = 50     (dim to 50%)
LwM2M Observe /10241/0/5805        (monitor power consumption)
```

Cross-reference: [lpwan.md](lpwan.md) section 1.4 (LwM2M Object Model).

---

## 10. oneM2M Platform (IoT Service Layer Standard)

**Reference**: ETSI TS 118.1xx / ISO/IEC 30128 series (oneM2M Release 4/5)

**Purpose**: oneM2M is a comprehensive IoT platform standard providing a horizontal service layer between IoT applications and underlying connectivity protocols. It's the standardized equivalent of commercial IoT platforms (AWS IoT Core, Azure IoT Hub) — enabling vendor-neutral IoT deployments.

**Standardization Bodies**: Joint standards initiative of 8 SDOs (ETSI, ARIB, ATIS, CCSA, TIA, TSDSI, TTA, TTC) — recognized globally

**oneM2M Architecture**:

```
┌────────────────────────────────────────────────────────────────┐
│                    APPLICATION LAYER                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Application Entity (AE)                                 │   │
│  │   - Smart City Dashboard, Fleet Management App         │   │
│  │   - Analytics Engine, ML Inference                     │   │
│  └──────────────────────┬──────────────────────────────────┘   │
│                         │ Mca (Application Interface)          │
├─────────────────────────▼──────────────────────────────────────┤
│              COMMON SERVICES ENTITY (CSE)                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Common Service Functions (CSFs):                        │  │
│  │   ├─ Registration (resource tree management)            │  │
│  │   ├─ Discovery (query-based resource search)            │  │
│  │   ├─ Subscription/Notification (event-driven)           │  │
│  │   ├─ Data Management (container, contentInstance CRUD)  │  │
│  │   ├─ Device Management (firmware update, reboot)        │  │
│  │   ├─ Security (authentication, authorization, ACL)      │  │
│  │   ├─ Group Management (bulk operations)                 │  │
│  │   ├─ Location (geospatial queries, proximity search)    │  │
│  │   └─ Semantics (ontology-based discovery, NGSI-LD)      │  │
│  └──────────────────────────────────────────────────────────┘  │
│                         │ Mcc (CSE-to-CSE)                      │
├─────────────────────────▼──────────────────────────────────────┤
│              NETWORK SERVICES LAYER                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Underlying Network Service Entity (NSE)                 │  │
│  │   - Transport: HTTP, CoAP, MQTT, WebSocket              │  │
│  │   - Security: TLS, DTLS, OSCORE                          │  │
│  │   - Connectivity: Cellular, Wi-Fi, LoRaWAN, etc.        │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

**Resource Model** (Hierarchical Tree):

```
CSE (Common Services Entity)
  ├─ <CSEBase> (root resource, CSE-ID: /CSE001)
  │
  ├─ <AE> (Application Entity: /CSE001/AE_SmartCityApp)
  │  ├─ <container> (data storage: /CSE001/AE_SmartCityApp/Temperature)
  │  │  ├─ <contentInstance> (timestamped sensor reading, immutable)
  │  │  ├─ <contentInstance> (next reading)
  │  │  └─ <subscription> (notify on new content)
  │  │
  │  ├─ <container> (Humidity)
  │  ├─ <group> (Sensor_Group_ZoneA)
  │  └─ <accessControlPolicy> (who can read/write this AE's resources)
  │
  ├─ <AE> (Device: /CSE001/AE_TempSensor01)
  │  ├─ <container> (SensorData)
  │  └─ <mgmtObj> (firmware version, battery, memory)
  │
  ├─ <node> (physical device representation)
  │  └─ <mgmtObj> (device management: firmware, reboot, diagnostics)
  │
  └─ <remoteCSE> (federated CSE: /CSE001/CSE002_EdgeGateway)
     └─ (resource tree of edge CSE)
```

**Reference Points** (Interfaces):

| Ref Point | Connection | Transport | Use Case |
|-----------|------------|-----------|----------|
| **Mca** | AE ↔ CSE | HTTP, CoAP, MQTT, WebSocket | Application subscribes to sensor data, sends commands |
| **Mcc** | CSE ↔ CSE | HTTP, CoAP | Federation (cloud CSE ↔ edge CSE), resource discovery across CSEs |
| **Mcc'** | CSE ↔ CSE (different SP) | HTTP | Service Provider interworking (roaming) |
| **Mcn** | CSE ↔ NSE | Protocol-specific | CSE uses underlying network (LTE, NB-IoT, Wi-Fi) |

**Protocol Bindings**:

```
HTTP Binding (default):
  - RESTful CRUD: POST (CREATE), GET (RETRIEVE), PUT (UPDATE), DELETE
  - URL structure: http://{cse-host}:8080/~/CSE001/AE_App/Temp
  - Content-Type: application/json, application/cbor, application/xml

CoAP Binding:
  - Lightweight for constrained devices
  - DTLS security
  - Observe for subscriptions (CoAP Observe → oneM2M <subscription>)

MQTT Binding:
  - Topic structure: /oneM2M/{operation}/{to}/{from}
  - QoS 0/1 for telemetry vs commands
  - Retained messages for device shadow (last-known-state)

WebSocket Binding:
  - Bi-directional for real-time applications
  - Server-push notifications without polling
```

**CRUD Operations** (RESTful Semantics):

```http
CREATE <container>:
  POST http://cse.example.com/~/CSE001/AE_App
  Content-Type: application/json; ty=3
  Body:
  {
    "m2m:cnt": {
      "rn": "Temperature",  // Resource Name
      "mni": 100,           // Max Number of Instances (auto-delete oldest)
      "mbs": 10000          // Max Byte Size
    }
  }
  Response: 2001 Created, Location: /CSE001/AE_App/Temperature

CREATE <contentInstance> (sensor reading):
  POST http://cse.example.com/~/CSE001/AE_App/Temperature
  Content-Type: application/json; ty=4
  Body:
  {
    "m2m:cin": {
      "con": "23.5",        // Content (sensor value)
      "cnf": "text/plain",  // Content Info (MIME type)
      "lbl": ["zone-A", "floor-3"]  // Labels (semantic tags)
    }
  }
  Response: 2001 Created, ri: /CSE001/AE_App/Temperature/cin_12345

RETRIEVE <container> (latest data):
  GET http://cse.example.com/~/CSE001/AE_App/Temperature/la
  (la = latest contentInstance)
  Response:
  {
    "m2m:cin": {
      "ri": "cin_12345",
      "ct": "20240115T103000",  // Creation Time
      "con": "23.5"
    }
  }

SUBSCRIBE to <container>:
  POST http://cse.example.com/~/CSE001/AE_App/Temperature
  Content-Type: application/json; ty=23
  Body:
  {
    "m2m:sub": {
      "rn": "TempAlertSub",
      "nu": ["http://app.example.com/notify"],  // Notification URI
      "enc": {  // Event Notification Criteria
        "net": [3],  // Notification Event Type: 3 = Update of Resource
        "om": {      // Operation Monitor
          "ops": [1]  // 1 = CREATE operation
        }
      }
    }
  }
  → When new <cin> created, CSE POSTs notification to http://app.example.com/notify

DISCOVERY (find all temperature sensors in zone-A):
  GET http://cse.example.com/~/CSE001?fu=1&ty=4&lbl=temperature&lbl=zone-A
  (fu=1 = filterUsage: discovery, ty=4 = contentInstance, lbl = label match)
  Response: List of URIs matching query
```

**Subscription/Notification**:

```
Notification Triggers:
  ├─ CREATE of resource (new sensor reading)
  ├─ UPDATE of attribute (device status change)
  ├─ DELETE of resource (device deregistered)
  ├─ Batch notification (collect N events before notify)
  └─ Rate limit (notify at most once per X seconds)

Notification Delivery:
  ├─ HTTP POST to AE callback URL
  ├─ CoAP CON message
  ├─ MQTT publish to subscribed topic
  └─ Blocking notification (wait for AE acknowledgment)

Event Notification Content:
  {
    "m2m:sgn": {  // Notification
      "nev": {    // Notification Event
        "rep": {  // Representation (full resource or delta)
          "m2m:cin": { "con": "24.1" }
        },
        "net": 3  // CREATE
      },
      "sur": "/CSE001/AE_App/Temperature/sub_001"  // Subscription URI
    }
  }
```

**Security**:

```
Authentication:
  ├─ M2M-Token (bearer token in HTTP header: X-M2M-Origin)
  ├─ OAuth 2.0 (for AE ↔ CSE)
  └─ Certificate-based (mTLS for CSE ↔ CSE)

Authorization (Access Control Policies):
  <accessControlPolicy>:
    ├─ privileges: CREATE, RETRIEVE, UPDATE, DELETE, NOTIFY, DISCOVER
    ├─ accessControlOriginators: [AE_ID list, wildcards]
    └─ accessControlContexts: time-based, location-based

Example:
  {
    "m2m:acp": {
      "pv": {  // Privileges
        "acr": [
          {
            "acor": ["AE_Admin", "AE_Analyst"],  // Origins
            "acop": 63  // Binary: 111111 = all operations
          },
          {
            "acor": ["AE_Viewer"],
            "acop": 2  // Binary: 000010 = RETRIEVE only
          }
        ]
      }
    }
  }

Encryption:
  ├─ TLS 1.3 for HTTP/WebSocket
  ├─ DTLS 1.3 for CoAP
  └─ End-to-end: Application-layer encryption in <cin> content field
```

**Device Management** (mgmtObj):

```
<node> resource represents physical device:
  ├─ <mgmtObj> firmware
  │  └─ version, URL, update, state (IDLE, DOWNLOADING, DOWNLOADED, UPDATING)
  ├─ <mgmtObj> battery
  │  └─ level, status (charging, discharging, full)
  ├─ <mgmtObj> memory
  │  └─ available, used
  ├─ <mgmtObj> reboot
  │  └─ trigger, factoryReset
  └─ <mgmtObj> deviceInfo
     └─ manufacturer, model, serialNumber, firmwareVersion

Firmware Update Flow:
  1. AE updates <firmware> mgmtObj with Package URL
  2. <node> downloads firmware (state: DOWNLOADING)
  3. <node> verifies signature (state: DOWNLOADED)
  4. AE triggers update command (state: UPDATING)
  5. <node> reboots, applies firmware
  6. <node> reports new version (state: IDLE)
```

**Interworking with Other Protocols** (IPE — Interworking Proxy Entity):

```
LwM2M IPE (ETSI TS 118 114):
  LwM2M Device (CoAP, Object 3303)
    ↓ LwM2M IPE translates
  oneM2M <container> + <contentInstance>

MQTT IPE:
  MQTT Device (topic: sensors/temp)
    ↓ MQTT IPE translates
  oneM2M <AE> + <container>

HTTP IPE:
  RESTful sensor (GET /api/temperature)
    ↓ HTTP IPE polls and translates
  oneM2M <contentInstance>

Benefit: Heterogeneous devices unified under single oneM2M resource tree
```

**Semantic Interoperability** (oneM2M Base Ontology):

```
Ontology-based discovery using NGSI-LD:
  - Device capabilities described via semantic tags
  - Standardized vocabulary (SAREF, SSN/SOSA)
  - Query: "Find all actuators controlling water valves in Building A"
    → Returns resources with semantic annotations matching query

Example Annotation:
  <container> labeled with:
    - schema:device = saref:TemperatureSensor
    - schema:location = geo:Building_A_Floor_3
    - schema:measuredProperty = qudt:Temperature
```

**Real-World Deployments**:

```
1. OCEAN (Open Connectome Engine) — Smart City Platform (South Korea)
   ├─ 10M+ IoT devices (traffic, parking, environment)
   ├─ oneM2M CSE at city level, edge CSEs at district level
   └─ Multi-vendor device integration via IPEs

2. FIWARE + oneM2M — European Smart City Deployments
   ├─ FIWARE Context Broker ↔ oneM2M CSE integration
   ├─ NGSI-LD semantic layer on top of oneM2M
   └─ Cities: Santander, Aarhus, Helsinki

3. KDDI IoT Platform (Japan) — Carrier-Grade oneM2M
   ├─ NB-IoT devices → oneM2M CSE (LwM2M IPE)
   ├─ 5M+ connected devices (utilities, logistics, agriculture)
   └─ Multi-tenant CSE with operator-level federation

4. AIUT (China) — Industrial IoT
   ├─ Manufacturing execution systems (MES) via oneM2M
   ├─ OPC UA IPE for PLC integration
   └─ Cross-factory data federation via Mcc
```

**oneM2M vs Commercial IoT Platforms**:

| Feature | oneM2M | AWS IoT Core | Azure IoT Hub |
|---------|--------|--------------|---------------|
| Standardization | SDO standard (vendor-neutral) | Proprietary | Proprietary |
| Resource Model | Hierarchical tree (<AE>, <container>, <cin>) | Thing Shadow (JSON doc) | Device Twin (JSON doc) |
| Federation | Native (Mcc reference point) | Cross-account roles (AWS-specific) | IoT Hub federation (Azure-specific) |
| Protocol Support | HTTP, CoAP, MQTT, WebSocket (standard bindings) | MQTT, HTTPS, WebSocket | MQTT, AMQP, HTTPS |
| Semantic Discovery | Native (ontology-based) | Via Lambda + Elasticsearch | Via Azure Digital Twins |
| Device Management | mgmtObj (firmware, reboot, battery) | Jobs, Fleet Indexing | Device Twin desired/reported |
| Deployment | Self-hosted or cloud | Cloud-only (AWS) | Cloud-only (Azure) |
| Interoperability | IPEs for LwM2M, MQTT, HTTP, OPC UA | SDK-based (device-specific) | SDK-based (device-specific) |

**Benefits of oneM2M**:
- **Vendor Neutrality**: Avoid cloud provider lock-in; deploy on-premise, edge, or multi-cloud
- **Horizontal Platform**: Single service layer across all IoT verticals (smart city, industrial, healthcare)
- **Global Standard**: Recognized by EU, Asia-Pacific governments for public IoT infrastructure
- **Longevity**: Standards-based architecture outlives proprietary platforms

**oneM2M Release Roadmap**:
```
Release 1 (2014): Core architecture, HTTP/CoAP bindings
Release 2 (2016): MQTT, semantics, interworking (LwM2M IPE)
Release 3 (2018): Edge computing, cross-domain discovery
Release 4 (2021): 5G integration, AI/ML at edge, time-series data
Release 5 (2023-2024): Enhanced security, NGSI-LD alignment, blockchain integration
```

Cross-reference: [lpwan.md](lpwan.md) for LwM2M core protocol; [edge-gateway.md](edge-gateway.md) for IPE integration patterns; [data-encoding.md](data-encoding.md) for NGSI-LD and semantic models

---

## 11. OMA LwM2M Servers (Bootstrap, Registration, Management)

**Reference**: OMA LwM2M v1.2 (2020), LwM2M v2.0 (2026), LwM2M Transport Bindings

**Purpose**: Lightweight M2M (LwM2M) is the de facto standard for LPWAN device management, adopted by cellular operators (NB-IoT, LTE-M), smart metering utilities, and industrial IoT platforms. It provides a complete device management framework over CoAP with significantly lower overhead than TR-069.

**Three-Server Architecture**:
```
┌──────────────────────────────────────────────────────────────────┐
│                    LwM2M ECOSYSTEM                               │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────┐      ┌──────────────────┐                  │
│  │ LwM2M Bootstrap │      │ LwM2M Server     │                  │
│  │ Server          │      │ (DM Server)      │                  │
│  │                 │      │                  │                  │
│  │ Port 5683/5684  │      │ Port 5683/5684   │                  │
│  │ CoAP/CoAPs      │      │ CoAP/CoAPs       │                  │
│  └────────┬────────┘      └────────┬─────────┘                  │
│           │ Bootstrap              │ Registration               │
│           │ (once per device)      │ (periodic)                 │
│           │                        │                            │
│     ┌─────▼────────────────────────▼─────┐                      │
│     │        LwM2M Client Device         │                      │
│     │  ┌──────────────────────────────┐  │                      │
│     │  │ Security Object (0)          │  │                      │
│     │  │ Server Object (1)            │  │                      │
│     │  │ Device Object (3)            │  │                      │
│     │  │ Connectivity Monitoring (4)  │  │                      │
│     │  │ Firmware Update (5)          │  │                      │
│     │  │ Location (6)                 │  │                      │
│     │  │ + Application Objects        │  │                      │
│     │  └──────────────────────────────┘  │                      │
│     └────────────────────────────────────┘                      │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**Server Types and Roles**:

| Server Type | Port | Role | When Used |
|------------|------|------|-----------|
| **Bootstrap Server** | 5683/5684 (CoAP/CoAPS) | Provisions initial security credentials and server URI list to device. Runs once per device lifecycle (factory provisioning or field bootstrap). | Zero-touch provisioning, device transfer between networks, credential rotation |
| **Registration Server** | 5683/5684 (CoAP/CoAPS) | Primary LwM2M management server. Handles device registration, lifecycle management, firmware updates, observations, and telemetry. | Ongoing device management, configuration updates, monitoring |
| **Application Server** | User-defined | Receives telemetry via LwM2M Observe notifications or northbound APIs from Registration Server. Not part of LwM2M spec but common deployment pattern. | Time-series databases, analytics platforms, SCADA integration |

**Bootstrap Modes**:

```
1. Factory Bootstrap (embedded credentials)
   ├─ LwM2M Client shipped with pre-configured Security Object (0)
   ├─ Contains Bootstrap Server URI, PSK/RPK/X.509 credentials
   └─ Device contacts Bootstrap Server on first boot

2. SmartCard Bootstrap (UICC-based)
   ├─ SIM card contains LwM2M credentials in secure storage
   ├─ Device reads from UICC and configures Security/Server Objects
   └─ Common in cellular IoT (NB-IoT, LTE-M)

3. Server-Initiated Bootstrap
   ├─ Device sends Bootstrap-Request to known Bootstrap Server
   ├─ Server provisions Security (0) + Server (1) Objects via CoAP
   └─ Used for device re-provisioning or network migration

4. Client-Initiated Bootstrap (LwM2M 2.0)
   ├─ Device discovers Bootstrap Server via DNS-SD or static config
   ├─ Sends Bootstrap-Request with device identity
   └─ Bootstrap Server provisions full object set
```

**Bootstrap Sequence Flow**:
```
LwM2M Client                  Bootstrap Server           LwM2M Server
     │                              │                          │
     ├─ 1. CoAP POST              ──>│                          │
     │    /bs?ep={endpoint_name}    │                          │
     │    (Bootstrap-Request)        │                          │
     │                              │                          │
     │<─ 2. 2.04 Changed (ACK)    ──┤                          │
     │                              │                          │
     │<─ 3. CoAP DELETE /0/0      ──┤  (Delete Security Obj 0) │
     │<─ 4. CoAP DELETE /1/0      ──┤  (Delete Server Obj 0)   │
     │                              │                          │
     │<─ 5. CoAP PUT /0/0         ──┤  Write Security Object   │
     │    {                          │  for LwM2M Server:      │
     │      LwM2M Server URI,        │    - Server URI         │
     │      Bootstrap Server,        │    - Security Mode (PSK)│
     │      Security Mode,           │    - Public Key ID      │
     │      Public Key,              │    - Secret Key         │
     │      Secret Key               │                          │
     │    }                          │                          │
     │                              │                          │
     │<─ 6. CoAP PUT /1/0         ──┤  Write Server Object:   │
     │    {                          │    - Short Server ID=1  │
     │      Short Server ID=1,       │    - Lifetime=86400s    │
     │      Lifetime=86400,          │    - Binding=U (UDP)    │
     │      Notification Storing=true│                          │
     │    }                          │                          │
     │                              │                          │
     │<─ 7. CoAP POST /bs         ──┤  Bootstrap-Finish        │
     │    (signals bootstrap done)   │                          │
     │                              │                          │
     ├─ 8. CoAP POST              ────────────────────────────>│
     │    /rd?ep={endpoint}&lt=86400│       Register           │
     │    &b=U&lwm2m=1.2            │                          │
     │    Payload: </1/0>,</3/0>... │                          │
     │                              │                          │
     │<─ 9. 2.01 Created           ────────────────────────────┤
     │    Location: /rd/aBc123      │    (Registration Success)│
     │                              │                          │
```

**LwM2M Server Operations (CRUDN + Execute + Observe)**:

| Operation | CoAP Method | LwM2M Usage | Example |
|-----------|-------------|-------------|---------|
| **Create** | POST | Create new Object Instance | `POST /3303` → Create new Temperature Sensor instance |
| **Read** | GET | Read resource value(s) | `GET /3/0/0` → Read Manufacturer name |
| **Update (Write)** | PUT | Write resource value(s) | `PUT /5/0/3` → Write Firmware Package URI |
| **Delete** | DELETE | Delete Object Instance | `DELETE /3303/1` → Remove sensor instance 1 |
| **Execute** | POST | Trigger action on resource | `POST /3/0/4` → Execute Reboot |
| **Observe** | GET + Observe | Subscribe to value changes | `GET /3303/0/5700 Observe:0` → Monitor temperature |
| **Discover** | GET + Accept: link-format | Query object/resource structure | `GET /3` → Discover Device Object resources |
| **Write-Attributes** | PUT + query params | Set notification rules | `PUT /3303/0/5700?pmin=10&pmax=60&gt=30` |

**LwM2M Object Model (Mandatory Objects)**:

```
Object ID | Object Name              | Instances | Purpose
----------|--------------------------|-----------|------------------------------------------
0         | Security                 | Multiple  | Bootstrap/Server credentials (PSK/RPK/X.509)
1         | Server                   | Multiple  | Server configuration (lifetime, binding, notification)
2         | Access Control           | Multiple  | Per-object ACLs (which server can access which objects)
3         | Device                   | Single    | Device metadata (manufacturer, model, serial, firmware version)
4         | Connectivity Monitoring  | Single    | Network stats (radio signal, IP address, cell ID)
5         | Firmware Update          | Single    | FOTA state machine (package URI, update, state)
6         | Location                 | Single    | GPS coordinates (latitude, longitude, altitude)
7         | Connectivity Statistics  | Single    | Data usage (TX/RX bytes, SMS count)

Application Objects (Examples):
3303      | Temperature Sensor       | Multiple  | IPSO Smart Object - temperature readings
3305      | Power Measurement        | Multiple  | Voltage, current, power factor
3306      | Actuation                | Multiple  | On/Off control, dimmer
3308      | Set Point                | Multiple  | Target temperature, pressure setpoints
3313      | Accelerometer            | Multiple  | X/Y/Z axis acceleration
3336      | Humidity Sensor          | Multiple  | Relative humidity %
10241     | uCIFI Base Object        | Multiple  | Smart city lighting control
10242     | uCIFI Lamp Controller    | Multiple  | LED streetlight parameters
```

**Security Object (0) Structure**:
```
Resource ID | Name                   | Type    | Operations | Description
------------|------------------------|---------|------------|----------------------------
0           | LwM2M Server URI       | String  | -          | coap://server.example.com:5683
1           | Bootstrap Server       | Boolean | -          | true=Bootstrap, false=LwM2M Server
2           | Security Mode          | Integer | -          | 0=PSK, 1=RPK, 2=X.509, 3=NoSec
3           | Public Key or Identity | Opaque  | -          | PSK Identity or RPK/X.509 cert
4           | Server Public Key      | Opaque  | -          | Server's public key (RPK/X.509)
5           | Secret Key             | Opaque  | -          | PSK secret or private key
6           | SMS Security Mode      | Integer | -          | 0-3 (SMS binding security)
7           | SMS Binding Key Param  | Opaque  | -          | SMS encryption params
8           | SMS Binding Secret Key | Opaque  | -          | SMS encryption key
10          | Short Server ID        | Integer | -          | Unique server identifier (1-65534)
11          | Client Hold Off Time   | Integer | -          | Delay before registration (seconds)
12          | Bootstrap Server Account Timeout | Integer | - | Bootstrap session timeout
```

**Registration Lifecycle**:
```
1. REGISTER (Client → Server)
   CoAP POST /rd?ep=device001&lt=86400&lwm2m=1.2&b=U
   Payload (CoRE Link Format):
   </1/0>,</3/0>,</4/0>,</5/0>,</3303/0>,</3303/1>

   Server Response: 2.01 Created
   Location-Path: /rd/aBcDeF123

2. UPDATE (Client → Server, before lifetime expires)
   CoAP POST /rd/aBcDeF123?lt=86400

   Server Response: 2.04 Changed

3. DE-REGISTER (Client → Server, graceful shutdown)
   CoAP DELETE /rd/aBcDeF123

   Server Response: 2.02 Deleted
```

**Firmware Update Flow (Object 5)**:
```
LwM2M Client                       LwM2M Server
     │                                    │
     │<─ 1. Write /5/0/1 (Package URI) ──┤
     │      coap://fw.example.com/v2.bin │
     │                                    │
     ├─ 2. Download firmware            ──┤
     │    (background CoAP GET)           │
     │                                    │
     ├─ 3. Write /5/0/3 = 1 (Downloaded)─>│
     │    (Update State = Downloaded)     │
     │                                    │
     │<─ 4. Execute /5/0/2 (Update)     ──┤
     │                                    │
     ├─ 5. Apply firmware               ──┤
     │    Write /5/0/3 = 2 (Updating)     │
     │    Reboot...                       │
     │                                    │
     ├─ 6. Re-register (new firmware)   ─>│
     │    Write /5/0/3 = 0 (Idle)         │
     │    Write /3/0/3 = "v2.0.1"         │
     │    (Firmware Version updated)      │
     │                                    │
```

**Observe Notification with Attributes**:
```
Server subscribes to temperature:
  GET /3303/0/5700 Observe:0
  Query params: pmin=10&pmax=60&lt=25&gt=30

Notification rules:
  - pmin=10:  Minimum 10 seconds between notifications
  - pmax=60:  Maximum 60 seconds between notifications (even if no change)
  - lt=25:    Notify when value < 25°C (less than)
  - gt=30:    Notify when value > 30°C (greater than)

Client sends notifications:
  CON Observe:123 2.05 Content
  Token: 0x4A
  Payload (TLV): 23.5

  CON Observe:124 2.05 Content
  Token: 0x4A
  Payload (TLV): 31.2  (crossed gt=30 threshold)
```

**LwM2M 2.0 Enhancements (2026)**:

| Feature | Description | Benefit |
|---------|-------------|---------|
| **Profile IDs** | `com.example.smart-meter.v1` - Device profiles map to pre-defined object trees | Zero-touch provisioning - server auto-configures based on device type |
| **CBOR Encoding** | Mandatory alongside TLV/JSON | 30-40% payload reduction vs JSON |
| **OSCORE (RFC 8613)** | Application-layer E2E encryption | Security through untrusted intermediaries (concentrators) |
| **Composite Operations** | `ReadComposite(/3/0/0,/3/0/1,/5/0/3)` | Single request for multiple resources - reduces round-trips |
| **Enhanced Bootstrap** | Profile ID in Bootstrap-Request | Server selects provisioning template based on device class |
| **eSIM Support (Object 504)** | GSMA SGP.32/33 remote SIM provisioning | OTA carrier switching for cellular devices |
| **Gateway Objects (25, 26)** | LwM2M Gateway + Routing Objects | Proxy non-IP devices (Modbus, Zigbee) through LwM2M gateway |

**LwM2M Server Implementations**:

| Implementation | Type | Language | Key Features | Use Case |
|----------------|------|----------|--------------|----------|
| **Eclipse Leshan** | Open Source | Java | Bootstrap + Registration servers, web UI, Redis persistence | Development, testing, small deployments |
| **AVSystem Coiote** | Commercial | C++ | Carrier-grade, multi-tenant, LwM2M 1.0-2.0, OMA CTT certified | Telco/MNO device management, smart metering |
| **Friendly Technologies One-IoT** | Commercial | Java/C++ | LwM2M + TR-369 unified, eSIM (Object 504), smart grid focus | Utility smart metering, broadband gateways |
| **Thingstream (U-blox)** | Cloud SaaS | Managed | MQTT-SN bridge, location services, cellular connectivity | Asset tracking, LPWAN devices |
| **IoTerop ALASKA** | Commercial | C | Embedded LwM2M server, on-premise or cloud | Industrial IoT, private networks |
| **Anjay (AVSystem)** | Open Source Client | C | LwM2M 1.0-1.2 client library | Device firmware integration |
| **Wakaama (Eclipse)** | Open Source Client | C | LwM2M 1.0/1.1 reference implementation | Embedded devices, PoC |

**Transport Bindings**:

| Binding | Protocol | Port | Use Case |
|---------|----------|------|----------|
| **U** | CoAP over UDP | 5683 (CoAP), 5684 (CoAPs) | Default for NB-IoT, Wi-Fi, Ethernet |
| **T** | CoAP over TCP | 5683, 5684 | Firewalled networks requiring TCP |
| **S** | CoAP over SMS | - | Extreme coverage (2G fallback), configuration updates |
| **N** | CoAP over Non-IP (NIDD) | - | 3GPP CIoT Non-IP Data Delivery (NB-IoT optimization) |
| **Q** | CoAP over MQTT | 1883, 8883 | LwM2M-over-MQTT for existing MQTT infrastructure |

**Real-World Deployments**:

```
1. NB-IoT Smart Metering (Europe)
   ├─ Landis+Gyr E360 meters → LwM2M 1.2 (CBOR)
   ├─ Diehl IZAR RMU → LwM2M + DLMS/COSEM hybrid
   ├─ AVSystem Coiote DMP (Deutsche Telekom, Orange)
   └─ 15M+ devices under management

2. Asset Tracking (Logistics)
   ├─ U-blox SARA-R5 modules → Thingstream LwM2M cloud
   ├─ LwM2M Object 6 (Location) + 3336 (Humidity) + 3313 (Accelerometer)
   └─ LTE-M Cat-M1 connectivity

3. Smart Cities (uCIFI)
   ├─ Philips CityTouch streetlights → LwM2M Object 10241/10242
   ├─ 500K+ streetlights across France, Netherlands, Dubai
   └─ AVSystem Coiote + city-specific application servers

4. Industrial IoT Gateway (Modbus → LwM2M)
   ├─ Friendly One-IoT Gateway → LwM2M Object 25 (Gateway), 26 (Routing)
   ├─ Virtualizes 50K Modbus devices as LwM2M endpoints
   └─ Profile ID auto-provisioning for device classes
```

**Security Considerations**:

```
1. DTLS 1.2/1.3 with PSK/RPK/X.509
   ├─ Pre-Shared Key (PSK): Embedded secret, low overhead
   ├─ Raw Public Key (RPK): ECDSA certificates without CA chain
   └─ X.509: Full PKI, enterprise deployments

2. OSCORE (LwM2M 2.0)
   ├─ Application-layer encryption (survives proxy/concentrator)
   ├─ AES-CCM-16-64-128 (COSE AEAD)
   └─ Sequence number replay protection

3. Access Control (Object 2)
   ├─ Per-object ACLs: Server 1 can Read /3, Server 2 can Write /5
   └─ Multi-server deployments (bootstrap + primary + backup)

4. Bootstrap Security
   ├─ Bootstrap Server holds master credentials
   ├─ Provisions unique per-device keys to LwM2M Server
   └─ Rotation: Device re-bootstraps to update credentials
```

**Comparison: LwM2M vs TR-069 vs OMA DM**:

| Feature | LwM2M | TR-069 | OMA DM |
|---------|-------|--------|--------|
| Transport | CoAP/UDP (binary) | SOAP/HTTP/TLS (XML) | HTTP/OBD-C (XML) |
| Overhead | 30-50 bytes/message | 500-1000 bytes/message | 200-400 bytes/message |
| Target | LPWAN, NB-IoT, constrained | Broadband gateways, CPE | Mobile phones, tablets |
| Object Model | Numeric IDs (3303/0/5700) | Hierarchical paths (Device.Temp.1.Value) | Tree structure (./DevDetail/SwV) |
| Bootstrap | Dedicated Bootstrap Server | ACS URL + credentials | Bootstrap DM server |
| Firmware Update | Object 5 (state machine) | Download RPC + TransferComplete | FUMO (Firmware Update MO) |
| Observe | Native (CoAP Observe) | Active Notification + STUN | DM Notifications |
| Market | IoT (metering, industrial, tracking) | Broadband (DSL, fiber, cable) | Mobile devices (legacy) |

Cross-reference: [lpwan.md](lpwan.md) section 1 (LwM2M core protocol, object model).

---

## Key Specification References

| Protocol       | Primary Spec                     | Publisher | Year |
|----------------|----------------------------------|-----------|------|
| TR-069 CWMP    | BBF TR-069 Amendment 6           | Broadband Forum | 2018 |
| TR-369 USP     | BBF TR-369 Issue 1.3             | Broadband Forum | 2023 |
| OMA DM         | OMA-TS-DM-Protocol-V2_0          | OMA       | 2016 |
| NETCONF        | RFC 6241                         | IETF      | 2011 |
| YANG           | RFC 7950                         | IETF      | 2016 |
| YANG Push      | RFC 8641                         | IETF      | 2019 |
| RESTCONF       | RFC 8040                         | IETF      | 2017 |
| SNMPv3         | RFC 3411-3418                    | IETF      | 2002 |
| gRPC           | gRPC Core Spec                   | CNCF      | 2023 |
| gNMI           | gNMI v0.8.0                      | OpenConfig| 2020 |
| uCIFI          | OMA LwM2M Object 10241           | uCIFI     | 2019 |
| oneM2M LwM2M IPE | ETSI TS 118 114 v3.6.1         | ETSI      | 2022 |

---

## Related Files
- **[lpwan.md](lpwan.md)**: LwM2M (OMA Lightweight M2M) - dominant protocol for NB-IoT/LTE-M device management
- **[cellular-wan.md](cellular-wan.md)**: OMA DM legacy usage in 2G/3G M2M modules
- **[industrial-ot.md](industrial-ot.md)**: SNMP v3 in SCADA/PLC, OPC UA nodeset management
- **[application-messaging.md](application-messaging.md)**: MQTT integration with TR-369 USP MTP binding
- **[security-provisioning.md](security-provisioning.md)**: X.509 certificate provisioning for NETCONF/TLS, USP E2ESession
