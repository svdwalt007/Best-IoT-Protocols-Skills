# Data Encoding & Modeling

**Scope**: Data serialization formats, schema languages, and information models for IoT telemetry, configuration, and semantic interoperability. Covers binary encodings (CBOR, Protobuf), text formats (JSON, XML), industrial models (OPC UA, YANG), semantic web (NGSI-LD, Haystack), and sensor metadata (IEEE 1451, IPSO Smart Objects). Cross-references [application-messaging.md](application-messaging.md) for MQTT/CoAP payload formats, [industrial-ot.md](industrial-ot.md) for OPC UA information model, and [device-management.md](device-management.md) for YANG/NETCONF.

## Table of Contents
1. CBOR (Concise Binary Object Representation)
2. SenML (Sensor Measurement Lists)
3. Protocol Buffers (Protobuf)
4. JSON / JSON-LD / JSON Schema
5. YANG (Data Modeling Language)
6. OPC UA Information Model
7. FIWARE NGSI-LD
8. Project Haystack 4.0
9. SCHC (Static Context Header Compression)
10. IEEE 1451 TEDS (Transducer Electronic Data Sheet)
11. IPSO Smart Objects / OMNA Registry

---

## 1. CBOR (Concise Binary Object Representation)

**Reference**: RFC 8949 (CBOR), RFC 8610 (CDDL - CBOR Data Definition Language)

**Purpose**: Compact binary encoding for constrained IoT devices. 30-70% smaller than JSON, self-describing, extensible.

**Data Types**:
```
Major Types:
  0 - Unsigned integer (0-18446744073709551615)
  1 - Negative integer (-1 to -18446744073709551616)
  2 - Byte string (binary data)
  3 - Text string (UTF-8)
  4 - Array (ordered sequence)
  5 - Map (key-value pairs)
  6 - Tag (semantic annotation, e.g., datetime, UUID)
  7 - Float / Simple values (true, false, null)
```

**Encoding Examples**:
```
Integer 42:
  Hex: 0x18 2A
  Binary: 000_11000 00101010
          │    │      └─ 42 in binary
          │    └─ Additional info: 1-byte uint8 follows
          └─ Major type 0 (unsigned int)

Text "hello":
  Hex: 0x65 68 65 6C 6C 6F
       │     └─ "hello" in ASCII
       └─ Major type 3, length 5

Array [1, 2, 3]:
  Hex: 0x83 01 02 03
       │     └─ Elements: 1, 2, 3
       └─ Major type 4, array of 3 items

Map {"temp": 23.5}:
  Hex: 0xA1 64 74656D70 F9 4DF0
       │    │            └─ Half-precision float 23.5
       │    └─ Key: "temp" (4-char string)
       └─ Major type 5, map with 1 pair
```

**Tags** (Major Type 6):
```
Tag 0:  DateTime (RFC 3339 text string)
        0xC0 74 323032332D31322D30315431303A30303A30305A
        → "2023-12-01T10:00:00Z"

Tag 1:  Epoch timestamp (Unix time, integer or float)
        0xC1 1A 656B8B80
        → 1701432000 (seconds since 1970-01-01)

Tag 37: UUID (16-byte binary)
        0xD8 25 50 550E8400E29B41D4A716446655440000
        → UUID: 550e8400-e29b-41d4-a716-446655440000

Tag 55799: CBOR magic number (self-describe CBOR)
        0xD9 D9F7 ...
        (helps identify CBOR streams, skipped by decoders)
```

**Deterministic Encoding** (RFC 8949 §4.2):
```
Rules:
  1. Map keys sorted by lexicographic byte order
  2. Integers encoded in smallest form
  3. Floats encoded as smallest IEEE 754 (half/single/double)
  4. No indefinite-length encoding

Use case: Cryptographic signatures (COSE RFC 8152), blockchain
```

**CDDL Schema Example**:
```cddl
sensor-reading = {
  device-id: tstr,
  timestamp: #6.1(uint),  ; Tag 1 (epoch time)
  ? location: coordinate,
  measurements: [+ measurement]
}

coordinate = {
  lat: float,
  lon: float
}

measurement = {
  name: tstr,
  value: float,
  unit: tstr
}
```

**Use Case**: CoAP payloads (Content-Format 60), EDHOC (ephemeral key exchange), COSE (CBOR Object Signing and Encryption).

---

## 2. SenML (Sensor Measurement Lists)

**Reference**: RFC 8428 (SenML), RFC 8790 (SenML Features)

**Purpose**: Lightweight format for sensor data. Optimized for time-series telemetry with base-value compression.

**SenML JSON**:
```json
[
  {
    "bn": "urn:dev:mac:001122334455",  // Base Name
    "bt": 1638360000,                   // Base Time (Unix epoch)
    "bu": "Cel",                        // Base Unit (Celsius)
    "n": "temperature",                 // Name
    "v": 23.5,                          // Value
    "t": 0                              // Time offset from bt
  },
  {
    "n": "humidity",
    "u": "%RH",                         // Unit (relative humidity)
    "v": 45.2,
    "t": 0
  },
  {
    "n": "temperature",
    "v": 23.7,
    "t": 60                            // bt + 60 = 1 minute later
  }
]
```

**SenML CBOR** (Content-Format 112):
```
Same structure, CBOR-encoded:
[
  {-2: "urn:dev:mac:001122334455", -3: 1638360000, -4: "Cel",
   0: "temperature", 2: 23.5, 6: 0},
  {0: "humidity", 1: "%RH", 2: 45.2, 6: 0},
  {0: "temperature", 2: 23.7, 6: 60}
]

Label mapping (RFC 8428 §6):
  -2 = bn (Base Name)
  -3 = bt (Base Time)
  -4 = bu (Base Unit)
   0 = n (Name)
   1 = u (Unit)
   2 = v (Value - float)
   6 = t (Time offset)
```

**SenML Units** (IANA Registry):
```
Common Units:
  m      - Meter (length)
  kg     - Kilogram (mass)
  s      - Second (time)
  A      - Ampere (electric current)
  K      - Kelvin (temperature)
  Cel    - Degrees Celsius
  %RH    - Relative Humidity (percent)
  W      - Watt (power)
  kWh    - Kilowatt-hour (energy)
  lat    - Latitude (degrees)
  lon    - Longitude (degrees)
  pH     - pH value (acidity)
  dBm    - Decibel milliwatt (signal strength)
```

**SenML Pack** (Base Value Compression):
```
Without pack (3 records, 240 bytes JSON):
[
  {"n": "urn:dev:mac:001122334455/temp", "v": 23.5, "t": 1638360000},
  {"n": "urn:dev:mac:001122334455/hum", "v": 45.2, "t": 1638360000},
  {"n": "urn:dev:mac:001122334455/pres", "v": 1013.2, "t": 1638360000}
]

With pack (120 bytes JSON, 50% reduction):
[
  {"bn": "urn:dev:mac:001122334455/", "bt": 1638360000,
   "n": "temp", "v": 23.5},
  {"n": "hum", "v": 45.2},
  {"n": "pres", "v": 1013.2}
]
```

**Use Case**: CoAP Observe notifications, MQTT telemetry (JSON topic payload), LwM2M Composite Read (RFC 8790).

---

## 3. Protocol Buffers (Protobuf)

**Reference**: Protobuf Language Guide v3 (Google)

**Purpose**: Strongly-typed binary serialization. Efficient, backward-compatible schema evolution, code generation for 10+ languages.

**Proto3 Syntax**:
```protobuf
syntax = "proto3";

message SensorReading {
  string device_id = 1;
  int64 timestamp = 2;         // Unix epoch milliseconds
  Location location = 3;
  repeated Measurement measurements = 4;
}

message Location {
  double latitude = 1;
  double longitude = 2;
  optional float altitude = 3;  // Optional field (proto3)
}

message Measurement {
  string name = 1;
  oneof value {
    float float_value = 2;
    int32 int_value = 3;
    bool bool_value = 4;
    string string_value = 5;
  }
  string unit = 6;
}
```

**Wire Format** (Varint Encoding):
```
Field tag encoding:
  Tag = (field_number << 3) | wire_type

Example: field_number=1, wire_type=0 (varint)
  Tag = (1 << 3) | 0 = 0x08

Message bytes:
  0x08 0x96 0x01     → field 1 (device_id, varint), value=150
  0x10 0xE0 0x97 ... → field 2 (timestamp, varint), value=...

Varint encoding (7 bits per byte, MSB=continuation):
  150 (decimal):
    Binary: 10010110
    Varint: 0x96 0x01 (little-endian, 7-bit chunks)
```

**Well-Known Types**:
```protobuf
import "google/protobuf/timestamp.proto";
import "google/protobuf/duration.proto";
import "google/protobuf/struct.proto";

message Event {
  google.protobuf.Timestamp occurred_at = 1;
  google.protobuf.Duration processing_time = 2;
  google.protobuf.Struct metadata = 3;  // Arbitrary JSON-like data
}
```

**Backward Compatibility**:
```
Rules:
  1. Never change field numbers (wire format breaks)
  2. New fields → use new field numbers (old decoders ignore)
  3. Deprecated fields → mark "reserved" to prevent reuse
  4. Enum: always include 0 value (proto3 default)

Example evolution:
  // v1
  message Sensor {
    string id = 1;
    float temperature = 2;
  }

  // v2 (backward-compatible)
  message Sensor {
    string id = 1;
    float temperature = 2;
    float humidity = 3;       // New field (old decoders ignore)
    reserved 4;               // Reserved for future
    reserved "old_field";     // Prevent name reuse
  }
```

**Use Case**: gRPC services (see [device-management.md](device-management.md)), Sparkplug B (see [edge-gateway.md](edge-gateway.md)), Kafka IoT telemetry.

---

## 4. JSON / JSON-LD / JSON Schema

**JSON** (JavaScript Object Notation):
```json
{
  "device": "sensor-001",
  "timestamp": "2023-12-01T10:00:00Z",
  "readings": [
    {"parameter": "temperature", "value": 23.5, "unit": "Cel"},
    {"parameter": "humidity", "value": 45.2, "unit": "%RH"}
  ]
}
```

**JSON-LD** (Linked Data):
```json
{
  "@context": {
    "schema": "https://schema.org/",
    "fiware": "https://uri.fiware.org/ns/data-models#"
  },
  "@id": "urn:ngsi-ld:Device:sensor-001",
  "@type": "fiware:Device",
  "schema:name": "Temperature Sensor 001",
  "fiware:temperature": {
    "@type": "schema:QuantitativeValue",
    "schema:value": 23.5,
    "schema:unitCode": "CEL"
  }
}
```

**JSON Schema** (Validation):
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "SensorReading",
  "type": "object",
  "required": ["device", "timestamp", "readings"],
  "properties": {
    "device": {
      "type": "string",
      "pattern": "^sensor-[0-9]{3}$"
    },
    "timestamp": {
      "type": "string",
      "format": "date-time"
    },
    "readings": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["parameter", "value"],
        "properties": {
          "parameter": {"type": "string"},
          "value": {"type": "number"},
          "unit": {"type": "string"}
        }
      }
    }
  }
}
```

---

## 5. YANG (Data Modeling Language)

**Reference**: RFC 7950 (YANG 1.1)

**Purpose**: Model configuration and state data for network devices. Used with NETCONF, RESTCONF, gNMI.

**(See [device-management.md](device-management.md) section 4 for full YANG coverage)**

**Example Module**:
```yang
module iot-sensor {
  namespace "http://example.com/ns/iot-sensor";
  prefix "sensor";

  import ietf-inet-types { prefix inet; }

  revision 2023-12-01 { description "Initial release"; }

  container measurements {
    config false;  // Read-only operational data

    leaf temperature {
      type decimal64 { fraction-digits 2; }
      units "celsius";
    }

    leaf-list co2-ppm {
      type uint16 { range "400..5000"; }
      max-elements 10;
    }
  }
}
```

---

## 6. OPC UA Information Model

**Reference**: IEC 62541-3 (OPC UA Part 3: Address Space Model)

**Purpose**: Object-oriented information model for industrial automation. Defines device capabilities, relationships, and semantics.

**NodeClass Types**:
```
Object:       Represents physical or logical entity (e.g., "Boiler")
Variable:     Data value (e.g., "Temperature" = 23.5)
Method:       Callable function (e.g., "StartPump()")
ObjectType:   Template for Objects (e.g., "BoilerType")
VariableType: Template for Variables (e.g., "AnalogItemType")
DataType:     Type definition (Int32, Float, custom structures)
ReferenceType: Relationship (e.g., "HasComponent", "HasProperty")
View:         Subset of AddressSpace for navigation
```

**AddressSpace Example**:
```
Server (Object)
  └─ Objects (Object)
      └─ Boiler1 (Object, TypeDefinition: BoilerType)
          ├─ HasComponent → PipeX001 (Object)
          │   └─ HasProperty → Temperature (Variable)
          │       ├─ Value: 87.3 (Float)
          │       ├─ EngineeringUnits: °C
          │       └─ EURange: [0, 150]
          ├─ HasComponent → Drum (Object)
          │   └─ HasProperty → LevelIndicator (Variable)
          └─ HasMethod → StartBoiler (Method)
              └─ InputArguments: [Duration (UInt32)]
```

**NodeId Format**:
```
ns=2;i=1234          → Numeric (namespace 2, identifier 1234)
ns=3;s=Boiler1.Temp  → String (namespace 3, identifier "Boiler1.Temp")
ns=1;g=<UUID>        → GUID (namespace 1, UUID identifier)
ns=0;i=58            → Namespace 0 (OPC UA built-in types)
```

**Data Encodings**:
```
1. Binary (UA Binary): Compact, efficient (default for OPC UA TCP)
2. XML (OPC UA XML): Interoperable, verbose
3. JSON (OPC UA JSON): Web-friendly, human-readable
```

**Use Case**: Factory automation (PLC data modeling), building automation (BACnet-to-OPC UA gateway).

---

## 7. FIWARE NGSI-LD

**Reference**: ETSI GS CIM 009 (NGSI-LD API)

**Purpose**: Semantic context information management for smart cities. JSON-LD-based entity/attribute/relationship model.

**Entity Structure**:
```json
{
  "id": "urn:ngsi-ld:AirQualityObserved:Madrid-AmbientObserved-28079004-2023-12-01T10:00:00.00Z",
  "type": "AirQualityObserved",
  "@context": [
    "https://uri.etsi.org/ngsi-ld/v1/ngsi-ld-core-context.jsonld",
    "https://raw.githubusercontent.com/smart-data-models/dataModel.Environment/master/context.jsonld"
  ],
  "dateObserved": {
    "type": "Property",
    "value": {
      "@type": "DateTime",
      "@value": "2023-12-01T10:00:00Z"
    }
  },
  "location": {
    "type": "GeoProperty",
    "value": {
      "type": "Point",
      "coordinates": [-3.7038, 40.4168]
    }
  },
  "NO2": {
    "type": "Property",
    "value": 42,
    "unitCode": "GP",  // Microgram per cubic meter
    "observedAt": "2023-12-01T10:00:00Z"
  },
  "refDevice": {
    "type": "Relationship",
    "object": "urn:ngsi-ld:Device:AirQualitySensor-001"
  }
}
```

**Context Broker API**:
```http
POST /ngsi-ld/v1/entities
Content-Type: application/ld+json
Link: <https://uri.etsi.org/ngsi-ld/v1/ngsi-ld-core-context.jsonld>; rel="http://www.w3.org/ns/json-ld#context"

GET /ngsi-ld/v1/entities?type=AirQualityObserved&q=NO2>50

PATCH /ngsi-ld/v1/entities/urn:ngsi-ld:Device:001/attrs
{
  "temperature": {
    "type": "Property",
    "value": 24.1
  }
}

POST /ngsi-ld/v1/subscriptions
{
  "type": "Subscription",
  "entities": [{"type": "AirQualityObserved"}],
  "watchedAttributes": ["NO2"],
  "q": "NO2>100",
  "notification": {
    "endpoint": {
      "uri": "http://my-app.example.com/notify",
      "accept": "application/json"
    }
  }
}
```

**Use Case**: Smart city platforms (FIWARE Orion Context Broker), digital twins (asset relationship modeling).

---

## 8. Project Haystack 4.0

**Reference**: Project Haystack 4.0 Specification

**Purpose**: Semantic tagging for building automation, energy systems. Uses tags, dicts, and grids for equipment modeling.

**Tagging Ontology**:
```
site           → Building or facility
equip          → Physical equipment
point          → Data point (sensor/actuator)
sensor         → Sensing point
cmd            → Command/setpoint
ahu            → Air Handling Unit
vav            → Variable Air Volume box
temp           → Temperature
sp             → Setpoint
cur            → Current value
```

**Example (Zinc Format)**:
```zinc
id: @ahu-001
dis: "AHU-1 Rooftop Unit"
ahu
equip
siteRef: @site-building-a
---
id: @ahu-001.sat
dis: "Supply Air Temp"
point
sensor
temp
cur
equipRef: @ahu-001
unit: "°F"
curVal: 55.2
```

**Haystack JSON**:
```json
{
  "_kind": "grid",
  "meta": {"ver": "3.0"},
  "cols": [
    {"name": "id"},
    {"name": "dis"},
    {"name": "point"},
    {"name": "curVal"},
    {"name": "unit"}
  ],
  "rows": [
    {
      "id": "r:ahu-001.sat",
      "dis": "s:Supply Air Temp",
      "point": "m:",
      "curVal": "n:55.2",
      "unit": "s:°F"
    }
  ]
}
```

**Use Case**: Building Management Systems (BMS), energy analytics platforms (SkySpark).

---

## 9. SCHC (Static Context Header Compression)

**Reference**: RFC 8724 (SCHC)

**Purpose**: Compress IPv6/UDP/CoAP headers for LPWAN (LoRaWAN, NB-IoT, Sigfox). Reduces 40+ byte headers to 1-2 bytes.

**Compression Context** (Rules):
```
Rule ID 1:
  IPv6 Source:      fc00::1/128           → Elided (fixed)
  IPv6 Dest:        fc00::2/128           → Elided (fixed)
  UDP Src Port:     5683 (CoAP)           → Elided (fixed)
  UDP Dst Port:     5683                  → Elided (fixed)
  CoAP Token:       Sent (variable)       → Sent as-is
  CoAP Uri-Path:    /temperature          → Mapped to ID 0x01

Compressed header:
  [Rule ID: 1 byte] [CoAP Token: 2 bytes] [Uri-Path ID: 1 byte] = 4 bytes
  (vs uncompressed: 40 IPv6 + 8 UDP + 10 CoAP = 58 bytes)
```

**Fragmentation** (for payloads >LPWAN MTU):
```
SCHC Fragmentation Header:
  [Rule ID][DTag][W][FCN][Payload]

  DTag: Datagram Tag (identifies fragment series)
  W:    Window (for ACK)
  FCN:  Fragment Counter (0 = All-1, last fragment)

Example (LoRaWAN 51-byte payload):
  Fragment 0: [RuleID][DTag=0][W=0][FCN=3][48B payload]
  Fragment 1: [RuleID][DTag=0][W=0][FCN=2][48B payload]
  Fragment 2: [RuleID][DTag=0][W=0][FCN=1][48B payload]
  Fragment 3: [RuleID][DTag=0][W=0][FCN=0][20B payload + RCS checksum]
```

**Use Case**: LoRaWAN CoAP (see [lpwan.md](lpwan.md)), NB-IoT UDP compression.

---

## 10. IEEE 1451 TEDS (Transducer Electronic Data Sheet)

**Reference**: IEEE 1451.0-2007 (Common Functions), IEEE 1451.4 (Mixed-mode Communication)

**Purpose**: Self-describing sensor metadata (calibration, units, accuracy). Plug-and-play sensor identification.

**TEDS Structure**:
```
Meta-TEDS (Transducer Interface Module):
  ├─ Manufacturer ID
  ├─ Model Number
  ├─ Serial Number
  └─ Number of Channels

Channel TEDS (per sensor):
  ├─ Physical Units (SI units, UCUM)
  ├─ Measurement Range (min, max)
  ├─ Accuracy / Uncertainty
  ├─ Calibration Coefficients
  └─ Update Rate

Example (Temperature Sensor):
  Manufacturer: "Acme Sensors Inc."
  Model: "TMP-200"
  Channel 1:
    PhysicalUnits: Kelvin (UCUM code)
    LowerLimit: 233.15 K (-40°C)
    UpperLimit: 398.15 K (125°C)
    Accuracy: ±0.5 K
    CalibrationDate: 2023-06-15
    CalibrationCoeffs: [a0=0.001, a1=1.0, a2=0]
      (T_actual = a0 + a1*T_raw + a2*T_raw²)
```

**TEDS Encoding**:
```
Bitstream (TLV - Type-Length-Value):
  [Type: 1 byte][Length: 2 bytes][Value: variable]

Example:
  Type 10 (Manufacturer ID): 0x0A 0x0008 "Acme Inc"
  Type 13 (Physical Units):  0x0D 0x0001 0x01 (Kelvin)
```

**Use Case**: Industrial sensor networks (auto-configuration), aerospace (MIL-STD-1553 sensor buses).

---

## 11. IPSO Smart Objects / OMNA Registry

**Reference**: OMA LwM2M IPSO Smart Objects (OMNA Registry)

**Purpose**: Reusable resource IDs for common sensor/actuator types in LwM2M. Standardized object model (temperature, humidity, switch, etc.).

**Object Structure**:
```
Object 3303 (Temperature Sensor):
  Resource 5700 (Sensor Value):          Float, R, Mandatory
  Resource 5701 (Sensor Units):          String, R, Optional (e.g., "Cel")
  Resource 5601 (Min Measured Value):    Float, R, Optional
  Resource 5602 (Max Measured Value):    Float, R, Optional
  Resource 5603 (Min Range Value):       Float, R, Optional (-40°C)
  Resource 5604 (Max Range Value):       Float, R, Optional (125°C)
  Resource 5605 (Reset Min/Max Values):  Opaque, E, Optional

Object 3311 (Light Control):
  Resource 5850 (On/Off):                Boolean, RW, Mandatory
  Resource 5851 (Dimmer):                Integer (0-100), RW, Optional
  Resource 5706 (Colour):                String (RGB hex), RW, Optional
  Resource 5852 (On Time):               Integer (seconds), R, Optional
```

**LwM2M URI**:
```
/3303/0/5700           → Temperature Sensor instance 0, Sensor Value
/3311/0/5850           → Light Control instance 0, On/Off
/3200/0/5501           → Digital Input instance 0, Digital Input State
```

**Common IPSO Objects**:
```
3200 - Digital Input
3201 - Digital Output
3202 - Analog Input
3203 - Analog Output
3300 - Generic Sensor
3301 - Illuminance Sensor
3302 - Presence Sensor
3303 - Temperature Sensor
3304 - Humidity Sensor
3305 - Power Measurement
3306 - Actuation
3308 - Set Point
3310 - Load Control
3311 - Light Control
3312 - Power Control
3313 - Accelerometer
3314 - Magnetometer
3315 - Barometer
```

**Use Case**: LwM2M device profiles (see [lpwan.md](lpwan.md)), IoT sensor interoperability.

---

## Key Specification References

| Standard         | Title                                  | Publisher       | Year |
|------------------|----------------------------------------|-----------------|------|
| RFC 8949         | CBOR (Concise Binary Object Rep)      | IETF            | 2020 |
| RFC 8428         | SenML (Sensor Measurement Lists)       | IETF            | 2018 |
| Protobuf v3      | Protocol Buffers Language Guide        | Google          | 2023 |
| RFC 7950         | YANG 1.1 Data Modeling Language        | IETF            | 2016 |
| IEC 62541-3      | OPC UA Part 3: Address Space Model     | OPC Foundation  | 2020 |
| ETSI GS CIM 009  | NGSI-LD API Specification              | ETSI            | 2021 |
| Haystack 4.0     | Project Haystack 4.0 Specification     | Project Haystack| 2021 |
| RFC 8724         | SCHC (Static Context Header Compress)  | IETF            | 2020 |
| IEEE 1451.0      | Smart Transducer Interface Standards   | IEEE            | 2007 |
| IPSO Smart Obj   | OMNA LwM2M Object & Resource Registry  | OMA             | 2023 |

---

## Related Files
- **[application-messaging.md](application-messaging.md)**: CoAP Content-Formats (CBOR, SenML), MQTT JSON payloads
- **[device-management.md](device-management.md)**: YANG/NETCONF data models, gRPC protobuf
- **[industrial-ot.md](industrial-ot.md)**: OPC UA information model, BACnet object model
- **[lpwan.md](lpwan.md)**: LwM2M IPSO Smart Objects, CBOR encoding for CoAP
- **[edge-gateway.md](edge-gateway.md)**: Sparkplug B protobuf, EdgeX device profiles
