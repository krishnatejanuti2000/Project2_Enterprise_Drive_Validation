# MODULE 08 — SSD HEALTH AND ENDURANCE


---

## 1. Objective

Understand how to assess SSD health, interpret device-reported health indicators, evaluate endurance consumption, and use the resulting evidence to support a drive qualification decision.

This module covers:

- SSD health monitoring
- Temperature
- Critical warnings
- Available spare
- Percentage used
- Media errors
- Power cycles
- Unsafe shutdowns
- Endurance indicators
- TBW and DWPD
- Health assessment and qualification decisions

---

## 2. What Is SSD Health Monitoring?

SSD health monitoring is the process of examining device-reported information to understand the drive's operating condition, reliability indicators, and endurance consumption.

The SSD may report health information such as:

- Temperature
- Critical warnings
- Available spare capacity
- Percentage used
- Media errors
- Power-cycle count
- Unsafe-shutdown count

These indicators help a validation engineer identify unusual conditions and determine whether further investigation is required.

```text
SSD Health Information
         |
         v
Review Reported Indicators
         |
         v
Compare Against Expectations
         |
         v
Investigate Anomalies
         |
         v
Assess Drive Condition
```

Health monitoring is an evidence-gathering activity. Individual indicators should not automatically be treated as complete pass/fail decisions unless the qualification criteria define them that way.

---

## 3. Temperature

Temperature is an important operating-condition indicator.

An SSD may report its current temperature through its supported health interface.

Temperature monitoring helps a validation engineer determine whether the drive is operating within the expected thermal conditions.

### Assessment Considerations

- Compare reported temperature with the applicable device specifications.
- Observe temperature during different workloads.
- Correlate high temperature with workload and performance behavior.
- Investigate thermal warnings or unexpected changes.
- Consider whether the test environment and cooling conditions are appropriate.

A temperature reading should be interpreted using the drive's specifications and test requirements. A particular temperature value cannot be classified as safe or unsafe without that context.

---

## 4. Critical Warnings

Critical warnings indicate that the SSD is reporting a condition requiring attention.

A warning should be investigated using the specific reported condition rather than treated as a generic failure message.

### Assessment Workflow

```text
Critical Warning Reported
          |
          v
Identify the Reported Condition
          |
          v
Check Device Specifications
          |
          v
Collect Supporting Health Information
          |
          v
Review Logs and Test Results
          |
          v
Determine Required Action
```

Possible actions depend on the warning and qualification criteria.

A drive reporting no critical warnings is not automatically proven to be fully qualified. Functional, compatibility, performance, reliability, and other required tests must also be considered.

---

## 5. Available Spare

Available spare indicates the spare capacity reported by the SSD for internal management purposes.

This is different from the free space visible to the operating system.

```text
Available Spare
       |
       v
Device-Reported Internal Spare Indicator
```

The value should be compared with the drive's specified threshold and expected behavior.

### Assessment Considerations

- Record the reported available-spare value.
- Compare it with the applicable threshold.
- Investigate unexpected changes.
- Correlate the value with other health and error indicators.

A reduced available-spare value can indicate that the drive's internal reserve has decreased. Its significance depends on the device specifications and reported status.

---

## 6. Percentage Used

Percentage used is a device-reported indicator of estimated endurance consumption.

It helps assess how much of the drive's rated endurance has been consumed according to the device's estimation.

Conceptually:

```text
NAND Wear and Usage
         |
         v
Endurance Consumption
         |
         v
Percentage Used
```

### Important Distinction

Percentage used is an estimate. It is not a direct measurement of the exact amount of physical wear in every NAND block.

It should be interpreted alongside:

- Workload history
- Host writes
- Available spare
- Media errors
- Manufacturer endurance specifications
- Other device health information

A high percentage-used value does not, by itself, prove that the drive has failed.

The applicable device specification and qualification requirements determine the significance of the value.

---

## 7. Media Errors

Media errors are device-reported errors associated with media or data-integrity-related problems, according to the device's reporting definition.

The reported count is useful evidence when assessing drive reliability.

### Assessment Considerations

- Record the reported media-error count.
- Compare the value with the previous baseline.
- Investigate unexpected increases.
- Correlate reported errors with workload failures and available logs.
- Determine whether the behavior violates qualification criteria.

A zero media-error count is useful evidence, but it does not prove that all possible data-integrity or functional failures are absent.

Likewise, a nonzero count requires contextual investigation rather than an automatic conclusion about the root cause.

---

## 8. Power Cycles

The power-cycle counter records the number of power-cycle events reported by the drive.

This information helps establish the drive's operational history.

```text
Power Applied
     |
     v
Drive Operates
     |
     v
Power Removed
     |
     v
Subsequent Power Cycle
```

The counter should be interpreted according to the device's reporting behavior.

A high power-cycle count alone does not prove that the drive is defective. During qualification, it may be useful to correlate the counter with the test plan and the expected number of power cycles.

---

## 9. Unsafe Shutdowns

An unsafe shutdown is an event in which the drive loses power or is shut down without the normal shutdown procedure completing as expected.

The unsafe-shutdown counter records such events according to the device's reporting behavior.

This indicator matters because sudden power loss can interrupt internal operations.

### Assessment Considerations

- Record the counter before a test.
- Perform the planned shutdown or power-loss scenario.
- Collect the counter after the event.
- Compare the counter change with the expected result.
- Check whether the drive recovers and remains accessible.
- Perform applicable data-integrity checks.

An unsafe-shutdown count indicates reported history. A nonzero count does not by itself prove current drive failure.

Recovery and data-integrity requirements must be evaluated separately.

---

## 10. Endurance

SSD endurance describes the drive's ability to sustain writes over its specified endurance life.

NAND Flash has finite Program/Erase (P/E) capability. Internal NAND writes and erases contribute to wear.

```text
Host Workload
      |
      v
Internal SSD Operations
      |
      v
NAND Program / Erase Activity
      |
      v
NAND Wear
      |
      v
Endurance Consumption
```

Endurance is influenced by multiple factors, including:

- NAND technology
- Workload characteristics
- Internal data movement
- Write Amplification
- Garbage Collection
- Wear Leveling
- Operating conditions

Health indicators provide supporting evidence, but actual endurance assessment must also consider the drive's specification and qualification workload.

---

## 11. TBW — Terabytes Written

TBW stands for **Terabytes Written**.

It is a rated endurance metric representing the total amount of data the SSD is specified to write under the manufacturer's applicable endurance conditions.

Conceptually:

```text
SSD Endurance Specification
           |
           v
Total Rated Data Writes
           |
           v
TBW
```

TBW should not be interpreted as a universal guarantee that every workload will produce the same service life.

The applicable workload assumptions, qualification conditions, and manufacturer specification must be considered.

---

## 12. DWPD — Drive Writes Per Day

DWPD stands for **Drive Writes Per Day**.

It expresses the rated number of complete drive-capacity writes per day over the specified endurance period.

For example, consider an illustrative 1 TB drive rated at 3 DWPD.

```text
Drive Capacity = 1 TB
DWPD           = 3

Rated Daily Write Amount
= 1 TB x 3
= 3 TB/day
```

This is an illustrative calculation under the applicable endurance rating, not a guarantee that every workload will produce the same endurance result.

### TBW vs DWPD

| Metric | Meaning |
|---|---|
| TBW | Total rated amount of data written over the specified endurance period |
| DWPD | Rated full-drive-capacity writes per day over the specified endurance period |

Both metrics help characterize the SSD's specified write endurance.

---

## 13. Health Indicators vs Endurance Indicators

Health and endurance are related but distinct.

| Indicator | Primary Purpose |
|---|---|
| Temperature | Indicates thermal operating condition |
| Critical warnings | Reports specified critical conditions |
| Available spare | Reports internal spare-capacity status |
| Percentage used | Estimates consumed endurance |
| Media errors | Provides evidence of reported media-related errors |
| Power cycles | Records reported power-cycle history |
| Unsafe shutdowns | Records reported unsafe-shutdown history |
| TBW / DWPD specification | Defines rated write endurance under specified conditions |

No single field provides a complete view of SSD health.

A reliable assessment uses multiple indicators and compares them with specifications, test expectations, and supporting evidence.

---

## 14. Interpreting Health Information

Health values must be interpreted in context.

### Example A — No Reported Media Errors

```text
Critical Warnings : 0
Media Errors      : 0
```

These values are encouraging, but they do not demonstrate that the drive has passed all qualification requirements.

The drive must still satisfy its required functional, compatibility, performance, and reliability tests.

### Example B — High Percentage Used

A high percentage-used value indicates that the drive estimates a substantial portion of its endurance has been consumed.

The engineer should compare this value with the applicable endurance specification and qualification requirements.

### Example C — Critical Warning Reported

The engineer should identify the specific warning, collect supporting information, correlate logs and test results, and determine the appropriate action.

### Example D — Increased Unsafe Shutdown Count

The engineer should compare the counter before and after the test, verify the expected behavior, inspect relevant logs, and perform the required recovery and data-integrity checks.

These are illustrative scenarios. Actual acceptance criteria depend on the product requirements and qualification plan.

---

## 15. SSD Health Assessment Workflow

The following workflow summarizes how a validation engineer can use health information.

```text
Collect Initial Health Data
           |
           v
Record the Baseline
           |
           v
Execute the Planned Workload
           |
           v
Collect Health Data Again
           |
           v
Compare Values and Counter Changes
           |
           v
Identify Unexpected Conditions
           |
           v
Correlate with Logs and Test Results
           |
           v
Determine Pass / Investigate / Fail
```

The final decision depends on the explicit qualification criteria.

A value should not be classified as a failure solely because it differs from an arbitrary expectation. It must be evaluated against the relevant device specifications and test requirements.

---

## 16. Health Information and Drive Qualification

Health assessment contributes to the broader SSD qualification lifecycle.

```text
Drive Identification
        |
        v
Firmware Check
        |
        v
Health Assessment
        |
        v
Compatibility Testing
        |
        v
Functional Testing
        |
        v
Performance Testing
        |
        v
Stress / Reliability Testing
        |
        v
Data Integrity and Recovery
        |
        v
Qualification Decision
```

The sequence is a high-level qualification model. Specific projects may perform some activities in a different order or repeat health checks throughout testing.

Health information helps determine whether additional investigation is necessary and whether the drive continues to satisfy the applicable requirements.

It does not replace the other qualification activities.

---

## 17. Module 8 — Key Takeaways

- Health information helps assess the drive's reported operating condition.
- Temperature must be compared with applicable device specifications.
- Critical warnings must be interpreted according to the specific condition reported.
- Available spare is distinct from operating-system-visible free space.
- Percentage used estimates endurance consumption; it is not a direct measurement of every NAND block's wear.
- Media errors provide evidence that should be correlated with logs and test results.
- Power-cycle and unsafe-shutdown counters describe reported operational history.
- TBW and DWPD describe rated write endurance under specified conditions.
- A single health indicator cannot prove that an SSD is fully qualified.
- Qualification decisions must follow explicit requirements and supporting evidence.
- Health monitoring should be correlated with functional behavior, compatibility, performance, recovery, and data-integrity results.

---


