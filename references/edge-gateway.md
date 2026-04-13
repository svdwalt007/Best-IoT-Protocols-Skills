# Edge Computing & Gateways

**Scope**: Edge computing platforms, IoT gateways, protocol adapters, and hybrid cloud-edge architectures. Covers Sparkplug B (MQTT for industrial), EdgeX Foundry, Azure IoT Edge, AWS IoT Greengrass, Eclipse Hono, OPC UA PubSub, and KNX IoT bridging. Cross-references [application-messaging.md](application-messaging.md) for MQTT/AMQP, [industrial-ot.md](industrial-ot.md) for OPC UA, and [device-management.md](device-management.md) for LwM2M gateway patterns.

## Table of Contents
1. MQTT Sparkplug B (spBv1.0 / 3.0)
2. EdgeX Foundry (Microservices Architecture)
3. Azure IoT Edge
4. AWS IoT Greengrass
5. Eclipse Hono (Protocol Adapter Framework)
6. OPC UA PubSub (TSN/MQTT Bindings)
7. KNX IoT (KNX-to-IP Bridging)
8. LwM2M Gateway Patterns
9. Protocol Adapter Architecture (Multi-Protocol Gateway Design)
10. Extended Gateway Protocol Coverage

---

## 1. MQTT Sparkplug B (spBv1.0 / 3.0)

**Reference**: Eclipse Sparkplug Specification B 1.0 (2016), Sparkplug 3.0 (2024)

**Purpose**: Standardized MQTT topic namespace and payload for industrial SCADA/IIoT. Enables bi-directional command/control, state management, and auto-discovery.

**Architecture**:
```
Enterprise (Primary Host Application)
  ↑ MQTT Broker (spBv1.0 namespace)
  ↓
Edge of Network (EoN) Node
  ├─ SCADA Node / HMI
  ├─ Gateway to PLC (Modbus, OPC UA)
  └─ Devices (sensors, actuators)
```

**Topic Namespace**:
```
spBv1.0/{group_id}/{message_type}/{edge_node_id}
spBv1.0/{group_id}/{message_type}/{edge_node_id}/{device_id}

Examples:
  spBv1.0/My Factory/NBIRTH/Edge Node 1
  spBv1.0/My Factory/DBIRTH/Edge Node 1/Sensor A
  spBv1.0/My Factory/NDATA/Edge Node 1
  spBv1.0/My Factory/NCMD/Edge Node 1
  spBv1.0/My Factory/STATE/scada_host_1
```

**Message Types**:
```
NBIRTH (Node Birth):        Edge node comes online, publishes full metric list
NDEATH (Node Death):        Will message (LWT) when edge node disconnects
DBIRTH (Device Birth):      Device within edge node publishes metrics
DDEATH (Device Death):      Device offline within edge node
NDATA (Node Data):          Incremental metric updates from edge node
DDATA (Device Data):        Incremental metric updates from device
NCMD (Node Command):        Command from primary host to edge node
DCMD (Device Command):      Command from primary host to device
STATE (State):              Primary host application state (online/offline)
```

**Sparkplug B Payload (Protobuf)**:
```protobuf
message Payload {
  uint64 timestamp = 1;          // Unix epoch ms
  repeated Metric metrics = 2;
  uint64 seq = 3;                // Sequence number (BIRTH resets to 0)
  string uuid = 4;               // Session UUID (unique per edge node session)
  bytes body = 5;                // Reserved
}

message Metric {
  string name = 1;               // "Temperature", "Pressure", "Valve1/Status"
  uint64 alias = 2;              // Optional numeric alias (reduce bandwidth)
  uint64 timestamp = 3;          // Metric-specific timestamp
  uint32 datatype = 4;           // 1=Int8, 2=Int16, 9=Float, 11=Boolean, 12=String
  bool is_historical = 5;
  bool is_transient = 6;
  bool is_null = 7;

  MetaData metadata = 8;         // Optional (units, min, max, engineering units)

  PropertySet properties = 9;    // Custom metadata (OEM extensions)

  oneof value {
    uint32 int_value = 10;
    uint64 long_value = 11;
    float float_value = 12;
    double double_value = 13;
    bool boolean_value = 14;
    string string_value = 15;
    bytes bytes_value = 16;
    DataSet dataset_value = 17;  // Table/matrix data
    Template template_value = 18;// Reusable metric structure
  }
}
```

**Session Lifecycle**:
```
1. Edge Node connects to MQTT broker
   - Sets LWT (Last Will Testament):
     Topic: spBv1.0/Factory1/NDEATH/Gateway1
     Payload: { seq: 123, timestamp: 0 }

2. Edge Node publishes NBIRTH
   Topic: spBv1.0/Factory1/NBIRTH/Gateway1
   Payload: {
     timestamp: 1638360000000,
     seq: 0,
     uuid: "550e8400-e29b-41d4-a716-446655440000",
     metrics: [
       { name: "Node Control/Rebirth", alias: 0, datatype: Boolean },
       { name: "bdSeq", alias: 1, datatype: UInt64, long_value: 0 }
     ]
   }

3. Device Birth
   Topic: spBv1.0/Factory1/DBIRTH/Gateway1/PLC1
   Payload: {
     seq: 1,
     metrics: [
       { name: "Temperature", alias: 100, datatype: Float, float_value: 22.5 },
       { name: "Pressure", alias: 101, datatype: Float, float_value: 101.3 }
     ]
   }

4. Incremental Data (using aliases to reduce bandwidth)
   Topic: spBv1.0/Factory1/DDATA/Gateway1/PLC1
   Payload: {
     seq: 2,
     metrics: [
       { alias: 100, float_value: 23.1 },  // Temperature update only
     ]
   }

5. Command from SCADA
   Topic: spBv1.0/Factory1/DCMD/Gateway1/PLC1
   Payload: {
     metrics: [
       { name: "Valve1/Command", datatype: Boolean, boolean_value: true }
     ]
   }

6. Disconnect (NDEATH published by broker via LWT)
```

**Primary Host State**:
```
Primary Host publishes STATE message:
  Topic: spBv1.0/STATE/scada_host_1
  Payload: { timestamp: 1638360123456, online: true }

If Primary Host crashes → STATE message changes to offline
Edge nodes detect state → stop publishing NDATA (wait for new primary)
```

**Sparkplug 3.0 Enhancements (2024)**:
```
- MQTT 5.0 support (user properties, reason codes)
- Enhanced security (OAuth 2.0 integration)
- Improved alias management
- State-based synchronization (eventual consistency)
```

**Use Case**: Manufacturing (SCADA data historian), oil & gas (SCADA RTU), building automation.

---

## 2. EdgeX Foundry (Microservices Architecture)

**Reference**: EdgeX Foundry 3.0 (LF Edge)

**Architecture**:
```
┌────────────────────────────────────────────────────────┐
│ Application Services (export, transformation)          │
│   ├─ app-service-configurable                          │
│   ├─ app-rfid-llrp-inventory (RFID integration)        │
│   └─ Custom app services (Kuiper rules engine)         │
├────────────────────────────────────────────────────────┤
│ Core Services                                          │
│   ├─ core-data (event/reading persistence)             │
│   ├─ core-command (actuator control)                   │
│   ├─ core-metadata (device/profile registry)           │
│   └─ support-notifications (alerts, email/SMS)         │
├────────────────────────────────────────────────────────┤
│ Device Services (southbound adapters)                  │
│   ├─ device-modbus (Modbus RTU/TCP)                    │
│   ├─ device-mqtt (MQTT devices)                        │
│   ├─ device-rest (RESTful sensors)                     │
│   ├─ device-bacnet (BACnet MS/TP, BACnet/IP)           │
│   ├─ device-opcua (OPC UA client)                      │
│   ├─ device-snmp (SNMP v2c/v3)                         │
│   ├─ device-camera (ONVIF, RTSP)                       │
│   ├─ device-gpio (Raspberry Pi GPIO)                   │
│   └─ device-coap (CoAP/LwM2M)                          │
├────────────────────────────────────────────────────────┤
│ Supporting Services                                    │
│   ├─ support-scheduler (cron-like job scheduling)      │
│   ├─ support-rulesengine (eKuiper CEP)                 │
│   └─ sys-mgmt-agent (EdgeX lifecycle management)       │
├────────────────────────────────────────────────────────┤
│ Security Services                                      │
│   ├─ security-secretstore-setup (Vault initialization) │
│   ├─ security-proxy-setup (Kong API gateway)           │
│   └─ security-spire (SPIFFE/SPIRE identity)            │
└────────────────────────────────────────────────────────┘
```

**Device Profile (YAML)**:
```yaml
name: "Modbus-Temperature-Sensor"
manufacturer: "Acme Corp"
model: "TempSensor-X1"
labels:
  - "temperature"
  - "industrial"
description: "Modbus RTU temperature sensor"

deviceResources:
  - name: "Temperature"
    description: "Current temperature in Celsius"
    attributes:
      { primaryTable: "HOLDING_REGISTER", startingAddress: 0, rawType: "Int16" }
    properties:
      valueType: "Float32"
      readWrite: "R"
      units: "Celsius"
      minimum: "-40"
      maximum: "125"
      scale: "0.1"  # Convert Int16 to Float (divide by 10)

  - name: "SensorStatus"
    attributes:
      { primaryTable: "HOLDING_REGISTER", startingAddress: 1, rawType: "Uint16" }
    properties:
      valueType: "String"
      readWrite: "R"
      mapping:
        "0": "OK"
        "1": "Warning"
        "2": "Error"

deviceCommands:
  - name: "ReadTemperature"
    readWrite: "R"
    resourceOperations:
      - { deviceResource: "Temperature" }
```

**REST API Examples**:
```bash
# Query device reading
GET http://localhost:59880/api/v3/reading/device/name/Modbus-Temp-01?limit=10

Response:
{
  "apiVersion": "v3",
  "readings": [
    {
      "id": "8f3b3c1a-...",
      "deviceName": "Modbus-Temp-01",
      "resourceName": "Temperature",
      "valueType": "Float32",
      "value": "23.5",
      "origin": 1638360123456000000
    }
  ]
}

# Send command
PUT http://localhost:59882/api/v3/device/name/HVAC-Controller-01/SetpointTemp
Body: { "SetpointTemp": "22.0" }
```

**MessageBus Integration** (Redis Streams / MQTT):
```go
// App service subscribes to EdgeX MessageBus
for event := range messageBus.Subscribe("edgex-events") {
    // Event contains readings from core-data
    for _, reading := range event.Readings {
        if reading.ResourceName == "Temperature" && reading.Value > 30.0 {
            // Trigger alert
            sendNotification("High temperature alert")
        }
    }
}
```

**Use Case**: Industrial gateway (multi-protocol aggregation), building automation (BACnet + Modbus unification).

---

## 3. Azure IoT Edge

**Reference**: Azure IoT Edge Runtime 1.4, IoT Edge Hub

**Architecture**:
```
┌─────────────────────────────────────────────────────┐
│ Azure IoT Hub (Cloud)                               │
│   ├─ Device Twin (desired/reported properties)     │
│   ├─ Direct Methods                                │
│   └─ Cloud-to-Device messages                      │
└──────────────────┬──────────────────────────────────┘
                   │ AMQP/MQTT/HTTPS
┌──────────────────▼──────────────────────────────────┐
│ IoT Edge Device (Gateway)                           │
│  ┌───────────────────────────────────────────────┐  │
│  │ IoT Edge Hub (local message broker)          │  │
│  │   ├─ Offline message buffering               │  │
│  │   ├─ Protocol translation (MQTT ↔ AMQP)      │  │
│  │   └─ Module-to-module routing                │  │
│  ├───────────────────────────────────────────────┤  │
│  │ IoT Edge Agent (module lifecycle manager)    │  │
│  ├───────────────────────────────────────────────┤  │
│  │ Custom Modules (Docker containers)           │  │
│  │   ├─ Simulated Temperature Sensor            │  │
│  │   ├─ OPC Publisher (OPC UA → IoT Hub)        │  │
│  │   ├─ Modbus Module (Modbus RTU/TCP adapter)  │  │
│  │   ├─ Azure Stream Analytics (edge analytics) │  │
│  │   ├─ Azure Functions (serverless edge logic) │  │
│  │   └─ Custom ML inference (ONNX model)        │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

**Deployment Manifest (JSON)**:
```json
{
  "modulesContent": {
    "$edgeAgent": {
      "properties.desired": {
        "modules": {
          "opcPublisher": {
            "version": "2.8",
            "type": "docker",
            "status": "running",
            "restartPolicy": "always",
            "settings": {
              "image": "mcr.microsoft.com/iotedge/opc-publisher:2.8",
              "createOptions": "{\"HostConfig\":{\"NetworkMode\":\"host\"}}"
            }
          },
          "filterModule": {
            "version": "1.0",
            "type": "docker",
            "status": "running",
            "settings": {
              "image": "myregistry.azurecr.io/filtermodule:1.0"
            }
          }
        }
      }
    },
    "$edgeHub": {
      "properties.desired": {
        "routes": {
          "opcToFilter": "FROM /messages/modules/opcPublisher/outputs/* INTO BrokeredEndpoint(\"/modules/filterModule/inputs/input1\")",
          "filterToIoTHub": "FROM /messages/modules/filterModule/outputs/output1 INTO $upstream"
        }
      }
    }
  }
}
```

**Module-to-Module Communication**:
```csharp
// C# module sending message
var messageBody = new { temperature = 25.3, deviceId = "sensor01" };
var message = new Message(Encoding.UTF8.GetBytes(JsonConvert.SerializeObject(messageBody)));
message.Properties.Add("temperatureAlert", "false");
await moduleClient.SendEventAsync("temperatureOutput", message);

// Receiving module
await moduleClient.SetInputMessageHandlerAsync("input1", async (message, userContext) =>
{
    var body = Encoding.UTF8.GetString(message.GetBytes());
    var data = JsonConvert.DeserializeObject<SensorData>(body);
    // Process data
    return MessageResponse.Completed;
}, null);
```

**OPC Publisher Configuration** (publishednodes.json):
```json
[
  {
    "EndpointUrl": "opc.tcp://192.168.1.100:4840",
    "UseSecurity": false,
    "OpcNodes": [
      {
        "Id": "ns=2;s=Temperature",
        "OpcSamplingInterval": 1000,
        "OpcPublishingInterval": 2000
      },
      {
        "Id": "ns=2;s=Pressure",
        "OpcSamplingInterval": 1000
      }
    ]
  }
]
```

**Use Case**: Factory floor gateway (OPC UA aggregation), oil rig edge (Modbus + analytics), retail store (camera + POS integration).

---

## 4. AWS IoT Greengrass

**Reference**: AWS IoT Greengrass Core v2.11

**Architecture**:
```
┌──────────────────────────────────────────────────────┐
│ AWS IoT Core (Cloud)                                 │
│   ├─ Device Shadow (state synchronization)          │
│   ├─ Jobs (remote deployment)                       │
│   └─ IoT SiteWise (industrial data modeling)        │
└─────────────────┬────────────────────────────────────┘
                  │ MQTT/HTTPS
┌─────────────────▼────────────────────────────────────┐
│ Greengrass Core Device (Edge Gateway)                │
│  ┌───────────────────────────────────────────────┐   │
│  │ Greengrass Nucleus (runtime manager)         │   │
│  ├───────────────────────────────────────────────┤   │
│  │ AWS IoT Greengrass Components                │   │
│  │   ├─ Stream Manager (local data buffering)   │   │
│  │   ├─ Lambda Manager (run Lambda@Edge)        │   │
│  │   ├─ Secret Manager (local secrets cache)    │   │
│  │   ├─ IoT SiteWise Connector (OPC UA/Modbus)  │   │
│  │   ├─ SageMaker Edge Agent (ML inference)     │   │
│  │   ├─ Docker Application Manager              │   │
│  │   └─ Custom components (Java/Python/Node)    │   │
│  └───────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────┘
```

**Component Recipe (YAML)**:
```yaml
RecipeFormatVersion: "2020-01-25"
ComponentName: com.example.ModbusReader
ComponentVersion: "1.0.0"
ComponentDescription: "Reads Modbus RTU devices and publishes to IoT Core"
ComponentPublisher: "Acme Corp"
ComponentConfiguration:
  DefaultConfiguration:
    modbusPort: "/dev/ttyUSB0"
    baudRate: 9600
    publishInterval: 5000

Manifests:
  - Platform:
      os: linux
    Lifecycle:
      Install: |
        pip3 install pymodbus
      Run: |
        python3 {artifacts:path}/modbus_reader.py --port {configuration:/modbusPort} --interval {configuration:/publishInterval}
    Artifacts:
      - URI: "s3://mybucket/modbus_reader.py"
```

**Lambda Function (Python)**:
```python
import greengrasssdk
import json

client = greengrasssdk.client('iot-data')

def function_handler(event, context):
    # Process sensor data
    temperature = event['temperature']

    if temperature > 30:
        # Publish alert to IoT Core
        client.publish(
            topic='alerts/temperature',
            queueFullPolicy='AllOrException',
            payload=json.dumps({'alert': 'High temperature', 'value': temperature})
        )

    return {'status': 'processed'}
```

**Stream Manager (Local Data Buffer)**:
```python
from stream_manager import StreamManagerClient, MessageStreamDefinition, ExportDefinition, S3ExportTaskDefinition

# Create stream that exports to S3 when offline
client = StreamManagerClient()
stream_name = "sensor_data_stream"

client.create_message_stream(
    MessageStreamDefinition(
        name=stream_name,
        max_size=268435456,  # 256 MB
        export_definition=ExportDefinition(
            s3_task_executor=[
                S3ExportTaskDefinition(
                    identifier="s3Export",
                    bucket="my-sensor-data-bucket"
                )
            ]
        )
    )
)

# Append data
client.append_message(stream_name, json.dumps(sensor_reading).encode())
```

**IoT SiteWise Connector**:
```json
{
  "sources": [
    {
      "name": "OPC_UA_Source",
      "endpoint": {
        "uri": "opc.tcp://192.168.1.100:4840",
        "securityPolicy": "None"
      },
      "measurementDataStreamPrefix": "factory1",
      "destination": {
        "type": "SiteWise"
      }
    }
  ]
}
```

**Use Case**: Manufacturing (SiteWise OPC UA ingestion), agriculture (drone image processing at edge), retail (local ML inference for inventory).

---

## 5. Eclipse Hono (Protocol Adapter Framework)

**Reference**: Eclipse Hono 2.4

**Architecture**:
```
┌────────────────────────────────────────────────────────┐
│ Business Applications                                  │
│   ├─ Telemetry Consumer (AMQP 1.0 client)            │
│   ├─ Command Sender (AMQP 1.0 client)                │
│   └─ Device Registry (HTTP REST API)                 │
└────────────────┬───────────────────────────────────────┘
                 │ AMQP 1.0 (Messaging Network)
┌────────────────▼───────────────────────────────────────┐
│ Eclipse Hono (Multi-Tenant IoT Gateway)                │
│  ┌─────────────────────────────────────────────────┐   │
│  │ Protocol Adapters (southbound)                  │   │
│  │   ├─ MQTT Adapter (MQTT 3.1.1/5.0)             │   │
│  │   ├─ HTTP Adapter (REST API)                   │   │
│  │   ├─ AMQP Adapter (AMQP 1.0)                   │   │
│  │   ├─ CoAP Adapter (CoAP/LwM2M)                 │   │
│  │   └─ Lora Adapter (Lorawan Network Server)     │   │
│  ├─────────────────────────────────────────────────┤   │
│  │ AMQP 1.0 Messaging Network (Qpid Dispatch)     │   │
│  │   ├─ Telemetry Endpoint (fire-and-forget)      │   │
│  │   ├─ Event Endpoint (guaranteed delivery)      │   │
│  │   └─ Command & Control (bidirectional)         │   │
│  ├─────────────────────────────────────────────────┤   │
│  │ Device Registry (MongoDB)                       │   │
│  │   ├─ Tenant management                          │   │
│  │   ├─ Device credentials (PSK, X.509)           │   │
│  │   └─ Device metadata                            │   │
│  └─────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────┘
```

**Device Telemetry Flow**:
```
MQTT Device → MQTT Adapter (port 1883/8883)
  ├─ Topic: telemetry/tenant1/device001
  ├─ Payload: {"temperature": 23.5}
  └─ QoS: 0 (telemetry) or 1 (event)

MQTT Adapter validates credentials (device registry lookup)
  ↓
Converts MQTT → AMQP 1.0 message
  ├─ Address: telemetry/tenant1
  ├─ Content-Type: application/json
  ├─ Device-ID: device001
  └─ Message annotation: x-opt-tenant=tenant1

Publishes to AMQP Messaging Network
  ↓
Business App receives AMQP message on telemetry/tenant1 address
```

**Command & Control**:
```
Business App sends command:
  AMQP Address: command/tenant1/device001
  Payload: {"action": "setSetpoint", "value": 22.0}
  Reply-To: command_response/tenant1/${correlation-id}

MQTT Adapter receives command → forwards to MQTT device
  Topic: command///req/${request-id}/setSetpoint
  Payload: {"value": 22.0}

Device responds:
  Topic: command///res/${request-id}/200
  Payload: {"status": "ok"}

MQTT Adapter forwards response to Business App via AMQP reply address
```

**Device Registry API**:
```bash
# Register device
POST /v1/tenants/tenant1/devices/device001
{
  "enabled": true,
  "ext": {
    "model": "TempSensor-X1",
    "location": "Building A"
  }
}

# Add credentials (PSK)
POST /v1/credentials/tenant1/device001
{
  "type": "psk",
  "auth-id": "device001-auth",
  "secrets": [
    {
      "key": "cGFzc3dvcmQ="  # base64("password")
    }
  ]
}
```

**Use Case**: Multi-tenant SaaS IoT platform, smart city (multi-protocol sensor aggregation), fleet management.

---

## 6. OPC UA PubSub (TSN/MQTT Bindings)

**Reference**: IEC 62541-14 (OPC UA PubSub)

**Transport Bindings**:
```
1. UADP (UDP/Ethernet):
   - Raw Ethernet frames (TSN 802.1Qbv scheduled)
   - UDP/IP multicast (224.0.2.100:4840)
   - Binary encoding (compact, low overhead)

2. JSON (MQTT/AMQP):
   - MQTT topic: opcua/dataset/Publisher1/DataSet1
   - JSON payload (interoperable, human-readable)

3. AMQP 1.0:
   - Address: opcua/telemetry/Publisher1
   - Used in cloud integration scenarios
```

**UADP Frame Structure** (Binary):
```
NetworkMessage Header:
  ├─ UADPVersion: 1
  ├─ UADPFlags: 0x05 (PublisherId enabled, DataSetMessage payload)
  ├─ PublisherId: 42 (Uint16)
  └─ DataSetClassId: UUID (optional)

DataSetMessage:
  ├─ DataSetWriterId: 1 (maps to DataSetMetaData)
  ├─ SequenceNumber: 12345
  ├─ Timestamp: 2023-12-01T10:30:00Z
  ├─ Status: Good (0x00)
  └─ Payload:
      Field 1 (Temperature): 23.5 (Float)
      Field 2 (Pressure): 101.3 (Float)
      Field 3 (Status): 1 (Byte: OK)
```

**JSON Payload** (MQTT):
```json
{
  "MessageId": "00000001",
  "MessageType": "ua-data",
  "PublisherId": "Publisher1",
  "Messages": [
    {
      "DataSetWriterId": 1,
      "SequenceNumber": 12345,
      "Timestamp": "2023-12-01T10:30:00Z",
      "Status": 0,
      "Payload": {
        "Temperature": { "Value": 23.5 },
        "Pressure": { "Value": 101.3 },
        "Status": { "Value": 1 }
      }
    }
  ]
}
```

**Publisher Configuration** (OPC UA Information Model):
```
PubSubConnection: "MQTTConnection1"
  ├─ TransportProfileUri: http://opcfoundation.org/UA-Profile/Transport/pubsub-mqtt-json
  ├─ Address: mqtt://broker.example.com:1883
  ├─ WriterGroup: "SensorData"
  │   ├─ PublishingInterval: 1000 ms
  │   ├─ DataSetWriter: "DataSet1"
  │   │   └─ DataSetName: "PLC1_Data"
  │   └─ MessageSettings: JSON encoding
  └─ SecurityMode: Sign (SecurityKeyService for group keys)
```

**Use Case**: TSN factory floor (deterministic publishing), MQTT cloud telemetry, SCADA interop.

---

## 7. KNX IoT (KNX-to-IP Bridging)

**Reference**: KNX-IoT-TP (CoAP + OSCORE over IPv6)

**Architecture**:
```
KNX Twisted Pair (TP) Bus
  ├─ KNX devices (actuators, sensors)
  └─ KNX/IP Router (bridge to IP network)
       ↓ IPv6 / 6LoWPAN
KNX IoT Point API (CoAP resources)
  ├─ /p/{iid} (datapoint resources)
  ├─ /dev (device discovery)
  └─ OSCORE security (Group OSCORE for multicast)
```

**CoAP Resource Example**:
```
GET coap://[fe80::1]/p/1
Response: {"v": true}  (Switch state: ON)

PUT coap://[fe80::1]/p/1
Payload: {"v": false}  (Turn switch OFF)
```

**Use Case**: Building automation (KNX legacy integration with cloud), smart home (bridge KNX to Matter).

---

## 8. LwM2M Gateway Patterns

**Reference**: OMA LwM2M v1.2 §9 (Gateway Architecture)

**Pattern 1: Proxy Gateway**:
```
Non-IP Devices (Zigbee, BLE)
  ↓ (Gateway internal protocol)
LwM2M Gateway (exposes devices as LwM2M Objects)
  ↓ CoAP/UDP (port 5683)
LwM2M Server

Example: BLE heart rate sensor → Gateway Object 3303 (Temperature)
```

**Pattern 2: Extended Proxy**:
```
LwM2M Client (NB-IoT sensor)
  ↓ CoAP/UDP (direct to gateway)
LwM2M Gateway (registers on behalf of clients)
  ↓ CoAP/UDP
LwM2M Server

Gateway acts as registration proxy, forwards Observe notifications
```

Cross-reference: [lpwan.md](lpwan.md) section 1 (LwM2M core protocol).

---

## 9. Protocol Adapter Architecture (Multi-Protocol Gateway Design)

**Reference**: Inspired by ThingsBoard Gateway, Friendly One-IoT Gateway, EdgeX Device Services

**Purpose**: Modern IoT edge gateways must aggregate data from heterogeneous field devices speaking incompatible protocols (Modbus RTU, BACnet, OPC UA, DLMS/COSEM, etc.) and translate to a unified northbound protocol (LwM2M, MQTT, HTTP). This section covers the canonical architecture pattern for multi-protocol adapter frameworks.

**Layered Adapter Architecture**:
```
┌───────────────────────────────────────────────────────────────────┐
│                    NORTHBOUND (Cloud/Platform)                     │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  Unified Protocol Layer                                     │  │
│  │    ├─ LwM2M 2.0 (CoAP/CBOR) - Objects 25 (Gateway), 26 (Routing)│
│  │    ├─ MQTT + Sparkplug B (protobuf payload)               │  │
│  │    ├─ HTTP/REST (JSON API)                                │  │
│  │    └─ AMQP 1.0 (Eclipse Hono integration)                 │  │
│  └─────────────────────────────────────────────────────────────┘  │
├───────────────────────────────────────────────────────────────────┤
│                    GATEWAY CORE SERVICES                          │
│  ┌──────────────────┐ ┌──────────────────┐ ┌─────────────────┐   │
│  │ Device Registry  │ │ Schema Registry  │ │ Command Router  │   │
│  │  (Device Twin)   │ │  (Profile IDs)   │ │  (Fan-out)      │   │
│  └──────────────────┘ └──────────────────┘ └─────────────────┘   │
│  ┌──────────────────┐ ┌──────────────────┐ ┌─────────────────┐   │
│  │ Normalisation    │ │ Payload Codec    │ │ Offline Buffer  │   │
│  │  (Protocol→LwM2M)│ │  (CBOR/JSON/TLV) │ │  (Store-forward)│   │
│  └──────────────────┘ └──────────────────┘ └─────────────────┘   │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │ Virtual Endpoint Manager (LwM2M 2.0)                        ││
│  │  - Object 25 (Gateway): Single gateway identity            ││
│  │  - Object 26 (Routing): Per-device routing table           ││
│  │  - Profile ID Framework: Device class auto-provisioning    ││
│  └──────────────────────────────────────────────────────────────┘│
├───────────────────────────────────────────────────────────────────┤
│                    PROTOCOL ADAPTERS (Southbound)                 │
│  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐        │
│  │ MQTT Adapter   │ │ OPC UA Adapter │ │ Modbus Adapter │        │
│  │  Port 1883     │ │  Port 4840     │ │  RTU/TCP       │        │
│  └────────────────┘ └────────────────┘ └────────────────┘        │
│  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐        │
│  │ BACnet Adapter │ │ DLMS Adapter   │ │ ANSI C12 Adptr │        │
│  │  Port 47808    │ │  HDLC/TCP      │ │  Optical/TCP   │        │
│  └────────────────┘ └────────────────┘ └────────────────┘        │
│  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐        │
│  │ Wi-SUN Adapter │ │ Wi-MBus Adptr  │ │ DNP3 Adapter   │        │
│  │  6LoWPAN/RPL   │ │  EN 13757-4    │ │  TCP/Serial    │        │
│  └────────────────┘ └────────────────┘ └────────────────┘        │
└───────────────────────────────────────────────────────────────────┘
                              │
                   ┌──────────▼──────────┐
                   │  Field Device Layer │
                   │  (Sensors, Meters,  │
                   │   PLCs, Actuators)  │
                   └─────────────────────┘
```

**Adapter Lifecycle State Machine**:
```
┌──────────────────────────────────────────────────────────────┐
│               ADAPTER LIFECYCLE STATES                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌───────────┐   ┌───────────┐   ┌──────────┐              │
│  │ INSTALLED ├──>│ CONFIGURED├──>│ CONNECTED│              │
│  └───────────┘   └───────────┘   └─────┬────┘              │
│                                         │                   │
│                                    ┌────▼────┐              │
│                                    │ RUNNING │              │
│                                    └────┬────┘              │
│                                         │                   │
│                          ┌──────────────┼──────────────┐    │
│                          │              │              │    │
│                     ┌────▼─────┐  ┌─────▼────┐  ┌─────▼────┐
│                     │ POLLING  │  │ LISTENING│  │ OBSERVING│
│                     │ (active) │  │ (passive)│  │ (subscr.)│
│                     └────┬─────┘  └─────┬────┘  └─────┬────┘
│                          │              │              │    │
│                          └──────────────┼──────────────┘    │
│                                         │                   │
│                                    ┌────▼────┐              │
│                                    │DRAINING │              │
│                                    │ (flush) │              │
│                                    └────┬────┘              │
│                                         │                   │
│                                    ┌────▼────┐              │
│                                    │ STOPPED │              │
│                                    └─────────┘              │
└──────────────────────────────────────────────────────────────┘
```

**Virtual Endpoint Lifecycle (LwM2M 2.0 Gateway Pattern)**:
```
Each discovered field device gets a virtual LwM2M endpoint:

DISCOVERED → PROFILING → BOOTSTRAPPING → REGISTERING → ACTIVE
    ↑              │              │              │           │
    │              └──────────────┴──────────────┴───────────┤
    │                     (Error recovery loop)              │
    └────────────────────────────────────────────────────────┘

State Descriptions:
├─ DISCOVERED:      Adapter detects device on field network
│                   (Modbus scan response, OPC UA browse, MQTT connect)
│
├─ PROFILING:       Resolve Profile ID from {protocol, device_model}
│                   Example: "Modbus + Landis+Gyr E350" → "com.lg.e350.modbus.v1"
│                   Construct LwM2M object tree from profile template
│
├─ BOOTSTRAPPING:   Provision security credentials via LwM2M Bootstrap
│                   DTLS/OSCORE context established per virtual endpoint
│
├─ REGISTERING:     LwM2M Register to DMP Server with lifetime, binding, object list
│                   Object 26 routing entry created (maps endpoint → adapter)
│
└─ ACTIVE:          Device operational - REPORTING, COMMANDED, OBSERVED sub-states
                    - REPORTING: Periodic telemetry (pmin/pmax attributes)
                    - COMMANDED: Processing SET/EXECUTE from DMP
                    - OBSERVED: Active CoAP Observe subscription
```

**Profile ID Framework**:
```yaml
# Device Profile CRD (Kubernetes Custom Resource)
apiVersion: gateway.iot.example.com/v1alpha1
kind: DeviceProfile
metadata:
  name: landis-gyr-e350-modbus
spec:
  profileId: "com.landis-gyr.e350.modbus.v1"
  lwm2mVersion: "2.0"
  protocol: modbus-tcp

  # Device identification rules
  deviceMatchers:
    - field: holding_register_40050  # Manufacturer ID
      value: "0x4C47"                # "LG" = Landis+Gyr
    - field: holding_register_40051  # Model register
      value: "0xE350"

  # LwM2M encoding
  encoding: CBOR  # Primary: CBOR (30-40% smaller), fallback: JSON

  # LwM2M object tree template
  objectTree:
    - objectId: 3          # Device Object
      resources: [0, 1, 2, 3, 17]  # Manufacturer, Model, Serial, FW Ver, Device Type
    - objectId: 3316       # Generic Sensor (Energy Meter)
      instances: auto       # One per meter channel
      resources: [5700, 5701, 5601, 5602]  # Sensor Value, Unit, Min/Max
    - objectId: 4          # Connectivity Monitoring
      resources: [0, 1, 4, 8]  # Network Bearer, Available Network Bearer, IP Addresses

  # Modbus → LwM2M mapping rules
  mappings:
    - source_register: "40001-40002"  # 2 registers = 32-bit float
      decode: float32_big_endian
      scale_factor: 0.01
      unit: "kWh"
      target_lwm2m:
        object_id: 3316
        instance_id: 0
        resource_id: 5700  # Sensor Value
        resource_type: float
      notification_attrs:
        pmin: 30   # Min 30s between notifications
        pmax: 300  # Max 5 min between notifications
        edge: false  # Not edge-triggered
```

**Message Flow (Device → Cloud)**:
```
Field Device (Modbus)          Adapter             Gateway Core         LwM2M Server
      │                          │                      │                     │
      │<─ Modbus Read Holding ──┤ Poll on schedule     │                     │
      │   Registers 40001-40002  │ (e.g., every 60s)   │                     │
      │                          │                      │                     │
      ├─ Response: 0x42480000  ─>│ (42.48 kWh as IEEE) │                     │
      │   (32-bit float BE)      │                      │                     │
      │                          │                      │                     │
      │                          ├─ Normalise ────────>│                     │
      │                          │  {                   │                     │
      │                          │    protocol: modbus, │                     │
      │                          │    endpoint: meter001│                     │
      │                          │    profile_id: ...   │                     │
      │                          │    lwm2m_objects: [  │                     │
      │                          │      {obj:3316, inst:0, res:5700, val:42.48}│
      │                          │    ]                 │                     │
      │                          │  }                   │                     │
      │                          │                      │                     │
      │                          │                      ├─ CBOR encode ─────>│
      │                          │                      │ CoAP NOTIFY        │
      │                          │                      │ /3316/0/5700       │
      │                          │                      │ Observe: 123       │
      │                          │                      │ Payload (CBOR):    │
      │                          │                      │   0xFA422C0000     │
      │                          │                      │   (42.48 as CBOR)  │
```

**Command Flow (Cloud → Device)**:
```
LwM2M Server              Gateway Core            Adapter            Field Device
      │                         │                    │                     │
      ├─ CoAP PUT /3306/0/5850 ─>│ Write resource    │                     │
      │   Payload: true          │ (turn on actuator)│                     │
      │                         │                    │                     │
      │                         ├─ Lookup routing  ──┤                     │
      │                         │  Object 26 entry   │                     │
      │                         │  endpoint → adapter│                     │
      │                         │                    │                     │
      │                         ├─ Command ────────>│ Translate to       │
      │                         │  {                 │ native protocol    │
      │                         │    target: valve01,│                     │
      │                         │    operation: WRITE│                     │
      │                         │    resource: on/off│                     │
      │                         │    value: true     │                     │
      │                         │  }                 │                     │
      │                         │                    │                     │
      │                         │                    ├─ Modbus Write ────>│
      │                         │                    │  Coil 0x0001 = ON  │
      │                         │                    │                     │
      │                         │                    │<─ Response: OK ────┤
      │                         │                    │                     │
      │                         │<─ ACK ─────────────┤                     │
      │<─ 2.04 Changed ──────────┤                    │                     │
```

**Composite Operations (LwM2M 1.2+)**:
```
Single request for multiple resources across objects:

READ-COMPOSITE:
  CoAP POST /dp (composite read path)
  Content-Format: SenML JSON
  Payload:
  [
    {"bn":"/3/0/", "n":"0"},      # Device Manufacturer
    {"bn":"/3/0/", "n":"1"},      # Device Model
    {"bn":"/3316/0/", "n":"5700"}, # Sensor Value
    {"bn":"/3316/1/", "n":"5700"}  # Sensor Value (instance 1)
  ]

Gateway scatter-gather:
  1. Parse SenML path list
  2. Group by backing adapter and physical device
  3. Issue parallel protocol-native bulk reads:
     - Modbus multi-register read (FC 0x03)
     - OPC UA ReadValueId[] array
     - BACnet ReadPropertyMultiple
  4. Assemble SenML-CBOR response
  5. Return composite response to DMP

Benefits: 4 round-trips → 1 round-trip (75% reduction)
```

**Data Encoding Strategy**:
```
┌────────────────────────────────────────────────────────────┐
│            CANONICAL MESSAGE FORMAT (Internal)             │
├────────────────────────────────────────────────────────────┤
│  All adapters produce this format after normalisation:     │
│                                                            │
│  {                                                         │
│    "schema_version": "2.0",                                │
│    "tenant_id": "utility-abc",                             │
│    "device_id": "meter-12345",                             │
│    "endpoint_name": "meter-12345.utility-abc.lwm2m",       │
│    "profile_id": "com.landis-gyr.e350.modbus.v1",          │
│    "lwm2m_version": "2.0",                                 │
│    "source_protocol": "modbus-tcp",                        │
│    "timestamp_utc": "2024-01-15T10:30:00Z",                │
│    "operation": "REPORT",                                  │
│    "lwm2m_objects": [                                      │
│      {                                                     │
│        "object_id": 3316,                                  │
│        "object_instance": 0,                               │
│        "resource_id": 5700,                                │
│        "resource_value": 42.48,                            │
│        "resource_type": "float",                           │
│        "encoding": "CBOR"                                  │
│      }                                                     │
│    ],                                                      │
│    "notification_attrs": {                                 │
│      "pmin": 30, "pmax": 300                               │
│    },                                                      │
│    "security_context": {                                   │
│      "transport_security": "DTLS_1_3",                     │
│      "cipher_suite": "TLS_AES_128_GCM_SHA256",             │
│      "oscore_context_id": "abc123"                         │
│    }                                                       │
│  }                                                         │
│                                                            │
│  → Internal event bus (NATS JetStream): CBOR-encoded      │
│  → Northbound to LwM2M Server: CoAP + CBOR payload        │
│  → Debug/diagnostic endpoints: JSON (human-readable)      │
└────────────────────────────────────────────────────────────┘
```

**Offline Buffering & Store-Forward**:
```
Gateway loses connectivity to LwM2M Server:
  ↓
1. Adapter continues polling devices, events flow to buffer-service
2. Buffer persists to local storage (Redis + disk WAL)
3. Configurable buffer depth: 1M messages per adapter
4. Overflow policy: drop oldest (FIFO) or reject new (back-pressure)
5. Metadata preserved: original timestamp, sequence number, device endpoint

Gateway reconnects:
  ↓
6. Buffer replays messages in original order
7. LwM2M Register + full object tree (catch-up)
8. NDATA/DDATA messages with historical flag
9. Server applies messages to device twin with original timestamps

At-least-once delivery semantics (idempotency via message UUID)
```

**Security Layers**:
```
Southbound (Device-facing):
  ├─ Per-protocol security:
  │  ├─ Modbus: VPN overlay (no native security)
  │  ├─ OPC UA: Security Policies (Basic256Sha256, Aes128-Sha256-RsaOaep)
  │  ├─ MQTT: TLS 1.3 + username/password or X.509 client certs
  │  ├─ DLMS/COSEM: HLS-GMAC authentication (DLMS v8)
  │  ├─ BACnet: BACnet/SC (Secure Connect) - TLS 1.3 + X.509
  │  └─ DNP3: Secure Authentication v5 (SAv5) - HMAC-SHA256
  │
  ├─ Hardware Root of Trust (optional):
  │  └─ Secure Element / TPM for private key storage (non-extractable)

Internal Service Mesh:
  ├─ mTLS between all microservices (Istio/Linkerd)
  ├─ SPIFFE/SPIRE workload identity
  └─ Network policy: deny-all default, explicit allow per service pair

Northbound (LwM2M Server):
  ├─ DTLS 1.3 (primary): TLS_AES_128_GCM_SHA256, TLS_CHACHA20_POLY1305_SHA256
  ├─ DTLS 1.2 (fallback)
  ├─ OSCORE (RFC 8613): Application-layer E2E through untrusted proxies
  │  ├─ AEAD: AES-CCM-16-64-128 (COSE)
  │  ├─ Sender/Recipient ID: derived from endpoint name + context ID
  │  └─ Replay protection: sequence number tracking (persisted in Redis)
  │
  └─ Authentication modes:
     ├─ PSK (Pre-Shared Key): Low overhead, embedded secret
     ├─ RPK (Raw Public Key): ECDSA without CA chain
     └─ X.509: Full PKI for enterprise deployments
```

**Observability (OpenTelemetry)**:
```
Metrics (Prometheus):
  ├─ Per-adapter: msg/sec, error rate, latency histogram, connected devices
  ├─ Virtual endpoints: count, state distribution (ACTIVE, DRAINING, ERROR)
  ├─ Buffer: depth, overflow events, replay lag
  ├─ CBOR compression ratio vs JSON baseline
  ├─ DTLS handshake success rate, session resumption rate
  └─ OSCORE: context establishment rate, replay rejection count

Traces (Jaeger/Tempo):
  ├─ Full distributed trace per message (adapter → core → DMP)
  ├─ Trace ID propagated through all services
  ├─ OSCORE context ID in trace metadata for E2E correlation
  └─ Spans: adapter.poll, normalise, encode, transmit, ack

Logs (Loki/Elasticsearch):
  ├─ Structured JSON logs (timestamp, level, service, trace_id, message)
  └─ Log aggregation with trace correlation

Alerts (Alertmanager):
  ├─ Adapter disconnect (> 5 min)
  ├─ Buffer overflow (depth > 90%)
  ├─ Error rate > 1%
  ├─ Certificate expiry < 30 days
  └─ Virtual endpoint churn > 5% (devices rapidly connecting/disconnecting)
```

**Deployment Models**:
```
1. Edge Gateway (Docker Compose)
   ├─ Single-node, on-premise deployment
   ├─ Industrial gateway hardware (ARM/x86)
   ├─ Local SQLite/PostgreSQL for device registry
   └─ Use case: Factory floor, substation, building controller

2. Kubernetes Cluster (Cloud/Hybrid)
   ├─ Multi-tenant, carrier-grade
   ├─ Horizontal Pod Autoscaling (HPA) per adapter type
   ├─ PodDisruptionBudget for rolling updates
   ├─ StatefulSet for buffer-service (persistent volumes)
   ├─ Helm chart with CRDs (ProtocolAdapter, DeviceProfile, VirtualEndpoint)
   └─ Use case: Telco DMP, smart city platform, utility head-end system

3. Edge Kubernetes (K3s/MicroK8s)
   ├─ Lightweight Kubernetes for edge devices
   ├─ Reduced resource footprint (512MB RAM minimum)
   └─ Use case: Industrial IoT gateway clusters, distributed deployments
```

**Use Cases**:
```
1. Smart Metering (LwM2M 2.0 Gateway + Modbus)
   ├─ 50,000 Modbus RTU meters (Landis+Gyr, Itron, Elster)
   ├─ Profile ID auto-provisioning per meter model
   ├─ Gateway Object 25 (identity) + Object 26 (routing table)
   ├─ CBOR encoding → 30% bandwidth reduction vs JSON
   └─ AVSystem Coiote DMP (Deutsche Telekom, Orange)

2. Industrial SCADA (OPC UA + Modbus + DNP3)
   ├─ OPC UA PLCs (Siemens S7-1500, Allen-Bradley ControlLogix)
   ├─ Modbus RTUs (legacy RTUs, power meters)
   ├─ DNP3 outstations (SCADA field devices)
   ├─ Sparkplug B northbound (MQTT to historian)
   └─ Unified namespace: spBv1.0/Factory1/NDATA/Gateway1

3. Building Automation (BACnet + KNX + Modbus)
   ├─ BACnet/IP HVAC controllers
   ├─ KNX lighting/blinds (via KNX IoT Point API)
   ├─ Modbus energy meters
   ├─ Azure IoT Edge northbound (cloud analytics)
   └─ EdgeX Foundry device services framework

4. Smart City (Multi-Protocol + uCIFI)
   ├─ DALI-2 streetlights (IEC 62386)
   ├─ LoRaWAN environmental sensors
   ├─ BLE asset trackers (parking sensors, waste bins)
   ├─ LwM2M Object 10241/10242 (uCIFI streetlight control)
   └─ Eclipse Hono multi-tenant gateway
```

---

## 10. Extended Gateway Protocol Coverage

**Purpose**: Comprehensive protocol adapter inventory covering building automation, agriculture, automotive, and industrial IoT verticals. Based on market coverage analysis identifying gaps in existing edge gateway platforms.

### 10.1 Building Automation Protocols

**DALI-2 / DALI+ (IEC 62386)** — Digital Addressable Lighting Interface:
```
Purpose: Commercial lighting control (LED drivers, luminaires, emergency lighting)
Market:  Dominant in modern commercial buildings (>60% of installations)

Architecture:
  ┌─────────────────┐      ┌──────────────────┐      ┌─────────────────┐
  │  DALI Gateway   │ 2-wire│  DALI Devices   │      │ BMS Integration │
  │  (Bus Master)   │──────>│  (Ballasts, LED  │<────>│ (BACnet/KNX)    │
  │  USB/Ethernet   │ bus   │   Drivers, Sensors)     │                 │
  └─────────────────┘       └──────────────────┘      └─────────────────┘

Addressing:
  - 64 short addresses per bus (0-63)
  - 16 groups (scene control)
  - Broadcast address (254)

Commands (8-bit frames, Manchester encoding, 1200 baud):
  ├─ DAPC (Direct Arc Power Control): Set brightness 0-254
  ├─ OFF / UP / DOWN / STEP UP / STEP DOWN
  ├─ GO TO SCENE (16 scenes per device)
  ├─ QUERY STATUS / QUERY LAMP FAILURE
  └─ STORE DTR AS FADE TIME (color control DT8)

DALI-2 Enhancements:
  - Device Type 8 (DT8): Color control (tunable white, RGB, RGBW)
  - Event-based push notifications (motion detection, daylight sensing)
  - Extended memory banks (fault logs, energy consumption)

Gateway Mapping:
  DALI Device Address 12 → LwM2M Object 3311 (Light Control)
    ├─ /3311/0/5850 (On/Off) ← DALI OFF/ON command
    ├─ /3311/0/5851 (Dimmer) ← DALI DAPC 0-254 → 0-100%
    └─ /3311/0/5706 (Color) ← DALI DT8 color commands
```

**LonWorks / LonTalk (EN 14908)** — Legacy BAS:
```
Purpose: Legacy BAS protocol (1995-2015 era), 15% of global installed base
Market:  North American retrofit, Honeywell/Siemens/Johnson Controls estates

Architecture:
  - Neuron chip (discontinued by Renesas → driving gateway demand)
  - Network Variables (NVs): Published/subscribed data points
  - LNS (LonWorks Network Services): Configuration database

Challenges:
  - Complex protocol stack (7 layers)
  - SNVT/UNVT data types (standard/user network variable types)
  - LNS database integration for NV bindings

Gateway Strategy:
  - Use open-source LON stacks (lon4linux)
  - Map NVs to LwM2M objects via configuration
  - Translate LonTalk frames to BACnet/IP or LwM2M for cloud integration
```

**EnOcean (ISO 14543-3-1X)** — Energy Harvesting Wireless:
```
Purpose: Battery-free sensors (occupancy, temperature, switches)
Market:  Green buildings, LEED certifications, 5000+ products

Radio: 868 MHz (EU), 902 MHz (US), 315 MHz (Asia)
Energy Harvesting:
  ├─ Solar cells (light switches)
  ├─ Piezo generators (push buttons)
  ├─ Thermoelectric (temperature differential)
  └─ Electromagnetic (motion)

EnOcean Equipment Profiles (EEPs):
  - A5-02-05: Temperature sensor (0-40°C, 0.1°C resolution)
  - A5-07-01: Occupancy sensor with illumination
  - D2-01-12: Electronic switches and dimmers
  - D2-14-41: Multi-sensor (temp, humidity, illumination, motion)

Gateway Integration:
  USB receiver (TCM310, ESP3 protocol) → Parse telegrams → LwM2M Object 3303/3304
```

### 10.2 Agriculture Protocols

**SDI-12 (Serial Data Interface at 1200 Baud)**:
```
Purpose: Environmental/soil sensors (THE standard for precision agriculture)
Market:  Every major soil sensor manufacturer (Meter Group, Campbell Scientific)

Physical:
  - 2-wire bus (data + ground), max 60m
  - 1200 baud, 7-bit ASCII
  - Daisy-chain up to 10+ sensors per bus

Command/Response:
  aM!     → Start measurement on sensor 'a', get number of values + wait time
  aD0!    → Retrieve measurement values (CSV format)
  aI!     → Identify sensor (manufacturer, model, version, serial)

Example Session:
  Gateway → 0M!      (Measure command to sensor address 0)
  Sensor  → 0014     (14 seconds until measurement ready, 0 values immediately)
  [Wait 14 seconds]
  Gateway → 0D0!     (Retrieve data)
  Sensor  → 0+23.5+45.2+1.234  (Temperature °C, Humidity %, EC mS/cm)

Gateway Mapping:
  SDI-12 Sensor 0 → LwM2M Object 3303 (Temperature) + 3304 (Humidity) + 3327 (EC)
```

**ISOBUS (ISO 11783)** — Farm Machinery CAN:
```
Purpose: Precision farming (tractors, implements, seeders, sprayers)
Market:  Managed by AEF (Agricultural Industry Electronics Foundation)

Based on: CAN 2.0B (250 kbps) + J1939 transport layer
Key Subsystems:
  ├─ Task Controller (TC): Prescription maps, variable rate application
  ├─ Virtual Terminal (VT): Tractor display protocol
  ├─ Working Set Management: Implement discovery and address claiming
  └─ Data logging: Section control, yield monitoring

PGNs (Parameter Group Numbers):
  - PGN 65267: Vehicle Position (GPS latitude, longitude, altitude)
  - PGN 65215: Wheel-Based Speed and Distance
  - PGN 64734: Product Control (application rate, section on/off)

Gateway Requirements:
  - SocketCAN on Linux (CAN interface)
  - ISO 11783 stack (commercial: AGCO CCI-A3, open-source: ISOBUS++)
  - Map PGNs to LwM2M Object 6 (Location) + 3336 (Humidity) + custom app objects
```

**BLE 5.x (Bluetooth Low Energy)** — Livestock/Sensor Beacons:
```
Use Cases:
  - Animal tracking (ear tags, collar sensors)
  - Greenhouse micro-sensor networks
  - Soil sensor beacons
  - Irrigation valve controllers

BLE Features:
  - BLE 5.0: 2x speed (2 Mbps), 4x range (240m outdoor)
  - BLE 5.1: Direction finding (AoA/AoD for asset location)
  - BLE Mesh: Many-to-many relay (extends range)

GATT Profiles:
  - Environmental Sensing Service (0x181A): Temperature, humidity, pressure
  - Device Information Service (0x180A): Manufacturer, model, firmware version
  - Battery Service (0x180F): Battery level %

Gateway Pattern:
  BLE Central (gateway) scans for advertising devices
  → Connects to GATT server
  → Subscribes to characteristic notifications
  → Maps to LwM2M Object 3303 (Temperature), 3304 (Humidity), 3320 (Battery)
```

**4-20mA Analog Current Loop** — Industrial Sensors:
```
Purpose: Industry-standard analog interface (soil probes, flow meters, pressure)
Signal:  4 mA = 0% (zero scale), 20 mA = 100% (full scale)

Gateway Hardware:
  - ADC module (16-bit resolution recommended)
  - 250Ω precision resistor (converts current to voltage: 1-5V)
  - Calibration per channel (offset, gain)

Mapping:
  ADC Channel 0 (4-20mA soil moisture) → LwM2M Object 3304 Instance 0
  Formula: moisture_% = ((I_mA - 4) / 16) * 100
```

### 10.3 Automotive / Fleet Telematics Protocols

**SAE J1939** — Heavy-Duty Vehicle CAN:
```
Purpose: Commercial trucks, buses, construction equipment (THE protocol)
Market:  Every fleet telematics platform (Geotab, Samsara, Trimble)

CAN Physical:
  - 250 kbps or 500 kbps
  - 29-bit extended CAN identifiers
  - 1000+ Parameter Group Numbers (PGNs)

Key PGNs:
  - PGN 61444: Engine Speed (RPM)
  - PGN 65265: Cruise Control/Vehicle Speed
  - PGN 65266: Fuel Economy
  - PGN 65276: Dash Display (odometer, fuel level, coolant temp)
  - PGN 65226: Active Diagnostic Trouble Codes (DTCs)
  - DM1/DM2: Diagnostic messages (active/previously active faults)

Transport Protocol:
  - BAM (Broadcast Announce Message): One-to-many (max 1785 bytes)
  - CMDT (Connection Mode Data Transfer): Point-to-point with flow control

Gateway Architecture:
  SocketCAN + J1939 decoder library → Parse PGNs
  → Map to LwM2M Objects:
    - 3336 (Generic Sensor) for engine RPM, fuel rate
    - 3337 (Fuel Level)
    - 3338 (Vehicle Diagnostics) for DTCs
```

**FMS Standard (Fleet Management Systems — ACEA)**:
```
Purpose: European standardized subset of J1939 for truck telematics
Defined by: ACEA (European Automobile Manufacturers Association)

Difference from J1939:
  - Controlled, OEM-approved interface (no proprietary CAN access)
  - Dedicated CAN bus or FMS gateway
  - Subset of PGNs: fuel, speed, distance, engine hours, PTO status

OEMs Supporting FMS:
  MAN, DAF, Volvo, Scania, Iveco, Mercedes-Benz

Gateway Pattern:
  FMS is J1939 subset → Use J1939 adapter with FMS PGN filter configuration
```

**V2X (C-V2X / DSRC / IEEE 802.11p)** — Vehicle-to-Everything:
```
Purpose: Connected vehicles (safety, traffic management, autonomous driving)
Market:  25% CAGR, European eCall/C-ITS mandates

Technologies:
  ├─ C-V2X (Cellular V2X): 3GPP Release 14+ PC5 sidelink (forward-looking)
  ├─ DSRC (Dedicated Short-Range Communications): IEEE 802.11p / ETSI ITS-G5 (legacy)
  └─ 5G NR-V2X: 3GPP Release 16+ (high bandwidth, low latency)

Message Types (ASN.1 Encoding):
  - BSM (Basic Safety Message / CAM): Vehicle position, speed, heading, accel (10 Hz)
  - MAP: Intersection geometry (lanes, stop lines, crosswalks)
  - SPaT: Signal Phase and Timing (traffic light state)
  - DENM: Decentralized Environmental Notification Message (hazard warnings)

Gateway Role:
  - Receive V2X messages (OBU → Gateway)
  - Parse ASN.1 (UPER encoding)
  - Forward to traffic management center or fleet platform
  - Map to LwM2M Object 6 (Location) + custom V2X objects
```

**NMEA 2000 / NMEA 0183** — Marine / Agricultural GPS:
```
NMEA 2000:
  - Based on CAN 2.0B (250 kbps)
  - PGN-based (similar to J1939)
  - Common PGNs:
    - PGN 129029: GNSS Position Data (lat, lon, altitude)
    - PGN 130306: Wind Data (speed, direction)
    - PGN 127488: Engine Parameters, Rapid Update
    - PGN 127505: Fluid Level (fuel, water)

NMEA 0183 (Legacy Serial):
  - RS-422, 4800 baud
  - ASCII sentences: $GPGGA (GPS fix), $GPRMC (recommended minimum), $GPVTG (track/speed)

Gateway: Parse PGNs or NMEA sentences → LwM2M Object 6 (Location)
```

### 10.4 Additional Industrial / Enterprise Protocols

**SNMP v2c/v3** — Network Equipment Monitoring:
```
Covered in device-management.md section 6, but relevant for gateway integration:
  - Monitor gateway uptime, interface stats, system load
  - SNMP traps for alerts (port down, high CPU, disk full)
  - Integrate with enterprise NMS (Nagios, Zabbix, PRTG)
```

**WirelessHART / ISA100.11a** — Industrial Wireless Mesh:
```
Covered in pan-short-range.md, but gateway pattern:
  - Gateway acts as WirelessHART/ISA100 Network Manager
  - TDMA + DSSS on 2.4 GHz (IEEE 802.15.4)
  - Maps process variables to Modbus/OPC UA northbound
  - Cross-reference: [pan-short-range.md](pan-short-range.md)
```

**LoRaWAN** — Long-Range LPWAN:
```
Covered in lpwan.md section 2, but gateway role:
  - LoRa Gateway (Concentrator IC like SX1301/SX1303)
  - Packet Forwarder → Network Server (ChirpStack, The Things Network)
  - Application Server decodes payloads → LwM2M/MQTT
  - Cross-reference: [lpwan.md](lpwan.md)
```

**Sigfox** — Ultra-Narrowband LPWAN:
```
Covered in lpwan.md section 4
  - Gateway forwards to Sigfox Cloud
  - Callback API delivers payloads to application server
  - Cross-reference: [lpwan.md](lpwan.md)
```

---

## Key Specification References

| Technology      | Primary Spec                         | Publisher       | Year |
|-----------------|--------------------------------------|-----------------|------|
| Sparkplug B     | Eclipse Sparkplug B 1.0 / 3.0        | Eclipse         | 2016/2024 |
| EdgeX Foundry   | EdgeX 3.0 (Minnesota)                | LF Edge         | 2023 |
| Azure IoT Edge  | IoT Edge Runtime 1.4                 | Microsoft       | 2023 |
| AWS Greengrass  | Greengrass Core v2.11                | AWS             | 2023 |
| Eclipse Hono    | Hono 2.4                             | Eclipse         | 2023 |
| OPC UA PubSub   | IEC 62541-14                         | OPC Foundation  | 2020 |
| KNX IoT         | KNX IoT Point API Spec 1.0           | KNX Association | 2021 |
| LwM2M Gateway   | OMA LwM2M v1.2 / v2.0                | OMA SpecWorks   | 2020/2026 |
| DALI-2          | IEC 62386 (all parts)                | IEC             | 2014-2022 |
| LonWorks        | EN 14908 (ISO/IEC 14908)             | Echelon/ISO     | 2012 |
| EnOcean         | ISO 14543-3-1X                       | EnOcean Alliance| 2012-2020 |
| SDI-12          | SDI-12 Version 1.4                   | SDI-12 Support  | 2016 |
| ISOBUS          | ISO 11783 (all parts)                | ISO/AEF         | 2007-2019 |
| SAE J1939       | SAE J1939 (J1939-71, -73, -81)       | SAE International| 2016 |
| FMS Standard    | FMS Interface Specification v4       | ACEA            | 2018 |
| C-V2X           | 3GPP Release 14/16 (PC5, NR-V2X)     | 3GPP            | 2017/2020 |
| NMEA 2000       | NMEA 2000 Standard                   | NMEA            | 2012 |
| NMEA 0183       | NMEA 0183 Version 4.11               | NMEA            | 2018 |

---

## Related Files
- **[application-messaging.md](application-messaging.md)**: MQTT, AMQP, CoAP protocols used by edge platforms
- **[industrial-ot.md](industrial-ot.md)**: OPC UA, Modbus, BACnet device-level protocols
- **[device-management.md](device-management.md)**: LwM2M, TR-369 for edge device lifecycle management
- **[lpwan.md](lpwan.md)**: LwM2M gateway patterns, NB-IoT edge integration
- **[data-encoding.md](data-encoding.md)**: Protobuf (Sparkplug), CBOR, JSON encodings
