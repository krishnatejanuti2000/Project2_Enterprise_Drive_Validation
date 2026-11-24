# Project 2 — Practical Understanding
## LeCroy Summit T516 — Lab Setup and Recording Options

**Primary equipment:** Teledyne LeCroy Summit T516 PCIe Protocol Analyzer  
**Practical scope:** Enterprise NVMe SSD PCIe validation  
**PCIe generations:** Gen4 and Gen5  
**Reference software documentation:** Summit T5 PCI Express Analyzer User Manual, PCIe Protocol Analysis Software Version 13.22

## Purpose

The objective of this practical learning is to understand how an Enterprise SSD Validation Engineer prepares and uses a PCIe Protocol Analyzer during validation.

The focus is on actual engineering activities:

- Identifying the system and device under test.
- Understanding the physical PCIe connection.
- Selecting compatible analyzer equipment.
- Understanding interposer and analyzer connections.
- Configuring Recording Options.
- Selecting appropriate recording and triggering behavior.
- Capturing relevant PCIe traffic.
- Inspecting protocol evidence.
- Investigating abnormal link behavior.

The Core Understanding chat teaches the internal PCIe and LTSSM mechanisms. This practical learning connects those mechanisms to the analyzer, its software and laboratory observations.

---

# Lab Architecture and Equipment

## SUT and DUT

| Term | Meaning | Our environment |
|---|---|---|
| SUT | System Under Test | Host server |
| DUT | Device Under Test | Enterprise NVMe SSD |

The host server contains the PCIe Root Complex and Root Port.

The NVMe SSD operates as a PCIe Endpoint.

## Direct-attached PCIe topology

Our baseline learning setup is a direct-attached NVMe SSD.

```text
             HOST SERVER
                  |
             Root Complex
                  |
              Root Port
                  |
              PCIe Link
                  |
          PCIe Interposer
                  |
             NVMe SSD
              Endpoint
```

The physical connection may include motherboard traces, connectors, a passive backplane and a drive carrier.

However, without a PCIe switch in the path, there is one PCIe link between the Root Port and SSD Endpoint.

## Switch-based topology

Enterprise servers and storage platforms may use PCIe switches.

```text
                 HOST
                   |
              Root Complex
                   |
                Root Port
                   |
                Link 1
                   |
          Switch Upstream Port
             +------------+
             | PCIe Switch|
             +------------+
                /      \
               /        \
       Downstream 0   Downstream 1
            |              |
          Link 2         Link 3
            |              |
         NVMe SSD          HBA
         Endpoint        Endpoint
```

Each PCIe link has an independent LTSSM.

We use the direct-attached topology as our baseline because it provides a simpler starting point for understanding the analyzer setup.

## PCIe lane arrangement

PCIe communication is full duplex.

Each PCIe lane has:

- One differential transmit pair.
- One differential receive pair.

For a PCIe x4 link, there are four transmit differential pairs and four receive differential pairs.

The analyzer must capture both communication directions for a complete protocol investigation.

---

# Summit T516 Physical Equipment

## Equipment involved

| Equipment | Purpose |
|---|---|
| Host server | System under test |
| Enterprise NVMe SSD | Device under test |
| U.2/U.3 interposer | Provides analyzer access to the PCIe connection |
| Compatible analyzer cable | Connects interposer to Summit T516 |
| Summit T516 | Captures and decodes PCIe protocol traffic |
| Analyzer Control PC | Runs LeCroy PCIe Protocol Analysis software |
| Interposer power adapter | Supplies required external interposer power |
| T516 AC power connection | Supplies power to the analyzer |

## U.2/U.3 interposer

We examined the actual U.2/U.3 interposer photograph and identified three important connections.

| Interposer marking | Purpose |
|---|---|
| DUT | SSD mounting/connection location |
| TO ANALYZER | Analyzer cable connection |
| DC IN | External DC power input |

For the selected interposer arrangement:

- The SSD connects to the DUT side.
- The analyzer cable connects to TO ANALYZER.
- External power connects to DC IN.
- The interposer and SSD are installed using the compatible carrier and host-side connection arrangement specified for that interposer.

The exact installation procedure depends on the interposer model.

## Gen5 analyzer cable

We discussed the Gen5 PE028UCA-X Y-cable.

For the selected U.2/U.3 Gen5 single-link arrangement:

- The interposer connects to the single end of the Y-cable.
- The two analyzer-side branches are connected to the corresponding labeled T516 connections A and B.

Use the manufacturer's connection diagram for the exact interposer and analyzer arrangement.

Do not infer signal direction from the cable's A/B labels.

The Summit T5 manual also documents different Gen4 and Gen5 connection arrangements. The cable must match the selected interposer and analyzer configuration.

## Interposer power

The interposer receives external DC power through its DC IN connector.

The power adapter must match the exact interposer model and its specified voltage/current requirements.

A separate AC power connection supplies the Summit T516.

The interposer supply and analyzer supply are two different power connections.

## Analyzer Control PC

The Summit T516 connects to the Analyzer Control PC through USB for control and software communication.

The PCIe analyzer cable is a separate high-speed connection between the interposer and analyzer.

Remember:

| Connection | Purpose |
|---|---|
| USB | Analyzer control and software communication |
| PCIe analyzer cable | PCIe signal capture path |

USB recognition only confirms the analyzer control connection. It does not prove that the SSD has established a PCIe link or that a PCIe recording is successful.

## T516 front-panel indications

The T516 Status LED provides these indications:

| LED color | Meaning |
|---|---|
| Red | Error condition |
| Green | System configuring |
| Blue | Unit ready/connected |

The analyzer also has a Trigger LED that indicates when the analyzer triggers on an event.

An analyzer-ready indication does not prove that the NVMe SSD has reached LTSSM L0.

## REFCLK — basic laboratory understanding

REFCLK is a separate timing reference used by the analyzer to correctly track PCIe traffic.

The essential distinction is:

- PCIe data lanes carry PCIe communication.
- REFCLK supplies a timing reference.

The reference-clock source options discussed include:

- REFCLKA.
- REFCLKB.
- US_CLK.
- DS_CLK.

The interposer provides clock-source selection for the analyzer recording setup.

Actual interposer switch settings and clock connections must be checked against the exact interposer documentation before execution.

---

# Summit T516 Software — Recording Options

## Recording Options window

The reference screenshot is:

**Summit T5 User Manual — Figure 6.1**

Recording Options Dialog — General Tab — T516 Standard Mode — Simple Trigger Mode.

The window is opened through:

`Setup → Recording Options`

The General tab contains the recording configuration controls discussed below.

The main functional areas are:

| Area | Purpose |
|---|---|
| Recording Type | Analyzer configuration and recording behavior |
| PCIe Specification | Specification revision |
| Trigger Mode | Simple, Advanced or Script |
| Recording Mode | Type of recorded protocol information |
| Buffer Size | Recording memory allocation |
| Trigger Position | Pre-trigger/post-trigger allocation |
| Trigger On | Event conditions for triggering |
| Filter Out | Traffic/events excluded from recording |
| Link Settings | Speed, width, clock and per-direction configuration |
| Trace Filename & Path | Destination and filename for captured traces |
| Additional Trace File Settings | Trace-file management |
| Bottom buttons | Save, Load, apply and restore settings |

---

# Recording Type

## Summit Analyzer

The Summit Analyzer dropdown identifies the analyzer model and operating configuration for which the software prepares recording options.

Our baseline selection is:

`T516 Standard mode`

The T516 supports Standard Mode and MultiPort configurations.

The appropriate configuration depends on:

- Number of PCIe links to be captured.
- Width of each link.
- Bifurcated or independent link arrangement.
- Physical interposer.
- Analyzer port configuration and licensing.

For our baseline setup:

```text
One Host Server
      |
One PCIe x4 Link
      |
One U.2/U.3 NVMe SSD
      |
Summit T516 Standard Mode
```

Standard Mode is appropriate for this single-link setup.

The selection is based on the PCIe topology and recording requirement, not simply whether the test is functional, performance or reliability testing.

## U.2/U.3 Interposer Mode

This setting identifies the applicable U.2/U.3 interposer configuration.

The selection must match the actual interposer arrangement and the configurations supported by the analyzer software.

Consider:

- U.2 or U.3.
- Single-port or dual-port arrangement.
- PCIe generation.
- Actual interposer model.
- Analyzer configuration.

Do not assume that a Gen4 interposer option automatically applies to a Gen5 interposer.

The purpose is to make the recording configuration correspond to the physical interposer arrangement.

## Snapshot

Snapshot is a recording of a predetermined length.

The recording length is determined by the selected Buffer Size.

Workflow:

- Configure Recording Options.
- Click Rec in the analyzer toolbar.
- Recording begins immediately.
- The analyzer collects PCIe traffic.
- Recording ends when the selected buffer fills or when Stop is pressed.

**Example — Initial PCIe Link Training**

We want to investigate PCIe link initialization when the server powers on.

We can start recording before initiating the test condition.

The analyzer then captures the available PCIe activity during the startup sequence.

Snapshot does not wait for TS1, TS2 or any other selected protocol event before starting.

## Manual Trigger

Manual Trigger lets the engineer decide when the recording should switch from pre-trigger to post-trigger capture.

Workflow:

- Select Manual Trigger.
- Select Start in the application.
- The analyzer begins collecting pre-trigger traffic.
- Observe or reproduce the failure.
- Press Trigger when the event occurs.
- The analyzer switches to post-trigger recording.
- Recording continues until the post-trigger buffer is filled or Stop is pressed.

Both portions are preserved:

- Pre-trigger traffic.
- Post-trigger traffic.

**Example — Intermittent PCIe Failure**

Start recording before reproducing the issue.

When the failure occurs, press Trigger.

The final trace preserves the PCIe activity before and after the trigger point.

This is useful because the events leading up to a failure may provide important evidence for root-cause investigation.

## Event Trigger

Event Trigger automatically triggers recording when a configured event is detected in the CATC Trace.

Workflow:

- Select Event Trigger.
- Configure the required trigger event.
- Select Start in the application.
- The analyzer begins collecting pre-trigger traffic.
- A matching event is detected.
- The analyzer automatically switches to post-trigger recording.
- Post-trigger traffic is collected according to the configured Buffer Size and Trigger Position.

**Example — Trigger on TS2**

Configure TS2 as the trigger event.

When the analyzer detects a matching TS2 Ordered Set, it triggers automatically.

The engineer does not need to press the Manual Trigger button.

Important distinction:

- Start is still required to begin the recording.
- Event Trigger automates the trigger action after the configured event appears.

## Comparison of recording types

| Feature | Snapshot | Manual Trigger | Event Trigger |
|---|---|---|---|
| Recording begins | Rec | Start | Start |
| Trigger point | No separate trigger point | Engineer presses Trigger | Configured event |
| Pre-trigger capture | Not applicable | Yes | Yes |
| Post-trigger capture | Not applicable | Yes | Yes |
| Automatic event detection | No | No | Yes |

---

# PCIe Specification

The PCIe Specification dropdown selects the specification revision used in the analyzer recording configuration.

Our reference screenshot shows:

`Revision 5.0`

This is different from Link Speed.

| Setting | Purpose |
|---|---|
| PCIe Specification | Specification revision context |
| Link Speed | Analyzer speed setting |
| Link Width | Analyzer width setting |

Selecting Revision 5.0 does not force the SSD to operate at 32.0 GT/s.

---

# Trigger Mode — Simple, Advanced and Script

These three controls determine how recording conditions and trigger rules are configured.

## Simple

Simple Mode provides quick access to commonly used recording options.

Examples include:

- TS1.
- TS2.
- Any TLP.
- Any DLLP.
- Invalid Symbol.
- Disparity Error.
- Speed Change.
- Link Width Change.

For our initial LTSSM exercises, Simple Mode is the starting point.

## Advanced

Advanced Mode provides facilities for defining more complex recording rules.

It is useful when a validation scenario needs more detailed trigger conditions and recording behavior.

## Script

Script Mode provides the analyzer's recording-script configuration mechanism.

This is not the same as writing Python or Pytest automation scripts.

**Remember:**

- Snapshot / Manual Trigger / Event Trigger determine recording behavior.
- Simple / Advanced / Script determine how recording conditions are configured.

---

# Recording Mode

Recording Mode determines the type of protocol information captured and processed.

## Protocol Recording

Protocol Recording is our primary mode for PCIe protocol and LTSSM investigation.

It is an intelligent recording mode that understands the protocol and displays recorded data according to the specification.

By default, the software uses CATC Trace View to show traffic on the PCIe link.

Relevant information includes:

- TS1 and TS2 Ordered Sets.
- TLPs and DLLPs.
- Link-training activity.
- Protocol-level errors.
- Other supported PCIe events.

**Our baseline:** Protocol Recording.

## SMBus Subrecording

SMBus is a separate system-management communication bus.

The SMBus option under Protocol Recording allows the analyzer to record both data traffic and SMBus traffic for supported events, including:

- Any SMBus.
- Any MCTP over SMBus.

This is useful when the investigation requires visibility into management or sideband communication.

It is not required merely to investigate PCIe TS1 and TS2 exchange.

**Our baseline for initial LTSSM investigations:** Leave SMBus Subrecording unchecked.

## CCIX Recording

CCIX stands for Cache Coherent Interconnect for Accelerators.

It is a cache-coherent interconnect protocol used in supported accelerator/system architectures.

The CCIX option enables supported CCIX transaction recording and decoding.

For CCIX analysis, the applicable Traffic Class to Virtual Channel mapping must be configured.

Our current NVMe SSD LTSSM work does not require CCIX recording.

**Our baseline:** Leave CCIX Recording unchecked.

## NVMe Enhanced Mode

NVMe Enhanced Mode is designed for long-duration recording of selected PCIe traffic that is decoded as NVMe.

It allows the analyzer to filter traffic so that only the required NVMe resources and activities are retained.

Examples include:

- NVMe commands.
- Doorbells.
- Controller registers.
- MSI interrupts.
- Acknowledgements.
- Device configuration information.

This can extend recording depth to very long NVMe captures.

Before using NVMe Enhanced Mode, the PCIe link parameters should be established and verified using Protocol Recording during L0.

A boot-up recording is also used to obtain the SSD Device Configuration information required for NVMe resource mapping.

**Example:** Investigating long-duration NVMe read/write workload behavior and queue activity.

## NVMe Command Mode

NVMe Command Mode provides selective recording of specified NVMe commands and registers.

It also supports selected PCIe filtering, such as DLLPs and interrupts.

The mode supports NVMe 2.0 command definitions and is available for T516 Standard Mode.

**Example — Read/Write investigation**

We want to record only selected NVMe Read and Write commands, their completions and the required controller registers.

The engineer selects the relevant command and completion filters.

Important:

- A valid SSD Device Configuration improves available selections.
- At least one Filter In item must be selected.
- Selecting too many filters can exceed available bus-engine resources and reduce precision.

## Recording Mode selection for our validation work

| Validation objective | Recording mode |
|---|---|
| PCIe LTSSM and TS1/TS2 investigation | Protocol Recording |
| Long-duration NVMe traffic analysis | NVMe Enhanced Mode |
| Specific NVMe commands and registers | NVMe Command Mode |
| Sideband management investigation | SMBus Subrecording, when required |
| CCIX transaction investigation | CCIX Recording, when required |

---

# Buffer Size

Buffer Size determines how much PCIe traffic the analyzer can store in its recording memory before uploading the CATC Trace to the host machine.

The Figure 6.1 screenshot shows:

`32 MB of 128 GB / direction`

The selected size is allocated separately for each direction.

PCIe is full duplex, so the analyzer maintains separate recording memory areas for the two traffic directions.

**Example — 64 MB per direction**

| Traffic direction | Allocation |
|---|---:|
| Direction 1 | 64 MB |
| Direction 2 | 64 MB |
| Total | 128 MB |

Buffer allocation depends on:

- Expected traffic volume.
- Recording duration.
- Recording type.
- Trigger configuration.
- Required evidence.

A longer capture or high-volume workload may require a larger buffer.

The available capacity displayed by the software should be used for the actual analyzer configuration.

---

# Trigger Position

Trigger Position determines how the recording buffer is divided between pre-trigger and post-trigger traffic.

It is available for Manual Trigger and Event Trigger.

It is not used with Snapshot.

**Example**

Selected Buffer Size:

`100 MB per direction`

Trigger Position:

`75% post-triggering`

| Buffer portion | Allocation |
|---|---:|
| Pre-trigger | 25 MB |
| Post-trigger | 75 MB |
| Total | 100 MB |

This allocation applies independently to each direction.

Choose the trigger position according to the investigation objective:

- More pre-trigger allocation when investigating what happened before a failure.
- More post-trigger allocation when investigating what happened after an event.

---

# Trigger On

Trigger On identifies the events that can cause an Event Trigger recording to switch from pre-trigger to post-trigger capture.

In Simple Mode, the trigger options apply when Event Trigger is selected.

The options visible in our Figure 6.1 screenshot include:

- Enter Electrical Idle.
- Exit Idle.
- CLKREQ#.
- WAKE#.
- PERST#.
- TS1.
- TS2.
- Any DLLP.
- InitFC1.
- NAK.
- PM.
- Any TLP.
- Config.
- Memory.
- IO.
- Message.
- Disparity Error.
- Invalid Symbol.
- Block Align Error.
- Idle Symbol Error.
- Speed Change.
- Link Width Change.

## TS1 trigger

TS1 is particularly relevant to Polling.Active and subsequent Link-training activity.

When Event Trigger is configured for TS1:

- Start recording.
- The analyzer collects pre-trigger traffic.
- A matching TS1 event is detected.
- The analyzer triggers automatically.
- Post-trigger traffic is collected.

The analyzer does not wait for the Polling.Active transition conditions to finish.

## TS2 trigger

TS2 is particularly relevant to Polling.Configuration and subsequent Configuration activity.

When Event Trigger is configured for TS2, the analyzer triggers when it detects a matching TS2 Ordered Set.

It does not wait for the required eight consecutive TS2 sequences.

The actual LTSSM transition conditions must be verified by inspecting the captured trace.

## Multiple selected trigger events

When TS1 and TS2 are selected as trigger candidates, the analyzer can trigger when either matching event occurs.

It does not need to wait for both events.

**Important engineering distinction:**

An analyzer trigger identifies the configured event. It does not independently prove that the complete LTSSM transition requirement has been satisfied.

The engineer must inspect the recorded sequence and relevant fields to establish what actually happened.

---

# Filter Out

Filter Out controls which selected event types are excluded from the recording buffer.

The options shown in our screenshot include:

- SKIP Ordered Sets.
- UpdateFC DLLP.
- End of Data Stream (EDS).
- Compliance Pattern.
- Modified Compliance Pattern.

Filtering out SKIP Ordered Sets is particularly important to understand because the SKIP Ordered Set category includes training-related and other ordered-set activity such as:

- TS1.
- TS2.
- Electrical Idle.
- Fast Training Sequence.
- SKIP.

If these are excluded, relevant protocol evidence may be missing from the captured trace.

**Our baseline:** Leave all Filter Out options unchecked for initial LTSSM investigation.

This preserves the traffic and error evidence required for the investigation.

---

# Link Settings

The Link section configures the analyzer's recording parameters for the PCIe link.

## Speed

The available options include:

| PCIe generation | Data rate |
|---|---:|
| Gen1 | 2.5 GT/s |
| Gen2 | 5.0 GT/s |
| Gen3 | 8.0 GT/s |
| Gen4 | 16.0 GT/s |
| Gen5 | 32.0 GT/s |

**Auto**

The analyzer software automatically determines the link speed.

**Fixed speed**

The analyzer is configured to record at the selected speed.

This does not command the Root Port or SSD to negotiate that speed.

For a complete initial Link-training investigation, Auto is the appropriate starting choice.

For example, a Gen5 SSD initially trains at 2.5 GT/s. The link subsequently completes Configuration and reaches L0 before the higher-speed transition is performed through the applicable Recovery procedure.

Selecting a fixed 32.0 GT/s is not appropriate as the starting configuration when the test requires capture of the entire initial training sequence.

For Auto speed determination, the analyzer's Lane 0 polarity setting must be correct.

## Link Width

Link Width configures the physical width assumed or detected by the analyzer.

Examples:

- x1.
- x2.
- x4.
- x8.
- x16.

Our Figure 6.1 screenshot shows:

`Auto (x4 Maximum)`

The analyzer determines the actual link width up to the selected maximum.

**Example**

Expected NVMe SSD link width: x4  
Actual negotiated link width: x2

With Auto (x4 Maximum), the analyzer can detect the actual x2 link width.

A fixed width assumes that the link always operates at that width.

If the actual bus width differs from the fixed selection, the trace may contain extensive errors.

**Our baseline:** Auto (x4 Maximum) for the single U.2/U.3 x4 NVMe SSD setup.

---

# Upstream and Downstream Settings

The Recording Options window provides separate controls for Upstream and Downstream analyzer channels.

These controls allow low-level configuration of each link direction.

Important: The exact Upstream/Downstream mapping depends on the current analyzer configuration. Do not assume that the software labels, analyzer Ports A/B and generic host-to-device traffic directions are interchangeable.

Use the applicable T516/interposer connection diagram and calibration guide to determine the correct mapping.

## Inhibit Channel

This option means:

`Do not record/upload this channel.`

If enabled, the analyzer stops recording/uploading the selected channel.

It does not:

- Disable the PCIe link.
- Stop the SSD transmitter.
- Physically interrupt PCIe communication.

**Our baseline:** Leave unchecked on both sides because we need to capture both directions.

## Enable Precoding

Precoding is a PCIe Gen5 feature.

It helps reduce long sequences of zeros by balancing the number of zeros and ones transmitted. Only scrambled symbols are precoded.

The analyzer must be informed when the corresponding transmitter uses precoding so that the captured data can be interpreted correctly.

The checkbox configures analyzer decoding behavior. It does not enable or disable precoding inside the SSD or Root Port.

**Manufacturer's T516 Gen5 mapping:**

| Transmitter using precoding | Analyzer checkbox |
|---|---|
| Endpoint / NVMe SSD | Downstream — Enable Precoding |
| Root Complex / Host | Upstream — Enable Precoding |

Do not enable both automatically just because the SSD supports Gen5.

The setting must match the actual transmitter configuration for the tested link.

Incorrect precoding configuration can result in a trace containing extensive errors.

## Invert Polarity

The analyzer provides individual lane polarity controls.

For example, in an x4 link, the relevant lanes are Lane 0 through Lane 3.

Polarity inversion can occur independently on different lanes.

Example:

| Lane | Polarity |
|---|---|
| Lane 0 | Normal |
| Lane 1 | Inverted |
| Lane 2 | Normal |
| Lane 3 | Inverted |

The individual settings allow the analyzer to interpret each lane using its actual polarity.

They do not physically swap the positive and negative conductors.

## Reference Clock

The Reference Clock selection tells the analyzer which clock source to use to correctly track PCIe traffic.

The T516 Standard Mode options discussed include:

| Option | Purpose |
|---|---|
| Internal — No SSC | Internally generated reference clock without SSC |
| Internal — SSC -0.3 | Internal clock with 3000 ppm downspread |
| Internal — SSC -0.5 | Internal clock with 5000 ppm downspread |
| External | Clock supplied through the probe-card/interposer analyzer connection |
| SMA | SMA reference-clock input; Summit T54 only |

For the Summit T516, External uses the clock supplied through the probe-card cable from the Host or Root Complex under test.

The manufacturer recommends External when the PCIe link uses spread-spectrum clocking and for best capture performance.

**Our baseline:** External when the actual physical configuration supplies the required external reference clock.

The actual connection must be verified before recording.

## XCVR Setting

XCVR means Transceiver.

The Figure 6.1 screenshot shows:

`Default`

We retained Default during the walkthrough. The exact meaning of this dropdown was not established from the Recording Options section of the manual, so its function should not be inferred from the label alone.

---

# General Recording Options Below the Link Sections

## Set Reference Clocks Independently

Normally, the analyzer uses the same reference-clock option for both traffic directions.

Enabling this checkbox allows separate Upstream and Downstream reference-clock selections.

For example, the two directions could be configured independently when the physical clock architecture requires it.

When both directions use the same External reference clock, this option does not need to be enabled.

The manufacturer warns that selecting different Internal reference-clock profiles for the two directions is unsupported.

## Disable Descrambling

When checked, the analyzer assumes that none of the PCIe traffic is scrambled.

By default, the analyzer determines the scrambling state of the devices under test.

This affects analyzer interpretation; it does not disable scrambling inside the SSD or Root Port.

**Our baseline:** Leave unchecked.

## Auto-Configure Lane Polarity

When enabled, the analyzer determines lane polarity automatically.

This is useful when capturing initial Link training instead of manually configuring every lane.

The manufacturer specifies that automatic polarity discovery relies on Gen1 or Gen2 training sequences.

Therefore, when recording starts only after the link is already operating at a higher speed, the analyzer may not have the required training information for automatic discovery.

**Our baseline:** Enable when the capture includes the required initial training activity.

## Ignore Idle Errors

When enabled, the analyzer does not store errors occurring between packets.

This preserves analyzer memory so that more packets can be recorded.

The manual also specifies that this option filters out Compliance Patterns.

For initial failure investigation, preserving error evidence is important.

**Our baseline:** Leave unchecked.

## Decode OBFF

OBFF stands for Optimized Buffer Flush/Fill.

Selecting this option enables decoding of supported OBFF events in the trace.

OBFF provides system-level power-management hints that devices can use to coordinate activity and improve low-power operation.

This checkbox controls analyzer decoding. It does not enable OBFF inside the SSD or host.

**Our baseline:** Leave unchecked for initial LTSSM training analysis unless the test specifically investigates OBFF.

## Swizzling Config

Swizzling Config controls the mapping between physical lane positions and logical lane ordering.

The analyzer supports separate Upstream and Downstream lane mappings.

The configuration can be performed manually or automatically.

**Manual lane swizzling**

Use this when the physical lane connection and advertised logical lane numbers are known.

**Automatic lane swizzling**

The analyzer can detect lane mapping when it observes the LTSSM Configuration-state link-training sequence.

Important conditions:

- Auto-Configure Lane Swizzle requires the analyzer to observe Configuration-state link training.
- The physical location of at least one active lane must be identified.
- Automatic swizzling does not work properly for bifurcating links. The manufacturer recommends manual swizzling in that arrangement.

Do not manually change lane mapping without evidence from the physical setup or an appropriate trace.

---

# Trace Filename and Path

Trace Filename & Path specifies the directory and filename used for subsequent CATC Trace recordings.

The predefined filename shown in the manual is:

`data.pex`

Example of a descriptive validation filename:

`NVMe_Gen5_LTSSM_Initial_Training_01.pex`

Configuring this field does not immediately create or save a PCIe capture.

It changes the default name and location used for subsequent recordings.

For validation work, use filenames that identify the device, PCIe generation, test scenario and capture number.

---

# Options Name

Options Name is a descriptive label for the Recording Options configuration file.

The screenshot shows:

`Default`

For example, a saved recording configuration could have the descriptive label:

`Gen5 x4 NVMe LTSSM Initial Training`

This label helps the engineer remember the purpose of the saved options set.

It is separate from the actual captured PCIe trace filename.

---

# Additional Trace File Settings

Additional Trace File Settings opens the Advanced Trace File Settings dialog.

The controls include:

- Maximum number of trace files that can be opened at one time during recording: 1 to 10.
- Auto-Delete Trace File: enables or disables automatic deletion when the configured limit is exceeded.

This manages trace files. It does not change the PCIe recording buffer size.

For failure investigation, retaining important captures is preferable to automatically deleting them before they have been reviewed and archived.

---

# Saving and Loading Recording Options

Recording Options configurations can be saved and reused.

The documented file formats are:

| File extension | Purpose |
|---|---|
| `.rec` | General Recording Options file |
| `.xrec` | Recording Options in XML format |
| `.pex` | Actual captured CATC Trace |

## Save...

Saves the current Recording Options configuration to a file.

Example:

`Gen5_NVMe_LTSSM.rec`

## Save as Default

Saves the current options as the default Recording Options file.

The default is normally named:

`default.rec`

When PCIe Protocol Analysis starts, it automatically loads the default file if one exists.

## Load...

Loads a previously saved Recording Options configuration.

## Load Default

Loads the default Recording Options configuration.

## Restore Factory Settings

Restores the Recording Options to the factory configuration.

Use this carefully because customized recording settings may be lost.

## OK and Cancel

- OK applies the current options and closes the dialog.
- Cancel closes the dialog without applying the current unsaved changes.

**Important file distinction:**

- `.rec` / `.xrec` store the analyzer recording configuration.
- `.pex` stores the actual captured PCIe traffic.

A reusable analyzer configuration and an actual captured trace are two different engineering artifacts.

---

# Practical Baseline — Initial NVMe PCIe Link Training

For our single-link U.2/U.3 NVMe SSD learning scenario, the starting configuration is:

| Setting | Baseline selection | Reason |
|---|---|---|
| Summit Analyzer | T516 Standard Mode | One PCIe link |
| Interposer Mode | Exact matching interposer option | Matches the physical fixture |
| PCIe Specification | Revision 5.0 | Gen5 learning baseline |
| Trigger Mode | Simple | Straightforward initial trigger configuration |
| Recording Type | Snapshot | Capture the full initial startup sequence |
| Recording Mode | Protocol Recording | PCIe protocol and LTSSM analysis |
| Buffer Size | Based on expected traffic and duration | Preserve required evidence |
| Trigger Position | Not used with Snapshot | No separate trigger point |
| Trigger On | Not required for Snapshot | No event trigger |
| Filter Out | All unchecked | Preserve relevant traffic |
| Link Speed | Auto | Allow automatic speed determination |
| Link Width | Auto (x4 Maximum) | Detect actual width up to x4 |
| Inhibit Channel | Unchecked on both sides | Capture both directions |
| Enable Precoding | According to actual transmitter use | Correct Gen5 decoding |
| Invert Polarity | Auto-discovery where supported | Correct lane interpretation |
| Reference Clock | External when supplied by the setup | Match actual clock architecture |
| XCVR Setting | Default | Retain documented screenshot selection |
| Set Reference Clocks Independently | Unchecked when both use the same source | No independent selection required |
| Disable Descrambling | Unchecked | Allow automatic scrambling-state handling |
| Auto-Configure Lane Polarity | Enabled when initial training is captured | Discover lane polarity |
| Ignore Idle Errors | Unchecked | Preserve error evidence |
| Decode OBFF | Unchecked unless required | Not needed for initial LTSSM work |
| Swizzling Config | Automatic where supported by the topology and capture | Determine lane mapping |
| Trace Filename & Path | Descriptive test-specific `.pex` name | Organize captured evidence |
| Auto-Delete Trace Files | Disabled | Preserve investigation evidence |

The actual hardware configuration must be verified before applying these selections. In particular, interposer compatibility, reference-clock connection, precoding, lane mapping and calibration must match the physical test setup.

**Why Snapshot for the initial training capture?**

We want to start recording before the server initiates PCIe Link establishment.

This allows us to capture the available initial training activity without waiting for a selected protocol event.

For a targeted investigation, Event Trigger can instead be configured using an event such as TS1 or TS2.

---
