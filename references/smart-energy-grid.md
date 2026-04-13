# Smart Energy & Grid

**Scope**: Smart grid communication protocols, demand response, distributed energy resources (DER), advanced metering infrastructure (AMI), electric vehicle charging, and power line communication. Covers IEEE 2030.5, OpenADR, IEC 61968/61970 CIM, ANSI C12 metering, G3-PLC, SMETS2, SPINE/EEBUS, and EU Battery Passport. Cross-references [automotive-transport.md](automotive-transport.md) for ISO 15118/OCPP EV charging, [industrial-ot.md](industrial-ot.md) for IEC 61850 substation automation, and [lpwan.md](lpwan.md) for DLMS/COSEM metering.

## Table of Contents
1. IEEE 2030.5 / SEP 2.0 (Smart Energy Profile)
2. OpenADR 2.0a/2.0b (Automated Demand Response)
3. IEC 61968 / 61970 CIM (Common Information Model)
4. ANSI C12.18/C12.19/C12.22 (North American AMI)
5. G3-PLC (Power Line Communication)
6. SMETS2 / GBCS (UK Smart Metering)
7. SPINE / EEBUS (Home Energy Management)
8. IEC 61851 (EV Conductive Charging)
9. EU Battery Passport (EU 2023/1542)

---

## 1. IEEE 2030.5 / SEP 2.0 (Smart Energy Profile)

**Reference**: IEEE 2030.5-2018 (SEP 2.0), California Rule 21 mandate

**Purpose**: IP-based smart energy communication for DER (Distributed Energy Resources), demand response, pricing, EVSE (EV Supply Equipment), and metering. Mandated by California utilities (Rule 21) for grid-connected solar/storage/EV.

**Architecture**:
```
┌────────────────────────────────────────────────────┐
│ Utility / Aggregator (Server)                     │
│   ├─ DER Programs (volt-watt, freq-watt)          │
│   ├─ Pricing (TOU, RTP)                           │
│   ├─ Demand Response Events                       │
│   └─ Metering / Billing                           │
└──────────────────┬─────────────────────────────────┘
                   │ HTTPS / TLS 1.2+ (REST-like)
┌──────────────────▼─────────────────────────────────┐
│ Client (DER Gateway, Smart Inverter, EVSE)         │
│   ├─ DER Capabilities (max export/import kW)       │
│   ├─ DER Status (active power, reactive power)     │
│   ├─ Pricing Response (reduce load during high $)  │
│   └─ Metering Data (kWh consumption)               │
└────────────────────────────────────────────────────┘
```

**Resource Model** (RESTful hierarchy):
```
/dcap (Device Capability)
  ├─ /edev (End Device List)
  │   └─ /edev/0 (Specific End Device)
  │       ├─ /der (DER)
  │       │   ├─ /dercap (DER Capability)
  │       │   ├─ /derp (DER Program)
  │       │   └─ /ders (DER Status)
  │       ├─ /fsa (Function Set Assignments)
  │       ├─ /tm (Time - clock sync)
  │       ├─ /mr (Metering)
  │       │   ├─ /mup (Metering Usage Point)
  │       │   └─ /mr/0/r (Reading List)
  │       └─ /edev/0/dr (Demand Response)
  │           └─ /drlc (DR Load Control)
  ├─ /tm (Server Time)
  ├─ /mup (Metering Usage Point List)
  ├─ /dr (Demand Response)
  │   └─ /drlc (DR Load Control)
  └─ /tariff (Pricing / Rate Tariff)
      └─ /tou (Time of Use)
```

**DER Control Example** (Volt-Watt Curve):
```xml
GET /dcap/edev/0/derp/0

<DERProgram>
  <primacy>100</primacy>  <!-- Priority (higher = more important) -->
  <DERControlList>
    <DERControl>
      <interval>
        <start>1638360000</start>  <!-- Unix epoch -->
        <duration>3600</duration>  <!-- 1 hour -->
      </interval>
      <DERCurveList>
        <DERCurve>
          <curveType>0</curveType>  <!-- 0=opModVoltVar, 1=opModFreqWatt -->
          <CurveData>
            <xvalue>23000</xvalue>  <!-- 230V × 100 (voltage) -->
            <yvalue>0</yvalue>      <!-- 0% reactive power -->
          </CurveData>
          <CurveData>
            <xvalue>24000</xvalue>  <!-- 240V × 100 -->
            <yvalue>-4400</yvalue>  <!-- -44% Var (absorb reactive) -->
          </CurveData>
        </DERCurve>
      </DERCurveList>
    </DERControl>
  </DERControlList>
</DERProgram>
```

**Demand Response Load Control**:
```xml
POST /dr/drlc

<DemandResponseLoadControl>
  <interval>
    <start>1638360000</start>
    <duration>7200</duration>  <!-- 2 hours -->
  </interval>
  <eventStatus>1</eventStatus>  <!-- Active -->
  <creationTime>1638359900</creationTime>
  <LoadSheddingProgram>
    <primacy>50</primacy>
    <deviceCategory>0x02</deviceCategory>  <!-- HVAC -->
    <appliedTargetReduction>
      <value>5000</value>  <!-- 50.00% reduction (×100 for precision) -->
      <multiplier>-2</multiplier>
    </appliedTargetReduction>
  </LoadSheddingProgram>
</DemandResponseLoadControl>
```

**Security**:
```
TLS 1.2+ with mutual authentication (X.509 client certs)
IEEE 1547 mandate: Certificates must be from trusted CA
Certificate Expiry: Devices auto-renew via SCEP or EST (RFC 7030)
```

**Use Case**: California solar installations (Rule 21 compliance), Hawaii grid stability (high PV penetration), EVSE demand response.

---

## 2. OpenADR 2.0a/2.0b (Automated Demand Response)

**Reference**: IEC 62746-10-3 (OpenADR 2.0b Profile), OpenADR Alliance

**Purpose**: Automated demand response signaling between utilities (VTN - Virtual Top Node) and customers (VEN - Virtual End Node). Reduce peak load via price signals or direct load control.

**Architecture**:
```
VTN (Utility / ISO)              VEN (Customer Gateway)
  ├─ Event creation                ├─ Receives DR events
  ├─ Pricing signals               ├─ Sheds load (HVAC, lighting)
  └─ Report requests               └─ Sends telemetry (kW reduced)
       ↓ XMPP / HTTPS
  EiEvent (Event)
  EiReport (Telemetry)
  EiRegisterParty (Enrollment)
```

**Transport Bindings**:
```
Pull Model (Simple HTTP):
  ├─ VEN polls VTN every 5 minutes (oadrPoll)
  └─ VTN responds with pending events

Push Model (HTTP Long-Poll or XMPP):
  ├─ VEN maintains persistent connection
  └─ VTN pushes events immediately (low latency)
```

**Event Flow** (DR Event):
```xml
<!-- VTN sends oadrDistributeEvent -->
<oadrDistributeEvent>
  <oadrEvent>
    <eiEvent>
      <eventDescriptor>
        <eventID>evt-001</eventID>
        <eventStatus>active</eventStatus>
        <priority>1</priority>  <!-- 0=highest -->
      </eventDescriptor>
      <eiActivePeriod>
        <dtstart>2023-12-01T14:00:00Z</dtstart>
        <duration>PT2H</duration>  <!-- 2 hours -->
      </eiActivePeriod>
      <eiEventSignals>
        <eiEventSignal>
          <signalName>LOAD_DISPATCH</signalName>
          <signalType>level</signalType>
          <intervals>
            <interval>
              <dtstart>2023-12-01T14:00:00Z</dtstart>
              <duration>PT1H</duration>
              <signalPayload>
                <payloadFloat>
                  <value>2.5</value>  <!-- Shed 2.5 MW -->
                </payloadFloat>
              </signalPayload>
            </interval>
          </intervals>
        </eiEventSignal>
      </eiEventSignals>
    </eiEvent>
  </oadrEvent>
</oadrDistributeEvent>

<!-- VEN responds with oadrCreatedEvent (opt-in/out) -->
<oadrCreatedEvent>
  <eiResponse>
    <responseCode>200</responseCode>  <!-- OK -->
  </eiResponse>
  <eventResponses>
    <eventResponse>
      <responseCode>200</responseCode>
      <eventID>evt-001</eventID>
      <optType>optIn</optType>  <!-- Accept event -->
    </eventResponse>
  </eventResponses>
</oadrCreatedEvent>
```

**Reporting** (Telemetry):
```xml
<!-- VTN requests report -->
<oadrCreateReport>
  <reportSpecifierID>rpt-power</reportSpecifierID>
  <reportRequestID>req-001</reportRequestID>
  <reportBackDuration>PT15M</reportBackDuration>  <!-- Report every 15 min -->
  <reportSpecifier>
    <reportInterval>
      <dtstart>2023-12-01T14:00:00Z</dtstart>
      <duration>PT2H</duration>
    </reportInterval>
    <specifierPayload>
      <rID>power</rID>  <!-- kW demand -->
    </specifierPayload>
  </reportSpecifier>
</oadrCreateReport>

<!-- VEN sends oadrUpdateReport -->
<oadrUpdateReport>
  <oadrReport>
    <reportRequestID>req-001</reportRequestID>
    <intervals>
      <interval>
        <dtstart>2023-12-01T14:00:00Z</dtstart>
        <duration>PT15M</duration>
        <payloadFloat>
          <value>125.3</value>  <!-- 125.3 kW -->
        </payloadFloat>
      </interval>
    </intervals>
  </oadrReport>
</oadrUpdateReport>
```

**Use Case**: California ISO (CAISO) demand response, commercial building HVAC control, industrial load shedding.

---

## 3. IEC 61968 / 61970 CIM (Common Information Model)

**Reference**: IEC 61968 (Distribution Management), IEC 61970 (Energy Management System), CIM16

**Purpose**: Semantic data model for power grid operations (SCADA, DMS, OMS, ADMS). Enables interoperability between vendor systems.

**CIM Class Hierarchy**:
```
IdentifiedObject (base class)
  ├─ PowerSystemResource
  │   ├─ Equipment
  │   │   ├─ ConductingEquipment
  │   │   │   ├─ Switch (Breaker, Disconnector, LoadBreakSwitch)
  │   │   │   ├─ EnergyConsumer (load)
  │   │   │   ├─ PowerTransformer
  │   │   │   └─ ACLineSegment (transmission line)
  │   │   └─ Sensor (CurrentTransformer, PotentialTransformer)
  │   └─ ConnectivityNode (bus, topological node)
  ├─ Measurement
  │   ├─ Analog (voltage, current, power)
  │   └─ Discrete (breaker status: open/closed)
  └─ Document
      ├─ WorkOrder (field crew task)
      └─ OutageReport (customer outage)
```

**CIM RDF/XML Example** (Breaker):
```xml
<cim:Breaker rdf:ID="BRK-001">
  <cim:IdentifiedObject.name>Substation A Feeder 1 Breaker</cim:IdentifiedObject.name>
  <cim:Equipment.EquipmentContainer rdf:resource="#Substation-A"/>
  <cim:Switch.normalOpen>false</cim:Switch.normalOpen>
  <cim:Switch.retained>true</cim:Switch.retained>
  <cim:ConductingEquipment.Terminals>
    <cim:Terminal rdf:ID="BRK-001-T1">
      <cim:Terminal.ConnectivityNode rdf:resource="#BusBar-1"/>
    </cim:Terminal>
  </cim:ConductingEquipment.Terminals>
</cim:Breaker>

<cim:Analog rdf:ID="BRK-001-Current">
  <cim:Measurement.PowerSystemResource rdf:resource="#BRK-001"/>
  <cim:Measurement.measurementType>Current</cim:Measurement.measurementType>
  <cim:Analog.positiveFlowIn>true</cim:Analog.positiveFlowIn>
</cim:Analog>
```

**CIM Profiles**:
```
IEC 61970-452 (CIM XML):       Static network model exchange
IEC 61968-13 (CIM RDF):        Distribution network model
IEC 61968-100 (CPSM):          Common Power System Model (generation, load, topology)
IEC 61850 Mapping:             CIM ↔ IEC 61850 SCL (substation interop)
```

**Use Case**: Grid modernization (DMS/OMS integration), outage management (OMS → SCADA), renewable integration (DER CIM models).

---

## 4. ANSI C12.18/C12.19/C12.22 (North American AMI)

**Reference**: ANSI C12.18 (Optical Port), C12.19 (Data Tables), C12.22 (Network Protocol)

**Purpose**: Advanced Metering Infrastructure (AMI) for North America. Optical port communication, standardized data tables, mesh networking.

**C12.18 Optical Port**:
```
Physical: IR LED (940nm) on meter faceplate
Speed: 9600 baud (default), up to 115200 baud
Protocol: Half-duplex, byte-oriented
Commands:
  ├─ IDENT:     Request meter identity
  ├─ READ:      Read data table (e.g., table 0 for config)
  ├─ WRITE:     Write configuration
  └─ LOGON/LOGOFF: Authenticate session (password-based)
```

**C12.19 Data Tables**:
```
Manufacturer Tables (0-2047):
  ├─ Table 0:   General Configuration (meter type, version)
  ├─ Table 1:   General Manufacturer ID
  └─ Table 2:   Device Nameplate (serial number, form factor)

Standard Tables (2048-4095):
  ├─ Table 11:  Actual Sources (current readings)
  ├─ Table 13:  Current Register Data (kWh, kVARh)
  ├─ Table 21:  Actual Register (real-time demand)
  ├─ Table 23:  Current Data (voltage, current, PF)
  ├─ Table 61:  Load Profile Status
  └─ Table 64:  Load Profile Data (15-min interval kWh)

Example Read (Table 13, kWh):
  Request: 0x30 (READ) 0x00 0x0D (table 13) 0x00 0x00 (offset) 0xFF 0xFF (length)
  Response: 0x00 (ACK) [data: total kWh, rate A kWh, rate B kWh] 0xXX (CRC)
```

**C12.22 Network Protocol**:
```
Transport: UDP/IP, TCP/IP
Addressing: ApTitle (Application Title, 20-byte unique ID per device)
Message Types:
  ├─ EPSEM (Extended Protocol Specification for Energy Metering)
  ├─ Segmented (for large tables, fragmented across multiple packets)
  └─ Security: AES-128-CCM encryption, HMAC-SHA256

Example (read table 13 over C12.22):
  <epsem-data>
    <c1222-message>
      <calling-aptitle>Collector-001</calling-aptitle>
      <called-aptitle>Meter-12345</called-aptitle>
      <epsem>0x30 0x00 0x0D 0x00 0x00 0xFF 0xFF</epsem>
    </c1222-message>
  </epsem-data>
```

**Use Case**: North American smart meters (Itron, Landis+Gyr, Aclara), AMI backhaul (RF mesh, cellular).

---

## 5. G3-PLC (Power Line Communication)

**Reference**: ITU-T G.9903 (G3-PLC), IEEE 1901.2

**Purpose**: Narrowband PLC for smart metering. Uses power lines as communication medium (no new wiring). Deployed in Linky meters (France), smart grids (Spain).

**PHY Layer**:
```
Frequency: CENELEC A band (35-91 kHz, Europe), FCC band (150-487 kHz, US)
Modulation: OFDM (Orthogonal Frequency Division Multiplexing)
  ├─ Subcarriers: 36 (CENELEC A), 72 (FCC)
  ├─ Modulation per carrier: BPSK, QPSK, 8PSK
  └─ Data rate: 3.4 kbps - 400 kbps

Tone Masking:
  ├─ Adaptive (avoid noisy frequencies)
  └─ Regulatory (avoid ham radio bands)
```

**MAC Layer**:
```
CSMA/CA (Carrier Sense Multiple Access with Collision Avoidance)
  ├─ Backoff: Exponential (similar to Wi-Fi)
  └─ Collision detection via ACK timeout

Mesh Routing (LOADng - Lightweight On-demand Ad hoc Distance-vector):
  ├─ RREQ (Route Request): Broadcast to find path to destination
  ├─ RREP (Route Reply): Destination responds with route
  └─ Metric: LQI (Link Quality Indicator) + hop count
```

**Network Layer**:
```
6LoWPAN (IPv6 over PLC):
  ├─ Header compression (IPHC)
  ├─ Fragmentation (PLC MTU ~400 bytes)
  └─ RPL routing (optional, LOADng more common)

Application: UDP/IPv6 + DLMS/COSEM (see [lpwan.md](lpwan.md) for DLMS)
```

**Security**:
```
EAP-PSK (Extensible Authentication Protocol - Pre-Shared Key):
  ├─ Meter → PAN Coordinator: EAP-PSK handshake
  ├─ AES-128-CCM encryption (network key)
  └─ Group Key Update: Periodic rekeying via coordinator
```

**Use Case**: French Linky meters (35 million deployed), Spanish smart grid, Italian Enel meters.

---

## 6. SMETS2 / GBCS (UK Smart Metering)

**Reference**: SMETS2 (Smart Metering Equipment Technical Specification v2), GBCS (Great Britain Companion Specification)

**Purpose**: UK national smart metering rollout (53 million meters by 2025). Dual-fuel (electricity + gas) with Home Area Network (HAN).

**Architecture**:
```
┌────────────────────────────────────────────────────┐
│ DCC (Data and Communications Company)              │
│   ├─ National hub (all meter data aggregation)    │
│   ├─ North Network (Arqiva, WAN: cellular)        │
│   ├─ Central Network (Telefonica, WAN: cellular)  │
│   └─ South Network (Arqiva, WAN: cellular)        │
└──────────────────┬─────────────────────────────────┘
                   │ DLMS/COSEM over cellular
┌──────────────────▼─────────────────────────────────┐
│ ESME (Electricity Smart Metering Equipment)        │
│   ├─ SMETS2 meter (Elster A1700, L+G E470)        │
│   ├─ WAN module (cellular: 2G/3G/LTE-M)           │
│   └─ HAN interface (Zigbee SEP 1.x)               │
└──────────────────┬─────────────────────────────────┘
                   │ Zigbee SEP 1.x (Home Area Network)
┌──────────────────▼─────────────────────────────────┐
│ HAN Devices (In-Home Display, GSME gas meter)      │
│   ├─ IHD (In-Home Display): Real-time usage       │
│   ├─ GSME (Gas Smart Metering Equipment)          │
│   ├─ CAD (Consumer Access Device): HACS interface │
│   └─ PPMID (Prepayment Interface Device)          │
└────────────────────────────────────────────────────┘
```

**SMETS2 Meter Data**:
```
DLMS/COSEM Objects (accessed via DCC):
  ├─ 1.0.1.8.0.255: Active energy import (kWh)
  ├─ 1.0.2.8.0.255: Active energy export (kWh, for solar)
  ├─ 1.0.14.7.0.255: Instantaneous frequency (Hz)
  ├─ 1.0.32.7.0.255: Instantaneous voltage (V)
  └─ 0.0.96.1.0.255: Meter serial number
```

**HAN (Zigbee SEP 1.x)**:
```
Zigbee Cluster:
  ├─ Simple Metering (0x0702): CurrentSummationDelivered (kWh)
  ├─ Price (0x0700): Current price tier (p/kWh)
  └─ Messaging (0x0703): Utility messages (e.g., "High demand period")

Commissioning:
  1. ESME creates Zigbee network (PAN ID, channel 11-26)
  2. IHD scans, finds ESME beacon
  3. IHD sends Join Request
  4. ESME authenticates (Install Code → preconfigured link key)
  5. IHD binds to Simple Metering cluster
  6. ESME sends kWh updates every 30s (Zigbee Attribute Report)
```

**GBCS Security**:
```
Device Certificates:
  ├─ ESME: DCC-issued X.509 cert (ECC P-256)
  ├─ TLS 1.2 mutual auth (ESME ↔ DCC)
  └─ DLMS/COSEM Application Layer Security (AES-128-GCM)

HAN Security:
  ├─ Zigbee Install Code (16-byte, printed on meter label)
  ├─ Derived link key: AES-MMO(Install Code)
  └─ Network key encrypted with link key during join
```

**Use Case**: UK residential electricity + gas metering, prepayment (PAYG) meters, time-of-use tariffs.

---

## 7. SPINE / EEBUS (Home Energy Management)

**Reference**: SPINE (Smart Premises Interoperable Neutral-message Exchange), EEBUS Initiative

**Purpose**: Home energy management system (HEMS) protocol. Coordinates EV charger, heat pump, solar inverter, battery storage. Based on SHIP (Smart Home IP) over WebSocket.

**SHIP Protocol**:
```
Transport: WebSocket (wss://), mDNS discovery
Message Format: JSON-LD (SPINE data model)
Connection:
  1. mDNS advertisement: _ship._tcp.local
  2. Client connects to wss://[IPv6]:4712
  3. SHIP handshake (SKI exchange, PIN pairing for new devices)
  4. SPINE message exchange (JSON-LD)
```

**SPINE Data Model**:
```
Device (e.g., EV Charger):
  ├─ DeviceClassification (EVSE)
  ├─ Functions:
  │   ├─ ElectricalConnection (voltage, current, power)
  │   ├─ LoadControl (setpoint, schedule)
  │   └─ Measurement (active power, energy)
  └─ Features:
      ├─ LoadControl: Adjustable power (0-11 kW)
      └─ Measurement: Real-time kW

Example JSON-LD Message (Set EV Charger Power):
{
  "@context": "https://eebus.org/spine",
  "datagram": {
    "header": {
      "specificationVersion": "1.3.0",
      "addressSource": "d:_i:HEMS_12345",
      "addressDestination": "d:_i:EVSE_67890"
    },
    "payload": {
      "cmd": [{
        "function": "loadControlLimitListData",
        "filter": [{"cmdControl": {"partial": {}}}],
        "loadControlLimitListData": {
          "loadControlLimitData": [{
            "limitId": 1,
            "value": 3000,  // 3000W (3kW)
            "isLimitActive": true
          }]
        }
      }]
    }
  }
}
```

**Use Cases**:
```
Dynamic EV Charging:
  ├─ HEMS monitors grid load (smart meter)
  ├─ Solar inverter reports excess production (2 kW)
  ├─ HEMS increases EV charger setpoint (use solar surplus)
  └─ Grid export minimized (self-consumption optimized)

Heat Pump Scheduling:
  ├─ HEMS receives TOU pricing signal (cheap at night)
  ├─ Heat pump pre-heats water tank at 2 AM
  └─ Reduced daytime peak demand
```

**Use Case**: German HEMS (E.ON, SMA), Dutch smart grid (Eneco), EV-PV integration.

---

## 8. IEC 61851 (EV Conductive Charging)

**Reference**: IEC 61851-1 (General Requirements)

**Charging Modes**:
```
Mode 1: AC charging, no pilot signal (deprecated, unsafe)
Mode 2: AC charging, in-cable control box (PWM pilot, GFCI)
Mode 3: AC charging, fixed installation (Type 2 connector, PWM pilot)
Mode 4: DC fast charging (CHAdeMO, CCS, GB/T), digital communication

**(See [automotive-transport.md](automotive-transport.md) for ISO 15118 DC charging)**
```

**Mode 3 PWM Pilot Signal** (IEC 61851-1 §A.5):
```
Control Pilot (CP) Pin:
  ├─ +12V: EVSE ready, no vehicle
  ├─ +9V:  Vehicle connected, not ready to charge
  ├─ +6V:  Vehicle ready to charge
  ├─ +3V:  Vehicle charging with ventilation required
  └─ -12V: EVSE fault

PWM Duty Cycle → Max Current:
  ├─ 10%: 6A
  ├─ 20%: 12A
  ├─ 50%: 30A
  ├─ 90%: 54A
  └─ Formula: I = Duty% × 0.6A (for <85%), or (Duty%-64) × 2.5A (for ≥85%)

Example: 50% duty cycle → 30A × 230V = 6.9 kW max
```

**Proximity Pilot (PP)**:
```
Resistance coding (Type 2 cable):
  ├─ 220Ω: 32A cable
  ├─ 680Ω: 20A cable
  └─ 1.5kΩ: 13A cable

EVSE reads PP resistance → limits max current to cable rating
```

**Use Case**: Residential AC charging (Mode 2 portable, Mode 3 wallbox), fleet charging.

---

## 9. EU Battery Passport (EU 2023/1542)

**Reference**: EU Regulation 2023/1542 (Battery Regulation), mandatory 2027

**Purpose**: Digital product passport for batteries >2 kWh (EV, LMT - Light Means of Transport, industrial). Tracks SoH, SoC, carbon footprint, recycling info, supply chain transparency.

**Data Model**:
```
Battery Identity:
  ├─ Battery ID (unique serial, QR code + NFC)
  ├─ Manufacturer, Model, Manufacturing Date
  └─ Chemistry (Li-ion NMC, LFP, Solid-state)

Performance & State:
  ├─ Capacity (kWh, rated vs actual)
  ├─ SoH (State of Health, 0-100%, capacity fade + resistance increase)
  ├─ SoC (State of Charge, real-time %)
  ├─ Cycle Count (deep cycles, calendar aging)
  └─ Power Capability (kW, peak discharge/charge)

Sustainability:
  ├─ Carbon Footprint (kg CO2e per kWh, manufacturing + transport)
  ├─ Recycled Content (% cobalt, nickel, lithium from recycling)
  └─ Supply Chain (battery cell origin, responsible sourcing certs)

Circularity:
  ├─ Disassembly Instructions (PDF, 3D model)
  ├─ Spare Parts Availability (years)
  └─ Recycling Info (collection points, recycler contacts)
```

**LwM2M Integration** (proposed):
```
LwM2M Object 33001 (Battery Passport):
  Resource 0: Battery ID (String)
  Resource 1: Manufacturer (String)
  Resource 5700: SoH (%) (Float, R)
  Resource 5701: SoC (%) (Float, R)
  Resource 5750: Cycle Count (Integer, R)
  Resource 5751: Carbon Footprint (kg CO2e/kWh) (Float, R)

LwM2M Server queries:
  GET /33001/0/5700  → SoH: 87.5%
  GET /33001/0/5750  → Cycles: 1234
```

**Access**:
```
QR Code / NFC Tag on Battery:
  Scan → Redirect to web portal (HTTPS)
  Portal displays:
    ├─ Battery technical specs
    ├─ Real-time SoH (if connected)
    ├─ Service history (battery swaps, repairs)
    └─ Recycling instructions

API (for authorized parties):
  Fleet operators: Monitor battery health across 100+ vehicles
  Recyclers: Identify chemistry, prioritize high-value materials
  Regulators: Audit carbon footprint claims
```

**Use Case**: EV battery second-life (repurpose 80% SoH for stationary storage), compliance audit (EU market surveillance), consumer transparency (used EV purchase decision).

---

## Key Specification References

| Standard         | Title                                       | Publisher       | Year |
|------------------|---------------------------------------------|-----------------|------|
| IEEE 2030.5-2018 | Smart Energy Profile 2.0                    | IEEE            | 2018 |
| IEC 62746-10-3   | OpenADR 2.0b Profile                        | IEC             | 2018 |
| IEC 61968/61970  | CIM (Common Information Model)              | IEC             | 2023 |
| ANSI C12.19      | Utility Industry Metering Data Tables       | ANSI            | 2012 |
| ANSI C12.22      | Protocol Specification for Energy Metering  | ANSI            | 2012 |
| ITU-T G.9903     | G3-PLC Narrowband PLC                       | ITU-T           | 2017 |
| SMETS2           | Smart Metering Equipment Technical Spec v2  | UK BEIS         | 2017 |
| GBCS             | Great Britain Companion Specification       | UK DCC          | 2020 |
| SPINE 1.3        | Smart Premises Neutral-message Exchange     | EEBUS           | 2023 |
| IEC 61851-1      | EV Conductive Charging System Part 1        | IEC             | 2017 |
| EU 2023/1542     | Battery Regulation (EU Battery Passport)    | EU Commission   | 2023 |

---

## Related Files
- **[automotive-transport.md](automotive-transport.md)**: ISO 15118 (V2G Plug & Charge), OCPP (charge point management), IEC 61851 Mode 4
- **[industrial-ot.md](industrial-ot.md)**: IEC 61850 (substation automation), Modbus (meter gateways)
- **[lpwan.md](lpwan.md)**: DLMS/COSEM (European smart meters), Wireless M-Bus (gas/water meters)
- **[device-management.md](device-management.md)**: TR-369 USP (broadband gateway energy mgmt), LwM2M (battery passport)
- **[application-messaging.md](application-messaging.md)**: MQTT (OpenADR over MQTT), CoAP (6LoWPAN/G3-PLC)
