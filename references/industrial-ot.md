# Industrial/OT Protocols — Complete Reference

## Overview
Industrial IoT (IIoT) and Operational Technology (OT) protocols span factory automation, process control, building management, smart metering, and critical infrastructure. This reference covers the complete industrial protocol landscape from legacy fieldbus through modern Industrial Ethernet and cybersecurity frameworks.

---

## Modbus
### Overview
The most widely deployed industrial protocol. Originally developed by Modicon (now Schneider Electric) in 1979. Simple master-slave (client-server) architecture.

**Variants:**

| Variant | Transport | Use Case |
|---------|-----------|----------|
| Modbus RTU | RS-232/RS-485 (serial, binary) | Field devices, PLCs |
| Modbus ASCII | RS-232/RS-485 (serial, ASCII) | Legacy, debugging |
| Modbus TCP | TCP/IP (port 502) | Ethernet, SCADA integration |
| Modbus UDP | UDP | Time-critical applications |

### Function Codes (key)

| Code | Name | Description |
|------|------|-------------|
| 0x01 | Read Coils | Read discrete output (1-bit) |
| 0x02 | Read Discrete Inputs | Read discrete input (1-bit) |
| 0x03 | Read Holding Registers | Read 16-bit output registers |
| 0x04 | Read Input Registers | Read 16-bit input registers |
| 0x05 | Write Single Coil | Write one coil |
| 0x06 | Write Single Register | Write one holding register |
| 0x0F | Write Multiple Coils | Write multiple coils |
| 0x10 | Write Multiple Registers | Write multiple holding registers |

### Modbus TCP Frame
```
| MBAP Header (7 bytes) | PDU (max 253 bytes) |
| TxID(2) | Proto(2) | Len(2) | UnitID(1) | FC(1) | Data... |
Proto=0x0000 (Modbus), Len=remaining bytes
```

### Limitations
- No authentication; no encryption (use VPN/TLS tunnel at network layer)
- Master-slave only; no publish-subscribe
- 16-bit registers; limited data types
- No standard exception/alarm mechanism

### Modbus in IoT Gateway Context
Modbus-to-LwM2M, Modbus-to-MQTT, Modbus-to-OPC-UA bridges are standard in industrial IoT gateways (AWS IoT SiteWise, Azure IoT Edge OPC Publisher both support Modbus TCP)

---

## OPC UA (IEC 62541)
### Overview
**OPC Unified Architecture** — the modern industrial data exchange standard. Combines security, information modeling, service-oriented architecture, and transport independence. Successor to OPC Classic (DA/HDA/A&E).

### Architecture Layers
```
Application Layer: Information Model (NodeIDs, Attributes, References)
Service Layer: Services (Read/Write/Browse/Subscribe/Call/Method)
Security Layer: Security Channel (TLS, certificates, signing/encryption)
Transport Layer: Binary TCP (port 4840), HTTPS (443), WebSocket
```

### Information Model
- **NodeID:** Unique identifier (Numeric, String, GUID, ByteString)
- **Attribute:** Property of a Node (NodeId, BrowseName, DisplayName, Value, DataType, AccessLevel...)
- **Reference:** Typed relationship between Nodes (HierarchicalReference, NonHierarchicalReference)
- **Node classes:** Object, Variable, Method, ObjectType, VariableType, ReferenceType, DataType, View

### Security
OPC UA Security Policy options:

| Policy | Signing | Encryption |
|--------|---------|------------|
| None | No | No |
| Basic128Rsa15 | Deprecated | Deprecated |
| Basic256 | Deprecated | Deprecated |
| Basic256Sha256 | RSA-SHA256 | RSA-OAEP/AES-256-CBC |
| Aes128_Sha256_RsaOaep | RSA-SHA256 | RSA-OAEP/AES-128-CBC |
| Aes256_Sha256_RsaPss | RSA-SHA256-PSS | RSA-OAEP-SHA256/AES-256-CBC |

Mutual authentication via X.509 certificates. Application Instance Certificates (AIC) per application instance.

### OPC UA PubSub
Decoupled publish-subscribe over MQTT, AMQP, or UDP multicast:
- **Dataset:** Named collection of field values (FieldMetaData)
- **DataSetWriter → WriterGroup → PubSubConnection:** Publisher hierarchy
- **DataSetReader → ReaderGroup → PubSubConnection:** Subscriber hierarchy
- **Encoding:** JSON, UADP (binary, for TSN/multicast)
- **Security:** SecurityKeyService (SKS) for symmetric key distribution; DTLS/TLS for MQTT/AMQP transport

### OPC UA FX (Field eXchange — IEC 62541-80)
Field-level deterministic OPC UA over TSN:
- **Target:** Machine-to-machine (M2M) at field level; replaces proprietary fieldbus for motion control
- **Transport:** OPC UA UADP over IEEE 802.1Qbv TSN (scheduled traffic)
- **Latency:** Sub-millisecond deterministic; replacing EtherCAT/PROFINET RT in new installations
- **Configuration:** AutoSAR-based; IEEE 802.1Qcc centralized configuration
- **Adoption:** Siemens, Bosch Rexroth, Phoenix Contact early implementations

### OPC UA Companion Specifications (Vertical Information Models)

**Purpose**: Industry-specific information models extending the OPC UA base specification. Companion Specs define standardized semantic data models for equipment, processes, and data exchange within vertical domains — enabling plug-and-play interoperability between vendors.

**Why Companion Specs Matter**: Without them, every PLC/robot/machine vendor would define proprietary NodeIDs and methods. Companion Specs provide vendor-neutral ontologies — critical for brownfield/greenfield integration and avoiding proprietary lock-in.

#### Core Companion Specifications

**OPC UA for Machinery (OPC 40001-1)** — Machinery Information Model Foundation:
```
Purpose: Base types for all production machinery (CNC, presses, robots, packaging)
Key Object Types:
  ├─ MachineryItemType (base class for all machinery)
  │  ├─ MachineryOperationModeStateMachineType (Auto, Manual, Maintenance, Service)
  │  ├─ MachineryItemIdentificationType (serial number, asset ID, manufacturer)
  │  └─ LifetimeCounterType (operating hours, cycles, production count)
  │
  ├─ IMachineryVendorNameplateType (manufacturer info, CE marking)
  ├─ IMachineryItemStateType (power state, emergency stop, fault)
  └─ MachineryComponentIdentificationType (component within machine)

Standard Conditions (Alarms/Events):
  ├─ OffSpecConditionType (product quality out of spec)
  ├─ ProcessValueOutOfRangeType (temperature, pressure exceeded)
  └─ MaintenanceRequiredConditionType (predictive maintenance trigger)

Use Case: Machine builder exposes temperature sensor as:
  ns=2;s=Machine1/Spindle/Temperature (AnalogItemType, EURange 0-100°C)
  MES system knows to subscribe to this NodeID because it follows 40001-1 ontology
```

**OPC UA for PLCopen (OPC 40010)** — Motion Control:
```
Purpose: Standardized PLC motion control function blocks (IEC 61131-3 alignment)
Key Types:
  ├─ AxisType (single axis control: position, velocity, torque)
  │  ├─ PowerOn() / PowerOff() / Reset() methods
  │  ├─ ActualPosition, ActualVelocity (read-only variables)
  │  └─ MoveAbsolute(Position, Velocity, Acceleration) method
  │
  ├─ AxisStatusType (Enabled, Homed, Error, InMotion, Synchronized)
  ├─ AxisGroupType (multi-axis coordination: gantry, delta robot)
  └─ CamType (electronic cam profiles for packaging machines)

Standard Methods:
  - MC_MoveAbsolute, MC_MoveRelative, MC_MoveVelocity
  - MC_Home (axis homing sequence)
  - MC_Stop (deceleration ramp stop)
  - MC_Halt (immediate hold)

Real-World Flow:
  MES → OPC UA Server: Call MC_MoveAbsolute(Position=100mm, Velocity=50mm/s)
  PLC executes motion → updates ActualPosition variable
  SCADA subscribes to ActualPosition → real-time position monitoring

Deployment: Every major PLC vendor (Siemens TIA, Beckhoff TwinCAT, Allen-Bradley Studio 5000, Omron Sysmac) maps IEC 61131 motion FBs to OPC 40010 NodeIDs
```

**OPC UA for PackML (OPC 40082/40083)** — Packaging Machine State Model:
```
Purpose: ISA-TR88.00.02 PackML state machine exposed via OPC UA
PackML State Machine:
  ┌─────────────────────────────────────────────────┐
  │           STOPPED                               │
  │  ┌────────────────────────────────────┐         │
  │  │ IDLE → STARTING → EXECUTE →        │         │
  │  │  COMPLETING → COMPLETE → RESETTING │         │
  │  └────────────────────────────────────┘         │
  │           ↕ HOLDING ↔ HELD                      │
  │           ↕ SUSPENDING ↔ SUSPENDED              │
  │           ↕ ABORTING → ABORTED                  │
  │           ↕ STOPPING                            │
  └─────────────────────────────────────────────────┘

OPC UA Mapping:
  PackMLStateMachineType (base type from 40082)
    ├─ CurrentState: EXECUTE, HELD, ABORTED, etc.
    ├─ UnitMode: PRODUCTION, MAINTENANCE, MANUAL, DRY_RUN
    ├─ UnitModeTransitions[] (allowed mode changes)
    └─ Methods: Start(), Stop(), Reset(), Abort(), Clear()

PackML Tags (40083 — Tag Model):
  ├─ Admin/ProdProcessedCount (products packaged)
  ├─ Admin/ProdDefectiveCount (rejects)
  ├─ Status/CurMachSpeed (current line speed)
  └─ Command/MachSpeed (target speed setpoint)

Use Case: OEE (Overall Equipment Effectiveness) calculation
  SCADA reads: ProdProcessedCount, ProdDefectiveCount, state transition timestamps
  → Computes Availability, Performance, Quality
  → PackML standardization = OEE works across ALL packaging lines (no custom scripting)

Adoption: Food/beverage, pharmaceutical, consumer goods packaging lines (Tetra Pak, Bosch Packaging, Multivac all implement 40082)
```

**OPC UA for ISA-95 (OPC 40201)** — Manufacturing Operations Management:
```
Purpose: MES ↔ ERP integration using ISA-95 Part 2 object model
Key Object Types:
  ├─ ProductionOrderType (work order from ERP)
  │  ├─ OrderID, ProductionRequest, SegmentRequirements
  │  ├─ Start()/Pause()/Resume()/Cancel() methods
  │  └─ OrderState: WAITING, RUNNING, COMPLETED, ABORTED
  │
  ├─ ProductSegmentType (recipe/BOM for product)
  │  ├─ ProcessSegments[] (manufacturing steps)
  │  └─ MaterialRequirements (raw materials, quantities)
  │
  ├─ ProductionPerformanceType (actual vs planned)
  │  ├─ StartTime, EndTime, QuantityProduced
  │  └─ MaterialConsumed, EnergyConsumed
  │
  └─ JobOrderType (scheduled production job)
     └─ PlannedStart, PlannedEnd, Priority

Information Flow:
  ERP → OPC UA Server (MES): CREATE ProductionOrder (1000 units, Product_ABC)
  MES → PLC: Start production (via PackML or Machinery CS)
  PLC → MES: Report progress (ActualQuantity, CurrentState)
  MES → ERP: ProductionPerformance (1000 units complete, 50 kWh consumed)

Integration Pattern:
  SAP ME/MII → OPC UA 40201 Server (MES middleware)
  → Reads ProductionOrders, writes ProductionPerformance
  Replaces: Proprietary MES APIs, manual data entry, file-based integration

Benefit: Multi-vendor MES/ERP can consume same OPC UA model (SAP, Aveva, Rockwell FactoryTalk all support 40201)
```

**OPC UA for Robotics (OPC 40010-1)** — Robot Kinematics and Control:
```
Purpose: Standardized robot control interface (position, motion, safety, I/O)
Key Object Types:
  ├─ RobotType (base robot representation)
  │  ├─ MotionDeviceType (individual axis, gripper, turntable)
  │  │  ├─ AxisType[] (J1-J6 for 6-axis arm)
  │  │  │  └─ ActualPosition, ActualVelocity, ActualTorque
  │  │  ├─ TCP (Tool Center Point) Cartesian position
  │  │  └─ MoveL(Target, Speed) / MoveJ(JointAngles, Speed) methods
  │  │
  │  ├─ SafetyStateType (SafetyStop, ReducedSpeed, Normal)
  │  ├─ ControllerType (robot controller state, program loaded)
  │  └─ TaskControlType (program execution: Start, Pause, Stop)
  │
  ├─ GripperType (pneumatic, electric, vacuum gripper)
  │  └─ Open(), Close(), Grip(Force) methods
  │
  └─ ForceControlType (force/torque sensor data)
     └─ Fx, Fy, Fz, Tx, Ty, Tz (6-axis force readings)

Robot Program Execution:
  OPC UA Client → LoadProgram(ProgramID)
  Client → TaskControl.Start()
  Robot → Updates MotionDevice.Axes[].ActualPosition (real-time)
  Client subscribes → position feedback at 100 Hz

Safety Integration (ISO 10218, ISO/TS 15066):
  SafetyState transitions:
    - Normal → EmergencyStop (safety PLC triggers)
    - Normal → ReducedSpeed (human enters collaborative zone)
  OPC UA exposes safety state → SCADA logs safety events

Deployment: ABB, KUKA, FANUC, Universal Robots (UR) all expose 40010-1 server
  - Enables robot fleet management (heterogeneous brands, single SCADA)
  - Plug-and-play robot tool changers (standardized gripper interface)
```

**OPC UA for AutoID (OPC 40081)** — RFID / Barcode Scanners:
```
Purpose: Automatic identification devices (RFID readers, barcode scanners) as OPC UA servers
Key Object Types:
  ├─ RFIDReaderType
  │  ├─ Scan() method → returns TagID[]
  │  ├─ LastScanResult (timestamp, tag list)
  │  └─ AntennaType[] (multi-antenna RFID reader)
  │
  ├─ BarcodeReaderType
  │  ├─ Symbology (QR, DataMatrix, Code128, EAN-13)
  │  └─ Scan() → returns decoded barcode string
  │
  └─ AutoIDDeviceConfigurationType
     └─ ScanMode: CONTINUOUS, TRIGGERED, MANUAL

Integration with MES:
  Product enters assembly cell → RFID gate reads tag → OPC UA Scan() event
  → MES receives TagID (serial number) → Looks up work order → Configures machine
  → Eliminates manual barcode scanning; automated product tracking

Use Case: Automotive assembly (VIN tracking via RFID), pharmaceutical serialization (EU FMD compliance), baggage handling (airport RFID)
```

**OPC UA for IoT (OPC 40451)** — Constrained Devices:
```
Purpose: OPC UA profile for resource-constrained IoT devices (sensors, actuators)
Key Features:
  - Reduced footprint OPC UA stack (<128 KB RAM, <256 KB ROM)
  - Binary protocol over CoAP (not TCP)
  - Simplified security (PSK instead of certificates)
  - PubSub-only mode (no client-server sessions)

Use Case: Wireless temperature sensor (battery-powered)
  - Publishes temperature readings to MQTT broker (OPC UA PubSub JSON encoding)
  - Gateway translates to full OPC UA server for SCADA consumption
  - Sensor never establishes TCP session (saves power)
```

#### Additional Domain-Specific Companion Specs

| Spec | Number | Domain | Key Use Case |
|------|--------|--------|--------------|
| OPC UA for Weihenstephan | OPC UA WS | Food/beverage | Brewery/dairy machine data (OEE, CIP cleaning, batch tracking) |
| OPC UA for Plastics | OPC 40077-1 | Plastics machinery | Injection molding machines (cycle time, cavity pressure, melt temp) |
| OPC UA for Machine Vision | OPC 40100-1 | Vision systems | Camera configuration, image acquisition, defect detection results |
| OPC UA for Weighing | OPC 30270 | Industrial scales | Checkweighers, bulk scales (tare, gross, net weight with legal metrology) |
| OPC UA for Glass | OPC 40050 | Glass manufacturing | Float glass, container glass (furnace temp, forming pressure) |
| OPC UA for CNC Systems | OPC 40501 | Machine tools | CNC milling, turning (spindle load, tool life, G-code program status) |
| OPC UA for Mining | OPC 40500 | Mining equipment | Haul trucks, loaders, drilling rigs (load, fuel, GPS, fleet management) |
| OPC UA for FDT/DTM | OPC 40001-100 | Field device tools | Electronic Device Description (EDD) for Hart, Profibus, Foundation Fieldbus |

#### Companion Spec Development Process

```
1. Joint Working Group (JWG) formed:
   - OPC Foundation + Industry consortium (e.g., VDMA for Machinery)
   - Members: Equipment vendors, end users, system integrators

2. Information Model Design:
   - Define ObjectTypes, VariableTypes, Methods, Events
   - UML diagrams → NodeSet2 XML export

3. Reference Implementation:
   - Open-source server (C#, C++, Java)
   - Compliance test tool (OPC Foundation CTT)

4. Certification:
   - Vendors implement Companion Spec
   - OPC Foundation compliance testing
   - Certified products listed on OPC website

5. Maintenance:
   - New releases (e.g., PackML v2 adds digital twin support)
   - Backward compatibility maintained (old clients work with new servers)
```

**Benefits of Using Companion Specs**:
- **Plug-and-Play**: Replace machine → new machine exposes same NodeIDs → SCADA works without reconfiguration
- **Vendor Neutrality**: Multi-vendor lines use single OPC UA client (no vendor-specific drivers)
- **Future-Proof**: Standards evolve (new features added via versioned releases) but remain backward-compatible
- **Brownfield Integration**: Legacy machines with proprietary protocols get OPC UA gateways implementing Companion Spec → Unified data model

**Deployment Guidance**:
```
Selecting Companion Spec:
  1. Identify machine type: Packaging → PackML, Robot → Robotics, CNC → CNC Systems
  2. Check vendor support: Does PLC/machine vendor implement the spec? (check OPC Foundation certified products)
  3. If no spec exists: Use base OPC UA (custom NodeIDs) or petition OPC Foundation for new JWG

Integration Pattern:
  Legacy Machine (Modbus) → OPC UA Gateway (implements Companion Spec) → MES/SCADA
  - Gateway maps Modbus registers to Companion Spec NodeIDs
  - SCADA sees standards-compliant OPC UA server (vendor-agnostic)
```

Cross-reference: [tsn-deterministic.md](tsn-deterministic.md) for OPC UA FX over TSN; [edge-gateway.md](edge-gateway.md) for OPC UA gateway patterns

---

## DNP3 (IEEE 1815)
Distributed Network Protocol — power systems and utilities SCADA:
- **Designed for:** Substation automation, RTU communication, water/wastewater SCADA
- **Transport:** Serial (RS-232/RS-485), TCP (port 20000), UDP
- **Data objects:** Binary inputs/outputs, analog inputs/outputs, counters, time-stamped data
- **Time sync:** DNP3 time-stamping critical for sequence-of-events recording
- **Integrity polls:** Periodic data polls vs event-based reporting (Class 0/1/2/3 polls)
- **Secure Authentication v5 (SA v5):** HMAC-based authentication added to DNP3; SHA-256; prevents man-in-the-middle; **should always be enabled for critical infrastructure**
- **Deployment:** NERC CIP compliance for bulk electric system; AWIA 2018 for water utilities

---

## IEC 61850
International standard for power utility communication (substations + beyond):
- **GOOSE (Generic Object-Oriented Substation Events):** UDP multicast; fast tripping signals (<4ms); used for protection relay coordination
- **SV (Sampled Values):** High-speed digitized current/voltage samples from merging units
- **MMS (Manufacturing Message Specification):** TCP-based client-server for control and monitoring
- **Logical Nodes:** Standardised functional blocks (XCBR=Circuit Breaker, MMXU=Measurement, PDIF=Differential protection)
- **Edition 2:** Adds cybersecurity (IEC 62351 security), extended data modelling
- **XMPP Extension:** IEC 61850 over XMPP for WAN communication
- **Expanding scope:** IEC 61850-7-420 (distributed energy resources), 61850-90-7 (object models for EV charging)

---

## DLMS/COSEM (IEC 62056 / EN 13757)
**Device Language Message Specification / Companion Specification for Energy Metering**
The primary global protocol for smart electricity, gas, water, and heat meters.

### Architecture
```
Application Layer: DLMS (object model + services)
Presentation: COSEM (object encoding, OBIS addressing)
Data Link: HDLC (serial), TCP/IP, wM-Bus (wireless), G3-PLC/PRIME (PLC)
Physical: RS-485, Ethernet, NB-IoT/LTE-M, wM-Bus RF, PLC
```

### COSEM Object Model
COSEM (Companion Specification for Energy Metering) defines standardised meter objects:
- **OBIS Code:** Object Identification System — `A-B:C.D.E.F` format
  - A: Energy medium (1=electricity, 7=gas, 8=water, 6=heat)
  - B: Channel (0=total)
  - C: Physical quantity (1=active energy, 2=reactive energy, 3=apparent)
  - D: Measurement type (8=time integral, 11=cumulative maximum)
  - E: Tariff (0=total, 1=tariff 1, 2=tariff 2)
  - F: Storage number
Example: `1-0:1.8.0` = Channel 0, Active energy, Time integral, Total (electricity import kWh)

### DLMS Services
- **GET:** Read object attribute value
- **SET:** Write object attribute value
- **ACTION:** Execute object method
- **Notification:** Unsolicited data push (Data-Push Class ID 65)
- **Profile Generic (Class ID 7):** Time-series buffer for logged data (load profiles, event logs)

### Transport Options

| Transport | Use Case |
|-----------|----------|
| HDLC (IEC 62056-46) | Direct serial meter connection; walk-by/optical |
| TCP/IP (IEC 62056-47) | AMI head-end to meter; cellular backhaul |
| SMS/GPRS | Legacy AMI; NB-IoT/LTE-M increasingly used |
| wM-Bus (EN 13757-4) | Wireless AMI for gas/water/heat; OMS profile |
| G3-PLC / PRIME | Power line communication; electricity grid AMI |

### Wireless M-Bus / OMS and DLMS Relationship
- **wM-Bus** = RF transport (physical + data link)
- **DLMS/COSEM** = Application layer (object model, services)
- They are complementary: wM-Bus carries DLMS data records, or OMS defines its own simplified records
- For electricity meters: typically DLMS/COSEM over cellular (NB-IoT/LTE-M)
- For water/gas/heat meters: often OMS data records over wM-Bus without full DLMS stack

---

## BACnet (ANSI/ASHRAE 135)
Building Automation and Control Network — the dominant building management standard:
- **Objects:** Analog Input/Output/Value, Binary Input/Output/Value, Multi-state, Schedule, Calendar, Program, Event Log, Trend Log
- **Services:** ReadProperty/Multiple, WriteProperty, SubscribeCOV, ReadRange, TimeSynchronization
- **Transports:**
  - **BACnet/IP:** UDP port 47808 (default); most common; supports routing via BBMD (BACnet/IP Broadcast Management Device)
  - **BACnet MS/TP:** RS-485 serial; token-passing bus; low cost for field devices
  - **BACnet/SC (Annex AB):** WebSocket-based; TLS 1.3; certificates; modern cloud connectivity
- **Interoperability:** PICS (Protocol Implementation Conformance Statement) declares object/service support

---

## KNX (EN 50090)
European building automation standard for home and commercial buildings:
- **KNX TP (Twisted Pair):** 2-wire bus; 9600 baud; bus-powered devices; DIN rail installation
- **KNX RF:** 868 MHz wireless; bidirectional; battery devices
- **KNX IP:** Ethernet backbone; KNXnet/IP tunnelling and routing
- **KNX IoT (IETF standardised):** CoAP + OSCORE over IPv6; bridges KNX to cloud/IP networks
- **Data model:** Group Objects (GOs) mapped to Group Addresses; ETS (Engineering Tool Software) for configuration
- **Certification:** KNX Association certification required; ~500 certified manufacturers

---

## IEC 62443 (Industrial Cybersecurity)
The reference cybersecurity framework for Industrial Automation and Control Systems (IACS):

### Structure
- **IEC 62443-1:** General concepts, models, terminology
- **IEC 62443-2:** Policies and procedures (CSMS — Cybersecurity Management System)
- **IEC 62443-3:** System requirements (zones and conduits, SL levels)
- **IEC 62443-4:** Component requirements (SL levels for products)

### Security Levels (SL)

| SL | Protection Against |
|----|-------------------|
| SL 1 | Casual/unintentional violation |
| SL 2 | Intentional violation with simple means |
| SL 3 | Sophisticated means, IACS-specific skills |
| SL 4 | State-sponsored, advanced persistent threats |

### Zones and Conduits
- **Zone:** Logical grouping of assets with same security requirements
- **Conduit:** Communication path between zones; must be controlled/filtered
- **Zone SL:** Highest SL of assets within the zone determines zone SL target
- **Conduit SL:** Must match or exceed the SL of connected zones

### Mandatory for: Critical infrastructure procurement in EU (NIS2 Directive), US (NERC CIP, ICS-CERT guidelines), nuclear, water, energy sectors

---

## Additional Protocols
### PROFINET (IEC 61158-6-10)
Industrial Ethernet for factory automation:
- **RT (Real-Time):** 1ms cycle time; priority frames; no switch modifications
- **IRT (Isochronous Real-Time):** 31.25μs cycle time; requires PROFINET-capable switches with hardware timestamping
- **IO-Controller:** PLC/master; IO-Device: field device/slave
- **GSDML:** XML device description (like EDS for DeviceNet)

### EtherCAT (IEC 61158-12)
Ethernet-based fieldbus with distributed clock:
- **Processing on-the-fly:** Each slave reads/writes its data as Ethernet frame passes through
- **Cycle time:** <100μs; deterministic; suitable for motion control
- **Topology:** Line, ring, star (any Ethernet topology)
- **EtherCAT P:** Combined data + 24V power on single cable

### IO-Link (IEC 61131-9)
Point-to-point sensor/actuator interface:
- **Physical:** 3-wire (24V, GND, data); max 20m; COM1/COM2/COM3 (4.8/38.4/230.4 kbps)
- **IO-Link Master:** Connects up to 8 IO-Link ports; integrated in PLC or field device
- **IO-Link Wireless (IEC 61131-9-1):** 2.4 GHz; up to 5 Mbit/s; 5m range
- **IODD (IO Device Description):** XML device description; parameter access; diagnostics

### POWERLINK (IEC 61158-6-13)
Open-source Industrial Ethernet:
- **Standard Ethernet hardware** (no special ASICs required)
- **Polling master:** Distributed Real-time (MN/CN model)
- **Cycle time:** 200μs; 100Mbps
- **Implementation:** openPOWERLINK (open source); used in Kuka robots, B&R Automation

### WirelessHART (IEC 62591)
Wireless extension of HART for process instrumentation:
- **Frequency:** 2.4 GHz IEEE 802.15.4; DSSS; channel hopping
- **TDMA:** Time-synchronized mesh; deterministic access
- **Self-organizing mesh:** Graph routing for reliability
- **Security:** AES-128; per-network and per-join keys