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

## Key Specification References

| Technology      | Primary Spec                         | Publisher       | Year |
|-----------------|--------------------------------------|-----------------|------|
| Sparkplug B     | Eclipse Sparkplug B 1.0              | Eclipse         | 2016 |
| EdgeX Foundry   | EdgeX 3.0 (Minnesota)                | LF Edge         | 2023 |
| Azure IoT Edge  | IoT Edge Runtime 1.4                 | Microsoft       | 2023 |
| AWS Greengrass  | Greengrass Core v2.11                | AWS             | 2023 |
| Eclipse Hono    | Hono 2.4                             | Eclipse         | 2023 |
| OPC UA PubSub   | IEC 62541-14                         | OPC Foundation  | 2020 |
| KNX IoT         | KNX IoT Point API Spec 1.0           | KNX Association | 2021 |

---

## Related Files
- **[application-messaging.md](application-messaging.md)**: MQTT, AMQP, CoAP protocols used by edge platforms
- **[industrial-ot.md](industrial-ot.md)**: OPC UA, Modbus, BACnet device-level protocols
- **[device-management.md](device-management.md)**: LwM2M, TR-369 for edge device lifecycle management
- **[lpwan.md](lpwan.md)**: LwM2M gateway patterns, NB-IoT edge integration
- **[data-encoding.md](data-encoding.md)**: Protobuf (Sparkplug), CBOR, JSON encodings
