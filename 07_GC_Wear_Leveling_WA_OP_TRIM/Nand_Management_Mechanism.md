# MODULE 07 — GARBAGE COLLECTION / WEAR LEVELING / WRITE AMPLIFICATION / OVER-PROVISIONING / TRIM


---

## 1. Objective

Understand how SSD firmware manages NAND space, distributes NAND wear, controls internal write activity, and reclaims logical data that is no longer required.

This module covers:

- Garbage Collection (GC)
- Valid and invalid NAND pages
- Victim block selection
- Valid-page migration and block erasure
- Background and foreground Garbage Collection
- Dynamic and Static Wear Leveling
- Write Amplification (WA)
- Over-Provisioning (OP)
- TRIM / Deallocate
- Relationships with latency, throughput, endurance and steady-state performance

---

# PART 1 — GARBAGE COLLECTION

## 2. What Is Garbage Collection?

**Garbage Collection (GC)** is an SSD firmware process that reclaims NAND blocks containing invalid pages so those blocks can be reused for future writes.

NAND supports page-level read/program operations and block-level erase operations.

```text
Read    → Page
Program → Page
Erase   → Block
```

Because NAND cannot simply overwrite an already-programmed page, updated data may be written to a new physical page. The previous page then becomes invalid.

Over time, blocks can contain a mixture of valid and invalid pages.

Garbage Collection reclaims this space while preserving valid data.

---

## 3. Valid and Invalid Pages

### 3.1 Valid Page

A **valid page** contains current data that the SSD must preserve.

### 3.2 Invalid Page

An **invalid page** contains obsolete data that is no longer the current version associated with the logical data.

Example:

```text
Block 20
 ├── Page 0 → Valid
 ├── Page 1 → Invalid
 ├── Page 2 → Valid
 └── Page 3 → Invalid
```

The block contains both current and obsolete data.

Because erase occurs at block level, the SSD cannot erase only the invalid pages. It must preserve the remaining valid pages before reclaiming the block.

### Important Distinction

```text
Invalid Page
    → Obsolete data at page level

Bad Block
    → NAND block considered unreliable
```

An invalid page does not necessarily mean the entire block is defective.

---

## 4. Victim Block Selection

A **victim block** is a block selected by the SSD for Garbage Collection.

The firmware evaluates candidate blocks according to its internal policy.

Potential considerations include:

- Number of invalid pages
- Number of valid pages that must be preserved
- Available free blocks
- Internal scheduling and workload
- Block condition and wear-management requirements

For example:

```text
Block A
Valid Pages   = 20
Invalid Pages = 236

Block B
Valid Pages   = 180
Invalid Pages = 76
```

Block A may be an attractive reclamation candidate because it contains fewer valid pages to migrate. However, the SSD's actual selection depends on its implementation and current operating conditions.

A victim block is not necessarily a bad block. It is a block selected for reclamation.

---

## 5. Garbage Collection Workflow

The conceptual process consists of the following steps.

### Step 1 — Identify a Candidate Block

The SSD identifies a block suitable for reclamation.

### Step 2 — Preserve Valid Pages

Valid pages must remain accessible.

The SSD copies their data to another suitable physical location.

### Step 3 — Update Mapping Information

If valid data moves, the FTL updates the relevant mappings so the logical addresses continue to reference the correct physical data.

### Step 4 — Erase the Original Block

Once all required valid data has been preserved, the original block can be erased.

### Step 5 — Return the Block for Reuse

After erasure, the block can be returned to the pool of available blocks.

```text
Select Victim Block
        |
        v
Copy Valid Pages
        |
        v
Update FTL Mappings
        |
        v
Erase Original Block
        |
        v
Return Block to Free Pool
```

The exact selection strategy, scheduling and ordering details depend on the SSD implementation.

---

## 6. Background vs Foreground Garbage Collection

Garbage Collection may occur under different operating conditions.

### 6.1 Background Garbage Collection

Background GC reclaims space when the SSD's internal policy identifies an appropriate opportunity, often during idle or lighter workloads.

Purpose:

- Maintain reusable block availability.
- Reclaim space before it becomes urgently necessary.
- Reduce the need for immediate reclamation during active writes.

### 6.2 Foreground Garbage Collection

Foreground GC occurs when the SSD needs to reclaim space to service active workloads.

It can require additional internal work while host requests are waiting.

Potential effects include:

- Increased write latency
- Reduced write throughput
- Additional NAND reads and writes
- Increased internal resource usage

The precise conditions and terminology used for background and foreground GC vary by SSD implementation.

---

## 7. Why Is Garbage Collection Required?

Out-of-place writes create invalid pages.

Without Garbage Collection, obsolete pages would continue occupying blocks, reducing the number of erased blocks available for new writes.

```text
Out-of-Place Writes
        |
        v
Invalid Pages Accumulate
        |
        v
Reusable Space Decreases
        |
        v
Garbage Collection
        |
        v
Blocks Become Reusable
```

Garbage Collection is therefore essential for maintaining an ongoing supply of reusable NAND blocks.

---

# PART 2 — WEAR LEVELING

## 8. What Is Wear Leveling?

**Wear Leveling** is an SSD firmware technique that distributes NAND wear across usable blocks to help avoid premature wear concentration.

NAND blocks have limited Program/Erase (P/E) endurance.

If certain blocks experience substantially more erase operations than others, they may reach their reliability limits earlier.

Wear Leveling uses information such as block erase counts to guide data placement or relocation decisions.

```text
Monitor Block Wear
        |
        v
Evaluate Wear Distribution
        |
        v
Apply Wear Leveling Policy
        |
        v
Distribute Wear Across Usable Blocks
```

Wear Leveling does not eliminate NAND wear or increase the physical P/E limit of NAND. It helps the SSD use its available endurance more effectively.

---

## 9. Dynamic Wear Leveling

**Dynamic Wear Leveling (DWL)** focuses on distributing new writes among suitable available blocks.

The firmware can use information such as erase counts and block availability when choosing a physical destination.

Conceptually:

```text
Host Write
    |
    v
Evaluate Suitable Free Blocks
    |
    v
Apply Dynamic Wear Leveling Policy
    |
    v
Select Physical Destination
    |
    v
Program NAND
```

The exact block-selection algorithm is vendor-specific. The firmware does not necessarily select the block with the lowest erase count for every write.

Dynamic Wear Leveling primarily manages wear associated with blocks used for new writes.

---

## 10. Static Wear Leveling

**Static Wear Leveling** also considers data that does not change frequently.

Suppose:

```text
Block A → Low erase count, static data
Block B → Higher erase count, more frequently reused
```

If static data remains in a low-wear block, that block may not participate in new write activity very often.

Static Wear Leveling may relocate static data from a lower-wear block to a higher-wear block so that the lower-wear block can become available for future writes.

Conceptually:

```text
Identify Wear Imbalance
        |
        v
Select Static Data for Relocation
        |
        v
Move Data According to Wear Policy
        |
        v
Update FTL Mapping
        |
        v
Make Blocks Available According to Policy
```

This data movement consumes resources, so firmware weighs the endurance benefits against the additional write and erase overhead.

The exact algorithms and thresholds vary by SSD implementation.

---

## 11. Dynamic vs Static Wear Leveling

| Dynamic Wear Leveling | Static Wear Leveling |
|---|---|
| Focuses on new writes | Also considers infrequently changed data |
| Distributes new writes among suitable blocks | May relocate static data to improve wear balance |
| Avoids repeatedly concentrating new writes on the same blocks | Helps address imbalance across blocks that have different write activity |
| Has its own internal overhead | May require additional relocation and erase activity |

Both mechanisms aim to manage wear. They do not eliminate the physical limitations of NAND Flash.

---

## 12. Garbage Collection vs Wear Leveling

These mechanisms are complementary but solve different problems.

| Garbage Collection | Wear Leveling |
|---|---|
| Reclaims NAND blocks | Distributes NAND wear |
| Preserves valid pages before erasing a block | Uses wear information to guide placement or relocation |
| Helps maintain reusable capacity | Helps manage endurance |
| Can introduce internal copying and erasing | Can introduce additional internal data movement |

A block reclaimed by Garbage Collection may become a candidate for future writes. Wear Leveling then influences how the SSD uses its available blocks according to its policy.

---

# PART 3 — WRITE AMPLIFICATION

## 13. What Is Write Amplification?

**Write Amplification (WA)** is the amount of data written to NAND relative to the amount of data written by the host.

The formula is:

```text
WA = NAND Data Written / Host Data Written
```

Both values must refer to a consistent measurement period and comparable data-volume units.

### Example

Suppose:

```text
Host Data Written = 100 GB
NAND Data Written = 150 GB
```

Then:

```text
WA = 150 / 100

WA = 1.5
```

The SSD wrote 1.5 times as much data to NAND as the host supplied during the measured period.

The example is illustrative and does not represent a measurement from a specific drive.

---

## 14. Causes of Write Amplification

Write Amplification can result from several internal SSD operations.

### 14.1 Out-of-Place Writes

Updated data is written to a new physical page rather than directly overwriting the previous programmed page.

### 14.2 Garbage Collection

Valid pages may need to be copied before a block can be erased and reclaimed.

### 14.3 Wear Leveling

Relocating data to manage NAND wear can generate additional internal writes.

### 14.4 Limited Free Space

When fewer suitable free blocks are available, reclaiming space may require additional data movement.

### 14.5 Workload Characteristics

Workload patterns, including frequent random updates, can affect internal data placement and the amount of reclamation required.

The resulting WA depends on the workload, SSD design, firmware and operating state.

---

## 15. How Garbage Collection Creates Write Amplification

Consider a block containing:

```text
Block 30
 ├── Page 0 → Valid
 ├── Page 1 → Invalid
 ├── Page 2 → Valid
 └── Page 3 → Invalid
```

To reclaim it, the SSD must preserve the valid pages.

```text
Copy Valid Pages
        |
        v
Write Copies to Another Location
        |
        v
Update Mappings
        |
        v
Erase Original Block
```

The host did not request the copied pages as new user data, but those copies generate additional NAND writes.

This is one way Garbage Collection contributes to Write Amplification.

---

## 16. Why Write Amplification Matters

### Performance

Internal copying, programming and erasing consume NAND bandwidth and controller resources. These operations can increase latency or reduce throughput under some workloads.

### Endurance

Additional NAND programming and erase activity consumes some of the drive's finite endurance.

### Power and Internal Overhead

Extra internal work can consume additional power and processing resources.

### Endurance Relationship

```text
Host Workload
     |
     v
Internal NAND Operations
     |
     v
Write Amplification
     |
     v
Additional NAND Activity
     |
     v
NAND Wear
```

A higher WA generally means more NAND data is written for the same amount of host data.

However, WA alone does not fully determine the lifetime of an SSD. NAND technology, workload, wear distribution and the manufacturer's endurance specification also matter.

---

# PART 4 — OVER-PROVISIONING

## 17. What Is Over-Provisioning?

**Over-Provisioning (OP)** is reserving a portion of an SSD's physical NAND capacity for internal use rather than exposing all physical capacity as host-addressable storage.

The reserved capacity can support operations such as:

- Garbage Collection
- Wear Leveling
- Bad-block replacement
- Internal block management
- Maintaining performance under sustained workloads

The amount reserved and how it is used depend on the SSD design and configuration.

---

## 18. Over-Provisioning Example

Consider a hypothetical SSD with 1,000 GB of physical NAND capacity.

```text
Physical NAND Capacity = 1,000 GB

        |
        +-------------------------+
        |                         |
        v                         v
Host-Visible Capacity       Reserved Capacity
        960 GB                    40 GB
```

These numbers are illustrative, not specifications for a particular SSD.

The host sees the exposed capacity. Reserved capacity gives the SSD additional space for internal management.

---

## 19. Why Is Over-Provisioning Used?

### 19.1 Garbage Collection Support

Additional internal capacity provides flexibility for relocating valid data while reclaiming blocks.

### 19.2 Write Amplification

More available internal space can allow the SSD to reclaim blocks and place data more efficiently, potentially reducing unnecessary copying and WA.

### 19.3 Performance

A larger reserve can help maintain a supply of reusable blocks and reduce the need for urgent reclamation during active writes.

### 19.4 Endurance

If additional space reduces internal write activity, it can help reduce NAND wear.

### 19.5 Bad-Block Management

Reserved capacity can help provide replacement resources when blocks become unsuitable for normal use.

These are potential benefits, not guaranteed results for every workload. The effect depends on the SSD's firmware, workload, available capacity and configuration.

---

## 20. Over-Provisioning Trade-Off

Over-Provisioning has a capacity trade-off.

```text
More Capacity Reserved Internally
                 |
                 v
More Flexibility for Internal Management
                 |
                 v
Potential Benefits for GC / WA / Performance

                 |
                 v
Less Capacity Exposed to the Host
```

The appropriate amount of reserved capacity depends on the intended workload and drive requirements.

---

# PART 5 — TRIM / DEALLOCATE

## 21. What Is TRIM?

**TRIM** is a command mechanism used by the operating system to inform an SSD that specified logical blocks are no longer in use and do not need to be preserved as current data.

For NVMe, equivalent functionality is provided by the **Dataset Management command with the Deallocate attribute**.

For SATA, the device uses the ATA TRIM command.

The precise mechanism depends on the storage interface and device support.

---

## 22. Why Is TRIM Needed?

When a file is deleted, the file system can release its logical storage space. However, the SSD does not automatically know that the logical data is no longer needed unless it receives suitable information.

TRIM / Deallocate allows the host to notify the SSD about those unused logical blocks.

Conceptually:

```text
File Deleted
     |
     v
File System Identifies Released Logical Blocks
     |
     v
TRIM / Deallocate Notification
     |
     v
SSD Controller and Firmware
     |
     v
FTL Updates Internal State
     |
     v
Space Becomes Eligible for Reclamation
```

The command identifies logical blocks or ranges. It does not carry the actual user data.

---

## 23. What Happens Inside the SSD After TRIM?

After receiving the supported notification, the SSD updates its internal management information to reflect that the specified logical data is no longer required.

The corresponding physical storage becomes eligible for reclamation according to the SSD's policy.

```text
TRIM / Deallocate
        |
        v
Logical Data No Longer Needed
        |
        v
FTL Updates Internal State
        |
        v
Pages Become Reclaimable
        |
        v
Garbage Collection, When Appropriate
        |
        v
Block Erase and Reuse
```

The internal metadata representation and timing of reclamation vary by implementation.

---

## 24. Does TRIM Immediately Erase NAND?

**No.**

TRIM / Deallocate does not itself perform a NAND block erase.

It provides information about logical storage that is no longer needed.

Actual physical reclamation can occur later during Garbage Collection.

```text
TRIM / Deallocate
        |
        v
Notify SSD About Unused Logical Data
        |
        v
FTL Updates Internal State
        |
        v
Garbage Collection Later
        |
        v
Erase Block When Safe
```

The original physical data may remain present until the block is reclaimed. TRIM should not be treated as a secure-erasure or sanitization command.

---

## 25. TRIM vs Garbage Collection

| TRIM / Deallocate | Garbage Collection |
|---|---|
| Host-originated notification about unused logical blocks | Internal SSD process for reclaiming physical NAND space |
| Identifies logical data no longer needed | Preserves remaining valid pages and reclaims blocks |
| Does not itself erase NAND blocks | Performs block reclamation and erasure |
| Helps the SSD identify data that need not remain current | Returns erased blocks for future reuse |

The core distinction is:

```text
TRIM / Deallocate
    → Provides information

Garbage Collection
    → Reclaims physical storage
```

---

## 26. How Does TRIM Help Performance and Endurance?

TRIM helps the SSD identify logical data that is no longer needed.

This can reduce unnecessary data movement during Garbage Collection.

```text
TRIM / Deallocate
        |
        v
SSD Knows More About Unused Logical Data
        |
        v
GC May Need to Copy Less Valid Data
        |
        v
Potentially Fewer Internal Writes
        |
        v
Potentially Lower WA
        |
        v
Potential Performance and Endurance Benefits
```

The benefits depend on workload, filesystem support, command handling and SSD implementation.

---

# PART 6 — INTEGRATED SSD MANAGEMENT MODEL

## 27. How the Mechanisms Work Together

The five mechanisms address different but related SSD management problems.

```text
Host Writes
     |
     v
FTL Maps Logical Data
     |
     v
Out-of-Place Writes
     |
     v
Old Pages May Become Invalid
     |
     v
Garbage Collection Reclaims Blocks
     |
     +-----------------------------+
     |                             |
     v                             v
Write Amplification          Free Blocks
     |                             |
     v                             v
Additional NAND Activity     Wear Leveling
     |                             |
     v                             v
Endurance Consumption       Wear Distribution

Over-Provisioning
     |
     v
Additional Capacity for Internal Management

TRIM / Deallocate
     |
     v
Identifies Logical Data No Longer Needed
     |
     v
Helps Garbage Collection Reclaim Space
```

This is a conceptual relationship diagram, not a fixed execution sequence. The SSD firmware coordinates these mechanisms according to its implementation and operating conditions.

---

## 28. Impact on Latency, Throughput, Endurance and Steady-State Performance

The Master Syllabus requires these mechanisms to be connected to four important dimensions.

### 28.1 Latency

Latency is the time taken to complete an operation.

Garbage Collection and other internal activities can add work while a host request is waiting. More effective free-space management may reduce the need for urgent reclamation.

### 28.2 Throughput

Throughput is the amount of data transferred or processed over time.

Internal data movement consumes NAND bandwidth and controller resources. Under some workloads, this can reduce the resources available for host I/O.

### 28.3 Endurance

Endurance reflects the drive's ability to sustain writes over its specified life.

Additional NAND writes and erases consume endurance. Wear Leveling helps distribute wear, while OP and TRIM can help internal management operate more efficiently.

### 28.4 Steady-State Performance

Steady-state performance describes drive behavior after sustained operation under a defined workload rather than only during initial writes.

During sustained workloads, the SSD may perform repeated Garbage Collection, data movement and Wear Leveling activities.

These interactions can influence longer-term latency and throughput.

---

## 29. Quick Reference Table

| Mechanism | Primary Purpose | Important Relationship |
|---|---|---|
| Garbage Collection | Reclaim NAND blocks | Can generate additional writes and latency |
| Dynamic Wear Leveling | Distribute wear associated with new writes | Helps manage NAND endurance |
| Static Wear Leveling | Address wear imbalance involving static data | May require additional data movement |
| Write Amplification | Measure NAND writes relative to host writes | Relates to NAND activity and endurance |
| Over-Provisioning | Reserve capacity for internal SSD management | Supports GC, performance and endurance |
| TRIM / Deallocate | Notify SSD about unused logical blocks | Helps GC identify data that need not be preserved |

---

## 30. Module 7 — Key Takeaways

- Garbage Collection preserves valid pages, erases suitable blocks, and returns them for reuse.
- A victim block is selected for reclamation; it is not necessarily defective.
- Background and foreground Garbage Collection describe different operating conditions.
- Wear Leveling distributes NAND wear rather than eliminating it.
- Dynamic Wear Leveling focuses on new writes.
- Static Wear Leveling may relocate static data to help correct wear imbalance.
- Write Amplification is calculated as NAND data written divided by host data written.
- Garbage Collection and Wear Leveling can contribute to internal NAND writes.
- Over-Provisioning reserves physical capacity for internal SSD management.
- TRIM / Deallocate informs the SSD about logical data no longer needed; it does not itself erase NAND.
- These mechanisms interact and can influence latency, throughput, endurance and steady-state performance.
- Exact policies, thresholds, and internal sequencing depend on the SSD implementation.

---

