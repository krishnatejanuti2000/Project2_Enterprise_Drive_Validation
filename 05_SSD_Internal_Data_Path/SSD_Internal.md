# MODULE 05 — SSD INTERNAL DATA PATH


---

## 1. Objective

Understand how a host storage request enters an NVMe SSD, passes through the SSD Controller and its internal management mechanisms, reaches NAND when required, and completes back toward the host.

This module establishes the relationship between:

- Host requests and NVMe commands
- SSD Controller and firmware
- Logical-to-physical address translation
- NAND operations and scheduling
- Metadata and error handling
- Garbage Collection and Wear Leveling

---

## 2. SSD Internal Data Path — Overview

The SSD internal data path describes how the SSD receives, processes and services storage requests using its internal hardware and firmware mechanisms.

The host works with logical storage addresses. The SSD internally manages physical NAND locations.

```text
Host Request
     |
     v
NVMe Command
     |
     v
SSD Controller
     |
     v
Internal SSD Processing
     |
     v
FTL / Address Translation
     |
     v
NAND Allocation and Scheduling
     |
     v
NAND Operation
     |
     v
Metadata Management
     |
     v
Request Completion
```

This is a simplified conceptual path. Actual internal execution order and parallelism depend on the SSD implementation.

---

## 3. SSD Controller

The **SSD Controller** is the central hardware/control subsystem that processes host requests and coordinates internal SSD operations.

### 3.1 Main Responsibilities

| Responsibility | Purpose |
|---|---|
| Command processing | Receives and processes storage requests |
| Data movement | Coordinates data transfers between relevant host, controller and NAND resources |
| Address translation | Uses the FTL to resolve logical addresses to physical NAND locations |
| NAND scheduling | Coordinates NAND operations and available resources |
| Metadata management | Maintains internal information required to manage data and NAND state |
| ECC | Supports detection and correction of recoverable NAND data errors |
| Error handling | Handles and reports unsuccessful operations and errors |
| Garbage Collection | Coordinates the reclamation of reusable NAND blocks |
| Wear Leveling | Applies policies to distribute NAND wear |
| Firmware execution | Uses embedded control logic to manage SSD operations |

These responsibilities work together. They are not necessarily separate sequential steps performed for every request.

### 3.2 Controller vs Firmware

**SSD Controller:** Hardware/control subsystem responsible for processing and coordinating operations.

**SSD Firmware:** Embedded software that provides control logic and policies for the SSD's internal behavior.

Conceptually:

```text
SSD Controller
      +
SSD Firmware
      +
Internal Hardware / Management Mechanisms
      |
      v
SSD Operation
```

The exact division of work between firmware and dedicated hardware engines varies by SSD design.

---

## 4. Command Processing

The host submits a storage request through the NVMe interface.

For example, the host requests a write to LBA 5000.

```text
Host
 |
 | WRITE Request
 v
NVMe Command
 |
 v
SSD Controller
 |
 v
Request Processing
```

Command processing is the handling of the storage request. It is not the same as programming data into NAND.

---

## 5. Data Movement

The SSD must transfer the data associated with host requests between the relevant resources.

### Write Path

```text
Host Data
    |
    v
SSD Controller
    |
    v
Internal Data Movement
    |
    v
NAND Programming
```

### Read Path

```text
NAND Read
    |
    v
ECC Processing
    |
    v
Controller Data Path
    |
    v
Host Receives Data
```

The exact buffering, transfer and internal processing mechanisms depend on the SSD implementation.

---

## 6. Address Translation

The host identifies storage using logical addresses such as Logical Block Addresses (LBAs).

NAND uses physical structures such as channels, dies, planes, blocks and pages.

The SSD must resolve the logical address to the current physical NAND location.

```text
Host LBA
   |
   v
SSD Controller
   |
   v
Flash Translation Layer (FTL)
   |
   v
Physical NAND Location
```

An illustrative mapping is:

```text
LBA 5000
   |
   v
Block 20, Page 8
```

The example is conceptual and does not represent a physical mapping from a particular SSD.

Detailed logical-to-physical mapping is covered in **Module 6 — FTL**.

---

## 7. NAND Scheduling

NAND scheduling is the coordination of operations across available NAND resources.

```text
SSD Controller
      |
      +---- Channel 0
      |
      +---- Channel 1
      |
      +---- Channel 2
```

Multiple channels and dies can provide opportunities for internal parallelism.

The controller coordinates operations according to available resources, workload and firmware policies. Exact scheduling behavior is implementation-specific.

NAND scheduling can influence resource utilization, latency and throughput.

---

## 8. Metadata Management

**Metadata** is internal management information used by the SSD to track data placement and NAND state.

It is distinct from user data.

Examples include:

| Metadata | Purpose |
|---|---|
| Mapping information | Tracks logical-to-physical data placement |
| Free-block information | Identifies blocks available for reuse |
| Bad-block information | Identifies blocks excluded from normal allocation |
| Erase-count information | Helps track NAND block wear |
| Valid/invalid page information | Helps identify current data and obsolete physical pages |

Metadata supports FTL mapping, block allocation, Garbage Collection and Wear Leveling.

The exact representation and storage of metadata depend on the SSD implementation.

---

## 9. ECC and Error Handling

NAND reads can encounter bit errors. ECC helps detect and correct errors within its correction capability.

### Read Processing

```text
NAND Page Read
      |
      v
ECC Verification
      |
      +---- No Error / Correctable Error
      |                 |
      |                 v
      |             Data Accepted
      |
      +---- Uncorrectable Error
                        |
                        v
                 Error Handling
                        |
                        v
                Failure Reported
```

ECC and error handling are related but distinct.

- **ECC:** Attempts to detect and correct recoverable data errors.
- **Error handling:** Covers the broader response to failed or unsuccessful operations.

An unrecoverable error may cause the SSD to report an unsuccessful read. The precise behavior depends on the controller, firmware and applicable command behavior.

---

## 10. Garbage Collection and Wear Leveling

These are important internal SSD-management mechanisms.

### Garbage Collection (GC)

Garbage Collection reclaims NAND blocks containing invalid pages.

Conceptually:

```text
Select Candidate Block
        |
        v
Preserve Valid Pages
        |
        v
Update Affected Mappings
        |
        v
Erase Original Block
        |
        v
Return Block for Reuse
```

GC supports the availability of reusable NAND blocks.

### Wear Leveling

Wear Leveling distributes NAND wear across usable blocks according to firmware policies.

It helps prevent a small group of blocks from wearing out disproportionately early.

```text
Track Wear
    |
    v
Evaluate Wear Distribution
    |
    v
Apply Wear Leveling Policy
    |
    v
Manage Block Usage
```

GC and Wear Leveling work together but solve different problems. Their detailed behavior is covered in **Module 7**.

---

## 11. SSD Firmware

Firmware provides embedded control logic and policies for the SSD's internal operations.

Depending on the design, these include:

- FTL and address mapping
- NAND management
- Metadata updates
- Garbage Collection
- Wear Leveling
- ECC coordination and error handling
- Power management

The controller, firmware and internal hardware mechanisms cooperate to service host requests.

Understanding firmware architecture does not itself constitute hands-on firmware validation. Firmware update, activation, recovery and regression testing are separate validation activities.

---

## 12. Integrated Write Path

The following is the core write path defined in the Project 2 Master Syllabus.

```text
Host Write
     |
     v
NVMe Command
     |
     v
SSD Controller
     |
     v
FTL
     |
     v
NAND Allocation
     |
     v
NAND Program
     |
     v
Metadata Update
     |
     v
Completion
```

### Stage Summary

1. **Host Write:** The host requests that data be written to logical storage addresses.
2. **NVMe Command:** The request is represented by an NVMe command.
3. **SSD Controller:** The controller processes and coordinates the request.
4. **FTL:** The logical data is resolved to the appropriate physical NAND placement.
5. **NAND Allocation:** Suitable physical storage resources are selected.
6. **NAND Program:** Data is programmed into NAND.
7. **Metadata Update:** Internal mapping and management information is updated as required.
8. **Completion:** The request completes according to the device's command and implementation behavior.

This is a simplified architectural flow, not a guarantee of an identical internal timeline across all SSDs.

---

## 13. High-Level Read Path

```text
Host Read Request
       |
       v
SSD Controller
       |
       v
FTL Mapping Lookup
       |
       v
NAND Read, If Required
       |
       v
ECC Processing
       |
       v
Data Returned to Host
```

Reads and writes share controller resources but involve different internal operations. A read may be serviced without a NAND access when applicable cached data is available.

---

## 14. Performance and Reliability Relationships

The SSD internal data path connects several aspects of drive behavior.

| Mechanism | Potential relationship |
|---|---|
| Address translation | Mapping work can contribute to request processing overhead |
| NAND scheduling | Affects resource utilization and parallelism |
| Metadata management | Maintains the information needed for correct data placement and reuse |
| ECC | Supports data integrity when errors are within its correction capability |
| Garbage Collection | Reclaims blocks but may introduce additional work and latency |
| Wear Leveling | Helps distribute NAND wear but may involve data movement |
| Firmware | Coordinates internal policies and responses |

These mechanisms can influence latency, throughput, write amplification, endurance and sustained performance.

---

## 15. Module 5 — Key Takeaways

- The SSD Controller coordinates the processing of host storage requests.
- Firmware provides control logic and policies for internal SSD operations.
- The host uses logical addresses; the SSD manages physical NAND locations.
- The FTL handles logical-to-physical mapping.
- NAND scheduling coordinates operations across available NAND resources.
- Metadata tracks data placement and internal NAND state.
- ECC corrects recoverable errors; error handling manages broader operation failures.
- Garbage Collection reclaims blocks for reuse.
- Wear Leveling distributes NAND wear.
- Actual internal behavior and hardware/software boundaries vary by SSD implementation.

### Core Mental Model

```text
Host Request
     |
     v
NVMe Command
     |
     v
SSD Controller + Firmware
     |
     +---- Command Processing
     +---- Data Movement
     +---- FTL / Address Translation
     +---- NAND Scheduling
     +---- Metadata Management
     +---- ECC / Error Handling
     +---- Garbage Collection
     +---- Wear Leveling
     |
     v
NAND Operations
     |
     v
Request Completion or Error
```

---

