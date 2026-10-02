# MODULE 01 — DRIVE QUALIFICATION ENGINEERING

## 1. Objective

Drive Qualification Engineering is the process of taking a **new enterprise storage drive**, with **Enterprise NVMe SSD as the primary focus**, and systematically determining whether the drive satisfies the requirements for qualification.

Qualification is not one test.

The drive must progress through a defined lifecycle, and each stage must establish a specific part of the overall qualification evidence.

The engineering mindset is:

```text
Expected State
      ↓
Observe Actual State
      ↓
Collect Evidence
      ↓
Compare
      ↓
Find First Divergence
      ↓
Investigate
      ↓
Root Cause
      ↓
Recovery
      ↓
Revalidate
```

A command is only a tool for collecting evidence.  
The **validation question comes first**.

---

# 2. Qualification Lifecycle

```text
New Drive
 ↓
Enumeration
 ↓
Identification
 ↓
Firmware Check
 ↓
Health Check
 ↓
Compatibility
 ↓
Functional Tests
 ↓
Performance
 ↓
Stress
 ↓
Endurance
 ↓
Fault Injection
 ↓
Recovery
 ↓
Data Integrity
 ↓
Regression
 ↓
Qualification
```

Each stage answers a different question.

---

# 3. Qualification Dimensions

The final qualification is concerned with:

```text
Functionality
Compatibility
Reliability
Firmware
Health
Performance
Endurance
Recovery
Integrity
```

### Lifecycle vs Dimensions

These are related but different concepts.

**Lifecycle** = the process followed to qualify the drive.

**Qualification dimensions** = the qualities/aspects that must ultimately be established about the drive.

For example, **Reliability** is not a separate lifecycle stage. Reliability evidence is built through areas such as stress, endurance, fault injection, recovery, and integrity validation.

---

# 4. Stage 1 — New Drive

## What is it?

A new enterprise drive has arrived and becomes the **device under qualification**.

## Main question

> What drive has arrived, and what validation environment will be used?

Before performing destructive or workload-based testing, the device and the intended test environment must be clearly established.

The drive should eventually be associated with information such as:

```text
Model
Serial Number
Firmware
Platform
BIOS
OS
Kernel
Driver
PCIe configuration
```

The purpose is to ensure that later test results can be tied to the exact device and environment.

---

# 5. Stage 2 — Enumeration

## What is it?

Enumeration establishes that the newly installed NVMe SSD can successfully progress from **PCIe hardware discovery toward Linux storage visibility**.

## Fixed Engineering Flow

This is the working flow used throughout this project:

```text
PCIe device discovered
        ↓
PCIe configuration
        ↓
NVMe device identified
        ↓
NVMe driver binds
        ↓
Driver initializes NVMe controller
        ↓
Controller becomes usable
        ↓
Namespaces are discovered/registered
        ↓
Linux block device is created
```

This is the stable mental model for Module 1. The Master Syllabus describes the underlying discovery/initialization sequence as Power On → PCIe Discovery → Enumeration → Configuration → BAR assignment → Driver Binding → NVMe Initialization.

## Main question

> **How far has the newly installed SSD successfully progressed through the system?**

The goal is not simply to ask:

> "Is the SSD detected?"

The goal is to determine **where the drive successfully entered the system and where it may have stopped**.

## Evidence

Typical commands:

```bash
lspci
```

Used to inspect the PCIe device inventory.

```bash
lspci -vv -s <PCI_ADDRESS>
```

Used to inspect detailed PCIe configuration, resources, capabilities, and link state.

```bash
lspci -k -s <PCI_ADDRESS>
```

Used to identify the kernel driver associated with the device.

```bash
nvme list
```

Used to inspect NVMe device/namespace visibility from the Linux NVMe stack.

```bash
dmesg | grep -i nvme
journalctl -k | grep -i nvme
```

Used to investigate kernel/NVMe initialization evidence when something does not progress normally.

## What the evidence means

For example:

```text
PCIe visible
        ↓
lspci evidence

Driver bound
        ↓
Kernel driver in use: nvme

NVMe storage visible
        ↓
nvme list

Linux block device visible
        ↓
/dev/nvme0n1
```

## First-Divergence Troubleshooting

Never jump from symptom to root cause.

Example:

```text
PCIe ✓
NVMe Controller ✓
Namespace ✗
Block Device ✗
```

The first known divergence is:

```text
NVMe Controller
        ↓
Namespace discovery/registration
        ✗
```

Investigation should begin at this boundary.

Do not immediately investigate the filesystem or application because those are downstream.

### Core rule

> **Do not troubleshoot the entire system at once. Find the first divergence.**

The Master Syllabus explicitly distinguishes a device missing from `lspci` from a device visible in PCIe but missing from the NVMe path.

---

# 6. Stage 3 — Identification

## What is it?

Identification determines **exactly which drive/controller has been discovered**.

## Main question

> **What device am I actually validating?**

Important identity information includes:

```text
Vendor
Model
Serial Number
Controller ID
Namespace information
Firmware Revision
NVMe version
```

## Primary command

```bash
sudo nvme id-ctrl /dev/nvme0
```

This retrieves the NVMe controller's Identify information.

Important fields for initial qualification:

```text
vid
ssvid
sn
mn
fr
cntlid
ver
nn
```

### Meaning

```text
Model
↓
What product is this?

Serial Number
↓
Which individual physical drive is this?

Controller ID
↓
Which NVMe controller is being examined?

Firmware Revision
↓
Which firmware is currently installed?
```

### Controller vs Namespace

Do not confuse:

```text
NVMe Controller
```

with:

```text
NVMe Namespace
```

or:

```text
/dev/nvme0n1
```

Conceptually:

```text
/dev/nvme0
    ↓
NVMe Controller
    ↓
Namespace
    ↓
/dev/nvme0n1
```

Namespace-specific information can be inspected separately with:

```bash
sudo nvme id-ns /dev/nvme0n1
```

## Engineering principle

Identification creates the **device baseline** that later firmware, health, compatibility, and validation results are associated with.

---

# 7. Stage 4 — Firmware Check

## What is it?

Firmware Check determines the firmware revision currently installed on the identified drive and compares it with the required qualification baseline.

## Main question

> **Is this drive running the firmware required for this qualification?**

The validation method is:

```text
Expected Firmware
        ↓
Observed Firmware
        ↓
Compare
        ↓
Result
```

The installed firmware can be observed through:

```bash
sudo nvme list
```

or:

```bash
sudo nvme id-ctrl /dev/nvme0
```

with the firmware field:

```text
fr
```

## Important distinction

```text
Firmware identified
        ≠
Firmware qualified
```

Example:

```text
Observed FW = 41002131
```

This is an observation.

To determine qualification status, the approved baseline must also be known.

### Possible outcomes

```text
Expected = Observed
        ↓
Baseline matched
```

```text
Expected ≠ Observed
        ↓
Firmware mismatch
        ↓
Investigate required action
```

```text
Expected = Unknown
        ↓
Firmware identified
but qualification result cannot yet be determined
```

Detailed firmware lifecycle activities such as upgrade, activation, downgrade, rollback, recovery, and post-update verification are handled as deeper firmware validation activities.

---

# 8. Stage 5 — Health Check

## What is it?

Health Check establishes the **current health state reported by the SSD** and creates a baseline for later comparison.

## Main question

> **What health condition is the drive reporting at the beginning of qualification?**

## Primary command

```bash
sudo nvme smart-log /dev/nvme0
```

Important information includes:

```text
Critical Warning
Available Spare
Available Spare Threshold
Percentage Used
Temperature
Media Errors
Error Log Entries
Data Units Read
Data Units Written
Power Cycles
Power-On Hours
Unsafe Shutdowns
Thermal Information
```

## Example baseline

A health record can look conceptually like:

```text
Critical Warning       : 0
Available Spare        : 100%
Spare Threshold        : 50%
Percentage Used        : 2%
Temperature            : 34 °C
Media Errors           : 0
Error Log Entries      : 0
Power Cycles           : 1201
Power-On Hours         : 4556
Unsafe Shutdowns       : 157
```

These values are **observations reported by the drive**.

They are not automatically a qualification PASS/FAIL.

### Important principle

```text
Observed Health Value
        ↓
Compare with Requirement / Threshold
        ↓
Determine Significance
```

A historical counter being non-zero does not automatically mean drive failure.

The health stage establishes the baseline against which later stress, endurance, fault, recovery, and integrity behavior can be compared.

---

# 9. Stage 6 — Compatibility

## What is it?

Compatibility validates the SSD together with the **specific platform and software configuration** in which it is being tested.

## Main question

> **Can this drive operate correctly in the intended platform and configuration?**

Important dimensions include:

```text
SSD
Platform
PCIe Generation
PCIe Width
BIOS
Firmware
Driver
Operating System
Controller
```

## Environment capture

Useful commands include:

```bash
sudo dmidecode -s system-product-name
sudo dmidecode -s bios-version
uname -r
cat /etc/os-release
```

PCIe and driver state can be captured with:

```bash
lspci
lspci -vv -s <PCI_ADDRESS>
lspci -k -s <PCI_ADDRESS>
```

## Engineering principle

Do not simply report:

> "The SSD is compatible."

Instead associate the result with the exact configuration:

```text
SSD
+
Platform
+
BIOS
+
OS
+
Kernel
+
Driver
+
Firmware
+
PCIe configuration
```

The detailed compatibility matrix is covered later in the dedicated compatibility module.

---

# 10. Stage 7 — Functional Tests

## What is it?

Functional testing verifies that the SSD performs its required storage operations **correctly**.

## Main question

> **Does the drive perform the required storage functions correctly?**

Typical functional areas include:

```text
Read
Write
Flush
Deallocation
Format
Namespace operations
Reset / recovery behavior
Firmware-related operations
Negative tests
```

## Functional validation model

```text
Expected Behavior
        ↓
Perform Operation
        ↓
Observe Result
        ↓
Compare
```

The focus is **correct behavior**, not speed.

Before any destructive functional testing, the target device must be verified as safe.

Useful inspection commands:

```bash
lsblk -o NAME,TYPE,SIZE,FSTYPE,MOUNTPOINTS,MODEL,SERIAL
findmnt
```

Only an isolated, intentionally selected test device should be used for destructive operations.

### Important distinction

```text
Command completed successfully
        ≠
Everything about the storage behavior has been proven correct
```

Data correctness is treated separately under Data Integrity.

The functional validation scope is defined in the Master Syllabus.

---

# 11. Stage 8 — Performance

## What is it?

Performance validation characterizes the SSD under **controlled workloads and configurations**.

## Main question

> **How does the drive perform under the defined workload?**

Typical workload dimensions:

```text
Sequential Read / Write
Random Read / Write
Mixed Workloads
Block Size
Queue Depth
Job Count
Test Duration
```

Important measurements:

```text
IOPS
Throughput
Bandwidth
Latency
Percentile Latency
CPU Utilization
Achieved Queue Depth
```

The primary workload tool is commonly:

```bash
fio
```

with a controlled job/configuration.

## Engineering method

```text
Baseline
 ↓
Controlled Configuration
 ↓
Workload
 ↓
Measurement
 ↓
Repeat
 ↓
Compare
 ↓
Detect Anomaly
 ↓
Investigate
```

A performance result is meaningful only when the test conditions are captured.

Possible investigation areas include:

```text
Queue Depth
Block Size
Workload Pattern
Firmware
Garbage Collection
NAND Behavior
PCIe Link
CPU
NUMA
Driver
OS
Temperature
Background Operations
```

The Master Syllabus defines this controlled methodology explicitly.

---

# 12. Stage 9 — Stress

## What is it?

Stress validation subjects the drive and surrounding system to demanding and repeated conditions.

## Main question

> **Does the drive remain stable and functional under demanding conditions?**

Typical conditions:

```text
Sustained I/O
High Queue Depth
Mixed Workloads
Concurrency
Repeated Operations
Reboot
Reset
Power Cycle
Hotplug
Recovery Events
```

Performance is not the only concern.

We watch for:

```text
Errors
Timeouts
Device Disappearance
Unexpected Resets
Health Changes
Performance Degradation
Recovery Problems
```

Stress therefore extends normal functional validation into **sustained/repeated operating conditions**.

---

# 13. Stage 10 — Endurance

## What is it?

Endurance evaluates behavior over **prolonged workload and accumulated wear**.

## Main question

> **Does the drive continue to behave reliably as workload and wear accumulate?**

Typical observations:

```text
Health Evolution
Wear
Performance Evolution
Data Integrity
Long-Duration Behavior
```

Engineering model:

```text
Initial Baseline
      ↓
Long-Term Workload
      ↓
Wear Accumulates
      ↓
Measure Health / Performance / Integrity
      ↓
Compare with Baseline
```

Stress and endurance are related but not identical:

```text
Stress
↓
Can it remain stable under demanding conditions?

Endurance
↓
Can it remain reliable over prolonged use and accumulated wear?
```

---

# 14. Stage 11 — Fault Injection

## What is it?

Fault Injection deliberately introduces a controlled failure condition.

## Main question

> **How does the drive and system behave when a controlled fault occurs?**

Examples:

```text
Hot Removal / Insertion
Controller Reset
PCIe Reset
Reboot
Power Cycle
Firmware Interruption
Repeated Reset
Workload During Failure
```

Evidence may include:

```text
Device State
OS Behavior
Controller Behavior
Error Logs
Recovery Behavior
Data State
Health State
Regression Impact
```

The objective is not merely to make the device fail.

The objective is to verify **failure-handling behavior**.

---

# 15. Stage 12 — Recovery

## What is it?

Recovery validates what happens **after a failure occurs**.

## Main question

> **Does the system return to the expected operational state after the fault?**

Conceptual recovery path:

```text
Failure
 ↓
Detection
 ↓
Error Handling
 ↓
Reset / Reinitialization
 ↓
Queue Recreation
 ↓
Re-enumeration
 ↓
Controller / OS Recovery
 ↓
Revalidation
```

Useful evidence during recovery analysis:

```bash
dmesg
journalctl -k
lspci
nvme list
```

The exact commands depend on the failure being investigated.

### Relationship with Fault Injection

```text
Fault Injection
↓
Create / reproduce controlled failure

Recovery
↓
Verify correct return to operation
```

The Master Syllabus includes timeout handling, reset, queue recreation, reinitialization, PCIe recovery, re-enumeration, and OS recovery.

---

# 16. Stage 13 — Data Integrity

## What is it?

Data Integrity validates that the **data itself remains correct**, not merely that commands complete.

## Main question

> **Is the data written to and read from the drive still correct?**

Basic model:

```text
Known Data
   ↓
Write
   ↓
Read Back
   ↓
Compare
   ↓
Correct?
```

Integrity should also be considered across:

```text
Normal I/O
Reset
Power Cycle
Firmware Update
Stress
Recovery
```

### Core principle

```text
Command success
      ≠
Data correctness
```

Evidence may use known data patterns, checksums, verification methods, and before/after comparisons.

The Master Syllabus explicitly makes this distinction.

---

# 17. Stage 14 — Regression

## What is it?

Regression verifies that previously validated behavior has not been broken by a change.

## Main question

> **Did the change introduce a regression?**

Typical changes:

```text
Firmware
Driver
OS / Kernel
Platform
Feature
Configuration
```

Engineering model:

```text
Known-Good Baseline
        ↓
Apply Change
        ↓
Run Relevant Tests
        ↓
Compare Results
        ↓
Investigate Difference
```

Useful engineering tools later include:

```text
Git
Pytest
Jenkins / CI
```

The detailed automation/CI implementation is handled later in the roadmap.

---

# 18. Stage 15 — Qualification

## What is it?

Qualification is the final determination made from the accumulated validation evidence.

The drive is not qualified because:

```text
lspci works
```

or because:

```text
nvme list works
```

or because:

```text
one FIO test looks good
```

Qualification requires evidence across the required dimensions:

```text
Functionality
Compatibility
Reliability
Firmware
Health
Performance
Endurance
Recovery
Integrity
```

The exact acceptance criteria depend on the qualification requirements being applied.

---

# 19. Engineering Evidence Model

The entire module follows one common engineering method:

```text
WHAT SHOULD HAPPEN?
        ↓
WHAT ACTUALLY HAPPENED?
        ↓
WHAT EVIDENCE PROVES IT?
        ↓
DO THEY MATCH?
        ↓
YES → Continue
NO  → Find First Divergence
```

For troubleshooting:

```text
Observation
    ↓
Hypothesis
    ↓
Evidence
    ↓
Root Cause
    ↓
Recovery
    ↓
Revalidation
```

### Example

```text
PCIe                 ✓
NVMe Controller      ✓
Namespace            ✗
Block Device         ✗
```

Correct reasoning:

> The first known divergence is at the controller-to-namespace boundary; investigation should focus there.

Incorrect reasoning:

> The SSD is defective.

The second statement is unsupported without further evidence.

---

# 20. Practical Enumeration Evidence — Hands-on Learning Example

During the learning exercise on a real NVMe laptop, the following progression was observed:

```text
lspci
 ↓
NVMe PCIe device visible
```

```text
lspci -k -s <PCI_ADDRESS>
 ↓
Kernel driver in use: nvme
```

```text
lspci -vv -s <PCI_ADDRESS>
 ↓
PCIe configuration/resources/link information
```

```text
nvme list
 ↓
NVMe namespace / Linux block-device visibility
```

```text
nvme id-ctrl /dev/nvme0
 ↓
Controller identity
```

```text
nvme smart-log /dev/nvme0
 ↓
Health baseline
```

Platform configuration was captured using:

```bash
sudo dmidecode -s system-product-name
sudo dmidecode -s bios-version
uname -r
cat /etc/os-release
```

This hands-on work was used to understand the **validation reasoning and evidence chain**; it does not by itself constitute full enterprise qualification.

---

# 21. Definition of Done

A new enterprise NVMe SSD arrives.

The validation engineer should be able to reason through:

```text
New Drive
 ↓
Enumeration
 ↓
Identification
 ↓
Firmware Check
 ↓
Health Check
 ↓
Compatibility
 ↓
Functional Tests
 ↓
Performance
 ↓
Stress
 ↓
Endurance
 ↓
Fault Injection
 ↓
Recovery
 ↓
Data Integrity
 ↓
Regression
 ↓
Qualification
```

and answer for each stage:

```text
What is this stage?
Why does it exist?
What am I trying to prove?
What evidence proves it?
What could fail here?
Where is the first divergence?
What should I investigate?
How do I revalidate after recovery?
```

That is the foundation of **Drive Qualification Engineering**.
