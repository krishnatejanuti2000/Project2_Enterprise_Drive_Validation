# MODULE 02 — HDD FUNDAMENTALS FOR VALIDATION

## Priority: P1

HDD is part of enterprise-drive qualification knowledge, but it is **not the primary technical focus** of this track.

The objective is to build enough HDD understanding to **validate, troubleshoot, and discuss HDD behavior confidently** without spending the same depth of study used for NVMe SSDs.

---

# 1. Mechanical

## 1.1 Platter

The platter is the magnetic storage medium on which data is recorded.

```text
HDD
 ↓
Platter
 ↓
Magnetic recording
```

The platter rotates so that different physical locations can pass beneath the read/write head.

---

## 1.2 Spindle

The spindle rotates the platters at a controlled rotational speed.

```text
Spindle
   ↓
Rotates platter
   ↓
Platter continuously moves beneath the head
```

Stable rotation is important because the required sector must rotate underneath the head during an access.

Abnormal spindle behavior can affect HDD access latency and operation.

---

## 1.3 Actuator

The actuator moves the read/write head assembly across the platter surface.

```text
Actuator
   ↓
Moves head assembly
   ↓
Positions head over target track
```

The time required to move the head from its current position toward the required track is **seek time**.

```text
Current Track
      ↓
Actuator Movement
      ↓
Target Track
```

---

## 1.4 Head

The read/write head performs the physical interaction with the magnetic recording medium.

### Read

```text
Required track
      ↓
Required sector rotates under head
      ↓
Read head senses magnetic information
      ↓
Electrical signal
      ↓
Signal processing / decoding
      ↓
Recovered data
```

### Write

```text
Required track
      ↓
Required sector rotates under write head
      ↓
Write head generates magnetic field
      ↓
Magnetic recording is changed
      ↓
Data is recorded
```

The head therefore provides the physical read/write interaction with the magnetic media.

---

## 1.5 Track

A track is a circular path on the platter surface.

```text
Platter surface
   ↓
Tracks
   ↓
Sectors
```

Tracks are part of the physical organization of the magnetic media.

---

## 1.6 Sector

A sector is a defined/addressable storage unit associated with a track.

```text
Track
 ├── Sector
 ├── Sector
 ├── Sector
 └── Sector
```

Sector size depends on the drive's sector format; do not assume one universal size for every HDD.

---

## 1.7 Cylinder

A cylinder is the collection of corresponding tracks at the same radial position across multiple platter surfaces.

Example:

```text
Surface 1 → Track 15
Surface 2 → Track 15
Surface 3 → Track 15
Surface 4 → Track 15
       ↓
   Same radial position
       ↓
     Cylinder
```

The important concept is:

> **Cylinder represents corresponding track positions across platter surfaces.**

---

## 1.8 Servo

Servo information provides positioning information used by the drive's positioning system to accurately locate and maintain the read/write head over the required track.

```text
Servo Information
       ↓
Position Feedback
       ↓
Actuator Control
       ↓
Head Positioning
       ↓
Target Track
```

The important relationship is:

> **Servo → positioning feedback → actuator → head → correct track access**

---

# 2. Mechanical Access Mental Model

The mechanical components work together:

```text
Host Request
     ↓
Controller determines required location
     ↓
Actuator positions head
     ↓
Spindle rotates platter
     ↓
Required sector reaches head
     ↓
Head performs read/write
```

Two major mechanical contributors to HDD latency are:

```text
Seek Time
→ time associated with head movement to the target track

Rotational Latency
→ time waiting for the required sector to rotate under the head
```

These become important when analyzing HDD performance.

---

# 3. Electronic

## 3.1 Controller

The HDD controller is the central electronic control point between the host-facing interface and the internal HDD mechanisms.

```text
Host
  ↓
Interface
  ↓
HDD Controller
  ↓
Internal HDD Operations
```

The controller receives and processes host requests and coordinates the internal mechanisms required to perform operations such as read and write.

Firmware controls and guides the behavior of the drive.

---

## 3.2 Cache

HDD cache is temporary memory used by the controller for buffering and caching data.

For a read request, the controller may check whether the requested data is already available in cache.

```text
Read Request
     ↓
Controller
     ↓
Cache Lookup
   ↙       ↘
Hit       Miss
 ↓          ↓
Return     Access
Data       Mechanical Media
```

### Cache Hit

The requested data is available in cache.

Mechanical media access may be avoided, reducing latency for that request.

### Cache Miss

The requested data is not available in cache.

The drive must continue toward physical media access.

Cache can also be used for buffering and read-ahead/write handling depending on the drive implementation.

---

## 3.3 Interface

The interface provides the communication boundary between the host and the HDD electronics.

```text
Host
 ↓
HDD Interface
 ↓
HDD Controller
 ↓
Internal HDD
```

The interface carries commands, data, and status between the host and the drive.

For a SATA HDD, the storage path includes the SATA interface/controller/link.

---

# 4. Logical

## 4.1 LBA — Logical Block Addressing

The host works with **logical block addresses** rather than directly specifying the physical platter geometry.

```text
Host
 ↓
LBA
 ↓
HDD Controller
 ↓
Internal handling
 ↓
Physical media location
```

The host does not normally specify:

```text
Platter
Surface
Track
Sector
```

Instead, it requests a logical block.

Example:

```text
Read LBA 5000
```

means:

> Retrieve the data associated with logical block 5000.

---

## 4.2 Logical-to-Physical Relationship

The host sees a logical address space:

```text
LBA 0
LBA 1
LBA 2
LBA 3
...
```

The HDD internally works with physical media organization:

```text
Platter
 ↓
Surface
 ↓
Track
 ↓
Sector
```

The drive's controller/firmware handles the relationship between the logical request and the physical media location.

```text
Host
 ↓
LBA
 ↓
HDD Controller / Firmware
 ↓
Logical-to-Physical Translation
 ↓
Physical Location
 ↓
Track / Sector / Surface
```

The important distinction is:

> **LBA is a logical address; it should not be treated as the physical track or sector number.**

---

# 5. HDD Read Flow

The complete HDD read flow is a software + hardware journey.

The detailed flow covered separately in the read-flow document is:

```text
Application
 ↓
read() system call
 ↓
CPU / Kernel transition
 ↓
Linux Kernel
 ↓
VFS
 ↓
Page Cache Lookup
 ↓
File System
 ↓
File Metadata / Inode
 ↓
Logical File Blocks
 ↓
LBA
 ↓
BIO
 ↓
Linux Block Layer
 ↓
I/O Scheduler
 ↓
Request Queue
 ↓
SATA Driver
 ↓
ATA READ Command
 ↓
SATA Host Controller
 ↓
SATA Frames
 ↓
SATA Link
 ↓
HDD SATA Interface
 ↓
HDD Controller
 ↓
HDD Firmware
 ↓
LBA Validation
 ↓
Logical → Physical Translation
 ↓
HDD Cache Lookup
 ↓
Servo / Actuator Positioning
 ↓
Rotational Latency
 ↓
Sector Detection
 ↓
Read Head
 ↓
Signal Processing / Decoding
 ↓
ECC Verification
 ↓
Error Correction / Retry if required
 ↓
Data Assembly
 ↓
HDD Buffer
 ↓
SATA Data Transfer
 ↓
Host SATA Controller
 ↓
DMA → System RAM
 ↓
Linux Kernel
 ↓
Page Cache
 ↓
Application Buffer
 ↓
read() completes
 ↓
Application receives data
```

The complete read lifecycle, including Linux layers, SATA path, HDD firmware, mechanical access, ECC/retry handling, DMA, and the return path, is covered in the separate detailed read-flow document.     

### Core Read Mental Model

```text
LBA
 ↓
Controller determines physical location
 ↓
Actuator positions head
 ↓
Spindle rotates platter
 ↓
Required sector reaches head
 ↓
Magnetic information is read
 ↓
Data is processed / verified
 ↓
Data returns to host
```

---

# 6. HDD Write Flow

The write flow follows the same overall host-to-device path, but the media operation is different.

```text
Application
 ↓
Linux / File System
 ↓
Block Layer
 ↓
SATA Driver
 ↓
SATA Controller / Link
 ↓
HDD Interface
 ↓
HDD Controller
 ↓
HDD Firmware
 ↓
LBA Validation
 ↓
Logical → Physical Translation
 ↓
Physical Location
 ↓
Servo / Actuator Positioning
 ↓
Required Sector
 ↓
Write Head
 ↓
Magnetic Recording
 ↓
Data Completion
 ↓
SATA Response
 ↓
Host
```

### Core Write Mental Model

```text
LBA
 ↓
Controller determines where the logical data belongs
 ↓
Actuator positions head
 ↓
Spindle rotates platter
 ↓
Required sector reaches head
 ↓
Write head changes magnetic recording
 ↓
Controller completes operation
 ↓
Completion returns to host
```

### Read vs Write

```text
READ
→ sense/recover existing magnetic information

WRITE
→ create/change magnetic recording on the media
```

The existing detailed flow documents should be used for the complete step-by-step read/write lifecycle.

---

# 7. Validation

## 7.1 Detection

### Question

> **Has the system detected the newly installed HDD through the expected storage path?**

Detection depends on the drive interface.

For a SATA HDD, useful Linux evidence can include:

```bash
lsscsi
lsblk
```

For controller/interface visibility:

```bash
lspci
```

For kernel discovery/error evidence:

```bash
dmesg
journalctl -k
```

The commands are selected according to the layer being validated.

```text
Detection
 ↓
Which interface/path?
 ↓
Which layer should see the drive?
 ↓
Collect evidence
 ↓
Drive detected?
```

Do not automatically assume `lspci` is the command that identifies the individual SATA HDD. It may show the host SATA controller/interface rather than the individual drive.

---

## 7.2 Capacity

### Question

> **What capacity does the HDD expose, and does it match the expected capacity?**

Useful commands:

```bash
lsblk
lsblk -o NAME,SIZE,TYPE,MODEL,SERIAL
sudo fdisk -l
```

The basic validation model is:

```text
Expected Capacity
       ↓
Observed Capacity
       ↓
Compare
       ↓
Match / Mismatch
```

Capacity validation should distinguish the device-reported/OS-visible capacity from the amount of storage ultimately usable at higher layers.

---

## 7.3 Health

### Question

> **What condition is the HDD currently reporting?**

Health is the broader validation concern.

We look for evidence such as:

```text
Operating condition
Error condition
Degradation indicators
Temperature
Usage / lifetime information
```

Health should be treated as a **current condition/baseline**, not as a guarantee that the drive can never fail.

```text
Initial Health
     ↓
Workloads
     ↓
Later Health
     ↓
Compare
```

---

## 7.4 SMART

### Meaning

SMART provides device-reported monitoring and condition information used to assess HDD health and behavior.

```text
HDD Internal Information
        ↓
SMART Attributes / Status
        ↓
Validation Engineer
        ↓
Health Assessment
```

A common Linux command for a SATA HDD is:

```bash
sudo smartctl -a /dev/sdX
```

For basic device information:

```bash
sudo smartctl -i /dev/sdX
```

The exact attributes vary by drive.

### Engineering principle

A non-zero SMART attribute is **not automatically a failure**.

Use:

```text
SMART Observation
      ↓
Understand Attribute
      ↓
Check Threshold / Requirement
      ↓
Supporting Evidence
      ↓
Assessment
```

---

## 7.5 Interface

### Question

> **Is the HDD using the expected interface, and is the communication path functioning correctly?**

For a SATA HDD, think:

```text
Host
 ↓
SATA Controller
 ↓
SATA Link
 ↓
HDD SATA Interface
 ↓
HDD Controller
```

Useful evidence can include:

```bash
lspci
lsscsi
lsblk
dmesg
journalctl -k
```

Interface issues can affect:

```text
Detection
Command communication
Data transfer
Stability
Performance
```

Do not automatically blame the HDD media when the communication path itself may be the source of the problem.

---

## 7.6 Firmware

### Question

> **What firmware is installed on the HDD, and is it the expected firmware for the test configuration?**

Validation model:

```text
Expected Firmware
       ↓
Observed Firmware
       ↓
Compare
       ↓
Result
```

Useful evidence for a SATA HDD may include:

```bash
sudo smartctl -i /dev/sdX
```

where supported.

Record firmware together with:

```text
Drive Model
Serial Number
Platform
OS
Interface
```

### Important distinction

```text
Firmware observed
        ≠
Firmware accepted
```

The installed revision must be compared with the applicable qualification baseline.

---

## 7.7 Performance

### Question

> **How does the HDD perform under a defined workload and test condition?**

Important variables:

```text
Read / Write
Sequential / Random
Block Size
Queue Depth
Concurrency
Duration
Test Region
```

Typical measurements:

```text
Throughput
IOPS
Latency
```

A common Linux workload tool is:

```bash
fio
```

HDD performance is strongly influenced by:

```text
Seek Time
Rotational Latency
RPM
Workload Pattern
Queue Depth
Block Size
Cache Behavior
Interface
Drive Condition
Background Activity
```

Engineering method:

```text
Define Test Condition
 ↓
Baseline
 ↓
Run Workload
 ↓
Measure
 ↓
Repeat
 ↓
Compare
 ↓
Investigate Anomaly
```

A performance result without workload configuration is incomplete evidence.

---

## 7.8 Failure Behavior

### Question

> **How does the HDD and the host system behave when a failure condition occurs?**

The focus is not merely:

> “Did the HDD fail?”

The validation engineer wants to know:

```text
Failure Condition
      ↓
Drive Response
      ↓
Host / OS Response
      ↓
Logs / SMART / Device Evidence
      ↓
Recovery or Continued Failure
```

Possible investigation areas include:

```text
Media
Mechanical system
Controller
Interface
Communication path
```

Useful evidence may include:

```bash
dmesg
journalctl -k
smartctl -a /dev/sdX
lsblk
lsscsi
```

The first objective is to understand **what actually happened** before assigning a root cause.

---

## 7.9 Replacement

### Question

> **After a failed HDD is replaced, does the replacement drive become correctly recognized and usable?**

Basic flow:

```text
HDD Failure
 ↓
Identify Failed Drive
 ↓
Remove Failed Drive
 ↓
Install Replacement
 ↓
Detect Replacement
 ↓
Identify Replacement
 ↓
Check Capacity / Health / SMART / Interface / Firmware
 ↓
Restore Required Configuration
 ↓
Functional Validation
 ↓
Replacement Accepted
```

Useful evidence may include:

```bash
lsscsi
lsblk
lspci
dmesg
journalctl -k
sudo smartctl -i /dev/sdX
sudo smartctl -a /dev/sdX
```

The exact replacement and rebuild process depends on the surrounding storage architecture.

---

# 8. HDD Validation Mental Model

The entire module can be connected as:

```text
HDD Physical Structure
        ↓
Mechanical Behavior
        ↓
Electronic Control
        ↓
Logical Addressing
        ↓
Host Read / Write
        ↓
Validation
```

Or, from the validation perspective:

```text
Detect
 ↓
Check Capacity
 ↓
Establish Health / SMART Baseline
 ↓
Verify Interface
 ↓
Verify Firmware
 ↓
Characterize Performance
 ↓
Validate Failure Behavior
 ↓
Validate Replacement
```

At every point:

```text
Expected State
      ↓
Observed State
      ↓
Evidence
      ↓
Compare
      ↓
Pass / Investigate
```

---

# 9. Core Engineering Principles

### 1. Detection is not validation

```text
Drive detected
        ≠
Drive qualified
```

### 2. Identification is not health

```text
Known Drive
        ≠
Healthy Drive
```

### 3. SMART value is not automatically a failure

```text
Observed SMART Attribute
        ↓
Understand
        ↓
Compare with requirement
        ↓
Assess
```

### 4. Performance must be tied to workload

```text
Performance Result
+
Workload Configuration
=
Meaningful Performance Evidence
```

### 5. Failure observation is not root cause

```text
Symptom
 ↓
Evidence
 ↓
Hypothesis
 ↓
Isolation
 ↓
Root Cause
```

### 6. Replacement re-enters validation

A replacement drive must be re-established as a correctly identified and functioning device before it is accepted.

---

**Module 2 — HDD Fundamentals for Validation is complete at the required P1 working depth.**
