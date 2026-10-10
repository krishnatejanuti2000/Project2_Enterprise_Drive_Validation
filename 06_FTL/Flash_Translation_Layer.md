# MODULE 06 — FLASH TRANSLATION LAYER (FTL)


---

## 1. Objective

Understand how the Flash Translation Layer (FTL) connects host-visible logical addresses to physical NAND storage locations.

This module covers:

- Logical-to-physical mapping
- Mapping tables
- Page mapping
- Block management
- Metadata
- Mapping updates
- Valid and invalid pages
- Relationship between FTL, latency, Write Amplification, Garbage Collection, endurance, and steady-state performance

---

## 2. What Is the FTL?

The **Flash Translation Layer (FTL)** is an internal SSD mechanism that manages the mapping between logical addresses provided by the host and physical locations inside NAND Flash.

The host works with logical addresses, while the SSD internally manages NAND channels, dies, planes, blocks, and pages.

```text
Host Logical Address
        |
        v
SSD Controller
        |
        v
Flash Translation Layer
        |
        v
Physical NAND Location
```

For example:

```text
Logical Address: LBA 1000
             |
             v
Physical Location:
Block 20, Page 5
```

This physical address is illustrative and does not represent a particular SSD's actual NAND layout.

### Why Is the FTL Required?

The FTL allows the SSD to manage physical data placement independently of the logical addresses used by the host.

This is particularly important because NAND cannot simply overwrite an already-programmed page. Updated data may need to be stored at a new physical location.

The FTL maintains the mapping needed to locate the current data.

---

## 3. Logical-to-Physical Mapping

Logical-to-physical mapping is the relationship between host-visible logical addresses and physical NAND locations.

### Example

Suppose the SSD maintains the following illustrative mappings:

| Logical Address | Physical NAND Location |
|---|---|
| LBA 1000 | Block 20, Page 5 |
| LBA 1001 | Block 20, Page 6 |
| LBA 1002 | Block 45, Page 2 |

When the host requests LBA 1001, the SSD uses its mapping information to locate the corresponding physical data.

```text
Host requests LBA 1001
           |
           v
     SSD Controller
           |
           v
           FTL
           |
           v
     Mapping Lookup
           |
           v
   Block 20, Page 6
           |
           v
        NAND Read
```

The logical address identifies the data from the host's perspective. The physical location identifies where the SSD currently stores that data.

---

## 4. FTL Mapping Table

A **mapping table** maintains information relating logical addresses to physical NAND locations.

Conceptually:

| Logical Address | Mapped Physical Location |
|---|---|
| LBA 500 | Block 10, Page 4 |
| LBA 501 | Block 10, Page 5 |
| LBA 502 | Block 25, Page 2 |

The FTL uses this information to locate data and update its physical placement when required.

### Main Purposes

- Locate data for host read requests.
- Track the current physical location of logical data.
- Update mappings when data is rewritten or relocated.
- Support NAND block reclamation and internal data management.

The exact mapping-table organization, granularity, and storage mechanism depend on the SSD implementation.

---

## 5. Page Mapping

**Page mapping** is an FTL mapping approach in which logical mapping units are associated with physical NAND pages.

Conceptually:

```text
Logical Mapping Unit
          |
          v
    FTL Mapping Table
          |
          v
  Physical NAND Page
```

Example:

| Logical Mapping Unit | Physical NAND Location |
|---|---|
| Logical Page A | Block 10, Page 4 |
| Logical Page B | Block 20, Page 7 |
| Logical Page C | Block 15, Page 2 |

### Why Is Page Mapping Useful?

NAND supports page-level read and program operations, while erasing occurs at block level.

When data is updated, the SSD can program a new physical page and update the logical-to-physical mapping.

This allows the logical address to remain unchanged even when the physical location changes.

**Important:** A logical mapping unit must not automatically be assumed to have the same size as a physical NAND page. Logical address granularity and mapping design depend on the SSD implementation.

---

## 6. Block Management

The FTL and related SSD management mechanisms use information about NAND block availability and condition to manage physical storage.

Conceptually, blocks can have different states:

```text
Block A --> Contains Current Data
Block B --> Contains Valid and Invalid Pages
Block C --> Erased and Available for Reuse
Block D --> Marked Bad
```

Internal management information helps the SSD determine which blocks can be allocated and which require special handling.

### Important Information

| Information | Purpose |
|---|---|
| Free-block information | Identifies blocks available for future writes |
| Bad-block information | Identifies blocks excluded from normal allocation |
| Erase-count information | Tracks block erase activity |
| Valid/invalid page information | Helps determine which physical pages contain current data |

Block management is closely related to Garbage Collection and Wear Leveling, but these mechanisms have different responsibilities.

---

## 7. Metadata Management

**Metadata** is internal management information used by the SSD to track data placement and NAND state.

It is distinct from the actual user data stored in NAND.

Examples include:

- Logical-to-physical mapping information
- Free-block information
- Bad-block information
- Erase-count information
- Valid/invalid page information

Metadata supports correct data placement, mapping updates, block allocation, and internal NAND management.

The precise metadata structures and their storage locations vary by SSD design.

---

## 8. Mapping Updates

A **mapping update** changes the physical location associated with a logical address when data is rewritten or relocated.

### Example: Before the Update

```text
LBA 500
   |
   v
Block 10, Page 4
```

The host requests an update to LBA 500.

Because the existing NAND page cannot simply be overwritten, the SSD may program the new data at another physical location.

### New Physical Location

```text
Block 25, Page 2
```

### After the Mapping Update

```text
LBA 500
   |
   v
Block 25, Page 2
```

The host continues using LBA 500. The physical mapping changes internally.

After the update, the old physical page is no longer the current version and may become invalid.

The exact sequencing, metadata persistence, and recovery behavior depend on the SSD implementation.

---

## 9. Valid and Invalid Pages

These concepts are essential to understanding FTL behavior and Garbage Collection.

### 9.1 Valid Page

A valid page contains current data that the SSD must preserve.

Example:

```text
Block 20
 ├── Page 0 --> Valid
 ├── Page 1 --> Valid
 ├── Page 2 --> Valid
 └── Page 3 --> Valid
```

### 9.2 Invalid Page

An invalid page contains obsolete data that is no longer the current version associated with the relevant logical address.

For example, when LBA 500 is updated, the new data may be programmed into a different physical page and the old page becomes invalid.

```text
Old Physical Page --> Invalid
New Physical Page --> Current Data
```

### Important Distinction

An invalid page does not necessarily mean that the entire NAND block is bad.

A block containing invalid pages can still contain valid data.

```text
Block 20
 ├── Page 0 --> Valid
 ├── Page 1 --> Invalid
 ├── Page 2 --> Valid
 └── Page 3 --> Invalid
```

Garbage Collection may later preserve the valid pages, erase the block, and return it for reuse.

---

## 10. Complete FTL Write Example

Suppose the host repeatedly updates the same logical address, LBA 500.

### Initial Mapping

```text
LBA 500
   |
   v
Block 10, Page 4
```

### Step 1 — Host Sends an Update

The host sends a write request for LBA 500 with new data.

### Step 2 — FTL and Controller Process the Request

The SSD processes the request and determines suitable physical storage.

### Step 3 — New Data Is Programmed

The updated data is programmed into a suitable new physical page.

```text
New Location:
Block 25, Page 2
```

### Step 4 — Mapping Information Is Updated

The current mapping changes to:

```text
LBA 500
   |
   v
Block 25, Page 2
```

### Step 5 — Old Page Becomes Invalid

The old physical page may be marked invalid because it no longer contains the current version.

### Step 6 — Block Reclamation Happens Later

The old block may still contain other valid pages. Garbage Collection can later preserve those pages, erase the block, and return it for reuse.

### Complete Conceptual Flow

```text
Host Write to LBA 500
          |
          v
     SSD Controller
          |
          v
           FTL
          |
          v
 Determine Physical Destination
          |
          v
    Program New Page
          |
          v
    Update Mapping
          |
          v
 Old Page Becomes Invalid
          |
          v
 Garbage Collection Later
          |
          v
    Block Reclaimed
```

This demonstrates why the FTL is central to SSD data placement and management.

---

## 11. FTL and Latency

**Latency** is the time taken to complete an operation.

FTL mapping and related internal work can contribute to request-processing overhead.

```text
Host Request
     |
     v
FTL Mapping Lookup
     |
     v
Locate Physical Data
     |
     v
Required NAND Operation
     |
     v
Request Completion
```

The actual latency also depends on NAND behavior, workload, controller resources, caching, queueing, and other factors.

Mapping work is therefore one part of the overall latency picture, rather than the only source of delay.

---

## 12. FTL and Write Amplification

Write Amplification occurs when the SSD writes more data to NAND than the host requested.

The formula is:

```text
WA = NAND Data Written / Host Data Written
```

For example:

```text
Host Data Written = 100 GB
NAND Data Written = 150 GB

WA = 150 / 100

WA = 1.5
```

The SSD wrote 1.5 times as much data to NAND as the host supplied during the measured period.

### FTL Relationship

Out-of-place writes can leave old physical pages invalid. Garbage Collection may need to copy valid pages before erasing their blocks.

That additional copying can contribute to Write Amplification.

```text
Host Writes
     |
     v
Out-of-Place Updates
     |
     v
Invalid Pages
     |
     v
Garbage Collection
     |
     v
Additional NAND Writes
     |
     v
Write Amplification
```

Not every host write triggers Garbage Collection. The amount of internal work depends on the workload and the SSD's implementation.

---

## 13. FTL and Garbage Collection

FTL and Garbage Collection cooperate.

### FTL

Maintains logical-to-physical mapping information and updates mappings when data moves.

### Garbage Collection

Reclaims physical NAND blocks containing invalid pages.

### Combined Flow

```text
Invalid Pages Accumulate
          |
          v
Select Block for Reclamation
          |
          v
Copy Remaining Valid Pages
          |
          v
Update Affected FTL Mappings
          |
          v
Erase Original Block
          |
          v
Return Block for Reuse
```

Garbage Collection reclaims space; the FTL mapping information allows the SSD to continue locating data when physical locations change.

---

## 14. FTL and Endurance

NAND supports a limited amount of Program/Erase activity.

FTL data placement and internal management operations influence the amount of work performed on NAND.

```text
Host Workload
     |
     v
FTL Data Placement
     |
     v
Internal NAND Operations
     |
     v
Program / Erase Activity
     |
     v
NAND Wear
     |
     v
Endurance Consumption
```

Write Amplification can increase internal NAND write activity, while Wear Leveling helps distribute wear across usable blocks.

The FTL and related management mechanisms do not eliminate NAND's physical endurance limits.

---

## 15. FTL and Steady-State Performance

An SSD's performance can change as a workload continues.

Early in a workload, suitable free blocks may be readily available. Under sustained writes, the SSD may need to reclaim space, copy valid pages, update mappings, and perform other internal maintenance.

```text
Initial Workload
       |
       v
Sustained Host Writes
       |
       v
Internal Reclamation and Data Movement
       |
       v
Changing Latency and Throughput
       |
       v
Longer-Term Operating Pattern
```

Steady-state performance describes behavior after the drive has been operating under sustained workload conditions, rather than measuring only its initial performance.

FTL decisions, Garbage Collection, Write Amplification, available free space, and firmware policies can all influence this behavior.

Performance does not necessarily become perfectly constant at steady state.

---

## 16. FTL — Key Concepts

| Concept | Main Purpose |
|---|---|
| Logical-to-physical mapping | Connects host addresses to physical NAND locations |
| Mapping table | Stores mapping information |
| Page mapping | Maps logical units to physical pages in a page-mapped design |
| Block management | Tracks block availability and condition |
| Metadata | Stores internal management information |
| Mapping updates | Keeps physical mappings current when data moves |
| Valid pages | Contain data that must be preserved |
| Invalid pages | Contain obsolete data that may be reclaimed |
| Garbage Collection | Reclaims reusable NAND blocks |
| Write Amplification | Measures NAND writes relative to host writes |

---

## 17. Module 6 — Key Takeaways

- The FTL connects logical addresses used by the host to physical NAND locations.
- Mapping tables help the SSD locate current data.
- Page mapping is an FTL approach; mapping granularity varies by implementation.
- Block-management information supports allocation and NAND reuse.
- Metadata tracks logical/physical relationships and relevant NAND state.
- Mapping updates allow the physical location to change without changing the host-visible LBA.
- Valid pages contain current data; invalid pages contain obsolete data.
- FTL behavior connects to latency, Write Amplification, Garbage Collection, endurance, and steady-state performance.
- FTL and Garbage Collection cooperate but perform different functions.

### Core Mental Model

```text
Host Logical Address
          |
          v
     SSD Controller
          |
          v
           FTL
          |
          v
  Mapping Information
          |
          v
 Physical NAND Location
          |
          v
 Read / Program / Relocation
          |
          v
   Mapping Update
          |
          v
 Garbage Collection Later
          |
          v
   Block Reuse
```

---


