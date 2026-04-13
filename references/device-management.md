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

## 10. oneM2M Interworking (LwM2M IPE)

**Reference**: ETSI TS 118 114 (oneM2M Interworking with LwM2M)

**Architecture**:
```
LwM2M Device                 LwM2M IPE                   oneM2M CSE
┌──────────────┐            ┌────────────────┐          ┌────────────────┐
│ LwM2M Client │ CoAP/UDP   │ LwM2M Bootstrap│ HTTP/CoAP│ Common Services│
│ (Endpoint)   │───────────>│ + Reg Server   │<────────>│ Entity (CSE)   │
│              │ DTLS       │                │  mca ref │                │
│ Object 3303  │            │ Resource Mapper│  point   │ AE (Application│
│ (Temperature)│            │   └─ Translate │          │    Entity)     │
└──────────────┘            │      to oneM2M │          │                │
                            │      <container>│          │ Resource Tree  │
                            └────────────────┘          │  /CSE/AE/Temp  │
                                                        └────────────────┘
```

**Mapping Example**:
```
LwM2M: /3303/0/5700 (Temperature Sensor Value)
  ↓
oneM2M: /CSE001/AE_SensorGW/Container_Temp/ContentInstance_12345
  {
    "m2m:cin": {
      "con": "23.5",  // sensor value
      "cnf": "text/plain",
      "lbl": ["temperature", "zone-A"]
    }
  }
```

**Benefits**: Allows LwM2M-native devices to participate in oneM2M ecosystems (smart city platforms, industrial IoT).

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
