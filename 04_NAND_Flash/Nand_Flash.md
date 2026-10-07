# MODULE 04 — NAND FLASH

## Priority: P0

---

# 1. Overview

NAND Flash is the primary non-volatile storage medium used inside an SSD.

Unlike HDDs, which store data on magnetic media using mechanical components, NAND Flash stores information electronically in semiconductor memory cells.

The SSD Controller and Firmware manage NAND Flash internally to perform:

- Read
- Program
- Erase
- Data placement
- Error handling
- Block management
- Reliability management

The NAND organization used throughout this module is:

```text
SSD
 ↓
Controller
 ↓
Channels
 ↓
Dies
 ↓
Planes
 ↓
Blocks
 ↓
Pages
 ↓
Cells
```

---

# 2. NAND Cell

A **NAND cell** is the basic physical storage element of NAND Flash.

The cell stores an electrical state that represents information.

NAND Flash is **non-volatile**, meaning the stored information is retained even when power is removed.

Cells are organized into larger structures that allow the SSD Controller to read, program, and erase NAND efficiently.

Conceptually:

```text
Cell
 ↓
Multiple Cells
 ↓
Page
 ↓
Block
 ↓
Plane
 ↓
Die
 ↓
Channel
```

The actual electrical implementation of the cell may vary by NAND technology.

---

# 3. NAND Page

A **page** is a group of NAND cells that forms a fundamental unit for NAND read and program operations.

The important rule is:

```text
Read    → Page
Program → Page
```

A page should not be confused with a host logical block.

The host works with logical addresses such as LBAs, while the SSD internally manages physical NAND pages.

Therefore:

```text
Host Logical Address
        ↓
SSD Internal Mapping
        ↓
Physical NAND Page
```

Page size and organization can vary with NAND implementation.

---

# 4. NAND Block

A **block** is a collection of NAND pages.

The most important property of a block is:

```text
Erase → Block
```

For example:

```text
Block
 ├── Page 0
 ├── Page 1
 ├── Page 2
 ├── Page 3
 └── ...
```

Read and program operations are performed at page level, but erase occurs at block level.

This difference is fundamental to understanding SSD internal data management.

```text
Read    → Page
Program → Page
Erase   → Block
```

---

# 5. NAND Plane

A **plane** is a subdivision within a NAND die.

A plane contains multiple blocks and forms part of the internal organization of NAND Flash.

Conceptually:

```text
Die
 ↓
Plane
 ↓
Blocks
 ↓
Pages
```

Planes also support internal parallelism depending on the NAND/controller implementation.

---

# 6. NAND Die

A **die** is an individual NAND memory unit.

A die contains internal NAND structures such as:

```text
Die
 ↓
Planes
 ↓
Blocks
 ↓
Pages
 ↓
Cells
```

A NAND package may contain one or more dies.

Multiple dies can provide opportunities for internal parallel operation.

---

# 7. NAND Channel

A **NAND channel** is a communication path between the SSD Controller and NAND resources.

An SSD may contain multiple NAND channels.

Conceptually:

```text
SSD Controller
      │
 ┌────┼────┬────┐
 ↓    ↓    ↓    ↓
Ch0  Ch1  Ch2  Ch3
```

Multiple channels allow the controller to communicate with different NAND resources and can provide internal parallelism.

This can contribute to higher bandwidth and improved overall performance.

The exact channel/package topology varies by SSD design.

---

# 8. Complete NAND Organization

The complete logical hierarchy used in this module is:

```text
SSD
 ↓
Controller
 ↓
Channel
 ↓
Die
 ↓
Plane
 ↓
Block
 ↓
Page
 ↓
Cell
```

A useful way to remember the hierarchy is:

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
   ↓
 Cell
```

---

# 9. NAND Cell Types

NAND Flash can store different numbers of bits per cell.

## 9.1 SLC — Single-Level Cell

SLC stores:

```text
1 bit / cell
```

Therefore:

```text
2¹ = 2 states
```

---

## 9.2 MLC — Multi-Level Cell

MLC stores:

```text
2 bits / cell
```

Therefore:

```text
2² = 4 states
```

---

## 9.3 TLC — Triple-Level Cell

TLC stores:

```text
3 bits / cell
```

Therefore:

```text
2³ = 8 states
```

---

## 9.4 QLC — Quad-Level Cell

QLC stores:

```text
4 bits / cell
```

Therefore:

```text
2⁴ = 16 states
```

---

## 9.5 Summary

| NAND Type | Bits per Cell | Possible States |
|---|---:|---:|
| SLC | 1 | 2 |
| MLC | 2 | 4 |
| TLC | 3 | 8 |
| QLC | 4 | 16 |

The general relationship is:

```text
Number of states = 2^(bits per cell)
```

---

# 10. NAND Operations

NAND Flash fundamentally performs three important operations:

```text
Read
Program
Erase
```

Their units are:

```text
Read
 ↓
Page
```

```text
Program
 ↓
Page
```

```text
Erase
 ↓
Block
```

This is one of the most important rules in NAND Flash.

---

# 11. NAND READ

A NAND **read** retrieves the contents of a NAND page.

Conceptually:

```text
NAND Block
    ↓
Select Page
    ↓
Read Page
    ↓
Data Retrieved
```

The SSD Controller identifies the required physical NAND location and initiates the appropriate internal read operation.

The retrieved data is then processed by the SSD's error-correction mechanism before being returned through the host interface.

---

# 12. NAND PROGRAM

A NAND **program** operation writes data into a NAND page.

Conceptually:

```text
Data
 ↓
NAND Controller Logic
 ↓
Select Page
 ↓
Program Page
```

The term **program** refers specifically to the NAND-level operation.

Therefore:

```text
Host WRITE
      ≠
NAND PROGRAM
```

A host write is a command/request at the storage interface level.

The SSD internally converts that request into the operations necessary to program NAND.

---

# 13. NAND ERASE

NAND erase occurs at **block level**.

Conceptually:

```text
Block
 ├── Page 0
 ├── Page 1
 ├── Page 2
 └── Page 3

        ↓

      ERASE

        ↓

Block becomes reusable
```

The entire erase unit is the block.

This is why NAND cannot simply erase one individual page while preserving the other pages in the same block.

---

# 14. Why NAND Cannot Simply Overwrite a Programmed Page

This is the fundamental NAND question for this module.

Consider:

```text
Block 20
 ├── Page 0 → Valid
 ├── Page 1 → Valid
 ├── Page 2 → Valid
 └── Page 3 → Valid
```

Suppose the data in Page 1 must be changed.

A simple overwrite would require:

```text
Overwrite Page 1
```

But NAND erase operates at block level.

The SSD cannot simply do:

```text
Erase Page 1
```

because the erase unit is:

```text
Entire Block 20
```

Erasing Block 20 would also erase the other valid pages.

Therefore, the SSD uses an **out-of-place update** approach.

Conceptually:

```text
Old Page
   ↓
Updated data written to another physical page
   ↓
New Page becomes current
   ↓
Old Page becomes invalid
```

Later, Garbage Collection can reclaim the old block by moving remaining valid data and erasing the block.

The key relationship is:

```text
Program → Page
Erase   → Block
```

Therefore:

> NAND cannot simply overwrite an already-programmed page because reusing the page requires block-level erase, which would also affect the other pages in that block.

---

# 15. Out-of-Place Write

When existing logical data is updated, the SSD can write the new version to a different physical NAND page.

Conceptually:

```text
Before:

Block A
 ├── Page 0 → LBA X
 ├── Page 1 → LBA Y
 └── Page 2 → LBA Z
```

After updating LBA Y:

```text
New physical location
 └── New Page → LBA Y

Old Page 1
 └── Invalid
```

The SSD's internal mapping metadata is updated so that the logical address points to the current physical location.

This mechanism enables the SSD to manage NAND without performing a direct in-place overwrite.

---

# 16. Invalid Page vs Bad Block

These two concepts must not be confused.

## Invalid Page

An invalid page means the data currently stored there is no longer the active valid version.

Example:

```text
Block 50
 ├── Page 0 → Valid
 ├── Page 1 → Invalid
 ├── Page 2 → Valid
 └── Page 3 → Valid
```

The block can still contain useful valid pages.

Garbage Collection can later move the valid pages and reclaim the block.

---

## Bad Block

A bad block is a block considered unreliable and therefore removed from normal allocation.

Example:

```text
Block 50
   ↓
Unreliable
   ↓
Marked Bad
   ↓
Removed from normal allocation
```

Therefore:

```text
Invalid Page
→ Page-level data is no longer current

Bad Block
→ The block itself is considered unreliable
```

---

# 17. Bad Block Management

Not every NAND block remains usable for the entire life of the SSD.

Bad blocks can occur because:

- A block is defective from manufacturing.
- A block becomes unreliable after NAND wear.
- A block develops excessive bit errors.

The SSD firmware includes **Bad Block Management** to prevent such blocks from normal allocation.

Conceptually:

```text
NAND Block
   ↓
Health/Integrity Assessment
   ↓
Unreliable
   ↓
Marked Bad
   ↓
Removed from normal allocation
```

Bad Block Management is therefore an important SSD reliability mechanism.

---

# 18. Program/Erase (P/E) Cycles

A **P/E cycle** represents program-and-erase activity involving NAND.

Conceptually:

```text
PROGRAM
   ↓
Data programmed into NAND
   ↓
ERASE
   ↓
Block becomes reusable
```

Because NAND blocks support only a limited amount of P/E activity, repeated P/E operations contribute to NAND wear.

The relationship is:

```text
P/E Activity
   ↓
NAND Wear
   ↓
Reliability Challenges
   ↓
Finite SSD Endurance
```

The erase portion is performed at block level.

---

# 19. Host Writes vs P/E Cycles

A host write is not necessarily equivalent to one P/E cycle.

For example:

```text
Host
 ↓
WRITE
 ↓
SSD
```

Internally, the SSD may perform additional operations associated with data placement and NAND management.

Therefore:

```text
Host Writes
   ≠
P/E Cycles on a 1:1 basis
```

This distinction becomes especially important when understanding:

- Write Amplification
- Garbage Collection
- Wear Leveling
- SSD endurance

These topics are connected but are not covered in full in this NAND module.

---

# 20. SSD Endurance

**SSD endurance** describes the amount of data an SSD is designed to reliably write over its specified useful life.

Common endurance metrics include:

```text
TBW
DWPD
```

---

## 20.1 TBW — Terabytes Written

TBW represents the specified total amount of data that an SSD is rated to write over its endurance period.

Conceptually:

```text
SSD
 ↓
Total rated writes
 ↓
TBW
```

The actual rating depends on the manufacturer's qualification conditions and workload assumptions.

---

## 20.2 DWPD — Drive Writes Per Day

DWPD expresses the rated number of complete drive-capacity writes that can be sustained per day over the specified endurance period.

For example:

```text
1 TB Drive
3 DWPD

≈ 3 TB/day
```

under the applicable endurance specification.

---

## 20.3 Endurance Relationship

The overall relationship is:

```text
Workload
   ↓
Internal NAND Activity
   ↓
P/E Wear
   ↓
NAND Aging
   ↓
SSD Endurance Consumption
```

Factors such as workload type, write amplification, wear leveling, garbage collection, over-provisioning, temperature, and NAND technology can influence SSD lifetime.

---

# 21. ECC — Error Correction Code

NAND Flash is susceptible to bit errors.

These errors can result from factors such as:

- NAND wear
- Charge-related effects
- Read disturb
- Other flash-specific phenomena

**ECC (Error Correction Code)** is the mechanism used to detect and correct recoverable bit errors.

Conceptually:

```text
NAND Page Read
      ↓
Possible Bit Errors
      ↓
ECC
      ↓
Detect Errors
      ↓
Correct Recoverable Errors
      ↓
Valid Data
```

ECC therefore helps maintain:

- Data integrity
- Reliability
- Correct data delivery

---

# 22. ECC Correction Capability

ECC cannot necessarily correct an unlimited number of errors.

Conceptually:

```text
No Errors
   ↓
Data accepted
```

```text
Recoverable Errors
   ↓
ECC corrects
   ↓
Data accepted
```

```text
Errors exceed correction capability
   ↓
Unrecoverable Error
   ↓
Read failure/error reported
```

Thus:

```text
ECC
→ Corrects recoverable errors

Not:
→ Guarantees that every error can always be corrected
```

---

# 23. ECC and NAND Wear

As NAND ages, the error environment can become more challenging.

The relationship can be represented as:

```text
P/E Activity
      ↓
NAND Wear
      ↓
Bit Errors Can Increase
      ↓
ECC Becomes Important
      ↓
Recoverable Errors Corrected
```

ECC is therefore an essential reliability mechanism between NAND storage and correct data delivery.

---

# 24. LDPC — Low-Density Parity-Check

**LDPC (Low-Density Parity-Check)** is a type of error-correcting code that can be used as part of an SSD's ECC/error-control implementation.

The conceptual flow is:

```text
Data
 ↓
Error-correction coding
 ↓
NAND Storage
 ↓
Read Data
 ↓
LDPC / ECC Processing
 ↓
Recover Intended Data
```

LDPC does not prevent NAND from developing errors.

Instead, it provides an error-correction mechanism for recovering data when errors are within the correction capability.

---

# 25. ECC vs LDPC

Keep the distinction clear:

| Concept | Meaning |
|---|---|
| ECC | Error detection/error-correction mechanism |
| LDPC | A specific error-correcting code used within an ECC implementation |

Conceptually:

```text
ECC
 │
 └── Error-control mechanism
       │
       └── LDPC
```

The exact ECC/LDPC architecture, decoding algorithms, correction strength, and implementation are controller/vendor dependent.

---

# 26. NAND Reliability Chain

The major concepts learned in this module connect together:

```text
P/E Cycles
      ↓
NAND Wear
      ↓
Bit Errors
      ↓
ECC / LDPC
      ↓
Recoverable Errors Corrected
      ↓
Reliable Data
```

When a block becomes unreliable:

```text
NAND Wear / Defects
      ↓
Unreliable Block
      ↓
Bad Block Management
      ↓
Removed from Normal Allocation
```

Therefore, NAND reliability involves multiple mechanisms working together.

---

# 27. Module 4 — Complete Mental Model

The complete NAND model is:

```text
                 NAND FLASH

                     │
                     ▼
                  CHANNEL
                     │
                     ▼
                    DIE
                     │
                     ▼
                   PLANE
                     │
                     ▼
                   BLOCK
                     │
                     ▼
                   PAGE
                     │
                     ▼
                   CELL
```

Operations:

```text
READ
 ↓
PAGE

PROGRAM
 ↓
PAGE

ERASE
 ↓
BLOCK
```

Reliability:

```text
Program / Erase
      ↓
P/E Wear
      ↓
NAND Aging
      ↓
Bit Errors
      ↓
ECC / LDPC
      ↓
Correct Recoverable Errors
```

Block reliability:

```text
Defect / Wear
      ↓
Bad Block
      ↓
Bad Block Management
      ↓
Removed from Normal Allocation
```

Update behavior:

```text
Host Data Update
      ↓
New Physical Page
      ↓
Old Page Invalid
      ↓
Later Block Reclamation
```

---

# 28. Module 4 — Key Takeaways

The following concepts must be clear before moving to the next module:

### NAND Organization

```text
Channel → Die → Plane → Block → Page → Cell
```

### NAND Operations

```text
Read    → Page
Program → Page
Erase   → Block
```

### NAND Types

```text
SLC → 1 bit/cell
MLC → 2 bits/cell
TLC → 3 bits/cell
QLC → 4 bits/cell
```

### NAND Update Model

```text
Cannot directly overwrite programmed page
        ↓
Out-of-place update
        ↓
Old page becomes invalid
        ↓
Block later reclaimed
```

### Reliability

```text
P/E Cycles
    ↓
NAND Wear
    ↓
Bit Errors
    ↓
ECC / LDPC
```

### Bad Blocks

```text
Defective / unreliable block
        ↓
Bad Block Management
        ↓
Removed from normal allocation
```

### Endurance

```text
NAND Wear
   ↓
Finite SSD Endurance
   ↓
Measured using metrics such as TBW / DWPD
```

---

# 29. Module 4 Completion Status

```text
NAND Cell                 ✅
NAND Page                 ✅
NAND Block                ✅
NAND Plane                ✅
NAND Die                  ✅
NAND Channel              ✅

SLC                       ✅
MLC                       ✅
TLC                       ✅
QLC                       ✅

NAND Read                 ✅
NAND Program              ✅
NAND Erase                ✅

Erase-Before-Write        ✅
Bad Blocks                ✅
P/E Cycles                ✅
SSD Endurance             ✅
ECC                       ✅
LDPC Concept              ✅

Why NAND Cannot
Overwrite a Page         ✅
```

**Module 4 — NAND Flash: COMPLETE**

Next module in the Master Syllabus:

```text
MODULE 05 — SSD INTERNAL DATA PATH
```

Its focus is the internal processing performed by the SSD Controller, including command processing, data movement, address translation, NAND scheduling, metadata, ECC, error handling, Garbage Collection, Wear Leveling, and firmware.
