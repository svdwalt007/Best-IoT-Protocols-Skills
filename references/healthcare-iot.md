# Healthcare IoT Protocols

> **Scope:** This reference covers healthcare and medical device IoT protocols — from personal
> health devices (IEEE 11073 PHD), Bluetooth health profiles, to health informatics standards
> (HL7 FHIR), integration (IHE PCD), and regulatory compliance (FDA 21 CFR Part 11, IEC 62304,
> EU MDR). For BLE GATT physical layer, see [pan-short-range.md](pan-short-range.md).

---

## Table of Contents

1. [IEEE 11073 PHD — Personal Health Devices](#1-ieee-11073-phd--personal-health-devices)
   - [IEEE 11073-20601 — Optimized Exchange Protocol](#ieee-11073-20601--optimized-exchange-protocol)
   - [Device Specializations (10404–10417)](#device-specializations-1040410417)
   - [IEEE 11073-10103 Nomenclature](#ieee-11073-10103-nomenclature)
2. [Bluetooth SIG Health Profiles](#2-bluetooth-sig-health-profiles)
3. [HL7 FHIR — Health Informatics](#3-hl7-fhir--health-informatics)
4. [IHE PCD — Integration Profiles](#4-ihe-pcd--integration-profiles)
5. [Regulatory & Compliance](#5-regulatory--compliance)

---

## 1. IEEE 11073 PHD — Personal Health Devices

**IEEE 11073** is the family of standards for personal health device communication, enabling interoperability between medical devices and health IT systems.

### IEEE 11073-20601 — Optimized Exchange Protocol

**IEEE 11073-20601 (OEP)** is the base protocol for agent (device) to manager (gateway/phone) communication.

**Architecture:**
```
Agent (PHD: Pulse Ox, BP Monitor)    Manager (Phone, Gateway, EHR)
    │                                      │
    ├─ 1. Association Request          ──>│
    │    (System ID, config)               │
    │                                      │
    │  <── 2. Association Response         │
    │      (Accepted/Rejected)             │
    │                                      │
    ├─ 3. Configuration Phase           <──>│
    │    (Exchange MDS, VMDs, metrics)     │
    │                                      │
    ├─ 4. Operating Phase               <──>│
    │    Event Reports (measurements)      │
    │    Confirmed Actions (commands)      │
    │                                      │
    ├─ 5. Disassociation                ──>│
    │    (Normal/timeout)                  │
    └──────────────────────────────────────┘
```

**Transport Layers:**
- **USB**: IEEE 11073-20601 PHD over USB (CDC ACM)
- **Bluetooth HDP**: IEEE 11073-20601 over Bluetooth Health Device Profile
- **Bluetooth LE**: IEEE 11073-20601 over BLE GATT (see Bluetooth SIG profiles)
- **Zigbee**: IEEE 11073-20601 over Zigbee Health Care Profile
- **USB-PHDC**: Personal Healthcare Device Class

**State Machine:**
1. **Unassociated**: Initial state
2. **Associating**: Negotiating association
3. **Associated**: Association established (cfg-wait or operating sub-states)
4. **Disassociating**: Terminating association

**Data Model Hierarchy:**
```
MDS (Medical Device System)
  │
  ├─ VMD (Virtual Medical Device) — Optional grouping
  │   │
  │   └─ Channel — Grouping of related metrics
  │       │
  │       └─ Metric — Individual measurement (Numeric, RT-SA, Enumeration)
  │
  └─ PM-Store — Persistent storage of measurements
```

**Object Types:**
- **MDS**: Top-level device object (System-Id, System-Model)
- **Numeric**: Single scalar value (e.g., heart rate = 72 bpm)
- **RT-SA (Real-Time Sample Array)**: Waveform data (e.g., ECG waveform)
- **Enumeration**: Coded value (e.g., pacemaker detected = Yes/No)
- **PM-Store**: Persistent metric storage for offline data

**APDU (Application Protocol Data Unit) Structure:**
- **AARQ** (Association Request): Agent requests association
- **AARE** (Association Response): Manager accepts/rejects
- **RLRQ** (Release Request): Initiate disassociation
- **RLRE** (Release Response): Confirm disassociation
- **ABRT** (Abort): Abnormal termination
- **Event Report**: Agent sends measurements
- **Confirmed Action**: Manager sends commands (e.g., start measurement)

**Configuration:**
- **Standard Configuration**: Device sends known config ID; manager retrieves from database
- **Extended Configuration**: Device sends full configuration object (for unknown devices)

---

### Device Specializations (10404–10417)

Each IEEE 11073-104xx standard defines device-specific objects, attributes, and measurements.

**IEEE 11073-10404 — Pulse Oximeter:**
- **Metrics**:
  - SpO2 (Oxygen saturation): Numeric, unit = % (MDC_PULS_OXIM_SAT_O2)
  - Pulse Rate: Numeric, unit = bpm (MDC_PULS_OXIM_PULS_RATE)
  - Pleth Waveform: RT-SA (optional)
- **Attributes**:
  - Measurement site (finger, ear, toe)
  - Sensor operational state
- **Events**: Low SpO2 alarm, sensor off, probe fault

**IEEE 11073-10406 — Basic ECG:**
- **Metrics**:
  - Heart Rate: Numeric, unit = bpm (MDC_ECG_HEART_RATE)
  - ECG Waveform: RT-SA (lead I, II, III, aVR, aVL, aVF, V1–V6)
  - QRS Duration: Numeric, unit = ms
- **Attributes**:
  - Lead configuration (12-lead, 3-lead, 5-lead)
  - Sample rate (250 Hz, 500 Hz, 1000 Hz)
- **Events**: Arrhythmia detection, lead-off

**IEEE 11073-10407 — Blood Pressure Monitor:**
- **Metrics**:
  - Systolic BP: Numeric, unit = mmHg (MDC_PRESS_BLD_NONINV_SYS)
  - Diastolic BP: Numeric, unit = mmHg (MDC_PRESS_BLD_NONINV_DIA)
  - Mean Arterial Pressure (MAP): Numeric, unit = mmHg
  - Pulse Rate: Numeric, unit = bpm
  - Cuff pressure waveform: RT-SA (optional)
- **Attributes**:
  - Measurement site (arm, wrist)
  - Cuff size
  - User ID (multi-user devices)
- **Events**: Irregular heartbeat detected, cuff error, measurement failed

**IEEE 11073-10415 — Weighing Scale:**
- **Metrics**:
  - Body Weight: Numeric, unit = kg or lb (MDC_MASS_BODY_ACTUAL)
  - BMI (Body Mass Index): Numeric, unit = kg/m² (computed if height configured)
- **Attributes**:
  - Height (stored for BMI calculation)
  - User ID (multi-user)
- **Events**: Overload, unstable weight

**IEEE 11073-10417 — Glucose Meter:**
- **Metrics**:
  - Glucose Concentration: Numeric, unit = mg/dL or mmol/L (MDC_CONC_GLU_CAPILLARY_WHOLEBLOOD)
  - HbA1c: Numeric, unit = % (MDC_HBA1C)
- **Attributes**:
  - Meal context (fasting, before meal, after meal, bedtime)
  - Sample location (finger, forearm, AST)
  - Control solution test flag
- **Events**: High glucose alarm, low glucose alarm, test strip error

**Common Attributes (All Devices):**
- **System-Id**: Unique device identifier (EUI-64)
- **System-Model**: Manufacturer + model (e.g., "Acme-Corp-BP-2000")
- **Time Stamp**: Absolute or relative (Base-Offset-Time)
- **Measurement-Status**: Valid, not-available, questionable, invalid
- **Battery Level**: Percent remaining

---

### IEEE 11073-10103 Nomenclature

**MDC (Medical Device Communication)** nomenclature codes identify measurements and attributes.

**Code Structure:**
- **Partition**: High-level category (e.g., 2 = Dimension, 4 = Vital Signs, 8 = Device)
- **Term Code**: Specific term within partition

**Common MDC Codes:**

| MDC Code (hex) | MDC Code (decimal) | Term | Unit |
|---|---|---|---|
| MDC_PULS_OXIM_SAT_O2 | 150456 | Oxygen saturation (SpO2) | % |
| MDC_PULS_OXIM_PULS_RATE | 149530 | Pulse rate (from pulse oximeter) | bpm |
| MDC_ECG_HEART_RATE | 147842 | Heart rate (from ECG) | bpm |
| MDC_PRESS_BLD_NONINV_SYS | 150017 | Systolic blood pressure | mmHg |
| MDC_PRESS_BLD_NONINV_DIA | 150018 | Diastolic blood pressure | mmHg |
| MDC_MASS_BODY_ACTUAL | 188736 | Body weight | kg |
| MDC_CONC_GLU_CAPILLARY_WHOLEBLOOD | 160196 | Glucose (capillary whole blood) | mg/dL |
| MDC_TEMP_BODY | 150364 | Body temperature | °C |
| MDC_DIM_PERCENT | 544 | Percent | % |
| MDC_DIM_BEAT_PER_MIN | 2720 | Beats per minute | bpm |
| MDC_DIM_MMHG | 3872 | Millimeters of mercury | mmHg |

**Partition Codes:**
- **0 (MDC_PART_UNSPEC)**: Unspecified
- **1 (MDC_PART_OBJ)**: Object infrastructure
- **2 (MDC_PART_SCADA)**: SCADA (supervisory control)
- **4 (MDC_PART_EVT)**: Events
- **5 (MDC_PART_DIM)**: Dimensions (units)
- **8 (MDC_PART_PHD_DM)**: PHD device model
- **128 (MDC_PART_PHD_HF)**: Health and Fitness
- **129 (MDC_PART_PHD_AI)**: Aging Independently

---

## 2. Bluetooth SIG Health Profiles

**Bluetooth SIG** defines GATT-based profiles for health devices using BLE.

### Health Device Profile (HDP) — Classic Bluetooth

**HDP (IEEE 11073-20601 over Bluetooth Classic):**
- **L2CAP-based**: Uses MCAP (Multi-Channel Adaptation Protocol)
- **Roles**: Source (device) and Sink (gateway)
- **Deprecated**: Replaced by BLE GATT profiles

### BLE GATT Health Profiles

**GATT Services for Health:**

**Heart Rate Service (0x180D):**
- **Heart Rate Measurement (0x2A37)**: Characteristic (notify)
  - **Flags**: Bit 0 = HR format (uint8/uint16), Bit 1-2 = Sensor contact, Bit 3 = Energy expended, Bit 4 = RR-interval
  - **Heart Rate Value**: uint8 or uint16 (bpm)
  - **Energy Expended**: uint16 (kJ, optional)
  - **RR-Interval**: uint16 array (1/1024 sec units, for HRV)
- **Body Sensor Location (0x2A38)**: Characteristic (read)
  - Values: 0=Other, 1=Chest, 2=Wrist, 3=Finger, 4=Hand, 5=Ear Lobe, 6=Foot
- **Heart Rate Control Point (0x2A39)**: Characteristic (write)
  - Command: 0x01 = Reset Energy Expended

**Blood Pressure Service (0x1810):**
- **Blood Pressure Measurement (0x2A35)**: Characteristic (indicate)
  - **Flags**: Bit 0 = Unit (mmHg/kPa), Bit 1 = Timestamp, Bit 2 = Pulse rate, Bit 3 = User ID, Bit 4 = Measurement status
  - **Systolic**: SFLOAT (mmHg or kPa)
  - **Diastolic**: SFLOAT
  - **MAP**: SFLOAT (Mean Arterial Pressure)
  - **Timestamp**: Date/time (optional)
  - **Pulse Rate**: SFLOAT (bpm, optional)
  - **User ID**: uint8 (1–255, optional)
  - **Measurement Status**: uint16 bitfield (body movement, cuff too loose, irregular pulse, etc.)
- **Intermediate Cuff Pressure (0x2A36)**: Characteristic (notify, optional)
- **Blood Pressure Feature (0x2A49)**: Characteristic (read)
  - Bitfield: Body movement detection, cuff fit detection, irregular pulse detection, pulse rate range detection, multiple bond support

**Glucose Service (0x1808):**
- **Glucose Measurement (0x2A18)**: Characteristic (notify)
  - **Flags**: Bit 0 = Timestamp, Bit 1 = Glucose concentration, type, and location present, Bit 2 = Concentration units (mg/dL/mmol/L), Bit 3 = Sensor status annunciation, Bit 4 = Context info follows
  - **Sequence Number**: uint16 (monotonic)
  - **Base Time**: Date/time
  - **Time Offset**: int16 (minutes)
  - **Glucose Concentration**: SFLOAT (mg/dL or mmol/L)
  - **Type-Sample Location**: uint8 (nibble 0-3 = Type, nibble 4-7 = Location)
    - Type: 1=Capillary Whole blood, 2=Capillary Plasma, 3=Venous Whole blood, 4=Venous Plasma, 5=Arterial Whole blood, 6=Arterial Plasma, 7=Undetermined Whole blood, 8=Undetermined Plasma, 9=ISF (Interstitial Fluid), 10=Control Solution
    - Location: 1=Finger, 2=AST (Alternate Site Test), 3=Earlobe, 4=Control solution, 15=Not available
  - **Sensor Status Annunciation**: uint16 bitfield (battery low, sensor malfunction, insufficient sample, strip insertion error, strip type incorrect, result too high/low, temperature too high/low, read interrupted, general device fault, time fault)
- **Glucose Measurement Context (0x2A34)**: Characteristic (notify, optional)
  - Meal context (preprandial, postprandial, fasting, casual, bedtime)
  - Tester (self, healthcare professional, lab test, not available)
  - Health (minor health issues, major health issues, during menses, under stress, no health issues, not available)
  - Medication (rapid acting insulin, short acting insulin, intermediate acting insulin, long acting insulin, pre-mixed insulin)
  - Exercise duration/intensity
- **Record Access Control Point (RACP, 0x2A52)**: Characteristic (write/indicate)
  - Commands: Report stored records, Delete stored records, Abort operation, Report number of stored records, Response code
- **Glucose Feature (0x2A51)**: Characteristic (read)

**Continuous Glucose Monitoring (CGM) Service (0x181F):**
- **CGM Measurement (0x2AA7)**: Characteristic (notify)
  - Real-time glucose value (every 1–5 minutes)
  - Trend information (rate of change)
  - Quality (ok, suspect, warning, error)
- **CGM Feature (0x2AA8)**: Characteristic (read)
- **CGM Status (0x2AA9)**: Characteristic (read)
- **CGM Session Start Time (0x2AAA)**: Characteristic (read/write)
- **CGM Session Run Time (0x2AAB)**: Characteristic (read)
- **RACP (0x2A52)**: Record Access Control Point

**Pulse Oximeter Service (0x1822):**
- **PLX Spot-Check Measurement (0x2A5E)**: Characteristic (indicate)
  - SpO2 (percent)
  - Pulse Rate (bpm)
  - Timestamp
  - Measurement Status
  - Device and Sensor Status
- **PLX Continuous Measurement (0x2A5F)**: Characteristic (notify)
  - Real-time SpO2 + Pulse Rate (streaming)
- **PLX Features (0x2A60)**: Characteristic (read)
- **Record Access Control Point (0x2A52)**: Characteristic (write/indicate)

**Weight Scale Service (0x181D):**
- **Weight Measurement (0x2A9D)**: Characteristic (indicate)
  - Weight (kg or lb)
  - Timestamp (optional)
  - User ID (optional)
  - BMI (optional, if height stored)
  - Height (optional)
- **Weight Scale Feature (0x2A9E)**: Characteristic (read)
  - Resolution (0.5 kg, 0.2 kg, 0.1 kg, 0.05 kg, 0.02 kg, 0.01 kg, 0.005 kg)
  - Timestamp supported
  - Multiple users supported
  - BMI supported

**Body Composition Service (0x181B):**
- **Body Composition Measurement (0x2A9C)**: Characteristic (indicate)
  - Body Fat Percentage (%)
  - Muscle Mass (kg)
  - Bone Mass (kg)
  - Basal Metabolism (kcal)
  - Muscle Percentage (%)
  - Body Water Mass (kg)
  - Impedance (ohm)
  - Weight (kg)
  - Height (m)
- **Body Composition Feature (0x2A9B)**: Characteristic (read)

---

## 3. HL7 FHIR — Health Informatics

**HL7 FHIR (Fast Healthcare Interoperability Resources)** is the modern RESTful standard for health data exchange.

**FHIR Releases:**
- **FHIR R4 (v4.0.1)**: October 2019, most widely deployed
- **FHIR R5 (v5.0.0)**: March 2023, latest version

### FHIR Resource Model

**Core Resources for IoT:**

**Device Resource:**
```json
{
  "resourceType": "Device",
  "id": "pulse-ox-001",
  "identifier": [
    {
      "system": "urn:oid:1.2.840.10004.1.1.1.0.0.1.0.0.1.2680",
      "value": "00-1B-63-04-03-5E-9F-01"
    }
  ],
  "status": "active",
  "manufacturer": "Acme Health Devices",
  "modelNumber": "PulseOx-3000",
  "serialNumber": "SN123456789",
  "deviceName": [
    {
      "name": "Acme PulseOx 3000",
      "type": "user-friendly-name"
    }
  ],
  "type": {
    "coding": [
      {
        "system": "urn:iso:std:iso:11073:10101",
        "code": "65573",
        "display": "MDC_MOC_VMS_MDS_SIMP"
      }
    ]
  },
  "version": [
    {
      "type": {
        "text": "Firmware"
      },
      "value": "1.2.5"
    }
  ]
}
```

**Observation Resource (Measurement):**
```json
{
  "resourceType": "Observation",
  "id": "spo2-measurement-001",
  "status": "final",
  "category": [
    {
      "coding": [
        {
          "system": "http://terminology.hl7.org/CodeSystem/observation-category",
          "code": "vital-signs",
          "display": "Vital Signs"
        }
      ]
    }
  ],
  "code": {
    "coding": [
      {
        "system": "http://loinc.org",
        "code": "2708-6",
        "display": "Oxygen saturation in Arterial blood"
      },
      {
        "system": "urn:iso:std:iso:11073:10101",
        "code": "150456",
        "display": "MDC_PULS_OXIM_SAT_O2"
      }
    ]
  },
  "subject": {
    "reference": "Patient/patient-001"
  },
  "effectiveDateTime": "2026-04-13T10:30:00Z",
  "device": {
    "reference": "Device/pulse-ox-001"
  },
  "valueQuantity": {
    "value": 97,
    "unit": "%",
    "system": "http://unitsofmeasure.org",
    "code": "%"
  }
}
```

**DeviceMetric Resource:**
```json
{
  "resourceType": "DeviceMetric",
  "id": "spo2-metric",
  "identifier": [
    {
      "system": "http://acme.com/devices",
      "value": "pulse-ox-001-spo2-channel"
    }
  ],
  "type": {
    "coding": [
      {
        "system": "urn:iso:std:iso:11073:10101",
        "code": "150456",
        "display": "MDC_PULS_OXIM_SAT_O2"
      }
    ]
  },
  "unit": {
    "coding": [
      {
        "system": "http://unitsofmeasure.org",
        "code": "%"
      }
    ]
  },
  "source": {
    "reference": "Device/pulse-ox-001"
  },
  "operationalStatus": "on",
  "category": "measurement"
}
```

**Patient Resource:**
```json
{
  "resourceType": "Patient",
  "id": "patient-001",
  "identifier": [
    {
      "system": "http://hospital.org/mrn",
      "value": "MRN123456"
    }
  ],
  "name": [
    {
      "use": "official",
      "family": "Doe",
      "given": ["John"]
    }
  ],
  "gender": "male",
  "birthDate": "1980-01-01"
}
```

### IEEE 11073 to FHIR Mapping

| IEEE 11073 Concept | FHIR Resource | Notes |
|---|---|---|
| MDS (Medical Device System) | Device | Top-level device |
| VMD (Virtual Medical Device) | Device (contained) | Sub-device |
| Numeric Metric | DeviceMetric | Measurement channel |
| Measurement | Observation | Actual measurement value |
| MDC nomenclature code | Observation.code.coding | Map to LOINC + MDC |
| PM-Store entry | Observation (historical) | effectiveDateTime in past |

**LOINC Codes (Common):**
- **2708-6**: Oxygen saturation in Arterial blood (SpO2)
- **8867-4**: Heart rate
- **8480-6**: Systolic blood pressure
- **8462-4**: Diastolic blood pressure
- **29463-7**: Body weight
- **2339-0**: Glucose [Mass/volume] in Blood

### SMART on FHIR

**SMART (Substitutable Medical Applications, Reusable Technologies) on FHIR** enables OAuth2-based authorization for FHIR APIs.

**OAuth2 Flow (Authorization Code):**
```
1. App requests authorization → FHIR server /authorize endpoint
2. User authenticates + grants permission
3. FHIR server redirects to app with authorization code
4. App exchanges code for access token → /token endpoint
5. App accesses FHIR API with Bearer token
```

**SMART Scopes:**
- `patient/Observation.read`: Read observations for current patient
- `user/Device.read`: Read devices as authenticated user
- `system/Patient.read`: Read all patients (backend service)

**Launch Contexts:**
- **EHR Launch**: App launched from EHR, receives `launch` parameter
- **Standalone Launch**: App launched independently, user selects patient

### FHIR Bulk Data API

**FHIR Bulk Data Access (Flat FHIR)** for large-scale IoT telemetry export.

**Endpoints:**
- `GET /Patient/$export`: Export all patient data
- `GET /Group/[id]/$export`: Export data for patient group
- `GET /$export`: System-level export

**Flow:**
```
1. Client: POST /Patient/$export (with _type=Observation,Device)
2. Server: 202 Accepted, Content-Location: /bulkstatus/job-123
3. Client: Poll GET /bulkstatus/job-123
4. Server: 200 OK (when complete) with NDJSON file URLs
5. Client: Download NDJSON files (streamed observations)
```

**NDJSON Format:**
- Newline-delimited JSON (one FHIR resource per line)
- Efficient for large datasets
- Files can be streamed line-by-line

---

## 4. IHE PCD — Integration Profiles

**IHE (Integrating the Healthcare Enterprise) PCD (Patient Care Device)** defines integration profiles for medical device interoperability.

### IHE PCD Integration Profiles

**DEC (Device Enterprise Communication):**
- **Purpose**: Communicate patient care device observations to clinical systems
- **Actors**:
  - Device Observation Reporter (DOR): Device/gateway sending observations
  - Device Observation Consumer (DOC): EHR/clinical system receiving observations
- **Transactions**:
  - PCD-01: Communicate PCD Data (HL7 v2.6 ORU^R01 message)
- **Content**: HL7 v2.6 ORU message with OBX segments containing IEEE 11073 nomenclature

**ACM (Alarm Communication Management):**
- **Purpose**: Communicate physiological alarms to clinical systems
- **Actors**:
  - Alarm Reporter (AR): Device generating alarms
  - Alarm Consumer (AC): System receiving/displaying alarms
- **Transactions**:
  - PCD-04: Report Alarm (HL7 v2.6 ORU^R01 with alarm OBX)
- **Alarm Types**: Technical alarm (device fault), physiological alarm (vital sign threshold)

**PIV (Point-of-Care Infusion Verification):**
- **Purpose**: Verify infusion pump programming against clinical orders
- **Actors**:
  - Infusion Order Programmer (IOP): System programming pump
  - Infusion Order Consumer (IOC): Pump receiving programming
- **Transactions**:
  - PCD-03: Communicate Infusion Order (HL7 v2.6 RGV^O15 message)
- **Use Case**: Prevent medication errors by automating pump programming from CPOE (Computerized Physician Order Entry)

**WCM (Waveform Communication Management):**
- **Purpose**: Communicate real-time waveforms (ECG, SpO2 pleth, capnography)
- **Transactions**:
  - PCD-10: Communicate Waveform (HL7 v2.x ORU with OBX containing waveform data)
- **Encoding**: Waveform data as Base64-encoded samples

### HL7 v2.x Message Structure (PCD-01 Example)

```
MSH|^~\&|DeviceGateway|Hospital^ACME^^ISO|ClinicalSystem|Hospital^ACME^^ISO|20260413103000||ORU^R01^ORU_R01|MSG12345|P|2.6|||AL|NE|USA|UNICODE UTF-8|||IHE PCD ORU-R01 2006^IHE^1.3.6.1.4.1.19376.1.6.1.1.1^ISO
PID|1||MRN123456^^^Hospital^MR||Doe^John||19800101|M
PV1|1|I|ICU^BED5^A|||||||||||||||VISIT123456
OBR|1|ORD123456||528391^MDC_DEV_SPEC_PROFILE_BP^MDC|||20260413103000
OBX|1|NM|150017^MDC_PRESS_BLD_NONINV_SYS^MDC||120|266016^MDC_DIM_MMHG^MDC|||||F|||20260413103000
OBX|2|NM|150018^MDC_PRESS_BLD_NONINV_DIA^MDC||80|266016^MDC_DIM_MMHG^MDC|||||F|||20260413103000
OBX|3|NM|149530^MDC_PULS_OXIM_PULS_RATE^MDC||72|264864^MDC_DIM_BEAT_PER_MIN^MDC|||||F|||20260413103000
```

**Key Segments:**
- **MSH**: Message header (sender, receiver, timestamp, message type)
- **PID**: Patient identification
- **PV1**: Patient visit (location)
- **OBR**: Observation request (device profile)
- **OBX**: Observation/result (actual measurements with MDC codes)

### POCT1-A2 (Point-of-Care Testing)

**IHE POCT1-A2** profile for laboratory point-of-care devices (glucose meters, blood gas analyzers, coagulation meters).

**HL7 LRI (Laboratory Results Interface):**
- HL7 v2.x ORU^R01 messages
- LOINC codes for lab tests
- Integration with LIS (Laboratory Information System)

---

## 5. Regulatory & Compliance

### FDA 21 CFR Part 11 — Electronic Records & Signatures

**21 CFR Part 11** (US FDA) governs electronic records and signatures for medical devices and pharmaceuticals.

**Requirements:**
1. **Validation**: Systems must be validated (software validation, IQ/OQ/PQ)
2. **Audit Trails**: All record changes must be logged (who, what, when, why)
3. **Electronic Signatures**: Must be unique, verified, non-repudiable
4. **Access Controls**: Role-based access, password complexity, account lockout
5. **Data Integrity**: ALCOA+ principles (Attributable, Legible, Contemporaneous, Original, Accurate, Complete, Consistent, Enduring, Available)

**Applicability to IoT:**
- Medical device firmware must maintain audit logs of configuration changes
- Cloud-based storage of patient data must ensure data integrity
- User authentication required for device programming

---

### IEC 62304 — Medical Device Software Lifecycle

**IEC 62304** defines software development lifecycle for medical device software.

**Safety Classes:**
- **Class A**: No injury or damage to health possible
- **Class B**: Non-serious injury possible
- **Class C**: Death or serious injury possible

**Lifecycle Processes:**
1. **Software Development Planning**: Define development plan, standards to follow, risk management integration
2. **Software Requirements Analysis**: Functional, performance, interface requirements
3. **Software Architectural Design**: Top-level design, interface specifications
4. **Software Detailed Design**: Unit-level design
5. **Software Unit Implementation**: Coding, unit testing
6. **Software Integration Testing**: Verify interfaces between units
7. **Software System Testing**: Verify against requirements
8. **Software Release**: Version control, release notes

**Risk Management (per ISO 14971):**
- Software-related hazards identified during risk analysis
- Risk controls implemented (design, protective measures, information for safety)
- Residual risk evaluated

**Configuration Management:**
- Version control (Git, SVN)
- Change control board approvals for changes
- Traceability matrix (requirements ↔ design ↔ tests ↔ code)

---

### IEC 62443 — Cybersecurity for Medical Devices

**IEC 62443** (originally ISA/IEC 62443 for industrial automation) adapted for medical devices.

**Security Levels (SL):**
- **SL 1**: Protection against casual or coincidental violation
- **SL 2**: Protection against intentional violation using simple means
- **SL 3**: Protection against intentional violation using sophisticated means
- **SL 4**: Protection against intentional violation using sophisticated means with extended resources

**Key Requirements:**
- **IAC (Identification and Authentication Control)**: Unique user IDs, password policies
- **UC (Use Control)**: Authorization, least privilege
- **SI (System Integrity)**: Input validation, malware detection
- **DC (Data Confidentiality)**: Encryption at rest and in transit
- **RDF (Restricted Data Flow)**: Network segmentation, firewall rules
- **TRE (Timely Response to Events)**: Audit logging, intrusion detection

---

### EU MDR 2017/745 — Medical Device Regulation

**EU MDR** (effective May 2021, replacing MDD 93/42/EEC) governs medical devices sold in EU.

**Classification (Annex VIII):**
- **Class I**: Low risk (non-invasive, short-term contact)
- **Class IIa**: Medium risk (invasive, short-term)
- **Class IIb**: High risk (invasive, long-term, critical anatomy)
- **Class III**: Very high risk (life-sustaining, implantable)

**IoT Relevance:**
- **Standalone software** (mobile apps, cloud platforms) classified as medical devices if they have medical purpose
- **AI/ML algorithms** for diagnosis/treatment are Class IIa or higher
- **Cybersecurity** is part of essential requirements (Annex I, §17.4)

**CE Marking Process:**
1. Classify device
2. Demonstrate conformity (technical documentation, clinical evaluation)
3. Implement QMS (ISO 13485)
4. Involve Notified Body (for Class IIa/IIb/III)
5. Affix CE mark
6. Register in EUDAMED database

**UDI (Unique Device Identification):**
- **UDI-DI (Device Identifier)**: Identifies device model
- **UDI-PI (Production Identifier)**: Lot/serial number, expiration date
- Encoded in barcode (GS1, HIBCC, ICCBBA)
- Required on device label and packaging

---

### HIPAA (US Healthcare Data Privacy)

**HIPAA (Health Insurance Portability and Accountability Act)** protects PHI (Protected Health Information) in the US.

**PHI Identifiers (18 types):**
- Names, addresses, dates (except year), phone numbers, email addresses, SSN, medical record numbers, device identifiers/serial numbers, IP addresses, biometric identifiers, photos

**Security Rule (45 CFR 164.308–316):**
- **Administrative Safeguards**: Access controls, workforce training, risk analysis
- **Physical Safeguards**: Facility access, device security
- **Technical Safeguards**: Encryption, audit logs, transmission security

**IoT Implications:**
- **BAA (Business Associate Agreement)** required for cloud service providers storing PHI
- **Encryption**: PHI must be encrypted in transit (TLS 1.2+) and at rest (AES-256)
- **Access Logs**: All PHI access must be logged for 6 years
- **Breach Notification**: Breaches affecting >500 individuals must be reported to HHS within 60 days

---

## Cross-References

- **Bluetooth GATT/BLE**: See [pan-short-range.md](pan-short-range.md) for BLE GAP/GATT/ATT/SMP physical layer
- **MQTT for telemetry**: See [application-messaging.md](application-messaging.md) for MQTT v3.1.1/v5.0
- **DTLS/TLS security**: See [security-provisioning.md](security-provisioning.md) for DTLS 1.2/1.3, TLS, certificates

---

## Key Specification References

| Technology | Primary Specification | Notes |
|---|---|---|
| IEEE 11073-20601 | IEEE 11073-20601-2016 | Optimized Exchange Protocol |
| IEEE 11073-10404 | IEEE 11073-10404-2008 | Pulse oximeter |
| IEEE 11073-10406 | IEEE 11073-10406-2011 | Basic ECG |
| IEEE 11073-10407 | IEEE 11073-10407-2010 | Blood pressure monitor |
| IEEE 11073-10415 | IEEE 11073-10415-2010 | Weighing scale |
| IEEE 11073-10417 | IEEE 11073-10417-2011 | Glucose meter |
| IEEE 11073-10103 | IEEE 11073-10103a-2015 | Nomenclature |
| Bluetooth Heart Rate | GATT Spec Supplement v8 | Service 0x180D |
| Bluetooth Blood Pressure | GATT Spec Supplement v8 | Service 0x1810 |
| Bluetooth Glucose | GATT Spec Supplement v8 | Service 0x1808 |
| Bluetooth Pulse Oximeter | GATT Spec Supplement v8 | Service 0x1822 |
| HL7 FHIR R4 | HL7 FHIR R4 (v4.0.1) | RESTful health data |
| HL7 FHIR R5 | HL7 FHIR R5 (v5.0.0) | Latest release |
| SMART on FHIR | HL7 SMART App Launch IG | OAuth2 authorization |
| IHE PCD DEC | IHE PCD TF-1/TF-2 | Device Enterprise Communication |
| IHE PCD ACM | IHE PCD TF-1/TF-2 | Alarm Communication Management |
| FDA 21 CFR Part 11 | 21 CFR Part 11 | Electronic records/signatures |
| IEC 62304 | IEC 62304:2006 + Amd 1:2015 | Medical device software lifecycle |
| IEC 62443 | IEC 62443-4-2 | Security for medical devices |
| EU MDR | Regulation (EU) 2017/745 | CE marking, UDI |
| HIPAA Security Rule | 45 CFR 164.308–316 | PHI protection (US) |

---

*Cross-references:*
- *Bluetooth BLE GATT: `references/pan-short-range.md`*
- *MQTT telemetry: `references/application-messaging.md`*
- *Security (DTLS/TLS): `references/security-provisioning.md`*
