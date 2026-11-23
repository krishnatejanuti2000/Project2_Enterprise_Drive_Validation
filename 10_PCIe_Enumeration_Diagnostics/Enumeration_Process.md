# **PCIe Device Enumeration — Concepts, Process and Practical Validation**

**Project:** Project 2 — Enterprise NVMe SSD Drive Validation  
**Domain:** Domain 3 — NVMe Architecture and Protocol  
**Host Bring-up Stage:** Stage 2 — PCIe Device Enumeration  
**Learning Status:** In Progress

## **Objective of PCIe Device Enumeration**

A server may contain several PCIe devices:

- NVMe SSDs
- RAID controllers
- Host Bus Adapters (HBAs)
- Ethernet controllers
- Graphics cards
- PCIe switches
- PCIe bridges

During server power-on, platform firmware (UEFI/BIOS) commonly performs initial PCIe discovery and configures enough of the PCIe hierarchy for the platform to boot. After control passes to the operating system, the OS PCI subsystem discovers or adopts that hierarchy, checks the discovered Functions and resources, and may preserve or reassign configuration as required. The exact division of work depends on the platform firmware, operating system and boot configuration.

This overall discovery and configuration activity is called **PCIe Device Enumeration**.

The activities involved include:

- Discovering PCIe Functions and reading their identification information.
- Identifying non-bridge Functions and PCI-to-PCI Bridge Functions.
- Discovering the PCIe hierarchy and the buses reachable through bridges.
- Assigning or validating bus numbers for bridges.
- Discovering relevant standard and extended capabilities.
- Determining, assigning or validating BAR resources.
- Configuring bridge resource windows so downstream resources can be reached.
- Enabling PCI resources and bus mastering at the appropriate stage of platform or OS initialization.
- Publishing discovered PCI Functions in the operating system's PCI device inventory.

**Important scope distinction:** PCIe enumeration makes a PCI Function visible and configured in the PCI hierarchy. It does not, by itself, prove that an NVMe controller has completed NVMe initialization, that namespaces have been discovered, or that a Linux block device such as `/dev/nvme0n1` is available. Those depend on subsequent driver binding and NVMe initialization.

### **Why is enumeration required for NVMe SSD validation?**

Consider an enterprise server with multiple NVMe SSDs connected through PCIe Root Ports and PCIe switches.

Before Linux can bind the NVMe driver to a controller, the PCIe Function representing that controller must be discovered and made accessible through the PCI hierarchy.

If the Function is not enumerated, it will normally be absent from the OS PCI inventory. However, a Function can appear in `lspci` even when the NVMe driver later fails to bind or the controller fails during NVMe initialization. In that situation, PCIe enumeration succeeded far enough to expose the Function, but NVMe host bring-up did not complete.

Therefore, PCIe enumeration is an important part of NVMe host bring-up and an important failure-isolation area for SSD validation engineers.

**Important distinction:** PCIe enumeration discovers and configures the PCIe Function. PCI driver matching, driver binding, NVMe Controller Initialization, namespace discovery and block-device creation are subsequent stages of host bring-up.

---

## **PCIe Enumeration — Overall Process**

The following is a working conceptual sequence, not a rigid register-write script. Platform firmware and the operating system may divide the work differently, and some operations can be interleaved.

```text
PCIe Link Establishment
        |
        | (Covered in the other learning chat)
        |
        v
Platform firmware / OS PCI subsystem
        |
        v
Scan the Root Bus and discover Functions
        |
        v
Read Vendor ID, Device ID, Revision ID
        |
        v
Read Class Code and Header Type
        |
        v
Discover applicable Functions and capabilities
        |
        v
Identify the Function type
        |
        +-----------------------------+
        |                             |
        v                             v
 Non-bridge Function             Bridge Function
        |                             |
 Record Function                 Validate or assign
        |                          Primary / Secondary /
        |                          temporary Subordinate
        |                             |
        |                         Scan Secondary Bus
        |                             |
        |                      Repeat for any bridges
        |                             |
        +--------------+--------------+
                       |
                       v
            Continue scanning current bus
                       |
                       v
        Complete reachable PCIe hierarchy scan
                       |
                       v
        Determine / assign / validate BAR resources
                       |
                       v
        Configure bridge windows for downstream resources
                       |
                       v
        Complete required PCI configuration and publish
        discovered Functions in the OS PCI inventory
                       |
                       v
        Later: PCI driver matching and driver binding
```

**Important ordering note:** Capability discovery is performed as each Function is examined; it is not necessarily a final step after BAR allocation. Resource assignment and bridge-window configuration can also be interleaved by the implementation. MSE and BME may be set by firmware or later by OS/driver initialization, so they should not be treated as proof that enumeration itself performed those writes.

**Our current learning boundary:** We have covered Function discovery and identification, conventional multifunction discovery, bridge recursion, PCIe switch branches and Subordinate Bus Number finalization. We have not yet completed PCI resource allocation, bridge resource-window configuration or the detailed capability-discovery process.

---

## **PCI Configuration Space**

### **What is PCI Configuration Space?**

Every PCIe Function has its own PCI Configuration Space.

It contains information that allows the host to identify, configure and manage that Function.

Examples of information available in Configuration Space:

| Information | Purpose |
|---|---|
| Vendor ID | Identifies the manufacturer |
| Device ID | Identifies the device |
| Revision ID | Identifies the device revision |
| Class Code | Identifies the device category |
| Command Register | Controls certain PCI operations |
| Status Register | Reports PCI status information |
| BARs | Identify resource type/attributes and contain the programmed base address; resource size is determined separately |
| Capability structures | Describe supported PCI capabilities |

A PCIe Function provides a 4 KB Configuration Space:

- Offsets `0x000`–`0x0FF` (first 256 bytes): the conventional PCI-compatible Configuration Space.
- Offsets `0x100`–`0xFFF` (remaining 3,840 bytes): PCI Express Extended Configuration Space.

Each Function has its own independent Configuration Space. Standard and extended capability structures are discovered by following the relevant capability lists; they are not all located in one fixed sequence.

### **Practical — Display PCIe device inventory**

**Command:**

```bash
sudo lspci -D -nn
```

**Why do we use it?**

This command displays PCI devices and Functions currently discovered by the operating system.

Options:

| Option | Purpose |
|---|---|
| `-D` | Display PCI domain numbers |
| `-nn` | Display numeric Vendor ID, Device ID and Class Code |

Your actual NVMe SSD output:

```text
10000:e1:00.0 Non-Volatile memory controller [0108]:
SK hynix Gold P31/BC711/PC711 NVMe Solid State Drive [1c5c:174a]
```

Interpretation:

| Field | Value | Meaning |
|---|---|---|
| Domain | `10000` | PCI domain |
| Bus | `e1` | Bus number |
| Device | `00` | Device number |
| Function | `0` | Function number |
| Vendor ID | `1c5c` | SK hynix |
| Device ID | `174a` | Device identifier |
| Class Code | `0108` | Non-Volatile Memory Controller |

The BDF format is:

```text
Domain:Bus:Device.Function

10000:e1:00.0
```

The Domain is also commonly called the PCI segment or PCI domain number. A BDF identifies a Function within that PCI domain; the domain distinguishes separate PCI hierarchies that may reuse the same Bus/Device/Function numbers.

**Why is BDF important?**

Every PCIe Function needs a unique address within its PCI domain so the host can issue Configuration Space requests to the correct Function.

---

## **PCIe Functions and Multifunction Devices**

### **What is a PCIe Function?**

A PCIe Function is an independently identifiable and configurable unit of a PCIe device.

A physical device may contain one Function or multiple Functions.

Examples:

| Physical PCIe device | Possible Functions |
|---|---|
| NVMe SSD | NVMe controller Function |
| Graphics card | Graphics Function, audio Function |
| Ethernet adapter | One or more network Functions |
| HBA | One or more controller Functions |
| PCIe switch | Upstream and Downstream Port bridge Functions |

Each PCIe Function has:

- Its own BDF address.
- Its own PCI Configuration Space.
- Its own identification information.

For example:

```text
10000:e1:00.0
```

Here, `.0` identifies Function 0.

### **How does the host identify a Function?**

The host performs Configuration Space reads.

The Vendor ID is located at Configuration Space offset `0x00`.

If the host receives a valid Vendor ID, it identifies a Function at that BDF.

For a normal configuration read, a Vendor ID value of `0xFFFF` indicates that no Function is responding at that candidate address. Access failures or platform-specific configuration-access problems should still be distinguished from a genuine empty slot when investigating faults.

**Practical command:**

```bash
sudo setpci -s 10000:e1:00.0 0x00.w
```

Expected value from your SSD:

```text
1c5c
```

Read the Device ID:

```bash
sudo setpci -s 10000:e1:00.0 0x02.w
```

Expected:

```text
174a
```

Read the Class Code base-class and subclass bytes:

```bash
sudo setpci -s 10000:e1:00.0 0x0a.w
```

Your actual result:

```text
0108
```

The PCI Class Code is three bytes wide: Programming Interface, Subclass and Base Class. This 16-bit read at offset `0x0A` returns the Subclass and Base Class bytes (shown as `0108`); it does not include the Programming Interface byte at offset `0x09`.

These commands are Configuration Space reads. They do not modify the device.

### **How does the host discover multiple Functions?**

In the conventional PCI/PCIe non-ARI addressing model, one Device Number can contain up to eight Functions, numbered 0 through 7.

The Header Type field is located at Configuration Space offset `0x0E`.

Its bit 7 indicates whether the device is multifunction.

| Bit 7 | Meaning |
|---|---|
| `0` | Single-function device |
| `1` | Multifunction device |

For conventional multifunction discovery, the host examines **Function 0's Header Type**.

- If bit 7 of Function 0 is clear, the host treats that device number as single-function and does not probe Functions 1–7.
- If bit 7 of Function 0 is set, the host checks candidate Functions 1 through 7 by reading their Vendor IDs.

Each additional Function with a valid Vendor ID is discovered and identified independently.

The implemented Functions need not be consecutive.

**Important:** The multifunction discovery decision is based on Function 0's Header Type bit 7. Reading the Header Type of Function 4 does not replace this check.

**Practical command:**

```bash
sudo setpci -s 10000:e1:00.0 0x0e.b
```

Your actual NVMe result:

```text
00
```

This represents a Type 0 header with bit 7 clear.

A more important example from your Root Port:

```bash
sudo setpci -s 10000:e0:1c.4 0x0e.b
```

Your actual result:

```text
81
```

Interpretation:

```text
Hex:       81
Binary:    10000001
           |      |
           |      +-- Header layout: Type 1
           +--------- Bit 7: Set in the read Function's Header Type
```

Your inventory also showed:

```text
10000:e0:1c.0
10000:e0:1c.4
```

This confirms that multiple Functions are present at the same domain, bus and device number.

The Header Type value `81` from Function 4 establishes its Type 1 header layout. However, the multifunction bit of Function 0 must be inspected to verify the conventional multifunction discovery decision.

For a rigorous multifunction-bit check, inspect Function 0's Header Type:

```bash
sudo setpci -s 10000:e0:1c.0 0x0e.b
```

We have not yet captured the output of this command. The inventory already shows Function `0` and Function `4` at this Device number, so Function `4` is present in the current OS PCI inventory. In the conventional non-ARI model, Function 0's multifunction bit is the standard indication that Functions 1–7 may be present; capturing this read will document that bit directly rather than relying on Function 4's own Header Type value.

**PCIe-specific clarification:**

The above explanation describes conventional non-ARI Function numbering.

Advanced features such as Alternative Routing-ID Interpretation (ARI), when supported and enabled through the hierarchy, change the Routing ID interpretation so that the Function Number can use an extended 8-bit range (up to 256 Function Numbers) instead of the conventional 3-bit Function field. The conventional Device/Function split is not used in the same way under ARI.

ARI is outside our current learning scope.

---

## **Identifying Endpoint and Bridge Functions**

After discovering a Function, the host needs to determine what type of Function it is.

Two important fields are:

- Class Code.
- Header Type.

### **Class Code**

Class Code identifies the general category of a PCI Function. The standard Class Code field contains three bytes: Base Class, Subclass and Programming Interface. The `lspci` labels shown below display the Base Class and Subclass pair.

Examples:

| Class Code | Meaning |
|---|---|
| `0108` | Non-Volatile Memory Controller |
| `0200` | Ethernet Controller |
| `0604` | PCI-to-PCI Bridge |

Your NVMe SSD:

```text
Class Code = 0108
```

Your Intel Root Port:

```text
Class Code = 0604
```

**Practical command:**

```bash
sudo setpci -s 10000:e1:00.0 0x0a.w
sudo setpci -s 10000:e0:1c.4 0x0a.w
```

Previously observed results:

```text
0108
0604
```

### **Header Type**

Header Type determines the Configuration Space header layout.

| Header Type lower 7 bits | Header layout |
|---|---|
| `00` | Type 0 — Non-bridge device header (commonly used by Endpoints) |
| `01` | Type 1 — PCI-to-PCI Bridge header |

The multifunction indicator is bit 7 and is interpreted separately. A Type 0 header by itself does not prove that a Function is an ordinary PCIe Endpoint; interpret it together with the Class Code and, for PCIe-specific roles, the PCI Express Capability Device/Port Type.

Your actual observations:

| Function | Header Type | Class Code |
|---|---|---|
| NVMe SSD | `00` | `0108` |
| Intel Root Port | `81` | `0604` |

This establishes that:

- Your NVMe controller Function is a non-bridge Function and uses a Type 0 Configuration Space header. Its Class Code `0108` identifies it as a Non-Volatile Memory Controller.
- Your Root Port uses a Type 1 Bridge Configuration Space header.

### **Important distinction — Bridge classification versus PCIe Port Type**

Class Code `0604` and Header Type 1 identify a PCI-to-PCI Bridge Function.

However, these two fields alone do not identify the exact PCIe Port role.

For example, the PCIe Port may be:

- Root Port.
- Switch Upstream Port.
- Switch Downstream Port.

To identify the exact PCIe Port Type, the host examines the **Device/Port Type field in the PCI Express Capability structure**.

We will study this field in our upcoming Configuration Space Capability Discovery lesson.

Your earlier `lspci` inventory explicitly identified the Intel Function as:

```text
Intel Corporation 500 Series Chipset Family On-Package
PCI Express Root Port #5
```

That decoded device description identifies the Root Port.

The actual PCI Express Capability Port Type field has not yet been independently inspected for this Function.

### **Why does this distinction matter?**

An Endpoint provides device functionality.

A Bridge connects one PCI bus to another and allows the host to discover devices behind it.

Therefore, when the host discovers a Bridge, it must perform additional enumeration work to discover its downstream devices.

---

## **PCIe Topology and Root Port Relationship**

### **Understanding the PCIe hierarchy**

The Root Complex provides one or more Root Ports in the topologies considered here. A Root Complex can also contain integrated PCIe components and need not consist only of Root Ports.

A Root Port is a PCIe Downstream Port. It originates a PCIe Link from the Root Complex towards an Endpoint or a PCIe Switch.

For a direct-attached NVMe SSD:

```text
PCIe Root Complex
        |
        |
    Root Port
 (Downstream Port)
        |
        | PCIe Link
        |
   NVMe Endpoint
```

For a switch-based topology, the Upstream and Downstream Ports are Bridge Functions within the PCIe Switch:

```text
PCIe Root Complex
        |
    Root Port
  (Downstream Port)
        |
        | PCIe Link
        |
+----------------------------------+
|             PCIe Switch          |
|                                  |
|       Switch Upstream Port       |
|                |                 |
|        Internal Switch Fabric    |
|                |                 |
|      Switch Downstream Port      |
+----------------------------------+
        |
        | PCIe Link
        |
    NVMe Endpoint
```

Remember:
- In the conventional single-host example, the switch has one Upstream Port and one or more Downstream Ports.
- A multi-host or multi-root switch can expose more than one host/upstream port, depending on its design and configuration.
- A Root Port is a Downstream Port.
- PCIe Endpoints include devices such as NVMe SSDs.

The phrase *upstream side of a Root Port* refers to its side facing the Root Complex. It does not change the Root Port's role as a Downstream Port.

### **Practical — Display PCIe topology**

**Command:**

```bash
sudo lspci -D -t
```

Why do we use this command?

It displays the hierarchy of buses, bridges and downstream devices.

Your actual relevant output:

```text
-[10000:e0]-
             \-1c.4-[e1]----00.0
```

Mapping:

```text
PCI Domain 10000
       |
     Bus e0
       |
 Intel Root Port
 10000:e0:1c.4
       |
     Bus e1
       |
 NVMe SSD
 10000:e1:00.0
```

This is our observed Root Port-to-NVMe topology.

The domain value `10000`, together with the Intel VMD-related entries in your inventory, is a clue that this may be a VMD-managed PCI hierarchy. The domain number alone does not prove that VMD owns this path. We have not yet verified the VMD driver, its relationship to this Root Port, or the platform-specific path, so keep this conclusion provisional.

---

## **BAR — Base Address Registers**

### **Why does a PCIe Function need a BAR?**

The host needs to access resources provided by the PCIe Function.

For an NVMe controller, the host needs access to the controller's memory-mapped register interface.

Examples of NVMe Controller Registers:

| Register | Purpose |
|---|---|
| CAP | Controller capabilities |
| VS | NVMe version |
| CC | Controller configuration |
| CSTS | Controller status |
| AQA | Admin Queue Attributes |
| ASQ | Admin Submission Queue Base |
| ACQ | Admin Completion Queue Base |

These registers do not reside inside PCI Configuration Space.

They are part of a separate NVMe Controller Register Interface.

The PCI Configuration Space contains BAR registers. A BAR identifies the resource type and attributes (for example, I/O versus Memory, 32-bit versus 64-bit, and prefetchability) and holds the base address programmed by firmware or the operating system. The resource size is determined through the BAR sizing procedure or platform-provided resource information; the size is not simply stored as a separate size field in the BAR.

### **Relationship between BAR and NVMe Controller Registers**

```text
PCIe NVMe Function
       |
       |
PCI Configuration Space
       |
       |
     BAR0
       |
        | Holds the assigned base address
       |
       v
NVMe Controller Register Interface
       |
        +---- CAP
       |
        +---- VS
       |
        +---- CC
       |
        +---- CSTS
       |
        +---- AQA
       |
        +---- ASQ
       |
        +---- ACQ
```

The host CPU accesses the NVMe controller registers using Memory-Mapped I/O (MMIO).

For example, if the base address is `0x80000000`, the host can access the registers using their defined offsets.

| Register | Offset | Illustrative address |
|---|---|---|
| CAP | `0x00` | `0x80000000` |
| VS | `0x08` | `0x80000008` |
| CC | `0x14` | `0x80000014` |
| CSTS | `0x1C` | `0x8000001C` |

These addresses are illustrative, not your current machine's actual values.

### **Base address versus resource size**

BAR resource allocation involves two important things:

- Base address: where the resource starts.
- Resource size: how much address space is required.

For example:

| Resource | Illustrative base address | Size |
|---|---|---|
| NVMe MMIO | `0x80000000` | 16 KB |
| Ethernet MMIO | `0x90000000` | 4 KB |

The NVMe controller resource uses the range beginning at `0x80000000`.

The Ethernet resource uses a different range.

The host must allocate resources without conflicting address ranges.

We intentionally skipped the low-level BAR sizing algorithm. We do not need to calculate the all-ones mask bit by bit for our current learning objective.

### **Practical — Observe your actual BARs**

**Command:**

```bash
sudo lspci -D -vv -s 10000:e1:00.0
```

Your actual output:

```text
Region 0: Memory at 72000000
          (64-bit, non-prefetchable) [size=16K]

Region 2: Memory at 72004000
          (32-bit, non-prefetchable) [size=4K]

Region 3: Memory at 72005000
          (32-bit, non-prefetchable) [size=4K]
```

Interpretation:

| Region | Address | Type | Size |
|---|---|---|---|
| 0 | `0x72000000` | 64-bit MMIO | 16 KB |
| 2 | `0x72004000` | 32-bit MMIO | 4 KB |
| 3 | `0x72005000` | 32-bit MMIO | 4 KB |

Region 0 is a 64-bit BAR and occupies two BAR slots (BAR0 and BAR1).

For a standard PCIe NVMe controller, the controller register interface is mapped through BAR0. When BAR0 is a 64-bit BAR, BAR1 is the upper 32-bit half of that same BAR pair; it is not a separate resource.

Region 2 corresponds to BAR2, and Region 3 corresponds to BAR3.

The additional resource regions should not be assumed to contain particular registers without further device-specific evidence.

**Enumeration-state note:** These `lspci` Region values show the resources currently known and assigned to the Function when the command was run. They do not, by themselves, reveal whether firmware or Linux assigned the addresses, whether Linux changed them after boot, or the exact BAR-sizing writes used during enumeration.

---

## **PCI Command Register — Memory Space Enable and Bus Master Enable**

The 16-bit PCI Command Register is located in the PCI Function's Configuration Space at offset `0x04`. Two bits are particularly important for NVMe operation:

- Bit 1: Memory Space Enable (MSE).
- Bit 2: Bus Master Enable (BME).

### **Memory Space Enable (MSE)**

The host assigns a memory-space resource through BAR.

But address assignment by itself does not enable the Function to respond to accesses targeting that resource.

The host must enable Memory Space Enable.

When MSE is enabled, the Function can respond to memory-space accesses directed to its assigned BAR resources.

```text
Host CPU
   |
    | Memory-space access
   |
   v
NVMe PCIe Function
   |
   |
BAR0 address range
   |
   |
NVMe Controller Registers
```

When MSE is disabled, those memory-space accesses are not enabled for the Function.

### **Bus Master Enable (BME)**

The NVMe controller also needs to initiate PCIe transactions.

For example, during normal NVMe operation, it needs to:

- Fetch commands from Submission Queues in host memory.
- Transfer read data into host memory.
- Write Completion Queue entries into host memory.

The controller performs these operations using DMA.

For the Function to initiate PCIe transactions as a bus master, Bus Master Enable must be enabled.

```text
NVMe Controller
       |
        | DMA-related PCIe transactions
       |
       v
PCIe Fabric
       |
       v
Host Memory
```

BME is required for those controller-initiated DMA transactions. Actual DMA access is also subject to platform DMA address mapping and IOMMU permissions where applicable.

### **Difference between MSE and BME**

| Feature | MSE | BME |
|---|---|---|
| Full name | Memory Space Enable | Bus Master Enable |
| Direction | Host accesses Function resources | Function initiates PCIe transactions |
| Primary purpose | Enables response to BAR memory accesses | Enables bus-master operations such as DMA |
| NVMe example | Host accesses CAP or CC | Controller accesses host queue memory |

**Practical command:**

```bash
sudo lspci -D -vv -s 10000:e1:00.0
```

Your actual output:

```text
Control: I/O- Mem+ BusMaster+ SpecCycle-
```

This establishes the Command Register state **at the time this `lspci` output was captured**:

- `Mem+`: Memory Space Enable is currently set.
- `BusMaster+`: Bus Master Enable is currently set.

Both are bits in the PCI Command Register, not separate registers. This live snapshot does not tell us whether firmware, the Linux PCI core or a device driver set the bits, nor does it prove the exact state at the earlier enumeration stage. A driver commonly enables the resources it needs and enables bus mastering as part of device initialization.

---

## **PCI Bus Numbering**

This is where we began studying how the host discovers devices behind bridges.

### **What is a PCI bus?**

A PCI bus is a logical grouping used to organise and address PCIe Functions.

PCIe uses point-to-point links, not a shared electrical bus, but retains the Bus/Device/Function addressing model.

A PCI domain can contain multiple bus numbers. In the conventional PCI bus-number field, each domain can address bus numbers `00` through `FF`.

For example:

```text
Root Complex
     |
    Bus 00
     |
   Root Port
     |
    Bus 01
     |
   NVMe SSD
```

The Root Port connects the upstream bus to the downstream bus.

### **Primary Bus Number**

Primary Bus Number identifies the bus on the upstream or parent side of the bridge. For a Root Port this is normally the Root Bus; for a PCIe Switch Downstream Port it is the switch's internal logical bus. Therefore, “parent-side bus” is the general definition; it is not always the Root-Complex-facing bus directly.

Example:

```text
Root Complex
     |
    Bus 00
     |
   Root Port
```

Primary Bus Number = `00`.

The Primary Bus Number field is stored in the bridge's Type 1 Configuration Header. In a Type 1 header, the bus-number fields are located at:

| Field | Configuration Space offset |
|---|---|
| Primary Bus Number | `0x18` (byte) |
| Secondary Bus Number | `0x19` (byte) |
| Subordinate Bus Number | `0x1A` (byte) |

These offsets apply to PCI-to-PCI Bridge Functions such as PCIe Root Ports and Switch Ports.

### **Secondary Bus Number**

Secondary Bus Number identifies the bus immediately downstream of the bridge.

Example:

```text
Root Complex
     |
    Bus 00
     |
   Root Port
     |
    Bus 01
     |
   NVMe SSD
```

Secondary Bus Number = `01`.

It identifies the first downstream bus that the host needs to scan.

### **Subordinate Bus Number**

Subordinate Bus Number identifies the upper boundary of the bus-number range routed downstream through that bridge. The range is inclusive, from the Secondary Bus Number through the Subordinate Bus Number. It is an upper boundary, not a count of buses; gaps can exist, so not every bus number in the range must contain an implemented bus or device. In systems that reserve bus-number capacity for hot-plug, the range may also include numbers reserved for future downstream devices, so the Subordinate value does not always equal the highest bus number currently populated by a device.

Consider:

```text
Root Complex
     |
    Bus 00
     |
   Root Port
     |
    Bus 01
     |
  PCIe Bridge
     |
    Bus 02
     |
  NVMe SSD
```

The Root Port's numbers are:

| Field | Value | Meaning |
|---|---|---|
| Primary | `00` | Root-Complex-facing bus |
| Secondary | `01` | First downstream bus |
| Subordinate | `02` | Highest bus number in the downstream hierarchy |

For a simple topology with just one downstream bus, Secondary and Subordinate can have the same value.

For a topology with additional downstream bridges, Subordinate can be greater than Secondary.

---

## **PCI Bridge Enumeration — Work Completed So Far**

After learning the three bus-number fields, we started studying how the host uses them during enumeration.

### **Host discovers a bridge**

The host scans an existing bus and discovers a Function.

It reads:

- Vendor ID.
- Device ID.
- Class Code.
- Header Type.

If the Function is identified as a bridge, the host must explore its downstream hierarchy.

For example:

```text
Root Complex
     |
    Bus 00
     |
   Root Port
```

Our actual Root Port:

```text
BDF:        10000:e0:1c.4
Class Code: 0604
Header Type: 81
```

### **Host configures bus numbers**

The bridge needs valid bus-number information to route Configuration Requests to its downstream hierarchy. Firmware may already have assigned valid bus numbers. The operating system can preserve and adopt those assignments, validate them, or reassign numbers when required.

For a bridge that needs new bus numbers in our simplified example, the Root Port might be configured as:

| Field | Value |
|---|---|
| Primary | `00` |
| Secondary | `01` |
| Subordinate | A temporary upper limit during scanning |

When an implementation assigns bus numbers to a previously unconfigured bridge, it may temporarily program a wider Subordinate Bus Number range so that the downstream hierarchy can be scanned. After the scan, it programs the final upper boundary. The temporary value and exact write sequence are implementation-dependent; it is not safe to assume that every platform uses the same temporary value.

Linux also distinguishes bridges that were already configured by firmware from bridges that still need bus numbers. Its PCI scan code processes already-configured bridges before assigning numbers to remaining bridges, helping avoid conflicts with existing assignments. We have not captured the boot-time bus-number writes on your laptop.

### **Host scans the downstream bus**

After the downstream bus is configured, the host scans candidate BDF addresses on that bus.

It reads Vendor IDs to check whether Functions are present.

On a conventional PCI bus, or on a PCIe logical bus where multiple Device numbers are valid (for example, the switch's internal logical bus), possible candidate addresses include:

```text
0000:01:00.0
0000:01:01.0
0000:01:02.0
...
```

These are illustrative candidate BDFs on Bus `01` in PCI domain `0000`. The domain and bus values are examples, not additional observations from your machine.

A valid Vendor ID indicates a Function exists.

`0xFFFF` indicates no Function is present at that candidate address.

**PCIe-specific clarification:**

On an ordinary non-ARI PCIe link directly behind a Root Port or Switch Downstream Port, the link normally contains only Device 0.

Therefore, the host normally scans Device 0 on that downstream link rather than assuming that Device numbers 0–31 can all exist behind the link.

A PCIe switch's internal logical bus is different: it can contain multiple Switch Downstream Port Functions at different Device numbers.

The Linux PCI subsystem implements this optimization: for a normal PCIe Downstream Port, it normally scans only Device 0 on the link behind that port. Linux still checks additional Function numbers for Device 0 when Function 0 advertises a multifunction device, or uses the ARI function-discovery mechanism when ARI forwarding is enabled. Linux also has a platform flag for unusual topologies that require scanning all possible Device numbers. This implementation behaviour is consistent with the normal PCIe topology rule, but it must not be applied blindly to every bus in the hierarchy.

### **Host identifies the discovered Function**

After discovering a valid Function, it reads its Class Code and Header Type.

Possible outcomes:

- **Non-bridge Function (for example, an NVMe Endpoint):** Record and configure its resources as required; do not recurse through it as a bridge.
- **Bridge Function:** Validate or configure its bus numbers, scan its Secondary Bus and recursively explore any bridges found below it.

After either outcome, continue the applicable scan of the current parent bus. Discovering one Endpoint does not end the scan of that bus.

Your actual SSD:

```text
10000:e1:00.0
Class Code: 0108
Header Type: 00
```

Your actual Root Port:

```text
10000:e0:1c.4
Class Code: 0604
Header Type: 81
```

This gives us practical evidence for the distinction.

**Important:**

Discovering an Endpoint does not mean enumeration of the current bus is complete.

The host must continue scanning the remaining applicable Device numbers and Functions on the current bus.

When another Bridge Function is discovered, the host explores its downstream hierarchy.

After the child hierarchy has been explored, the enumeration process continues with any remaining work for the parent bus. Thus, finding an Endpoint does not cause the host to stop scanning the bus.

**Implementation-order clarification:** Recursive enumeration describes the parent/child bus hierarchy; it does not require every operating system to descend immediately when it first encounters a Bridge Function. For example, Linux scans the candidate Device/Function slots on the current bus, then iterates over the discovered PCI-to-PCI Bridges and scans their subordinate buses. Other firmware or OS implementations may organize the work differently. See the Linux PCI probe implementation: [Linux `drivers/pci/probe.c`](https://linux.googlesource.com/linux/kernel/git/torvalds/linux/%2B/3f7938b1aec7f06d5b23adca83e4542fcf027001/drivers/pci/probe.c).

Thus, the sequence in these notes is a conceptual traversal model, not a promise about the precise order of configuration reads and writes in every implementation.

---

## **Multiple PCIe Devices and Functions on the Same Bus**

### **Why do we need to understand this?**

So far, we have mostly considered a simple PCIe topology where one Root Port connects to one NVMe SSD.

However, enterprise servers contain more complex PCIe hierarchies.

A logical PCI bus can contain multiple PCIe Functions at different Device numbers, and a Device number can represent one or multiple Functions. This is common on a Root Bus or a PCIe switch's internal logical bus. It does not mean that one ordinary PCIe Downstream Port link can contain 32 independent devices.

The host must discover all implemented Functions on the bus. It cannot assume that a bus contains only one PCIe Function.

### **Understanding the relationship between Bus, Device and Function**

Remember the BDF format:

```text
Domain:Bus:Device.Function
```

Consider an example:

```text
PCI Domain: 0000
Bus:        01

Device 00, Function 0
Device 01, Function 0
Device 02, Function 0
Device 03, Function 0
```

All these Functions belong to the same Bus `01`, but they have different Device numbers. This is a generic conventional-bus or internal-logical-bus example; it is not the normal topology of a single PCIe link directly behind a Root Port or Switch Downstream Port.

Now consider a multifunction PCIe device:

```text
PCI Domain: 0000
Bus:        01
Device:     00

Function 0
Function 1
Function 2
```

Here, all three Functions belong to the same Device number, but have different Function numbers.

The distinction is:

| Address component | What it distinguishes |
|---|---|
| Bus | The logical PCI bus |
| Device | A device number within that bus |
| Function | An individual Function belonging to that Device number |

In the conventional non-ARI BDF addressing model:

- Bus numbers range from `00` to `FF` within a PCI domain.
- Device numbers range from `00` to `1F` (32 possible device numbers per bus).
- Function numbers range from `0` to `7` (8 possible Functions per device number).

The maximum theoretical conventional address space is therefore 256 buses × 32 device numbers × 8 Functions per PCI domain, although only implemented and reachable Functions are enumerated.

### **How does the host scan multiple Functions?**

Suppose the host is scanning a conventional PCI bus.

For each possible Device number, it starts by checking Function `0`.

It reads the Vendor ID from Function `0`.

There are two possible outcomes:

**Function 0 is absent**

The Vendor ID read returns `0xFFFF`.

The host treats that device number as unimplemented and moves to the next device number.

**Function 0 is present**

The host receives a valid Vendor ID.

It then reads Function 0's Header Type and checks the Multifunction bit.

- If bit 7 is clear, it treats the device as single-function and does not probe Functions 1–7.
- If bit 7 is set, it checks the possible additional Functions, 1 through 7, by reading their Vendor IDs.

Each additional Function with a valid Vendor ID is discovered and identified independently.

The implemented Functions need not be consecutive.

### **PCIe downstream-link scanning clarification**

The conventional Device-number range of 0–31 does not mean that every PCIe downstream link can contain 32 independent devices.

For a normal non-ARI PCIe link behind a Root Port or Switch Downstream Port, the connected PCIe device occupies Device 0 on that downstream bus. That Device 0 may itself implement multiple conventional Functions (0–7) when it is a multifunction device.

The internal logical bus of a PCIe switch is different: it can contain multiple Downstream Port bridge Functions at different Device numbers, because those are separate bridge Functions within the switch rather than multiple independent devices on one external downstream link.

This is why the host must understand the topology when scanning PCIe buses.

### **Why is this important?**

A single bus can contain multiple PCIe devices and multiple Functions.

Some of these Functions may be ordinary Endpoints, while others may be PCIe Bridges.

Therefore, the host must continue scanning the bus after discovering an Endpoint. It must also identify any Bridge Functions that lead to additional buses.

---

## **Recursive PCIe Bridge Enumeration**

### **Why do we need recursive enumeration?**

Consider a simple topology:

```text
Root Complex
     |
    Bus 00
     |
   Root Port
     |
    Bus 01
     |
  NVMe SSD
```

Here, the Root Port leads to Bus 01, and the host discovers an NVMe Endpoint on that bus.

But enterprise servers can have multiple levels of PCIe bridges and switches.

Consider:

```text
Root Complex
     |
    Bus 00
     |
   Root Port
     |
    Bus 01
     |
  PCIe Bridge
     |
    Bus 02
     |
  NVMe SSD
```

There are two bridges in the path:

- First bridge: Root Port.
- Second bridge: PCIe Bridge.

The NVMe SSD is behind the second bridge.

The host must discover both bridges and configure their downstream buses before it can discover the NVMe SSD.

This process is called **Recursive PCIe Bridge Enumeration**.

### **Host discovers the Root Port**

The host scans the Root Bus and discovers the Root Port.

It identifies the Root Port as a Bridge Function by examining its Class Code and Header Type.

The host configures its bus numbers.

For our example:

| Root Port field | Value |
|---|---|
| Primary | `00` |
| Secondary | `01` |
| Subordinate | Temporary upper limit |

The host can now scan Bus 01.

### **Host discovers another Bridge on Bus 01**

During scanning of Bus 01, suppose the host discovers another PCIe Bridge Function.

Example:

```text
BDF:         0000:01:00.0
Class Code:  0604
Header Type: 01
```

This is another Bridge Function, not an ordinary Endpoint.

Therefore, the host must configure this bridge and explore its downstream bus.

### **Host configures the second Bridge**

The second bridge connects Bus 01 to Bus 02.

Its bus-number fields become:

| Field | Value | Meaning |
|---|---|---|
| Primary | `01` | Bus containing this bridge |
| Secondary | `02` | Immediately downstream bus |
| Subordinate | Temporary upper limit | Downstream scanning is not complete |

The host then scans Bus 02.

Suppose it discovers an NVMe SSD:

```text
0000:02:00.0
Non-Volatile Memory Controller
```

Now the host has discovered the NVMe SSD behind the second bridge.

### **Why is this called recursive enumeration?**

When the host discovers a Bridge, it performs the following operations:

- Configures that bridge's bus numbers.
- Scans its Secondary Bus.
- Identifies Functions on that bus.
- Checks whether any discovered Function is another Bridge.
- If another Bridge is found, repeats the same process for that Bridge.
- After exploring a bridge's child hierarchy, continues with the remaining parent-bus work. The exact point at which the implementation descends into each bridge can vary; some implementations first scan all candidate Functions on the parent bus, then process the discovered bridges.

The same discovery procedure is applied again at each additional bridge level.

This repeated process is called Recursive PCIe Bridge Enumeration.

The fundamental sequence is:

```text
Discover Bridge
      |
      v
Configure Bus Numbers
      |
      v
Scan Secondary Bus
      |
      v
Discover Functions
      |
      v
Another Bridge?
      |
      +---- Yes ----> Repeat the process
      |
      +---- No -----> Process Endpoint
                         |
                         v
                 Continue scanning
                 remaining Functions
```

The host continues this process until it has explored the reachable downstream hierarchy.

---

## **PCIe Switch Enumeration — Upstream Port and Multiple Downstream Ports**

### **Why do we need to understand PCIe Switch enumeration?**

Enterprise servers commonly use PCIe switches to connect multiple NVMe SSDs through a common upstream connection.

In the conventional single-host, single-upstream-port topology used in this example, the PCIe Switch exposes:

- One Upstream Port Function.
- One or more Downstream Port Functions.

Each port is represented to the host as a separate PCIe Bridge Function, with its own Configuration Space and bus-number registers. The Upstream Port connects the switch to the parent bus; its Secondary Bus represents the switch's internal logical routing bus. Each Downstream Port bridges from that internal logical bus to a separate bus representing its external downstream PCIe Link.

The host discovers and configures each visible Bridge Function independently.

**Multi-host clarification:**

Some PCIe switches support multi-host or multi-root operation, with multiple host/upstream ports and associated downstream port groups. Depending on the switch architecture and configuration, the device can expose separate virtual switch domains for different hosts. Therefore, a physical PCIe switch does not always have exactly one Upstream Port. For example, Broadcom's PEX8733 product brief describes a multi-host mode with up to four host/upstream ports and downstream ports assigned to each host: [Broadcom PEX8733 Product Brief](https://docs.broadcom.com/doc/12351852).

### **Understanding the PCIe Switch topology**

Consider an enterprise server with:

- One PCIe Root Port.
- One PCIe Switch.
- Two Switch Downstream Ports.
- Two NVMe SSDs.

The PCIe Switch's Upstream Port connects to the Root Port.

The Downstream Ports connect to the NVMe SSDs.

Conceptual topology:

```text
               Root Complex
                   |
                  Bus 00
                   |
                Root Port
                   |
                  Bus 01
                   |
           Switch Upstream Port
                   |
              Internal Bus 02
                   |
           +--------+--------+
           |                 |
     Downstream Port 1  Downstream Port 2
           |                 |
         Bus 03            Bus 04
           |                 |
       NVMe SSD 1         NVMe SSD 2
```

The bus numbers and BDFs are illustrative.

### **Understanding the internal logical bus**

The Switch Upstream Port is a PCIe-to-PCIe bridge Function and has a Type 1 Configuration Header.

Its Primary Bus Number identifies the parent bus on the host-facing side. Its Secondary Bus Number identifies the bus on the switch's internal side.

This Secondary Bus represents the switch's internal logical routing bus in our topology. It is a logical PCI bus used to enumerate the switch's Downstream Port Functions; it is not a separate external electrical PCI bus. The Linux PCI Express Port guide describes the Upstream Port's Secondary Bus as the switch's internal routing bus and the Downstream Port as bridging from that internal bus to the external downstream link: [Linux PCI Express Port Bus Driver Guide](https://kernel.org/doc/html/v6.6/PCI/pciebus-howto.html).

In this example:

| Switch Upstream Port field | Value |
|---|---|
| Primary | `01` |
| Secondary | `02` |

The Downstream Port Functions are discovered on this internal Bus `02`. Each Downstream Port's own Secondary Bus then represents the external PCIe link on the downstream side of that port.

For example:

| Switch Function | BDF | Role |
|---|---|---|
| Downstream Port 1 | `02:00.0` | Bridge Function |
| Downstream Port 2 | `02:01.0` | Bridge Function |

Both Downstream Port Functions belong to the same logical bus, Bus `02`, but have different Device numbers.

Each Downstream Port leads to its own downstream bus.

**Important distinction:**

The Downstream Port Functions are discovered on the switch's internal logical bus. They are not discovered on the bus containing the Upstream Port.

### **Why does each Downstream Port need separate bus numbers?**

Each Downstream Port is an independent PCIe Bridge Function.

Therefore, each Downstream Port needs its own Primary, Secondary and Subordinate Bus Number configuration.

**Downstream Port 1**

| Field | Value | Meaning |
|---|---|---|
| Primary | `02` | Bus containing the Downstream Port |
| Secondary | `03` | Bus behind Downstream Port 1 |
| Subordinate | `03` | Highest bus in this simple branch |

The host scans Bus 03 and discovers NVMe SSD 1.

**Downstream Port 2**

| Field | Value | Meaning |
|---|---|---|
| Primary | `02` | Bus containing the Downstream Port |
| Secondary | `04` | Bus behind Downstream Port 2 |
| Subordinate | `04` | Highest bus in this simple branch |

The host scans Bus 04 and discovers NVMe SSD 2.

The two Downstream Ports receive separate Secondary Bus Numbers because they represent separate downstream branches.

The host assigns Bus `03` to the first branch. Since Bus `03` is already allocated, it selects another available bus number, Bus `04`, for the second branch.

The actual bus-number allocator can consider firmware-provided assignments, previously allocated ranges and resources reserved for hot-plug. Therefore, these sequential values are illustrative rather than mandatory.

### **Important understanding**

The host discovers the Downstream Port Functions on the switch's internal logical bus.

It then configures each Downstream Port separately and scans the corresponding downstream bus.

This is how one physical PCIe Switch can expose multiple downstream PCIe devices to the host.

---

## **Recursive Enumeration When a Downstream Branch Contains Another Bridge**

### **Why do we need to study this?**

So far, we have considered a PCIe Switch where each Downstream Port connects directly to an NVMe SSD.

However, a downstream branch may contain another PCIe Bridge instead of directly connecting to an Endpoint.

The host must continue enumeration through that additional Bridge.

### **Example topology**

Consider:

- One Root Complex.
- One Root Port.
- One PCIe Switch.
- Two Switch Downstream Ports.
- Downstream Port 1 connects to another PCIe Bridge.
- Downstream Port 2 connects directly to an NVMe SSD.
- The additional bridge connects to another NVMe SSD.

```text
Root Complex
     |
    Bus 00
     |
  Root Port
     |
    Bus 01
     |
  PCIe Switch
     |
    Bus 02
     |
  +---+---------------------+
  |                         |
  |                         |
Downstream Port 1       Downstream Port 2
  |                         |
Bus 03                    Bus 05
  |                         |
Additional Bridge        NVMe SSD 2
  |
Bus 04
  |
NVMe SSD 1
```

The bus numbers and BDFs are illustrative.

### **Host discovers Downstream Port 1**

The host scans the switch's internal Bus 02.

It discovers Downstream Port 1.

The host assigns:

| Field | Value |
|---|---|
| Primary | `02` |
| Secondary | `03` |
| Subordinate | Temporary upper limit |

The host scans Bus 03.

### **Host discovers another Bridge on Bus 03**

Instead of directly discovering an NVMe Endpoint, the host discovers another PCIe Bridge.

Example:

```text
BDF:         0000:03:00.0
Class Code:  0604
Header Type: 01
```

The host identifies this as another Bridge Function.

It must configure the newly discovered bridge.

### **Host configures the additional Bridge**

The additional bridge is located on Bus 03.

Its bus-number configuration becomes:

| Field | Value |
|---|---|
| Primary | `03` |
| Secondary | `04` |
| Subordinate | Temporary upper limit |

The host scans Bus 04 and discovers NVMe SSD 1.

Now the host has discovered an Endpoint two bridge levels below the Root Port.

### **Host scans the second Switch Downstream Port**

The host also needs to enumerate Downstream Port 2.

This is a separate branch of the PCIe hierarchy.

It assigns:

| Field | Value |
|---|---|
| Primary | `02` |
| Secondary | `05` |
| Subordinate | Temporary upper limit |

The host scans Bus 05 and discovers NVMe SSD 2.

This demonstrates that recursive enumeration can occur independently within different downstream branches of the same PCIe Switch.

---

## **Finalizing Subordinate Bus Numbers Across the Entire PCIe Hierarchy**

### **Why is Subordinate Bus Number finalization required?**

When the host encounters a Bridge that needs new bus-number assignments, it may not yet know how much bus-number space is needed below that bridge. A Root Port may lead to a PCIe Switch, whose Downstream Port may lead to another Bridge, which finally connects to an NVMe SSD.

The host must discover the reachable downstream hierarchy and account for any bus-number capacity reserved for hot-plug before it can determine the final Subordinate boundary for each parent bridge. An implementation may therefore use a temporary Subordinate Bus Number limit while scanning a newly configured bridge, then update it after the downstream scan and bus-number reservation decisions.

A bridge that already has valid firmware-provided bus numbers may be handled differently: the OS may first scan and preserve those assignments rather than treating every bridge as unconfigured. The exact sequence is implementation-dependent.

### **Understanding bottom-up finalization**

Consider the following nested PCIe bridge topology:

```text
Root Complex
     |
   Root Bus 00
     |
   Root Port
     |
    Bus 01
     |
Switch Upstream Port
     |
    Bus 02
     |
Downstream Port 1 (Bridge A)
     |
    Bus 03
     |
   Bridge B
     |
    Bus 04
     |
   Bridge C
     |
    Bus 05
     |
   NVMe SSD
```

The host recursively scans the hierarchy until it reaches the deepest bridge.

**Bridge C — Deepest Bridge**

The host scans Bus 05 and discovers an NVMe Endpoint.

There are no additional bridges below Bridge C.

Therefore, its final configuration becomes:

| Field | Value |
|---|---|
| Primary | `04` |
| Secondary | `05` |
| Subordinate | `05` |

Bridge C is finalized.

**Bridge B — Parent of Bridge C**

Bridge B's downstream hierarchy contains Bus 04 and Bus 05.

The highest bus number is `05`.

Therefore, its final configuration becomes:

| Field | Value |
|---|---|
| Primary | `03` |
| Secondary | `04` |
| Subordinate | `05` |

Bridge B is finalized.

**Bridge A — Parent of Bridge B**

Bridge A's downstream hierarchy contains Bus 03, Bus 04 and Bus 05.

The highest bus number is `05`.

Therefore, its final configuration becomes:

| Field | Value |
|---|---|
| Primary | `02` |
| Secondary | `03` |
| Subordinate | `05` |

Bridge A is finalized.

**Switch Upstream Port**

Its downstream hierarchy includes all buses from Bus 02 through Bus 05.

Its final configuration becomes:

| Field | Value |
|---|---|
| Primary | `01` |
| Secondary | `02` |
| Subordinate | `05` |

**Root Port**

The Root Port's downstream hierarchy includes Bus 01 through Bus 05.

Its final configuration becomes:

| Field | Value |
|---|---|
| Primary | `00` |
| Secondary | `01` |
| Subordinate | `05` |

### **Final Bus Number Configuration**

| Bridge | Primary | Secondary | Subordinate |
|---|---:|---:|---:|
| Bridge C | 04 | 05 | 05 |
| Bridge B | 03 | 04 | 05 |
| Bridge A | 02 | 03 | 05 |
| Switch Upstream Port | 01 | 02 | 05 |
| Root Port | 00 | 01 | 05 |

### **Bottom-Up Finalization Process**

For a newly allocated, recursively scanned hierarchy, a useful conceptual model is to finalize the Subordinate Bus Number from the deepest bridge towards its parent bridges.

The deepest bridge's downstream scan completes first, so the bus-number boundary needed for its subtree can be determined. That upper boundary is then reflected in the parent bridge's Subordinate Bus Number. This continues upwards until the Root Port's Subordinate Bus Number covers the complete hierarchy, including any bus-number capacity reserved for hot-plug where applicable.

This bottom-up model explains the final values in our example. It is not a claim that every firmware/OS implementation performs one simple, literal bottom-up write pass for every bridge. Linux, for example, has separate handling for bridges already configured by firmware and bridges that need new bus numbers.

The Root Bus itself does not have Primary, Secondary or Subordinate Bus Number fields. These fields belong to PCI Bridge Functions.

### **Why is this necessary?**

The Subordinate Bus Number establishes the inclusive upper boundary of the downstream bus-number range behind a PCIe bridge. Depending on platform policy, this range can include bus numbers reserved for future hot-plug devices, not only buses currently containing enumerated Functions.

The bridge's bus-number registers are used when routing PCI Configuration Requests through the hierarchy: Primary identifies the bus on the parent side, Secondary identifies the immediate downstream bus, and Subordinate bounds the downstream bus-number range that can be reached through that bridge.

### **Important Implementation Note**

The temporary Subordinate Bus Number used during enumeration is not necessarily always `0xFF` in every implementation.

The exact bus-number assignment sequence depends on the platform firmware and operating-system PCI enumeration implementation.

For example, Linux processes bridges that are already configured by firmware in a first pass, then assigns bus numbers to bridges that still need them in a later pass so new assignments do not overlap existing ones. Its bridge-scanning implementation recursively scans downstream buses and returns the highest subordinate bus number needed to cover the discovered hierarchy. See the Linux PCI probe implementation.
