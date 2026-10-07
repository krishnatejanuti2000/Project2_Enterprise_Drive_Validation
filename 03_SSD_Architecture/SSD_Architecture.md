# MODULE 03 — SSD ARCHITECTURE

## Priority

**P0 — Must Master**

## Objective

Understand the internal architecture of an SSD, the role of its major components, the NAND hierarchy, and the relationship between the host's logical storage view and the SSD's physical NAND storage.

The key mastery requirement is:

> **Explain how a host logical write eventually becomes NAND programming.**

---

# 1. SSD Architecture — Big Picture

An SSD is a complete storage device made of multiple cooperating components.

At a high level:

```text
Host
  ↓
NVMe / PCIe
  ↓
SSD Controller
  ↓
SSD Internal Management
  ↓
NAND Flash
```

The host works with a **logical storage view**, while the SSD internally manages the physical NAND storage system.

```text
Host / Logical World
        ↓
NVMe / PCIe
        ↓
SSD Control World
        ↓
Physical NAND World
```

---

# 2. SSD Controller

## Definition

The **SSD Controller** is the **central hardware/control subsystem** of the SSD.

Its main role is to receive and process host storage commands and coordinate the internal operations required to store and retrieve data.

## Main Responsibilities

- Command processing
- Data movement
- Address translation
- NAND scheduling and management
- Metadata handling
- ECC and error handling
- Coordination of garbage collection
- Coordination of wear leveling

These functions are implemented or coordinated through the SSD firmware running on the controller.

## Mental Model

```text
Host
  ↓
NVMe / Interface
  ↓
SSD Controller
  ↓
NAND
```

### Controller vs Firmware

```text
Controller
= Hardware / control subsystem

Firmware
= Specialized software / logic running on the controller
```

The controller provides the processing and control resources, while firmware provides the logic that controls SSD behavior.

---

# 3. NAND Flash

## Definition

**NAND Flash is the non-volatile physical storage medium inside the SSD.**

NAND is where persistent user data is ultimately stored.

The SSD controller does not expose the physical NAND organization directly to the host.

## NAND Organization

```text
Channel
   ↓
Die
   ↓
Plane
   ↓
Block
   ↓
Page
```

### Channel

A NAND **channel** is a communication/data path between the SSD controller and NAND resources.

> Do not confuse this with the transistor-channel concept from semiconductor physics.

### Die

A **die** is an individual NAND memory unit containing internal structures such as planes, blocks and pages.

### Plane

A **plane** is an internal subdivision within a NAND die and is part of the NAND organization and parallel operation.

### Block

A **block** is a collection of NAND pages.

A block is the smallest unit associated with NAND erase operations.

### Page

A **page** is the NAND granularity associated with read and program operations.

## Fundamental NAND Operation Relationship

```text
Read     → Page
Program  → Page
Erase    → Block
```

This distinction is fundamental to NAND behavior.

---

# 4. DRAM

## Definition

**DRAM is high-speed, volatile memory used as fast working memory in many SSDs.**

It is physically present on SSDs that use dedicated DRAM.

```text
SSD
├── Controller
├── DRAM
└── NAND
```

## Purpose

Dedicated DRAM can hold frequently accessed internal information such as:

- FTL mapping information
- Address translation information
- Command queues
- Temporary write buffers
- Internal management data

Keeping frequently accessed information in DRAM allows the controller to access it efficiently.

## Important Distinction

```text
DRAM
→ Volatile working memory

NAND
→ Non-volatile persistent storage
```

DRAM is not the permanent storage location for user data.

---

# 5. HMB — Host Memory Buffer

## Definition

**HMB (Host Memory Buffer)** is an NVMe mechanism that allows the SSD controller to use a designated portion of host system memory for controller use.

It is commonly associated with **DRAM-less NVMe SSDs**.

```text
Host System RAM
      ↓
   HMB Region
      ↓
SSD Controller
      ↓
NAND
```

## DRAM vs HMB

### Dedicated DRAM

```text
SSD Controller
      ↓
Local SSD DRAM
      ↓
NAND
```

### HMB

```text
Host RAM
   ↓
HMB
   ↓
SSD Controller
   ↓
NAND
```

The HMB memory is physically located in the **host**, not inside the SSD.

## Important Clarification

HMB is **not**:

- NAND storage
- additional SSD capacity
- permanent storage

Its exact use is implementation-dependent.

An HMB-enabled SSD may use host memory for supported internal purposes such as mapping-related information or other controller working data, but it should not be assumed that the entire FTL mapping table always resides in HMB.

## Core Mental Model

```text
HMB
→ Host memory made available to the SSD controller
```

---

# 6. SSD Firmware

## Definition

**SSD firmware is specialized embedded software that runs on the SSD controller.**

It provides the control logic required to manage SSD operation.

```text
SSD Controller
      ↓
SSD Firmware
      ↓
SSD Internal Operations
```

## Functions

Firmware can implement or coordinate functions such as:

- Flash Translation Layer (FTL)
- NAND management
- Error handling
- Garbage Collection
- Wear Leveling
- TRIM processing
- Background management
- Power management
- Health/statistics handling

## Important Distinction

```text
Controller
= Hardware / control subsystem

Firmware
= Software / control logic running on the controller
```

The FTL, garbage collection and wear leveling should not be described as separate physical hardware components inside the controller.

---

# 7. SSD Interface

## Definition

The **SSD interface is the communication boundary through which the host communicates with the SSD.**

For our primary NVMe SSD focus:

```text
Host
  ↓
PCIe
  ↓
NVMe
  ↓
SSD Controller
```

## PCIe vs NVMe

```text
PCIe
= Interconnect / transport

NVMe
= Storage protocol / command model
```

NVMe defines the storage communication model and commands used by the host and controller.

PCIe provides the interconnect through which this communication is transported.

## Simple Example

```text
Host
 ↓
NVMe READ command
 ↓
PCIe
 ↓
NVMe SSD Controller
 ↓
Internal SSD processing
```

Do not define the interface as a separate component that simply "decodes the electrical stream." At the Module 3 architecture level, it is better understood as the **communication boundary/path between the host and SSD controller**.

---

# 8. Power Management

## Definition

**Power management controls SSD behavior across different power conditions and transitions while managing power consumption.**

At a high level:

```text
Active
  ↕
Lower-power / Idle condition
```

Power management can affect:

- Device accessibility
- Latency
- Performance
- Temperature
- Recovery behavior

## Important Distinction

Power management is not the same as a complete power cycle.

```text
Power Management
→ Changes operating power conditions

Power Cycle
→ Power OFF → Power ON
```

Detailed PCIe and NVMe power-management mechanisms are covered later in their dedicated modules.

---

# 9. Telemetry

## Definition

**SSD telemetry provides diagnostic and operational information about the SSD's internal condition and behavior.**

Conceptually:

```text
SSD Internal State
       ↓
Telemetry / Diagnostic Information
       ↓
Host / Validation Engineer
       ↓
Analysis / Troubleshooting
```

## Why Telemetry Matters

Telemetry can provide evidence useful for understanding:

- Device condition
- Errors
- Operational behavior
- Diagnostic information
- Changes observed during workloads or failures

Telemetry is **evidence**, not automatically the root cause.

```text
Symptom
  ↓
Collect Evidence
  ↓
Telemetry / Logs / Health
  ↓
Correlation
  ↓
RCA
```

Health information and telemetry are related but should not be treated as exact synonyms.

---

# 10. Logical vs Physical Storage

The host works with a **logical storage view**.

For example:

```text
WRITE
LBA = 1000
```

The host does not specify:

```text
Channel
Die
Plane
Block
Page
```

Those are internal SSD physical storage concepts.

Therefore:

```text
Host View
---------
Logical Block Address

        ↓

SSD Internal View
-----------------
Physical NAND Location
```

The SSD internally manages the relationship between these two views.

---

# 11. Address Translation

The basic relationship is:

```text
Logical Address
      ↓
Address Translation
      ↓
Physical NAND Location
```

For example:

```text
LBA 1000
   ↓
Internal mapping
   ↓
Physical NAND location
```

The host remains unaware of the physical location.

The logical address may remain the same even when the underlying physical location changes.

The detailed implementation of this mapping is covered in the dedicated **FTL module**.

---

# 12. Metadata

The SSD needs internal information to manage its storage system.

Conceptually:

```text
Logical information
       ↕
Mapping / Metadata
       ↕
Physical NAND information
```

Metadata can include information required for:

- Address translation
- Storage management
- Internal state tracking
- NAND management

DRAM or HMB may be used to support access to some internal information depending on the SSD architecture and implementation.

---

# 13. High-Level Host Write Flow

The Module 3 mastery requirement is to understand how a logical host write eventually becomes NAND programming.

Example:

```text
Host
 ↓
WRITE to LBA
 ↓
NVMe command
 ↓
PCIe
 ↓
SSD Controller
 ↓
Firmware / Internal Control Logic
 ↓
Logical → Physical Translation
 ↓
Physical NAND Destination
 ↓
NAND Program Operation
 ↓
Metadata / Mapping Update
 ↓
Completion
```

At this stage, this is a **high-level architectural flow**.

The detailed responsibilities of the FTL, NAND allocation, mapping updates, garbage collection, wear leveling and write amplification are studied separately in later modules.

---

# 14. Host-Level WRITE vs NAND-Level PROGRAM

These terms belong to different layers.

```text
Host / NVMe Layer
→ WRITE command

NAND Layer
→ PROGRAM operation
```

Therefore:

```text
Host WRITE
    ↓
SSD internal processing
    ↓
NAND PROGRAM
```

This distinction should remain clear throughout the project.

---

# 15. Complete Module 3 Mental Model

```text
                         HOST
                           │
                       NVMe / PCIe
                           │
                           ↓
                  SSD CONTROLLER
                           │
                        Firmware
                           │
                Logical → Physical
                    Translation
                           │
                           ↓
                         NAND
                           │
          ┌────────────────┴────────────────┐
          ↓                                 ↓
   Physical NAND                      Persistent Data
   Organization
          │
      Channel
          ↓
         Die
          ↓
        Plane
          ↓
        Block
          ↓
        Page
```

Supporting components:

```text
DRAM
→ Fast volatile working memory

HMB
→ Host memory available to the controller

Power Management
→ Controls power-related behavior

Telemetry
→ Provides operational/diagnostic visibility
```

---

# 16. Key Concepts to Remember

### SSD Controller

> Central hardware/control subsystem that processes host commands and coordinates internal SSD operations.

### Firmware

> Specialized software running on the SSD controller that provides control logic for SSD operations.

### NAND

> Non-volatile physical storage medium where persistent data is stored.

### DRAM

> Fast volatile working memory used in many SSDs for internal information and management.

### HMB

> NVMe mechanism that allows the controller to use designated host system memory for internal purposes.

### Interface

> Communication boundary between the host and SSD.

### PCIe

> Interconnect/transport used by an NVMe SSD.

### NVMe

> Storage protocol/command model used for communication with an NVMe controller.

### Power Management

> Controls SSD behavior across different power conditions and transitions.

### Telemetry

> Diagnostic and operational information that provides visibility into SSD condition and behavior.

---

# 17. Final Engineering Mental Model

```text
Host
 ↓
NVMe / PCIe
 ↓
SSD Controller
 ↓
Firmware
 ↓
Internal SSD Management
 ↓
Logical → Physical Translation
 ↓
NAND
 ↓
Persistent Data
```

The essential principle is:

> **The host sees a logical storage device, while the SSD controller and firmware hide and manage the complexity of the physical NAND storage system.**
