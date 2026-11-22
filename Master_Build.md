# Project 2 — Enterprise NVMe SSD Validation
## Master Build Plan — Version 2.2
### Practical Engineering Competency Plan

---

# 1. Project Objective

Develop practical working competence to independently handle common Enterprise NVMe SSD Validation Engineering assignments at the expected 3–4-year experience level.

The objective is to learn how to perform actual validation engineering work, rather than only understand technical concepts.

The training will focus on:

- Understanding SSD functionality and failure mechanisms.
- Inspecting and configuring NVMe devices.
- Executing functional and negative tests.
- Performing boundary and recovery testing.
- Working with Linux storage and PCIe diagnostic tools.
- Generating workloads using FIO.
- Measuring IOPS, bandwidth and latency.
- Investigating failures across relevant system layers.
- Collecting logs, traces and technical evidence.
- Automating validation tasks using Python and Pytest.
- Verifying fixes and performing targeted regression.
- Preparing technical test reports.

## Reference Product

- Product: Solidigm D7-PS1010
- Capacity: 3.84 TB
- Form Factor: U.2, 15 mm
- Interface: PCIe Gen5 x4
- Protocol: NVMe

Actual manufacturer specifications must be verified using official documentation.

Internal firmware builds, release notes and defect information created for training must be clearly identified as simulated. They must never be represented as actual manufacturer releases.

---

# 2. Priority Structure

## Priority A — Main Preparation

These builds receive the greatest practical attention and execution priority.

- Build 1 — SSD Initialization and NVMe Detection
- Build 2 — PCIe Link Validation and Troubleshooting
- Build 3 — NVMe Protocol, Command Processing and I/O Validation
- Build 6 — Enterprise SSD Performance Validation
- Build 7 — Controller Reset and Recovery
- Build 8 — Power Loss Protection and Data Persistence
- Build 9 — Thermal and SSD Power Management
- Build 10 — Firmware Lifecycle, Update and Compatibility Validation

## Priority B — Supporting Core Competencies

These builds remain important, but their depth will be proportional to their practical relevance.

- Build 4 — FTL, Data Integrity and SSD Internal Storage Management
- Build 5 — NAND Media Reliability, ECC and Error Management

The objective is to understand common failure mechanisms, perform relevant observable tests and investigate available evidence.

Unnecessary deep dives into proprietary SSD algorithms will be avoided.

---

# 3. Master Build Plan

# Build 1 — Enterprise SSD Initialization and NVMe Detection

**Priority:** A

**Primary Ownership:** Device Discovery and Initialization

## Objective

Develop the ability to investigate and validate failures preventing an SSD from being detected, initialized or exposed to the host operating system.

## Practical Technical Coverage

### Host and Platform

- Server power and startup.
- Server BIOS configuration.
- Platform compatibility.
- PCIe slot configuration.
- Physical SSD connectivity.
- Backplane and cabling checks.

### PCIe Enumeration — Basic Scope

- PCIe device discovery.
- Endpoint visibility.
- Vendor ID and Device ID.
- PCI configuration space.
- BAR detection and assignment.
- PCI resource allocation.
- Linux PCI subsystem.

### Linux NVMe Driver

- NVMe driver availability.
- Kernel module loading.
- Module configuration.
- Driver binding.
- Driver probe.
- Probe failures.
- Kernel logs and driver errors.

### Controller Initialization

- Controller capability registers.
- Controller configuration registers.
- CC.EN.
- CSTS.RDY.
- Controller fatal status.
- Controller startup sequence.
- Controller readiness verification.

### Admin Queue Initialization

- Admin Submission Queue.
- Admin Completion Queue.
- Queue initialization.
- Admin command submission.
- Admin command completion.
- Initial controller communication.

### Controller and Namespace Discovery

- Identify Controller.
- Identify Namespace.
- Controller identification.
- NVM subsystem discovery.
- Controller association.
- Namespace discovery.
- Namespace attachment.
- Namespace availability.
- Namespace capacity.
- LBA format discovery.

### Linux Block Device Exposure

- NVMe block driver registration.
- Namespace block-device registration.
- /dev/nvme* device exposure.
- /dev/nvme*n* namespace exposure.
- Missing block-device investigation.
- Device availability to applications.

### Startup Failures

- Cold boot detection failure.
- Warm reboot detection failure.
- Repeated startup failures.
- Intermittent controller initialization failure.
- Firmware initialization failure.
- Controller hardware startup failure.
- Driver-related detection failure.
- Platform-related detection failure.

## Practical Work

- Inspect PCIe devices using lspci.
- Inspect PCI configuration space.
- Examine BAR information.
- Inspect Linux NVMe driver binding.
- Check module availability and loading.
- Investigate driver probe failures.
- Inspect controller registers where accessible.
- Execute supported NVMe Admin commands.
- Identify controllers and namespaces.
- Verify block-device exposure.
- Investigate initialization failures using system logs.
- Perform supported boot and rediscovery tests.
- Collect diagnostic evidence.

## Expected Competency

Given an SSD initialization or detection problem, independently investigate the host-to-controller initialization path and identify the likely failing layer.

## Ownership Boundary

- Basic PCIe visibility and enumeration belong here.
- Detailed PCIe link training, speed, width and link stability belong to Build 2.
- Runtime controller reset and recovery belong to Build 7.

---

# Build 2 — PCIe Link Validation and Troubleshooting

**Priority:** A

**Primary Ownership:** PCIe Connectivity and Link Behaviour

## Objective

Develop the ability to investigate PCIe link failures, speed/width downgrades, unexpected link drops and interoperability problems.

## Practical Technical Coverage

### PCIe Architecture and Topology

- PCIe Root Complex.
- Root Ports.
- PCIe switches.
- Upstream and Downstream Ports.
- PCIe retimers.
- Direct-attached topology.
- Switch-based topology.
- Retimer-based topology.
- NVMe backplanes.

### PCIe Generations

- PCIe Gen4.
- PCIe Gen5.
- Link capabilities.
- Link status.
- Speed negotiation.
- Width negotiation.

### LTSSM

- Detect.
- Polling.
- Configuration.
- L0.
- Recovery.
- Relevant substates.
- State transitions.
- Link-training failures.

### Training Sequences

- TS1.
- TS2.
- SKP.
- EIOS.
- EIEOS.
- FTS.
- Receiver detection.
- Bit lock.
- Symbol lock.
- Lane detection.
- Polarity inversion.
- Lane reversal.

### Link Negotiation and Recovery

- Speed negotiation.
- Width negotiation.
- Speed downgrade.
- Width downgrade.
- Link retraining.
- Link drops.
- Unexpected Recovery transitions.
- Gen4/Gen5 equalization.
- Lane-level failures.
- Signal-integrity-related issues.

### PCIe Error Handling

- AER correctable errors.
- AER non-fatal errors.
- AER fatal errors.
- TLP handling.
- DLLP handling.
- Replay mechanism.
- Data Link Layer errors.
- Link-level error recovery.

### PCIe Power Management

- ASPM.
- L0s.
- L1.
- L1.1.
- L1.2.
- Link entry and exit behaviour.
- Link recovery following power transitions.

### Protocol Analyzer

- Teledyne LeCroy analyzer configuration.
- Trigger configuration.
- Capture setup.
- LTSSM trace analysis.
- TS1/TS2 inspection.
- TLP/DLLP analysis.
- Error identification.
- Correlation with Linux logs.
- Link-drop investigation.

### Interoperability

- PCIe Gen4/Gen5 compatibility.
- Root Port interoperability.
- Switch interoperability.
- Retimer interoperability.
- Different server configurations.
- Repeated link stability testing.

## Practical Work

- Inspect PCIe Link Capabilities and Link Status.
- Compare expected and negotiated speed/width.
- Examine AER errors.
- Investigate link failures.
- Perform supported link recovery exercises.
- Configure analyzer captures.
- Observe LTSSM behaviour.
- Capture and analyse PCIe traces.
- Correlate analyzer evidence with Linux logs.
- Investigate link drops and downgrades.

## Expected Competency

Independently investigate PCIe link failures, understand the evidence and identify the likely source of the problem.

## Ownership Boundary

- Build 1 owns basic enumeration.
- Build 2 owns full PCIe link investigation.
- Build 7 owns controller recovery following resets.

---

# Build 3 — NVMe Protocol, Command Processing and I/O Validation

**Priority:** A

**Primary Ownership:** NVMe Protocol and I/O Execution

## Objective

Develop the ability to validate NVMe commands, queue management, data transfers, command completion and I/O error handling.

## Practical Technical Coverage

### NVMe Command Architecture

- Admin commands.
- I/O commands.
- Admin Submission Queue.
- Admin Completion Queue.
- I/O Submission Queues.
- I/O Completion Queues.

### Queue Management

- Queue creation and deletion.
- Queue head and tail management.
- Queue wraparound.
- Queue depth.
- Queue-full conditions.
- Multiple I/O queues.
- Concurrent command execution.
- Phase Tag.
- Command Identifier (CID).

### Doorbells and Completion

- SQ Tail Doorbell.
- CQ Head Doorbell.
- MSI-X notification.
- Polling.
- Interrupt handling.
- Completion processing.

### NVMe Commands

- Read.
- Write.
- Flush.
- Identify.
- Identify Controller.
- Identify Namespace.
- Get Log Page.
- Get Features.
- Set Features.
- Abort.
- Asynchronous Event handling.

### Data Transfer

- PRP.
- SGL.
- Data-buffer descriptors.
- DMA data transfer.
- Host-to-controller transfer.
- Controller-to-host transfer.
- Invalid descriptor handling.
- Data-transfer failures.

### Command Error Handling

- Invalid opcodes.
- Invalid command parameters.
- Invalid NSID.
- Invalid LBA.
- Invalid command fields.
- Command rejection.
- Error completion status.
- Command timeouts.
- Hanging commands.
- Missing completions.
- Duplicate completions.
- Delayed completions.
- Abort handling.

### Concurrent I/O

- Multiple queue operation.
- Mixed Read/Write.
- Queue-depth variation.
- Concurrent commands.
- Command ordering.
- Outstanding command handling.

## Practical Work

- Inspect NVMe controller and namespace information.
- Execute NVMe Admin commands.
- Execute Read/Write operations.
- Create and delete supported namespaces.
- Test valid and invalid command parameters.
- Observe command completion and errors.
- Exercise different queue configurations.
- Generate concurrent I/O.
- Investigate command timeouts.
- Collect NVMe error information.
- Develop selected command-validation scripts.

## Expected Competency

Independently execute NVMe commands, validate queue behaviour, identify command-processing failures and collect evidence.

## Ownership Boundary

- Build 1 owns initial Admin Queue setup.
- Build 3 owns I/O queue operation and command processing.
- Build 4 owns FTL mapping correctness.
- Build 10 owns firmware update functionality.

---

# Build 4 — FTL, Data Integrity and SSD Internal Storage Management

**Priority:** B

**Primary Ownership:** Logical-to-Physical Mapping and NAND Space Management

## Objective

Understand and validate logical data storage, mapping correctness, data relocation and SSD internal storage management.

## Practical Technical Coverage

### FTL

- FTL architecture.
- Logical-to-physical mapping.
- Mapping-table management.
- Mapping-cache behaviour.
- Physical page allocation.
- Write mapping.
- Overwrite mapping.
- Mapping updates.
- Mapping metadata consistency.
- Mapping checkpoints.
- Mapping recovery.

### Data Integrity

- Read-after-Write verification.
- Overwrite verification.
- Stale-data failures.
- Incorrect Read data.
- Partial writes.
- LBA boundary testing.
- Cross-LBA data isolation.
- Sequential data patterns.
- Random data patterns.
- Data integrity after internal relocation.

### Garbage Collection

- GC triggering conditions.
- GC thresholds.
- Victim-block selection.
- Valid-page relocation.
- Free-block management.
- Block erasure.
- Block reclamation.
- GC recovery behaviour.

### Wear Leveling and Space Management

- Dynamic Wear Leveling.
- Static Wear Leveling.
- TRIM.
- Deallocation.
- Overprovisioning.
- Free-space management.
- High-capacity-occupancy behaviour.

## Practical Work

- Create controlled data patterns.
- Execute sequential and random Read/Write tests.
- Perform repeated overwrites.
- Verify data correctness.
- Test deallocation behaviour where supported.
- Examine available health information.
- Understand observable effects of GC.
- Investigate stale-data and mapping-related symptoms.
- Compare data before and after supported recovery operations.

## Expected Competency

Perform data-integrity testing and understand FTL/GC-related failures, their observable symptoms and the evidence needed for investigation.

Detailed proprietary FTL algorithms are outside our target.

## Ownership Boundary

- Build 4 owns mapping correctness, GC and space management.
- Build 5 owns physical NAND media and ECC reliability.
- Build 6 owns formal performance characterization.

---

# Build 5 — NAND Media Reliability, ECC and Error Management

**Priority:** B

**Primary Ownership:** NAND Media Reliability

## Objective

Understand NAND-related failures and validate observable media-error handling and reliability behaviour.

## Practical Technical Coverage

### NAND Operations

- NAND Read.
- NAND Page Program.
- NAND Block Erase.
- NAND operation failures.
- NAND die/channel-level failures.
- NAND initialization and media identification.

### ECC

- ECC encoding.
- ECC decoding.
- Correctable errors.
- Uncorrectable errors.
- Error-correction capability.
- ECC error reporting.

### Read Retry

- Read-retry mechanisms.
- Read-retry behaviour.
- Read latency impact.
- Read recovery.

### NAND Reliability

- Bad-block detection.
- Bad-block management.
- Block retirement.
- Read Disturb.
- Data Retention.
- NAND wear.
- Wear-related degradation.
- NAND media reliability.

### Media Error Management

- Media-error detection.
- Media-error handling.
- Recovery behaviour.
- SMART media-error information.
- Error Information Log.
- NAND-related telemetry.
- Error reporting and diagnostics.

### Reliability Testing

- Applicable aged-media testing.
- Read-intensive workloads.
- Media-error observation.
- Data-integrity verification.
- Retention testing concepts.
- Endurance testing concepts.

## Practical Work

- Examine NVMe SMART/Health information.
- Examine Error Information Log.
- Inspect supported telemetry.
- Run Read-intensive workloads.
- Exercise supported host-level error scenarios.
- Analyse media-error evidence.
- Study Read-retry and ECC failure traces where available.
- Develop appropriate reliability test scenarios.

## Expected Competency

Recognize NAND-related failure symptoms, interpret media-error evidence and develop appropriate reliability validation tests.

Physical NAND ECC correction, retention and Read Disturb qualification cannot be claimed using virtual-device emulation.

## Ownership Boundary

- Build 4 owns GC and free-block allocation.
- Build 5 owns physical NAND reliability and ECC.

---

# Build 6 — Enterprise SSD Performance Validation

**Priority:** A

**Primary Ownership:** Performance Engineering and FIO

## Objective

Independently design, execute, measure and analyse enterprise SSD performance workloads.

## Practical Technical Coverage

### FIO

- FIO installation.
- FIO configuration.
- Job-file development.
- Job sections.
- I/O engines.
- Direct I/O.
- Runtime configuration.
- Result collection.
- JSON output.

### Workload Types

- Sequential Read.
- Sequential Write.
- Random Read.
- Random Write.
- Mixed Read/Write.
- Read/write ratios.
- Different workload patterns.

### Workload Parameters

- Block size.
- Queue depth.
- Number of jobs.
- Number of workers.
- I/O depth.
- Runtime.
- Workload duration.
- Data-set size.

### Performance Metrics

- IOPS.
- Bandwidth.
- Throughput.
- Average latency.
- Minimum latency.
- Maximum latency.
- P95 latency.
- P99 latency.
- P99.9 latency.
- Tail latency.
- QoS.

### Performance Testing Methodology

- Drive preconditioning.
- Steady-state testing.
- Short-duration workloads.
- Sustained workloads.
- Sequential performance.
- Random performance.
- Mixed workload performance.
- Four-corner testing.
- Queue-depth scaling.
- Performance consistency.
- Repeatability.
- Controlled benchmarking conditions.

### Performance Regression

- Old versus new firmware comparison.
- Performance degradation detection.
- Performance improvement verification.
- IOPS regression.
- Bandwidth regression.
- Latency regression.
- Tail-latency regression.
- Workload-specific regression.
- Performance report preparation.

### Automation

- FIO execution using Python.
- FIO JSON parsing.
- Result comparison.
- Automated performance checks.
- Pytest-based performance regression.
- Automated report generation.

## Practical Work

- Install and inspect FIO.
- Develop reusable job files.
- Execute sequential and random workloads.
- Execute mixed Read/Write workloads.
- Change block size and queue depth.
- Measure IOPS, bandwidth and latency.
- Analyse tail latency.
- Compare repeated workload runs.
- Generate and inspect JSON results.
- Build Python result parsers.
- Automate selected benchmarks.
- Compare results between different configurations or firmware builds.

## Expected Competency

Independently configure FIO, execute performance workloads, interpret the results, identify performance regressions and create meaningful performance reports.

## Ownership Boundary

- Build 4 owns GC functionality.
- Build 6 owns formal performance measurement and analysis.
- Other builds may use FIO for targeted workload generation.

---

# Build 7 — Controller Reset, Error Handling and Recovery

**Priority:** A

**Primary Ownership:** Runtime Controller Recovery

## Objective

Validate controller recovery following reset operations, runtime faults and unexpected controller conditions.

## Practical Technical Coverage

### Reset Types

- NVMe Controller Reset.
- PCIe Function Level Reset (FLR).
- NVM Subsystem Reset, where supported.
- PCIe Hot Reset.
- Reset sequence and recovery behaviour.

### Controller Recovery

- Controller readiness transitions.
- CC.EN and CSTS.RDY.
- Queue teardown.
- Queue recreation.
- Admin Queue recovery.
- I/O Queue recreation.
- Namespace rediscovery.
- Controller availability after reset.

### Runtime Scenarios

- Reset during idle.
- Reset during active Read.
- Reset during active Write.
- Reset during mixed Read/Write.
- Reset with outstanding commands.
- Repeated reset cycles.
- Reset under workloads.

### Controller Error Handling

- Controller hangs.
- Firmware watchdog.
- Fatal controller errors.
- Internal firmware faults.
- Controller recovery.
- Recovery timeout.
- Repeated recovery failures.

### Post-Recovery Validation

- Controller readiness.
- Namespace availability.
- Block-device availability.
- Data integrity.
- I/O resumption.
- Recovery time.
- Linux driver interaction.
- Repeated reset stability.

## Practical Work

- Execute supported controller reset operations.
- Observe controller readiness.
- Verify namespace availability.
- Verify block-device rediscovery.
- Run I/O before and after reset.
- Run workloads during supported reset scenarios.
- Verify data after recovery.
- Examine Linux logs.
- Investigate failed recovery.
- Develop reset/recovery automation where appropriate.

## Expected Competency

Independently perform supported reset tests, verify controller recovery, investigate failures and validate data integrity after recovery.

## Ownership Boundary

- Build 1 owns initial startup.
- Build 2 owns PCIe link-level Hot Reset behaviour.
- Build 7 owns runtime controller recovery.

---

# Build 8 — Power Loss Protection and Data Persistence

**Priority:** A

**Primary Ownership:** Data Durability During Unexpected Power Loss

## Objective

Understand and validate the protection of acknowledged data and critical metadata during unexpected power loss.

## Practical Technical Coverage

### Write Persistence

- Volatile Write Cache.
- NVMe Flush.
- FUA.
- Write completion versus persistence.
- Acknowledged Write data.
- Persistence requirements.

### PLP Architecture

- Power-fail detection.
- Input-voltage monitoring.
- Hold-up capacitor concepts.
- Capacitor health.
- Controller power-fail notification.
- Emergency shutdown sequence.
- Protection of pending user data.
- Protection of critical FTL metadata.

### Power-Loss Scenarios

- Power loss during active Writes.
- Power loss during mixed workloads.
- Power loss during metadata operations.
- Power loss during GC.
- Power loss after Write completion.
- Power loss before Flush completion.
- Repeated power-fail cycles.

### Recovery and Integrity

- Recovery after sudden power loss.
- Mapping consistency.
- Metadata consistency.
- Acknowledged Write verification.
- Missing data.
- Stale data.
- Corrupted data.
- PLP health reporting.
- Recovery evidence.

## Practical Work

- Execute Read/Write persistence tests.
- Exercise Flush semantics.
- Inspect volatile-cache capabilities.
- Develop power-loss test scenarios.
- Understand controlled power interruption procedures.
- Analyse recovery logs and data-integrity results.
- Verify persistence requirements against observed behaviour.

## Expected Competency

Understand data-persistence guarantees, prepare power-loss validation scenarios and investigate recovery and integrity failures.

Actual capacitor-backed PLP qualification requires suitable physical enterprise SSD hardware and controlled power-interruption equipment.

## Ownership Boundary

- Build 7 owns ordinary controller reset.
- Build 8 owns unexpected power failure and PLP.
- Build 10 owns interruption during firmware updates.

---

# Build 9 — Thermal and SSD Power Management

**Priority:** A

**Primary Ownership:** Temperature Management and NVMe Power States

## Objective

Understand and validate thermal behaviour, throttling, NVMe power states and recovery between operating conditions.

## Practical Technical Coverage

### Temperature Monitoring

- Temperature-sensor information.
- Temperature acquisition.
- SMART temperature reporting.
- Critical temperature warnings.
- Temperature logs and telemetry.

### Thermal Management

- Thermal thresholds.
- Throttling activation.
- Throttling levels.
- Performance reduction.
- Thermal recovery.
- Hysteresis.
- Performance under sustained temperature.
- Thermal protection.
- Thermal event reporting.

### NVMe Power Management

- NVMe power-state identification.
- Supported power states.
- Autonomous Power State Transition (APST).
- Idle power-state entry.
- Return from low-power states.
- Wake-up latency.
- I/O after idle.
- Repeated active/idle transitions.
- Power consumption.
- Performance under different policies.

### Performance and Stability

- Thermal throttling impact.
- Performance recovery.
- Sustained workload behaviour.
- Power-state transition failures.
- I/O timeout after idle.
- Temperature and performance correlation.

## Practical Work

- Inspect supported power states.
- Read SMART temperature information.
- Examine available thermal events.
- Execute supported idle/active transitions.
- Run workloads while observing temperature.
- Examine performance changes.
- Investigate APST-related issues.
- Collect logs and telemetry.

## Expected Competency

Understand thermal and power management, execute supported validation exercises and identify temperature/power-related performance or stability failures.

Actual thermal-throttling qualification requires physical hardware and controlled temperature conditions.

## Ownership Boundary

- Build 2 owns PCIe ASPM.
- Build 8 owns sudden power loss.
- Build 9 owns thermal management and NVMe power states.

---

# Build 10 — Firmware Lifecycle, Update and Compatibility Validation

**Priority:** A

**Primary Ownership:** Firmware Update and Activation Reliability

## Objective

Understand and validate firmware image management, downloading, activation, upgrade, downgrade and recovery.

## Practical Technical Coverage

### Firmware Image Management

- Firmware image identification.
- Firmware version.
- Build ID.
- Image checksum.
- Firmware integrity verification.
- Product compatibility.
- Hardware-revision compatibility.
- Build manifest.
- Release documentation.

### Firmware Download

- Firmware Download command.
- Transfer sizes.
- Firmware offsets.
- Alignment requirements.
- Image transfer validation.
- Invalid image handling.
- Unsupported image rejection.
- Download interruption.

### Firmware Slots

- Firmware slot management.
- Firmware Slot Information Log.
- Active firmware slot.
- Firmware revision reporting.
- Slot compatibility.

### Firmware Commit and Activation

- Firmware Commit operations.
- Activation without reset, where supported.
- Activation requiring reset.
- Firmware activation sequence.
- Active firmware verification.
- Controller readiness after activation.
- Namespace availability after activation.

### Firmware Update Scenarios

- Firmware upgrade.
- Supported downgrade.
- Firmware rollback, where supported.
- Interrupted firmware download.
- Power interruption during update.
- Update failure.
- Recovery after interrupted update.
- Firmware update under supported I/O conditions.
- User-data preservation.
- Post-update startup.
- Compatibility across supported configurations.

### Release Information

- Firmware release notes.
- Fixed defect review.
- Known issue review.
- Firmware version tracking.
- Update status.
- Firmware logs.
- Candidate build verification.

## Practical Work

- Inspect firmware manifests.
- Verify image checksums.
- Understand firmware compatibility information.
- Inspect firmware version and slot information.
- Execute supported firmware-management commands.
- Study activation mechanisms.
- Analyse firmware update failures.
- Prepare upgrade/downgrade test scenarios.
- Verify post-update device availability.
- Develop firmware-update evidence collection.

## Expected Competency

Understand firmware release information, verify images, perform supported firmware update procedures and investigate activation/recovery problems.

Real vendor firmware activation requires a compatible supported physical SSD and valid manufacturer-provided firmware image.

## Ownership Boundary

- Build 7 owns generic controller reset/recovery.
- Build 8 owns operational PLP.
- Build 10 owns update-specific activation and recovery.

---

# Cross-Cutting Practical Engineering Skills

These skills are not independent theory courses.

They must be applied within the ten builds whenever needed.

## Linux Storage and PCIe Diagnostics

- lspci
- lspci -vvv
- dmesg
- journalctl
- nvme-cli
- smartctl
- setpci
- Linux sysfs
- PCIe configuration inspection
- Controller and namespace discovery
- Error-log collection

## Python and Pytest Automation

- Python scripting.
- subprocess.
- Regex.
- JSON parsing.
- Log parsing.
- Command execution.
- FIO automation.
- NVMe command wrappers.
- Automated evidence collection.
- Pytest fixtures.
- Parametrization.
- Assertions.
- Regression automation.

## Protocol and Performance Evidence

- Teledyne LeCroy traces.
- PCIe logs.
- NVMe error logs.
- SMART and telemetry.
- FIO JSON results.
- Linux kernel logs.
- Firmware logs, where available.

## Defect Investigation

- Failure reproduction.
- Evidence collection.
- Failure classification.
- OS versus driver versus PCIe versus controller versus firmware isolation.
- Root-cause investigation.
- Defect documentation.
- Developer communication.
- Fix verification.

---

# Practical Learning Method

We will not repeat the complete 11-stage validation lifecycle for every build.

The focus is on performing the technical work.

For each practical task:

1. Understand the assigned engineering problem.
2. Identify the functionality being investigated.
3. Prepare the required environment.
4. Learn only the technical background necessary for execution.
5. Run the relevant commands or tests.
6. Observe actual behaviour and outputs.
7. Compare expected and actual results.
8. Investigate failures.
9. Collect evidence.
10. Automate suitable repeatable tasks.
11. Record the outcome.

Company-specific Agile ceremonies and validation lifecycle procedures will be introduced only when necessary to understand the actual work.

# Practical Execution Classification

Every exercise must be classified honestly:

- **Hands-on executed:** Physically or virtually performed and evidence collected.
- **Guided practical simulation:** A realistic failure or workflow is simulated.
- **Evidence-based investigation:** Existing logs, traces or outputs are analysed.
- **Not executed:** Requires unavailable hardware, firmware or laboratory equipment.

Virtual NVMe emulation must never be presented as physical SSD, NAND or PCIe electrical validation.

# Progress and Completion Rules

- Work one technical task at a time.
- Avoid unnecessary theory and repeated analogies.
- Do not give large information dumps.
- Explain commands before execution.
- Show expected outputs and failure indicators.
- Investigate common relevant sub-issues systematically.
- Do not skip important practical work simply to finish quickly.
- Do not repeat full investigations already owned by another build.
- Record actual execution evidence.
- Move to the next build when the relevant practical competency has been developed and reviewed.

A build does not need every conceivable failure scenario mastered. The objective is to develop the ability to handle representative common validation assignments and extend the same engineering method to unfamiliar issues.

---

# Final Project Objective

Develop the ability to independently handle common Enterprise NVMe SSD Validation Engineering assignments:

- Inspect and understand devices.
- Configure validation environments.
- Execute functional and negative tests.
- Generate performance workloads.
- Analyse performance metrics.
- Investigate PCIe, NVMe and storage failures.
- Collect and interpret diagnostic evidence.
- Automate suitable validation tasks.
- Verify firmware fixes.
- Perform relevant regression.
- Produce technical validation reports.

**Target Role:** Enterprise NVMe SSD / Storage Validation Engineer

**Target Experience Level:** 3–4-year working competency

**Project:** Project 2 — Enterprise Drive Validation

**Master Build Plan:** Version 2.2 — Practical Execution Plan
