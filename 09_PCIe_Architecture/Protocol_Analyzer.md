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

# Analyzer and Probe Settings — Preset Library

The Analyzer and Probe Settings area includes a Preset Library. It provides predefined analyzer/interposer configurations that can be selected for a supported Host, DUT and interposer combination.

A preset should match the actual equipment in the lab. It is not a generic setting that can safely be applied to any server, SSD or interposer.

## Host, DUT and Interposer selection

| Field | Meaning | Selection guidance |
|---|---|---|
| Host | Host platform containing the PCIe Root Complex/Root Port | Select the actual server/platform preset when an exact supported match is available |
| DUT | Device Under Test, such as the NVMe SSD or another PCIe endpoint | Select the actual device preset when an exact supported match is available |
| Interposer | The physical interposer used in the PCIe capture path | Select the exact interposer model and supported configuration |

The available Host and DUT choices are filtered according to the selected interposer and supported combinations.

When a connected interposer is detected, its matching entry is shown in bold and underlined in the Interposer list. This is an identification aid; the physical interposer model and board identity must still be checked before applying a configuration.

## Preset list columns

The preset list contains the following information:

| Column | Meaning |
|---|---|
| Host | Host platform associated with the preset |
| DUT | Device associated with the preset |
| Rank | Preset ranking related to observed error performance |
| Comment | Descriptive information about the preset |

The manual explains that Rank 1 represents few or zero errors, while higher ranks indicate increasing errors. A rank greater than 6 is not recommended. The manual does not define a precise meaning for every intermediate rank, so do not infer additional thresholds.

The Comment field is descriptive information. It is not itself a configuration control.

## Applying a preset

The Preset Library provides three application choices:

| Control | Effect |
|---|---|
| Apply Preset | Applies the selected preset to both Upstream and Downstream configurations |
| Apply US only | Applies the selected preset to the Upstream configuration only |
| Apply DS only | Applies the selected preset to the Downstream configuration only |

Applying a preset can prompt for a host reboot. The exact prompt and required action depend on the selected configuration.

A board-ID mismatch warning must be taken seriously. Do not apply a preset to an interposer whose identity does not match the preset's intended hardware.

## Export Library and Import Library

**Export Library** exports the preset library to a `.pelib` file.

**Import Library** imports a `.pelib` file into the local preset library. Importing replaces the local preset folder. Back up locally created presets before importing another library so that they are not unintentionally lost.

## Save Current

Save Current saves the manually selected configuration as a user-defined preset. The documented workflow asks for identifying information such as:

- Platform Name.
- Root Complex.
- DUT.
- Interposer.
- Notes.

The Save Current operation saves the selected values and writes the configuration to the interposer across the three configuration areas:

- Host-to-DUT path.
- Interposer-to-Analyzer path.
- Analyzer Calibration.

This is more than assigning a descriptive preset name. Because it also writes configuration to the interposer, use it only after confirming the physical setup and selected values.

## Choose File

Choose File selects a user-defined interposer configuration file for loading. The workflow allows the engineer to select which configuration tables to load.

Choosing a file loads its configuration into the software. It does not, by itself, mean that the selected values have been written to the physical interposer.

## Apply File

Apply File applies the selected configuration file. The operation saves the Recording Options and writes the selected preset values to the interposer for the selected configuration tables.

It may require a host reboot. A board-ID mismatch warning must be checked before proceeding.

**Important distinction:** Choose File selects/loads a configuration file; Apply File applies its selected configuration to the relevant equipment. Do not treat these two controls as interchangeable.

---

# Autotune and Probe Calibration Wizard

The Autotune and Probe Calibration Wizard is used to prepare and calibrate the analyzer/interposer signal paths for capturing PCIe traffic. It verifies the equipment, accepts the intended link parameters and polarity, assesses link quality, connects to the host when required, and performs the calibration operations.

The wizard walkthrough in this learning session was based on the Summit T5 PCI Express Analyzer User Manual, PCIe Protocol Analysis Software Version 13.22.

## Wizard sequence

The wizard contains the following stages:

| Stage | Wizard screen | Manual figure | Printed page |
|---|---|---:|---:|
| Analyzer Verification | Verify the Summit analyzer | 6.206 | 342 |
| Interposer Verification | Verify the interposer | 6.207 | 343 |
| Setting Parameters | Select calibration channels and target link parameters | 6.208 | 344 |
| Setting Polarity | Set lane polarity for each direction | 6.209 | 345 |
| Precalibration | Assess initial link quality and required calibration work | 6.210 | 346 |
| Setup Connection | Configure the software connection to the host | 6.211 | 347 |
| Host System and DUT Configuration | Identify the PCIe DUT and select applicable presets | 6.212 | 348 |
| Calibration Process | Run autotuning and calibration | 6.213 | 349 |
| Calibration Results | Review and save calibration results | 6.214 | 350 |

The detailed calibration report is shown separately in Figure 6.215, printed page 351.

## Analyzer Verification

The first stage identifies the physical Summit analyzer used for the calibration procedure.

Select and verify the correct analyzer model and serial number. The information must correspond to the actual Summit unit connected to the Control PC.

This is equipment identification. It does not establish that the SSD has trained successfully or that the PCIe signal path is ready for capture.

## Interposer Verification

The second stage identifies the interposer used in the setup.

Select the exact supported interposer model. The selection must agree with the physical interposer installed between the host and DUT and the analyzer connection arrangement.

Do not choose a similar-looking interposer entry simply because it is available in the list. The correct model and board identity matter when loading or applying interposer-specific settings.

## Setting Parameters

The third stage configures the parameters used by the calibration procedure.

The screen includes the following selections:

| Parameter | Purpose |
|---|---|
| Calibration channels | Select whether to calibrate the Upstream direction, Downstream direction or both |
| Dwell time | Sets the interval used for the error measurement during calibration |
| Analyzer Calibration maximum retries | Sets the maximum retry count for the Analyzer Calibration operation |
| DUT Autotuning maximum retries | Sets the maximum retry count for DUT autotuning |
| Intended Link Width | Selects the intended PCIe link width, such as x1, x2, x4, x8 or x16 |
| Intended Speed | Selects the intended PCIe link speed, from 2.5 GT/s through 32.0 GT/s |

The example shown in the manual uses a 100 ms dwell time, 10 maximum retries for Analyzer Calibration, 10 maximum retries for DUT Autotuning, x4 intended width and 16 GT/s intended speed.

These are example values from the manual screenshot, not universal settings for every lab setup. Select parameters that match the actual Host, DUT, interposer and intended test link.

The retry fields specify maximum retry counts. Do not infer additional retry behavior that is not described by the manual.

## Setting Polarity

The fourth stage is the Setting Polarity screen (Figure 6.209).

The screen presents two separate lane groups:

| Direction | Direction label in the wizard |
|---|---|
| Upstream | Device to Host |
| Downstream | Host to Device |

Each direction has individual lane controls. The displayed lane positions run from Lane 15 through Lane 0. The screen asks the engineer to invert polarity for lanes with negative polarity.

Select only the lanes whose polarity has been identified as inverted. For an x4 link, the active lanes of interest are Lane 0 through Lane 3; do not select unused lanes simply because their checkboxes are visible.

**Illustrative x4 example only:** If measurement or the verified setup identifies Upstream Lane 1 and Downstream Lane 2 as inverted, select those two lane controls in their respective direction groups. This is an example, not a finding about our institute hardware.

Incorrect polarity settings can introduce errors in the recorded trace. The wizard's polarity step is a manual lane-selection step; do not confuse it with the separate Recording Options control for automatic lane-polarity discovery.

After setting the required lane polarities, proceed to Precalibration.

## Precalibration

The fifth stage is the Precalibration Link Quality Check (Figure 6.210).

This stage assesses the existing link quality and determines which calibration operations are required. It is an initial assessment, not the final calibration result.

In the manual's example, the link quality is reported as not good. The screen identifies three required operations:

| Operation | Purpose in the calibration flow |
|---|---|
| Interposer: Host-to-DUT tuning | Tunes the signal path between the Host and DUT through the interposer |
| Interposer: Analyzer-path calibration | Calibrates the signal path between the interposer and analyzer |
| Analyzer: AutoCalibration | Performs the analyzer's AutoCalibration operation |

The example does not mean that every calibration session will require all three operations. The required work depends on the link-quality assessment.

A poor analyzer capture must not automatically be attributed to SSD firmware. The Host-to-DUT path and the interposer-to-analyzer observation path also need to be considered.

## Setup Connection

The sixth stage configures the software connection between the calibration wizard and the Host system. This is separate from the PCIe signal connection used to capture traffic.

The manual provides these host operating-system choices:

| Host selection | Connection information shown by the wizard |
|---|---|
| Linux | Login, password and IP address |
| Windows | Agent address |
| N/A | No additional host connection information |

Use authorized host credentials only. Do not place server passwords in shared notes or validation reports.

The screen also provides an option labelled `Disable Active State Power Management (ASPM) on Host target system`. ASPM is PCIe link power management. This option changes the Host's ASPM behavior; do not enable it automatically for every test. Use it only when justified by the calibration procedure and the test requirements.

Keep the three connections distinct:

| Connection | Purpose |
|---|---|
| Control PC to Summit T516 | Analyzer control and software communication, over the analyzer control connection |
| Calibration wizard to Host | Host-system interaction required by the calibration procedure |
| Host to interposer to DUT, with the analyzer connected to the interposer | Physical PCIe signal path being observed |

A successful Control PC connection to the T516 does not prove that the calibration wizard has connected to the Host. Likewise, a successful Host software connection does not prove that PCIe traffic is being captured correctly.

## Host System and DUT Configuration

The seventh stage identifies the PCIe DUT and selects the Host/DUT presets that may be used during calibration.

The PCIe Device dropdown lists detected PCIe devices. In the manual's example, the selected device is a Standard NVM Express Controller. Select the controller corresponding to the SSD actually connected to the interposer.

The `Show full list of PCIe devices` control displays the full list of detected devices, including device types that may not appear in the default list, such as Root Complex Root Ports and Event Collectors.

The screen provides local and cloud preset selections for the Host and DUT:

| Preset field | Source |
|---|---|
| Local Host | Host preset stored on the Control PC |
| Cloud Host | Host preset stored in the cloud library |
| Local DUT | DUT preset stored on the Control PC |
| Cloud DUT | DUT preset stored in the cloud library |

The `Calibrate using presets` option is enabled by default. When enabled, at least one preset must be selected to proceed. Unchecking it allows calibration to proceed without selecting presets. Selecting more presets can increase calibration time.

For our lab, we must first identify the actual server platform, SSD/controller and interposer. Do not select unrelated presets merely to satisfy the wizard.

## Calibration Process

The eighth stage runs the selected autotuning and calibration operations (Figure 6.213).

The screen displays progress and a log window. The log can show operations such as:

- The calibration service starting.
- DUT-path autotuning and analyzer-path calibration starting.
- Autotuning starting.
- Target and current link parameters being reported.
- The link being reset as part of the speed-adjustment process.
- CTLE and DCGain values being reported for Upstream and Downstream.

The manual's example includes a link reset while the calibration process adjusts speed. A link reset during this procedure is part of the example calibration workflow; it should not automatically be classified as an SSD failure.

Treat log entries as evidence of what the calibration software reports. Do not treat an intermediate CTLE/DCGain value as a final calibration result, and do not assume that progress-bar movement alone proves success.

Calibration can take time. Carry it out only in an approved lab window because link resets and calibration activity can interrupt normal PCIe operation.

## Calibration Results

The ninth stage presents the calibration outcome and provides access to the detailed calibration report.

Review the process status and open the report before treating the calibration as complete. The report records per-lane calibration settings separately for Upstream and Downstream.

The wizard provides options to save the calibration results locally or in the cloud:

| Save destination | Meaning |
|---|---|
| Local | Saves the calibration result as a local preset on the Control PC; it can be accessed there or copied to another PC |
| Cloud | Saves the result to the cloud so it can be accessed from another internet-connected PC |

Cloud saving is an explicit choice. The wizard includes an agreement/consent checkbox for posting calibration results to the cloud. Use it only when permitted by the lab's data-sharing policy.

The result screen also allows descriptive Host and DUT names and an optional comment. Finish saves the calibration results as a preset according to the selected save options.

A report being available is not, by itself, proof that the calibration completed successfully. Check the calibration outcome and any reported errors before recording the result as successful.

---

# Reading the Calibration Report

The detailed report is shown in Figure 6.215, printed page 351 of the Summit T5 User Manual.

The report presents calibration information for the Upstream and Downstream directions and records values for individual lanes. The three parameters discussed in this learning session are CTLE, DCGain and SigDet.

These are analyzer/probe signal-path calibration settings. They are not NVMe SSD firmware parameters, PCIe link-speed measurements or error counters.

## CTLE — Continuous Time Linear Equalization

CTLE adjusts the frequency response of the received signal. High-frequency signal components can be attenuated as a signal travels through the physical channel. CTLE provides frequency-dependent equalization to compensate for channel loss.

The report records the CTLE setting for each lane, expressed in dB.

Different lanes may require different CTLE values because their signal paths are not necessarily identical. A higher CTLE value must not automatically be interpreted as better calibration.

## DCGain — DC Gain

DCGain is the gain setting associated with the signal level. It is distinct from CTLE's frequency-dependent equalization.

The report records DCGain for each lane. Do not interpret it as a PCIe protocol metric or as a direct measurement of SSD performance.

## SigDet — Signal Detection threshold

SigDet means Signal Detection threshold. It is associated with the signal-detection circuitry's threshold for recognizing an incoming signal.

The Analyzer Calibration section of the manual describes the following values for Manual Mode:

| Item | Manual-mode information in the manual |
|---|---:|
| Default SigDet | 60 mV |
| Available range | 30–410 mV |
| Available steps | 7 |

The manual also notes that SigDet is not available for every device. These manual-mode values must not be assumed to be the correct setting for every automatic calibration or every interposer.

## Read the report lane by lane

For each direction:

- Identify whether the table is Upstream or Downstream.
- Identify the lane number.
- Read that lane's CTLE value.
- Read that lane's DCGain value.
- Read that lane's SigDet value, where available.
- Repeat for the other active lanes.

**Illustrative example only — not actual T516 calibration results:**

| Direction | Lane | CTLE | DCGain | SigDet |
|---|---:|---:|---:|---:|
| Upstream | 0 | 4.0 dB | 2.0 dB | 60 mV |

This row demonstrates how to read the three values for one lane. It is not a target, recommendation or expected result for our equipment.

## Engineering interpretation

Do not assume that:

- Higher CTLE or DCGain values always indicate better calibration.
- All lanes must have identical settings.
- A report with numerical values automatically means the calibration passed.
- The manual's example values are ideal values for every analyzer/interposer combination.

Use the calibration process outcome, reported errors and detailed report together. Preserve the report with the test record so that the signal-capture configuration can be reviewed alongside the analyzer model, interposer, Host, DUT, link speed and width.

The manual figures discussed here do not establish a universal numerical pass/fail limit for CTLE, DCGain or SigDet. Do not create such limits from the example values.

---

# Practical Learning Boundary — Calibration

The Preset Library, Autotune and Probe Calibration Wizard, and calibration report have been covered through guided software familiarization using the Summit T5 manual and screenshots.

**No physical T516 calibration has been performed as part of this walkthrough.** The illustrative lane polarity and report values are examples only, not measurements from the institute hardware.

Before physical calibration or capture, we must inspect and verify the actual Summit T516 model and serial number, interposer model and board identity, compatible analyzer cables, power connections, reference-clock arrangement, Host/DUT topology, lane polarity and the applicable calibration settings. We must also preserve the existing state of the institute servers before changing their configuration.

Our RHEL laptop's internal NVMe drive remains protected and is not part of any write, reset or reconfiguration activity for this lab exercise.

