
# PROJECT 2 — MODULE 09: PCIe ARCHITECTURE

# PCIe LINK TRAINING AND STATUS STATE MACHINE (LTSSM)

## REFERENCE BASELINE

These notes use the **PCI Express Base Specification Revision 5.0, Version 1.0** as the normative baseline for:

- LTSSM major states and substates.
- Link initialization and training.
- TS1 and TS2 Ordered Set exchanges.
- Link-width and Lane Number negotiation.
- Link recovery and data-rate changes.
- Equalization.
- Electrical Idle and receiver detection.
- Power-management state transitions.
- Loopback, Disabled and Hot Reset behaviour.
- State-transition conditions and timeout values.

The emphasis is enterprise PCIe architecture and NVMe SSD drive validation.

PCIe Gen4 and Gen5 are the primary generations for our practical learning.

Gen6 concepts will be addressed separately where behaviour differs. PCIe 6.0 rules must not be mixed into the PCIe 5.0 transition conditions documented here.

---

# 1. INTRODUCTION TO LTSSM

## 1.1 What Is LTSSM?

LTSSM stands for **Link Training and Status State Machine**.

It is hardware control logic associated with PCIe Link operation.

It manages:

- Initial Link establishment.
- Link training.
- Link-width and Lane negotiation.
- Link recovery and retraining.
- Data-rate changes.
- Equalization.
- Power-management transitions.
- Special-purpose operations such as Loopback, Disabled and Hot Reset.

A state describes the current operating condition of a PCIe Link.

A substate describes a more specific operation being performed within a major state.

A state transition occurs when the specified conditions for moving into another state or substate are satisfied.

## 1.2 Initial State and the 11 Major LTSSM States

The PCIe 5.0 Base Specification illustrates an Initial State as the entry/reset point of the LTSSM.

Excluding this Initial State marker, the specification defines **11 major LTSSM states**:

1. Detect
2. Polling
3. Configuration
4. Recovery
5. L0
6. L0s
7. L1
8. L2
9. Disabled
10. Loopback
11. Hot Reset

These are the complete major states included in our Module 09 learning scope.

A major state may contain one or more named substates, or it may be represented as an individual state without a separately named substate machine.

## 1.3 Complete LTSSM State and Substate Inventory

The following inventory is the complete PCIe 5.0 LTSSM scope.

### 1.3.1 Detect State

Purpose: Detect the presence of the expected far-end receiver termination.

Substates:

1. `Detect.Quiet`
2. `Detect.Active`

### 1.3.2 Polling State

Purpose: Establish initial synchronization and training conditions using TS1 and TS2 Ordered Sets.

Substates:

1. `Polling.Active`
2. `Polling.Compliance`
3. `Polling.Configuration`
4. `Polling.Speed`

Important: `Polling.Speed` is defined in the PCIe 5.0 specification but is unreachable in the normal PCIe 5.0 initial Link-up flow.

### 1.3.3 Configuration State

Purpose: Configure physical PCIe lanes into one or more logical Links through Link-width negotiation, Lane Number assignment and final Link configuration.

Substates:

1. `Configuration.Linkwidth.Start`
2. `Configuration.Linkwidth.Accept`
3. `Configuration.Lanenum.Accept`
4. `Configuration.Lanenum.Wait`
5. `Configuration.Complete`
6. `Configuration.Idle`

Important: The specification's subsection numbering lists `Lanenum.Accept` before `Lanenum.Wait`, but the normal state-machine progression after `Linkwidth.Accept` enters `Lanenum.Wait`. The two Lane Number substates can repeat as required before proceeding to `Configuration.Complete`.

### 1.3.4 Recovery State

Purpose: Recover and retrain an established PCIe Link.

Substates:

1. `Recovery.RcvrLock`
2. `Recovery.Equalization`
3. `Recovery.Speed`
4. `Recovery.RcvrCfg`
5. `Recovery.Idle`

**Recovery.Equalization — Internal Phases**

1. Phase 0 — Initial transmitter settings and equalization preparation.
2. Phase 1 — Link partners exchange and evaluate equalization information.
3. Phase 2 — Transmitter equalization requests and receiver evaluation.
4. Phase 3 — Final transmitter equalization evaluation and completion.

These are phases within `Recovery.Equalization`, not four additional major LTSSM states.

### 1.3.5 L0 State

Purpose: Normal operational PCIe Link state.

L0 has no separately named substate machine.

In L0, the configured PCIe Link is operational and data and control packets can be transmitted and received.

### 1.3.6 L0s State

Purpose: Low-latency, low-power PCIe Link state.

L0s contains two independently operating substate machines.

**A. Receiver L0s**

1. `Rx_L0s.Entry`
2. `Rx_L0s.Idle`
3. `Rx_L0s.FTS`

**B. Transmitter L0s**

1. `Tx_L0s.Entry`
2. `Tx_L0s.Idle`
3. `Tx_L0s.FTS`

The Transmitter and Receiver sides of a Port are not required to enter L0s simultaneously.

### 1.3.7 L1 State

Purpose: Lower-power Link state providing greater power savings than L0s, at the cost of additional resume latency.

Substates:

1. `L1.Entry`
2. `L1.Idle`

### 1.3.8 L2 State

Purpose: Aggressive PCIe power conservation.

Substates:

1. `L2.Idle`
2. `L2.TransmitWake`

`L2.TransmitWake` applies only to Upstream Ports and is associated with Beacon-based wake-up behaviour.

### 1.3.9 Disabled State

Purpose: Keep a PCIe Link disabled as directed, until the specified conditions permit a return to Detect.

The PCIe 5.0 specification defines Disabled as an individual state without separately named substates.

### 1.3.10 Loopback State

Purpose: PCIe testing and fault isolation.

Substates:

1. `Loopback.Entry`
2. `Loopback.Active`
3. `Loopback.Exit`

Loopback has two component roles:

- Loopback Master — the component requesting Loopback.
- Loopback Slave — the component performing the loopback operation.

Only the entry and exit behaviours are specified in detail; other Loopback behaviour is implementation-specific.

### 1.3.11 Hot Reset State

Purpose: Reset a configured PCIe Link and its associated downstream device using in-band signaling.

The PCIe 5.0 specification defines Hot Reset as an individual state without separately named substates.

## 1.4 Complete LTSSM Inventory — Summary Table

| No. | Major State | Substates |
|---|---|---|
| 1 | Detect | Quiet, Active |
| 2 | Polling | Active, Compliance, Configuration, Speed |
| 3 | Configuration | Linkwidth.Start, Linkwidth.Accept, Lanenum.Accept, Lanenum.Wait, Complete, Idle |
| 4 | Recovery | RcvrLock, Equalization (Phases 0–3), Speed, RcvrCfg, Idle |
| 5 | L0 | No separately named substates |
| 6 | L0s | Rx_L0s.Entry, Rx_L0s.Idle, Rx_L0s.FTS, Tx_L0s.Entry, Tx_L0s.Idle, Tx_L0s.FTS |
| 7 | L1 | L1.Entry, L1.Idle |
| 8 | L2 | L2.Idle, L2.TransmitWake |
| 9 | Disabled | No separately named substates |
| 10 | Loopback | Loopback.Entry, Loopback.Active, Loopback.Exit |
| 11 | Hot Reset | No separately named substates |

**Total major LTSSM states: 11.**

The Initial State is the entry/reset marker and is not counted among these 11.

## 1.5 Overall LTSSM Learning Scope

Our Module 09 coverage includes the complete LTSSM.

The learning sequence is:

**Initial Link Establishment**

```text
Initial State
     |
     v
Detect
     |
     v
Polling
     |
     v
Configuration
     |
     v
L0
```

**Link Retraining**

```text
L0
 |
 v
Recovery
 |
 v
L0
```

**Power Management**

```text
L0
 |
 +----> L0s
 |
 +----> L1
 |
 +----> L2
```

**Other States**

```text
Configuration / Recovery
          |
          +----> Disabled
          |
          +----> Loopback

Applicable Link operation
          |
          +----> Hot Reset
```

These are high-level relationships, not a complete transition diagram. The exact entry and exit conditions will be studied separately for every state.

## 1.6 LTSSM Status and LinkUp

`LinkUp` is a status indication associated with the Data Link Layer and tracked by the link-operating hardware.

For the normal initial Link-up path:

`Detect → Polling → Configuration`

`LinkUp = 0b` throughout initial Configuration.

When Configuration is entered from an already established Link through another applicable path, such as Recovery, `LinkUp = 1b`.

The validation engineer must distinguish:

- Initial Link establishment.
- Reconfiguration of an existing Link.
- Recovery of an existing Link.

The same Configuration substates can be used in different circumstances, but the conditions and active-lane rules can differ.

=====================================================================================================

# 2. PCIe DETECT STATE

## 2.1 Purpose of Detect

The purpose of Detect is to identify whether the expected far-end receiver termination is present on the relevant PCIe lanes.

This is an electrical receiver-detection procedure.

It does not confirm:

- NVMe controller readiness.
- Namespace availability.
- Successful PCIe packet transmission.
- Successful NVMe Read or Write commands.

Detect only establishes the physical receiver-presence condition necessary for subsequent Link training.

The Detect state contains two substates:

1. `Detect.Quiet`
2. `Detect.Active`

---

## 2.2 DETECT.QUIET

### 2.2.1 Purpose

`Detect.Quiet` is the initial Electrical Idle state in the normal Link initialization path.

The transmitter is placed in Electrical Idle.

The hardware establishes the electrical conditions required before performing Receiver Detection.

### 2.2.2 What Is Electrical Idle?

PCIe uses differential signaling.

Each lane contains:

- Positive signal conductor (P).
- Negative signal conductor (N).

During Electrical Idle, the transmitter is not actively transmitting normal differential data.

The differential voltage is approximately zero because the two conductors of the differential pair are at approximately the same electrical potential.

Electrical Idle does not mean that the host or NVMe SSD is powered off.

The DC common-mode voltage is not required to be within its normal operating specification while in this state.

### 2.2.3 Data Rate

Under the PCIe 5.0 baseline, 2.5 GT/s is selected as the frequency of operation.

If the frequency of operation was not 2.5 GT/s on entry to `Detect.Quiet`, the LTSSM must remain in this substate for at least 1 ms while changing the frequency of operation to 2.5 GT/s.

This frequency change does not affect the advertised data rates in TS1 and TS2 Ordered Sets.

### 2.2.4 Receiver Electrical Conditions

All Receivers must meet the applicable ZRX-DC specification for 2.5 GT/s within 1 ms of entering this substate.

The LTSSM must remain in `Detect.Quiet` until the required receiver electrical condition is met.

This ensures that the receivers are ready for the subsequent Receiver Detection procedure.

### 2.2.5 LinkUp Status

On entry into this substate:

`LinkUp = 0b`

The LinkUp status is cleared.

Relevant PCIe equalization status bits are also cleared as specified for the Detect initialization path.

These include the equalization status indicators for the applicable 8.0, 16.0 and 32.0 GT/s rates.

### 2.2.6 Conditions for Leaving Detect.Quiet

The LTSSM remains in `Detect.Quiet` until either of these conditions occurs:

1. The 12 ms timeout expires.
2. Electrical Idle is broken on any lane.

Electrical Idle is considered broken when the applicable electrical criteria indicate that the incoming signal has left the idle condition.

An arbitrary voltage change does not automatically mean the Electrical Idle exit criteria have been satisfied.

**Transition:**

`Detect.Quiet → Detect.Active`

### 2.2.7 Important Timeout Distinction

The 12 ms timeout used for the transition:

`Detect.Quiet → Detect.Active`

is separate from the 12 ms wait used in the partial Receiver Detection confirmation procedure inside `Detect.Active`.

These are two different timing requirements.

### 2.2.8 What Has Been Achieved?

At the end of Detect.Quiet:

- The transmitter is in Electrical Idle.
- The operating data rate is prepared at 2.5 GT/s.
- Required receiver electrical conditions are established.
- LinkUp is cleared.
- The LTSSM is ready to perform Receiver Detection.

However, the far-end receiver has not yet been confirmed.

### 2.2.9 Enterprise Storage Validation Perspective

Suppose an enterprise NVMe SSD does not establish a PCIe Link after power-on.

At this stage, the validation engineer must not immediately declare the SSD defective.

Possible investigation areas include:

- Host Root Port electrical behaviour.
- SSD Endpoint electrical behaviour.
- Connector and slot connectivity.
- Riser or backplane issues.
- Platform power and reset sequencing.
- Receiver termination behaviour.
- PCIe PHY or signal-integrity issues.

Ordinary Linux tools do not expose all internal Electrical Idle and Receiver Detection conditions.

Platform hardware-debug facilities or protocol/electrical analysis tools may be required for deeper investigation.

---

## 2.3 DETECT.ACTIVE

### 2.3.1 Purpose

In `Detect.Active`, the transmitter performs a Receiver Detection sequence on all relevant unconfigured lanes that can form one or more Links.

The detection circuitry checks the electrical response to determine whether the expected receiver termination is present at the far end of a lane.

This is an electrical detection procedure.

It does not check whether the remote component can execute NVMe commands.

### 2.3.2 Receiver Detection

Consider an enterprise host connected to an NVMe SSD through a PCIe interface.

The host Root Port has one or more physical PCIe lanes.

For every relevant unconfigured lane, the transmitter performs the specified Receiver Detection procedure.

The result indicates whether the expected receiver termination is detected.

The receiver-detection results determine how the LTSSM progresses.

### 2.3.3 Receiver-Detection Outcomes

There are three principal outcomes:

1. Receiver detected on all relevant unconfigured lanes.
2. Receiver not detected on any relevant lane.
3. Receiver detected on at least one, but not all, relevant unconfigured lanes.

### 2.3.4 Case 1 — Receiver Detected on All Relevant Lanes

If a receiver is detected on all relevant unconfigured lanes, the LTSSM proceeds to Polling.

**Transition:**

`Detect.Active → Polling.Active`

Receiver detection has succeeded.

However, the Link is not yet fully trained.

It has not reached the normal operational state L0.

### 2.3.5 Case 2 — No Receiver Detected on Any Lane

If a receiver is not detected on any relevant lane, the LTSSM returns to `Detect.Quiet`.

**Transition:**

`Detect.Active → Detect.Quiet`

The Receiver Detection process may be attempted again according to the Detect-state rules.

This result alone does not prove that the SSD is defective or physically absent.

Possible reasons include:

- SSD not powered or not ready.
- Incorrect platform power sequencing.
- Missing receiver termination.
- Connector or slot problem.
- Riser or backplane issue.
- Host or device PHY problem.
- Hardware or signal-integrity issue.

### 2.3.6 Case 3 — Partial Receiver Detection

Partial Receiver Detection occurs when at least one, but not all, relevant unconfigured lanes detect a receiver.

Consider an intended PCIe x4 connection.

**First Receiver Detection attempt:**

| Lane | First Attempt |
|---|---|
| Lane 0 | Receiver detected |
| Lane 1 | Receiver detected |
| Lane 2 | Receiver not detected |
| Lane 3 | Receiver not detected |

This is partial detection.

The PCIe specification defines a confirmation procedure.

### 2.3.7 Partial Detection Confirmation Procedure

When partial detection occurs:

1. Wait for 12 ms.
2. Repeat Receiver Detection on all relevant unconfigured lanes that can form one or more Links.
3. Compare the complete set of lanes that detected receivers during the first and second attempts.

The transmitter must not repeat Receiver Detection only on the lanes that failed the first attempt.

The confirmation procedure compares the identity of the detected lanes, not merely the total number of successful detections.

### 2.3.8 Case 3A — Same Set of Lanes Detects Receivers

Suppose the results are:

| Lane | First Attempt | Second Attempt |
|---|---|---|
| Lane 0 | Detected | Detected |
| Lane 1 | Detected | Detected |
| Lane 2 | Not detected | Not detected |
| Lane 3 | Not detected | Not detected |

The detected set is:

`{Lane 0, Lane 1}`

during both attempts.

Because exactly the same lanes detect receivers during both attempts, the LTSSM can transition to Polling on the detected lanes.

**Transition:**

`Detect.Active → Polling.Active`

Lanes that did not detect receivers are handled according to the optional multiple-Link behaviour.

If the optional feature is supported:

- The unused lanes may be associated with a new LTSSM.

If the optional feature is not supported:

- Lanes that cannot be associated with another LTSSM must transition to Electrical Idle.
- They must be re-associated with the LTSSM immediately after the LTSSM in progress returns to Detect.
- An EIOS is not required before those lanes transition to Electrical Idle.

**Important:** Detecting receivers on two lanes does not automatically guarantee that the final configured Link will be x2.

The final Link width is determined through the subsequent training and Configuration process.

### 2.3.9 Case 3B — The Detected Lane Set Changes

Suppose the results are:

| Lane | First Attempt | Second Attempt |
|---|---|---|
| Lane 0 | Detected | Detected |
| Lane 1 | Detected | Detected |
| Lane 2 | Detected | Not detected |
| Lane 3 | Not detected | Not detected |

The detected set changes from:

`{Lane 0, Lane 1, Lane 2}`

to:

`{Lane 0, Lane 1}`

The detected lane sets are not identical.

The LTSSM returns to `Detect.Quiet`.

**Transition:**

`Detect.Active → Detect.Quiet`

The purpose of the retry is to determine whether the partial detection result is repeatable.

The hardware compares the identities of the detected lanes.

It does not simply check whether the number of successful detections is the same.

### 2.3.10 Detect.Active Transition Summary

| Receiver-Detection Result | Next State |
|---|---|
| Receiver detected on all relevant unconfigured lanes | Polling.Active |
| Receiver detected on no relevant lane | Detect.Quiet |
| Partial detection; same detected lane set on both attempts | Polling.Active on detected lanes |
| Partial detection; detected lane set differs between attempts | Detect.Quiet |

The partial-detection confirmation procedure uses a 12 ms wait before the second Receiver Detection attempt.

### 2.3.11 Enterprise Troubleshooting Perspective

Receiver Detection provides evidence about the electrical Link initialization process.

It does not automatically establish the root cause of a failure.

For an enterprise NVMe SSD that fails to establish the expected Link, possible investigation areas include:

- Host Root Port behaviour.
- SSD Endpoint behaviour.
- PCIe lane connectivity.
- Connector or slot problems.
- Riser or backplane problems.
- Platform configuration.
- PCIe hardware or signal-integrity problems.

Ordinary Linux tools can report device enumeration, negotiated Link speed and width, kernel events and some PCIe errors.

However, ordinary Linux tools do not generally expose every internal per-lane Receiver Detection result.

A supported hardware-debug interface or protocol analyser may be needed to obtain additional evidence.

The validation engineer must establish what happened, collect evidence, isolate the responsible layer and only then attribute the defect to the SSD or platform.

---

## 2.4 DETECT STATE — COMPLETE TRANSITION SUMMARY

```text
                    Detect.Quiet
                         |
           12 ms timeout or EI exit
                         |
                         v
                    Detect.Active
                         |
          +--------------+--------------+
          |              |              |
          v              v              v
      All lanes       No lanes       Partial
      detected        detected       detection
          |              |              |
          v              v              v
    Polling.Active  Detect.Quiet    Wait 12 ms
                                         |
                                         v
                                 Repeat detection
                                         |
                              +----------+----------+
                              |                     |
                              v                     v
                          Same lane set       Different lane set
                              |                     |
                              v                     v
                        Polling.Active         Detect.Quiet
```

The diagram shows the normal Detect progression and partial-detection confirmation procedure.

======================================================================================================

# 3. POLLING STATE

## 3.1 Purpose of Polling

Detect checks whether the expected receiver termination is present at the opposite end of the PCIe connection.

Successful Receiver Detection does not prove that the receiver can synchronize with and correctly interpret the incoming serial data.

Polling establishes the initial synchronization and training conditions required for PCIe Link initialization.

During Polling, the transmitter sends predefined training patterns called TS1 and TS2 Ordered Sets.

- TS1 means Training Sequence 1.
- TS2 means Training Sequence 2.
- An Ordered Set is a specially structured sequence of symbols defined by the PCIe specification.

TS1 and TS2 Ordered Sets are not NVMe Read or Write commands.

They are used for PCIe Link training.

## 3.2 Bidirectional Training

Both link partners have their own transmitter and receiver.

The Root Port transmitter sends training sequences that the opposite Port's receiver attempts to recognize.

The opposite Port independently transmits its own training sequences, which are received and examined by the Root Port.

The SSD or switch does not simply echo the Root Port's training sequence.

Both link partners independently generate training sequences according to the PCIe LTSSM rules.

## 3.3 Polling Does Not Mean the Link Is Operational

Polling is an initial Link-training stage.

Successful transition out of Polling does not mean that the Link has reached L0.

The main Configuration state must complete its procedures before the Link enters L0, the normal operational state.

## 3.4 Initial Data Rate and PCIe Generations

Under the PCIe 5.0 baseline, the Link initially trains at 2.5 GT/s.

A Gen4 or Gen5 SSD does not normally begin its initial LTSSM training directly at 16 GT/s or 32 GT/s.

The Link initially reaches L0 at 2.5 GT/s.

Higher common supported data rates are advertised in the training Ordered Sets.

The other Port's speed capability is registered during `Configuration.Complete`.

Based on the highest supported common data rate, either side can initiate a speed change from L0 by transitioning to Recovery.

| PCIe Generation | Nominal Data Rate per Lane | Encoding / Signaling |
|---|---:|---|
| Gen1 | 2.5 GT/s | 8b/10b |
| Gen2 | 5 GT/s | 8b/10b |
| Gen3 | 8 GT/s | 128b/130b |
| Gen4 | 16 GT/s | 128b/130b |
| Gen5 | 32 GT/s | 128b/130b |
| Gen6 | 64 GT/s | PAM4, FLIT mode and FEC |

The Gen6 row is included for awareness.

Detailed transition conditions in these notes use the PCIe 5.0 baseline.

The initial 2.5 GT/s training process uses the 8b/10b training representation, including when the Link's components support higher generations.

## 3.5 POLLING.ACTIVE

### 3.5.1 Purpose

`Polling.Active` begins the initial exchange of TS1 Ordered Sets between the link partners.

The transmitter sends TS1 Ordered Sets on all lanes that detected a receiver during Detect.

The receiving Port examines these Ordered Sets to establish the initial training conditions required for further Link initialization.

### 3.5.2 Initial TS1 Ordered Set Transmission

During `Polling.Active`, the transmitter sends TS1 Ordered Sets with:

| TS1 Field | Initial Value |
|---|---|
| Link Number | PAD |
| Lane Number | PAD |

This applies to all lanes that detected a receiver during Detect.

PAD is a placeholder used during the early training process, before Link and Lane Numbers have been established.

The final association of physical lanes into a logical Link is determined during the subsequent Configuration process.

### 3.5.3 Data Rate Identifier

The Data Rate Identifier symbol in TS1 advertises all data rates supported by the Port, including rates that the Port does not intend to use.

Advertising a rate does not mean that the Link is currently operating at that data rate.

The current initial training rate remains 2.5 GT/s under the PCIe 5.0 baseline.

### 3.5.4 Transmitter Common-Mode Settling

Before exiting Electrical Idle and transmitting TS1 Ordered Sets, the transmitter must allow its TX common mode to settle.

The transmitter must drive patterns at the default voltage level of the Transmit Margin field within 192 ns of entry into this substate.

This transmit voltage level remains effective until Polling.Compliance or Recovery.RcvrLock is entered.

### 3.5.5 Bit Lock

#### What Problem Does Bit Lock Solve?

The transmitter sends a serial stream of bits over a PCIe lane.

The receiver initially receives an electrical waveform rather than a row of already-labelled binary digits.

The receiver must recover the timing required to sample the incoming signal correctly.

This function uses Clock and Data Recovery (CDR).

CDR recovers timing information from the incoming serial signal so that the receiver can sample the transmitted bits at the appropriate times.

**Bit lock** means the receiver has recovered the timing required to sample the incoming bit stream correctly.

Without suitable bit timing, the receiver cannot reliably recover the transmitted bits.

Bit lock does not mean that all Link-training operations are complete.

#### Validation Relevance

If the receiver cannot establish the required bit timing, it may not correctly recover the incoming training sequence.

As a result, the LTSSM may fail to progress through Link training.

A validation engineer should investigate available PCIe Link-training evidence instead of immediately concluding that the NVMe SSD is defective.

### 3.5.6 Symbol Lock

Bit lock and symbol lock solve different problems.

- Bit lock establishes the timing needed to recover individual bits.
- Symbol lock identifies the boundaries of encoded symbols in the recovered bit stream.

For the initial 2.5 GT/s PCIe training process, 8b/10b encoding is used.

In 8b/10b encoding, an 8-bit data or control value is represented by a 10-bit transmitted code group.

The receiver must identify where encoded symbols begin and end.

#### The COM Symbol

The COM symbol is the 8b/10b control character K28.5.

It provides a recognizable alignment pattern that the receiver can use to establish the correct symbol boundary.

Remember:

- Bit lock: Recover the timing of incoming bits.
- Symbol lock: Identify the boundaries of encoded symbols.

For PCIe Gen3 through Gen5, 128b/130b encoding is used during higher data-rate operation.

However, initial Polling begins at 2.5 GT/s in the PCIe 5.0 baseline.

### 3.5.7 Lane Polarity Inversion

#### What Is Polarity Inversion?

PCIe uses differential signaling.

Each PCIe lane has a transmit differential pair and a receive differential pair.

Each differential pair has two signal conductors:

- Positive (P).
- Negative (N).

If the P and N conductors of a differential pair are swapped, the polarity of the received differential signal is reversed.

This condition is called **lane polarity inversion**.

Polarity inversion is different from Lane Reversal:

- Polarity inversion swaps P and N within one differential lane.
- Lane Reversal changes the ordering of lanes within a multi-lane Link.

#### How Does the Receiver Detect Polarity Inversion?

During Polling, the receiver examines specific symbols in the received TS1 and TS2 Ordered Sets.

For the initial 8b/10b training representation, the PCIe specification defines recognizable normal-polarity and inverted-polarity symbol patterns.

For TS1:

- Normal polarity: Symbols 6–15 contain the expected D10.2 pattern.
- Inverted polarity: Symbols 6–15 are received as D21.5.

For TS2:

- Normal polarity: Symbols 6–15 contain the expected D5.2 pattern.
- Inverted polarity: Symbols 6–15 are received as D26.5.

These patterns allow the receiver to distinguish normal polarity from inverted polarity.

The receiver does not simply assume that the physical wires are swapped.

#### What Happens if Polarity Is Inverted?

If the receiver detects polarity inversion, it must invert the received data to compensate for the inversion.

The transmitter must not invert its transmitted data to perform this correction.

Support for lane polarity inversion is required independently for each PCIe receiver lane.

#### What Happens if Polarity Is Normal?

If no polarity inversion is present, the receiver does not need to apply polarity correction to that lane.

If the expected training patterns cannot be recognized, the receiver cannot automatically conclude that polarity inversion is the cause.

Other training or signal-integrity problems may need investigation.

#### When Must Polarity Correction Be Completed?

The PCIe 5.0 specification requires polarity correction, if necessary, before the receiver exits `Polling.Configuration`.

Detection and correction may begin earlier, including during `Polling.Active`.

Therefore, do not assume that all polarity correction must have been completed during Polling.Active.

#### Validation Relevance

Polarity inversion is not automatically a hardware failure.

A supported inversion can be compensated for by the receiver.

The validation engineer must establish whether the Link trains, reaches the expected Link speed and width, and operates reliably.

### 3.5.8 Normal Transition: Polling.Active to Polling.Configuration

The normal transition requires two main conditions.

**Condition 1 — Transmitter Requirement**

At least 1,024 TS1 Ordered Sets must have been transmitted.

**Condition 2 — Receiver Requirement**

Every lane that detected a receiver during Detect must receive eight consecutive qualifying training sequences, or their complement, according to the specified criteria.

The qualifying training sequences include:

1. TS1 with Link Number and Lane Number set to PAD and the Compliance Receive bit deasserted (0b).
2. TS1 with Link Number and Lane Number set to PAD and the Loopback bit asserted (1b).
3. TS2 with Link Number and Lane Number set to PAD.

For this Polling.Active transition, the eight qualifying training sequences may be a mixture of qualifying TS1 and TS2 Ordered Sets.

They do not necessarily have to be eight TS1 sequences or eight TS2 sequences exclusively.

When the required conditions are satisfied:

`Polling.Active → Polling.Configuration`

This transition can occur before the 24 ms timeout.

### 3.5.9 Polling.Active — 24 ms Timeout and Fallback Conditions

If the normal transition conditions are not satisfied before the 24 ms timeout expires, the LTSSM evaluates the specified fallback conditions.

The timeout is a decision point, not an automatic declaration that the SSD is defective.

There are three principal timeout outcomes.

#### Outcome A — Continue to Polling.Configuration

The LTSSM can transition to `Polling.Configuration` if all the following conditions are satisfied:

1. At least one lane that detected a receiver during Detect has received eight consecutive qualifying training sequences, or their complement.

2. At least 1,024 TS1 Ordered Sets have been transmitted after receiving one TS1 or TS2 Ordered Set.

3. At least a predetermined set of lanes that detected receivers during Detect has detected an exit from Electrical Idle at least once on every lane in that set since entering Polling.Active.

The predetermined lane set is implementation-specific.

It must not automatically be assumed to be identical to the complete set of initially detected lanes in every implementation.

The intent of this fallback is to prevent a defective receiver or transmitter on one or more lanes from holding up a valid Link, allowing further training to continue in Polling.Configuration.

Any lane that receives the eight consecutive TS1 or TS2 Ordered Sets should have detected an exit from Electrical Idle at least once since entering Polling.Active.

#### Outcome B — Enter Polling.Compliance

If Outcome A does not apply, the LTSSM enters Polling.Compliance if either of the following conditions is satisfied.

**Condition B1 — Electrical Idle Exit Condition**

Not all lanes in the implementation-defined predetermined lane set have detected an exit from Electrical Idle since entering Polling.Active.

**Condition B2 — Compliance Request Through TS1**

A lane that detected a receiver during Detect receives eight consecutive TS1 Ordered Sets, or their complement, with:

| TS1 Field | Required Value |
|---|---|
| Link Number | PAD |
| Lane Number | PAD |
| Compliance Receive | 1b |
| Loopback | 0b |

These conditions select the compliance-testing path.

**Important specification note:** If a passive test load is applied on all lanes, the device enters Polling.Compliance.

#### Outcome C — Return to Detect.Quiet

If neither Outcome A nor Outcome B applies, the LTSSM returns to `Detect.Quiet`.

**Transition:**

`Polling.Active → Detect.Quiet`

The Link begins the detection process again.

#### Important Difference Between Normal Transition and Timeout Fallback

**Normal transition:**

- At least 1,024 TS1 Ordered Sets transmitted.
- Eight qualifying training sequences received on every lane that detected a receiver during Detect.

**Timeout fallback:**

- At least one qualifying lane.
- The required TS1 transmission count after receiving one TS1 or TS2.
- The implementation-defined predetermined set of lanes satisfies the Electrical Idle exit requirement.

Do not confuse the two sets of conditions.

### 3.5.10 Direct Entry to Polling.Compliance

A separate route to Polling.Compliance exists through the Enter Compliance bit in the Link Control 2 register.

This is bit 4 of the Link Control 2 register.

If the Enter Compliance bit is set before entry into Polling.Active, the transition to Polling.Compliance must be immediate without sending any TS1 Ordered Sets.

If the bit is asserted while Polling.Active is in progress, the applicable compliance-entry rules must be followed.

This is a compliance-test control path, not the normal NVMe Link-initialization path.

### 3.5.11 Polling.Active — Transition Summary

| Condition | Next State |
|---|---|
| Normal transmitter and receiver training conditions satisfied | Polling.Configuration |
| 24 ms expires and fallback conditions permit further training | Polling.Configuration |
| 24 ms expires and a specified compliance condition applies | Polling.Compliance |
| 24 ms expires and neither fallback applies | Detect.Quiet |
| Enter Compliance set before Polling.Active entry | Polling.Compliance immediately, without sending TS1 |

### 3.5.12 Storage Validation Perspective

If an NVMe SSD fails to progress beyond Polling.Active, the validation engineer should determine whether the required TS1 exchange and receiver conditions were achieved.

Possible investigation areas include:

- TS1 Ordered Set transmission and reception.
- CDR and bit-lock behaviour.
- Symbol synchronization.
- Lane polarity.
- Electrical Idle exit detection.
- Root Port and Endpoint receiver behaviour.
- Compliance settings.
- Physical lane connectivity.
- Available PCIe protocol traces.

The 24 ms timeout alone does not identify the root cause.

A failure in Polling.Active must be investigated at the physical-link and training-sequence level before attributing it to the SSD.

---

## 3.6 POLLING.COMPLIANCE

### 3.6.1 Purpose

`Polling.Compliance` is intended for PCIe electrical compliance testing.

It allows the transmitter and interconnect to be evaluated using PCIe-defined compliance and test patterns.

It is not a required stage of ordinary NVMe SSD startup.

The test environment may use equipment such as:

- Oscilloscope.
- Pattern generator.
- Bit Error Rate Tester (BERT).
- Other PCIe compliance test equipment.

The compliance pattern and transmission settings depend on the component capabilities, selected compliance setting and the mechanism used to enter this substate.

### 3.6.2 Entry Conditions

Polling.Compliance can be entered through specified conditions, including:

1. The Enter Compliance bit in Link Control 2 is set before entry into Polling.Active.
2. The 24 ms Polling.Active timeout fallback selects Polling.Compliance.
3. The receiver observes eight consecutive qualifying TS1 Ordered Sets with Compliance Receive asserted and Loopback deasserted.
4. A compliance-test environment produces one of the specified compliance-entry conditions.

The precise behaviour depends on the entry mechanism.

### 3.6.3 Transmit Margin

On entry to Polling.Compliance, the Transmit Margin field of Link Control 2 is sampled.

The selected voltage level becomes effective at the transmitter package pins within 192 ns of entry and remains effective during Polling.Compliance.

### 3.6.4 Compliance Pattern Data Rate and Settings

The compliance transmission rate and de-emphasis or preset settings are selected according to the entry mechanism and the Port's supported data rates.

Examples include:

- A Port capable of transmitting only at 2.5 GT/s uses the 2.5 GT/s compliance pattern and -3.5 dB de-emphasis.
- When entry is triggered by the Compliance Receive TS1 mechanism, the highest common data rate indicated by the transmitted and received Data Rate Identifiers is used.
- The selectable de-emphasis or preset values are determined by the applicable received training information and compliance-test sequence.

The compliance pattern sequence can cycle through different rates and de-emphasis or preset settings on successive entries into Polling.Compliance.

The PCIe specification also defines Modified Compliance Pattern behaviour for applicable higher-speed testing.

These are electrical testing operations, not ordinary NVMe data transactions.

### 3.6.5 Compliance Pattern Types

Depending on the applicable compliance setting, the transmitter can send:

- Standard Compliance Pattern.
- Modified Compliance Pattern.
- Jitter Measurement Pattern on specified lanes.

The detailed pattern selection is defined by the compliance sequence setting and the PCIe specification.

### 3.6.6 Exiting Polling.Compliance

The exit conditions depend on why the Port entered Polling.Compliance.

#### Pattern-Driven Compliance Path

For the compliance-pattern cycling path, the LTSSM can return to Polling.Active when the receiver detects an exit from Electrical Idle on any lane that detected a receiver during Detect.

If the transmitter was operating above 2.5 GT/s, the required return-to-training-rate sequence must be performed.

The procedure includes:

1. Sending the required consecutive EIOSs.
2. Entering Electrical Idle.
3. Changing the data rate to 2.5 GT/s.
4. Allowing the new rate to stabilize.

For this applicable return sequence, the Electrical Idle period is greater than 1 ms and must not exceed 2 ms.

#### Register-Directed Compliance Path

When compliance testing was initiated through the Enter Compliance register mechanism, exit conditions depend on the register state and Port role.

Examples include:

- Enter Compliance changes from 1b to 0b.
- An Upstream Port receives an EIOS on any lane under the applicable register-directed exit rule.

When the specified conditions apply, the transmitter can be required to send eight consecutive EIOSs, enter Electrical Idle, change back to 2.5 GT/s and stabilize the operating rate before returning to Polling.Active.

If the Enter Compliance bit is cleared under the specified conditions, a transition to Detect may also be required.

The exact exit sequence must be interpreted according to the entry path defined by the PCIe specification.

### 3.6.7 Validation Relevance

If an NVMe SSD enters Polling.Compliance unexpectedly, first determine whether the platform intentionally requested compliance testing.

Do not immediately treat Polling.Compliance as proof that the SSD has failed.

If compliance testing was not intended, investigate:

- Link Control 2 compliance settings.
- TS1 Compliance Receive and Loopback fields.
- Electrical Idle behaviour.
- Training-sequence reception.
- Host Root Port behaviour.
- SSD Endpoint behaviour.
- Compliance-test load conditions.
- Available hardware-debug or protocol-analyser evidence.

---

## 3.7 POLLING.CONFIGURATION

### 3.7.1 Purpose

`Polling.Configuration` follows Polling.Active when the applicable normal or timeout-fallback transition requirements are satisfied.

Its purpose is to continue initial Link training using TS2 Ordered Sets and establish the conditions required to transition to the main Configuration state.

### 3.7.2 Entry Actions

At entry to Polling.Configuration:

1. The receiver must invert polarity if necessary.
2. The Transmit Margin field of the Link Control 2 register must be reset to 000b.
3. The transmitter's Polling.Compliance sequence setting is updated if required by the specification.

Polarity correction must be complete before the receiver exits Polling.Configuration.

Polarity inversion may have been detected and corrected earlier during Polling.Active.

### 3.7.3 TS2 Ordered Set Transmission

During Polling.Configuration, the transmitter sends TS2 Ordered Sets on all lanes that detected a receiver during Detect.

Both Link Number and Lane Number remain set to PAD.

| TS2 Field | Value |
|---|---|
| Link Number | PAD |
| Lane Number | PAD |

The Data Rate Identifier symbol advertises all data rates supported by the Port, including rates the Port does not intend to use.

The initial training rate remains 2.5 GT/s under the PCIe 5.0 baseline.

### 3.7.4 Normal Transition to Configuration

The normal transition requires two independent conditions.

**Condition 1 — Receiver Requirement**

Eight consecutive TS2 Ordered Sets, with Link Number and Lane Number set to PAD, must be received on any lane or lanes that detected a receiver during Detect.

This is different from the Polling.Active normal transition.

Polling.Active requires qualifying training sequences on every lane that detected a receiver during Detect.

Polling.Configuration requires the eight consecutive TS2 condition on any lane or lanes that detected a receiver during Detect.

It does not impose the same all-lanes requirement.

**Condition 2 — Transmitter Requirement**

The transmitter must send at least 16 TS2 Ordered Sets after receiving one TS2 Ordered Set.

These are separate requirements.

The transmitter is not required to send 16 TS2 Ordered Sets for every received TS2 Ordered Set.

When both conditions are satisfied:

`Polling.Configuration → Configuration.Linkwidth.Start`

This is the first substate of the main Configuration state.

### 3.7.5 Polling.Configuration — 48 ms Timeout

The 48 ms timer applies to Polling.Configuration.

It is separate from the 24 ms timer used in Polling.Active.

The normal transition can occur as soon as the required TS2 conditions are satisfied. It does not have to wait for the complete 48 ms timeout.

If the required transition conditions are not satisfied within 48 ms, the LTSSM returns to Detect.

**Transition:**

`Polling.Configuration → Detect`

The next entry into Detect begins with Detect.Quiet.

Unlike the 24 ms timeout in Polling.Active, the Polling.Configuration 48 ms timeout does not use the same three-way fallback decision.

Its specified timeout outcome is a return to Detect.

### 3.7.6 Validation Relevance

If an NVMe SSD repeatedly fails to leave Polling.Configuration, the evidence indicates that the TS2 training stage did not complete within the specified period.

The timeout alone does not identify the root cause.

The validation engineer should investigate:

- TS2 Ordered Set transmission and reception.
- Receiver recognition.
- Link and Lane PAD fields.
- Polarity correction.
- Electrical Idle exit conditions.
- Host and device training behaviour.
- Platform configuration.
- Available diagnostic evidence.

---

## 3.8 POLLING.SPEED

### 3.8.1 Purpose and Specification Status

`Polling.Speed` is included in the PCIe 5.0 LTSSM specification.

However, it is unreachable in the normal PCIe 5.0 Link-initialization flow.

A Link initially reaches L0 at 2.5 GT/s.

A subsequent change to a higher supported data rate is performed through Recovery.

### 3.8.2 Higher Data-Rate Support

The supported higher data rates are advertised in the TS Ordered Sets.

The other side's speed capability is registered during `Configuration.Complete`.

Based on the highest common supported data rate, either side can initiate a speed change from L0 by transitioning to Recovery.

### 3.8.3 Learning Scope

`Polling.Speed` remains part of the complete LTSSM inventory.

We will document its specification status and why it is unreachable in the normal PCIe 5.0 initial Link-up sequence.

It must not be described as a required intermediate stage between Polling.Configuration and Configuration.

---

## 3.9 POLLING STATE — COMPLETE TRANSITION SUMMARY

| Current State | Condition | Next State |
|---|---|---|
| Detect.Active | Receiver detected on all relevant unconfigured lanes | Polling.Active |
| Polling.Active | Normal training conditions satisfied | Polling.Configuration |
| Polling.Active | 24 ms timeout; fallback conditions permit further training | Polling.Configuration |
| Polling.Active | 24 ms timeout; a specified compliance condition applies | Polling.Compliance |
| Polling.Active | 24 ms timeout; neither fallback applies | Detect.Quiet |
| Polling.Active | Enter Compliance set before entry | Polling.Compliance immediately, without sending TS1 |
| Polling.Compliance | Applicable compliance exit condition satisfied | Polling.Active or specified alternate outcome, depending on entry path |
| Polling.Configuration | Normal TS2 transition conditions satisfied | Configuration.Linkwidth.Start |
| Polling.Configuration | 48 ms timeout without normal transition conditions | Detect |

====================================================================================================

# 4. CONFIGURATION STATE

## 4.1 Purpose of the Configuration State

The purpose of Configuration is to configure the physical PCIe lanes of a Port into one or more logical PCIe Links.

During Detect:

- The hardware identifies the presence of the expected receiver termination.

During Polling:

- The link partners exchange TS1 and TS2 Ordered Sets.
- Initial synchronization and training conditions are established.
- Lane polarity is configured.

However, successful completion of Polling does not mean that the final PCIe Link has already been formed.

Configuration performs the Link-width and Lane negotiation sequence.

It also performs additional operations such as:

- Lane-to-Lane de-skew.
- Scrambling control where permitted.
- N_FTS configuration.
- Data-rate capability exchange.
- Entry into Disabled or Loopback when directed.

Links are formed at the conclusion of Configuration.

## 4.2 Entry into the Configuration State

Under the normal initial Link-training path, Configuration is entered from Polling.Configuration.

The transition occurs after the required TS2 training conditions are satisfied.

**Transition:**

`Polling.Configuration → Configuration.Linkwidth.Start`

`Configuration.Linkwidth.Start` is the first substate of the main Configuration state.

There is no separate generic Configuration operation before Linkwidth.Start.

## 4.3 LinkUp During Configuration

The value of LinkUp depends on how the LTSSM entered Configuration.

**Initial Link establishment:**

`Detect → Polling → Configuration`

In this case:

`LinkUp = 0b`

**Reconfiguration of an existing Link:**

If Configuration is entered from another state, such as Recovery:

`LinkUp = 1b`

This distinction matters because certain Configuration procedures differ between initial Link establishment and reconfiguration of an existing Link.

## 4.4 Participating PCIe Ports — Enterprise Switch Example

Throughout our Configuration learning, we use this enterprise PCIe switch architecture.

```text
                       HOST
                  Root Complex
                        |
                    Root Port
                 (Downstream Port)
                        |
                    PCIe Link 1
                        |
                Switch Upstream Port
                +-------------------+
                |    PCIe SWITCH    |
                +-------------------+
                    |           |
             Downstream Port 0  Downstream Port 1
                    |           |
                PCIe Link 2  PCIe Link 3
                    |           |
                 NVMe SSD       HBA
                 Endpoint     Endpoint
```

There are three separate PCIe Links:

1. Root Port ↔ Switch Upstream Port.
2. Switch Downstream Port 0 ↔ NVMe SSD Endpoint.
3. Switch Downstream Port 1 ↔ HBA Endpoint.

Each Link has its own LTSSM.

The Configuration process on Link 1 does not configure Link 2 or Link 3.

For our current learning, we concentrate only on Link 1:

**Root Port (Downstream Port) ↔ Switch Upstream Port (Upstream Port).**

The two ports are the direct link partners for this LTSSM example.

## 4.5 Link Number and Lane Number

Before learning the Configuration substates, two TS1 fields must be understood.

### 4.5.1 Link Number

The Link Number identifies the logical PCIe Link to which a group of physical lanes belongs.

For our example, assume the Root Port selects Link Number N.

The physical lanes selected to form this Link carry the same Link Number N in their training sequences.

### 4.5.2 Lane Number

The Lane Number identifies an individual lane within a particular logical PCIe Link.

For an illustrative x4 Link:

| Physical Lane | Link Number | Lane Number |
|---|---|---|
| Lane 0 | N | 0 |
| Lane 1 | N | 1 |
| Lane 2 | N | 2 |
| Lane 3 | N | 3 |

All four lanes have the same Link Number because they belong to the same Link.

Each lane has a different Lane Number because each represents a different position within that Link.

### 4.5.3 Difference Between Link Number and Lane Number

| Field | Question Answered |
|---|---|
| Link Number | Which logical Link does this lane belong to? |
| Lane Number | Which position does this lane occupy within that Link? |

Remember:

- Link Number = Which Link?
- Lane Number = Which lane within that Link?

These are protocol identifiers, not security or encryption mechanisms.

### 4.5.4 PAD

PAD is a special placeholder value used in the training Ordered Sets.

For the initial 8b/10b training representation, PAD is encoded using K23.7.

Examples:

`Link Number = N, Lane Number = PAD`

The Link Number has been selected, but the Lane Number has not yet been assigned in that transmitted TS1.

`Link Number = PAD, Lane Number = PAD`

Neither identifier has yet been assigned in those transmitted TS1 fields.

---

# 5. CONFIGURATION.LINKWIDTH.START

## 5.1 Purpose

`Configuration.Linkwidth.Start` begins the initial Link-width negotiation process.

The two link partners must establish which physical lanes can be associated with a particular logical PCIe Link.

For our normal initial-Link-training example:

- Root Port is the Downstream Port.
- Switch Upstream Port is the Upstream Port.
- Four physical lanes are assumed to be available.
- All four lanes detected receivers during Detect.
- LinkUp = 0b because this is initial Link establishment.
- We assume one intended x4 Link, rather than multiple Links or optional crosslink operation.

The selected Link Number N is an illustrative value used to explain the protocol exchange.

## 5.2 Downstream Port — Initial TS1 Transmission

The Root Port is the Downstream Port.

On entering `Configuration.Linkwidth.Start` from Polling, any Downstream Lane that detected a receiver during Detect is considered an active lane.

For initial Link training, if LinkUp = 0b, the Root Port transmits TS1 Ordered Sets with:

| TS1 Field | Transmitted Value |
|---|---|
| Link Number | Selected Link Number N |
| Lane Number | PAD |

For our four-lane example:

| Root Port Physical Lane | Link Number | Lane Number |
|---|---|---|
| Lane 0 | N | PAD |
| Lane 1 | N | PAD |
| Lane 2 | N | PAD |
| Lane 3 | N | PAD |

The Root Port transmits its own TS1 Ordered Sets on the four active Downstream Lanes.

The Data Rate Identifier symbol advertises all data rates supported by the Port, including rates that the Port does not intend to use.

### 5.2.1 Important Initial-Link Rule

When the LTSSM enters this substate from Polling:

Any lane that detected a receiver during Detect is considered an active lane.

When the LTSSM enters from Recovery:

A lane that was part of the configured Link the previous time through Configuration.Complete is considered an active lane.

These two cases must not be confused.

### 5.2.2 Upconfiguration Case — Awareness

When LinkUp = 1b and the LTSSM is initiating an increase in Link width, the initial TS1 transmission rules are different.

The transmitter initially sends TS1 Ordered Sets with both Link and Lane Numbers set to PAD on:

- Current active lanes.
- Inactive lanes it intends to activate.
- Other specified inactive lanes that satisfy the Electrical Idle exit and received-PAD conditions.

After the required receiver conditions are satisfied, the transmitter begins sending the selected Link Number with the Lane Number set to PAD.

This is an existing-Link width-upconfiguration operation.

It is not the normal initial Link-up example being taught in our current lessons.

## 5.3 Upstream Port — Initial TS1 Transmission

The PCIe switch has an Upstream Port facing the Root Complex.

On entering this substate from Polling, any Upstream Lane that detected a receiver during Detect is considered active.

The Upstream Port independently transmits its own TS1 Ordered Sets.

Initially, it transmits:

| TS1 Field | Transmitted Value |
|---|---|
| Link Number | PAD |
| Lane Number | PAD |

For our four-lane example:

| Switch Physical Lane | Link Number | Lane Number |
|---|---|---|
| Lane 0 | PAD | PAD |
| Lane 1 | PAD | PAD |
| Lane 2 | PAD | PAD |
| Lane 3 | PAD | PAD |

The Upstream Port initially transmits PAD/PAD on its active Upstream Lanes.

The Root Port and Switch Upstream Port are not echoing each other's TS1 Ordered Sets.

Both Ports independently generate their own TS1 Ordered Sets.

## 5.4 Upstream Port — Recognizing the Downstream Port's Link Number

The Switch Upstream Port receives the Root Port's TS1 Ordered Sets through its physical lane receivers.

Initially, the Root Port is transmitting:

- Link Number = N.
- Lane Number = PAD.

The Upstream Port examines the received TS1 Ordered Sets on its lanes.

If any lane receives two consecutive TS1 Ordered Sets with:

1. A non-PAD Link Number.
2. Lane Number equal to PAD.

The Upstream Port selects a single Link Number.

For our example, the selected Link Number is N.

### 5.4.1 What Changes in the Upstream Port's TS1?

Initially, the Upstream Port transmits:

`PAD / PAD`

After receiving two consecutive qualifying TS1 Ordered Sets, it transmits:

`N / PAD`

For our example:

| Switch Physical Lane | Initial TS1 | Updated TS1 |
|---|---|---|
| Lane 0 | PAD/PAD | N/PAD |
| Lane 1 | PAD/PAD | N/PAD |
| Lane 2 | PAD/PAD | N/PAD |
| Lane 3 | PAD/PAD | N/PAD |

This table assumes all four lanes detected receivers and received the required qualifying sequences.

The Upstream Port transmits the selected Link Number and PAD Lane Number on all Upstream Lanes that:

1. Detected a receiver during Detect.
2. Also received two consecutive qualifying TS1 Ordered Sets with a non-PAD Link Number and PAD Lane Number.

Any remaining lanes that detected a receiver during Detect but did not satisfy the specified condition continue transmitting TS1 Ordered Sets with both Link and Lane Numbers set to PAD.

The Upstream Port then proceeds to `Configuration.Linkwidth.Accept`.

## 5.5 Downstream Port — Recognizing the Upstream Response

The Root Port is also receiving TS1 Ordered Sets from the Switch Upstream Port.

Initially, it receives:

`PAD / PAD`

It subsequently receives TS1 Ordered Sets containing:

- Link Number = N.
- Lane Number = PAD.

The Downstream Port's exact entry condition must be understood carefully.

For the normal initial-Link flow:

1. Any of the Downstream Lanes must first have received at least one TS1 Ordered Set with Link Number and Lane Number set to PAD.
2. After that, any of those same Downstream Lanes must receive two consecutive TS1 Ordered Sets with:
   - A non-PAD Link Number.
   - A Link Number matching one transmitted by the Downstream Lanes.
   - Lane Number = PAD.

When this condition is satisfied, the Downstream Port enters `Configuration.Linkwidth.Accept`.

**Special rule:** If crosslink configuration is not supported, the preliminary condition of first receiving PAD/PAD is always considered satisfied.

This is why, in our normal simplified example, the Root Port can progress after receiving the required two consecutive matching N/PAD TS1 Ordered Sets.

## 5.6 Independent LTSSM Behaviour

The Root Port and Switch Upstream Port each have their own LTSSM.

Therefore:

- The Upstream Port evaluates its own received training sequences.
- The Downstream Port evaluates its own received training sequences.
- Each Port makes its own transition decision.
- Their transitions are not one shared LTSSM event.

In our normal example, both Ports progress to `Configuration.Linkwidth.Accept` after satisfying their respective conditions.

## 5.7 Directed Transitions and Special Cases

The specification also defines special paths in Linkwidth.Start.

These include:

**Disabled**

A higher layer can direct the Downstream Port to assert the Disable Link bit in TS1 and TS2 Ordered Sets.

**Loopback**

A Port can be directed to assert the Loopback bit, subject to the applicable Loopback rules and its ability to act as a Loopback Master.

**Optional Crosslink**

If crosslink support is implemented, both connected Ports can initially attempt to behave as Downstream Ports.

The crosslink procedure resolves the two-port role conflict through the specified TS1 exchanges and random crosslink timeout.

Crosslink is an optional special case and is not part of our normal Root Port-to-switch learning example.

## 5.8 Linkwidth.Start — Timeout and Failure Behaviour

If the applicable transition conditions are not satisfied within 24 ms, the LTSSM returns to Detect.

**Transition:**

`Configuration.Linkwidth.Start → Detect`

The next entry into Detect begins with Detect.Quiet.

The timeout does not automatically prove that the SSD or switch is defective.

Possible investigation areas include:

- Root Port TS1 transmission behaviour.
- Switch Upstream Port TS1 transmission behaviour.
- TS1 Ordered Set reception and recognition.
- Link Number selection.
- Receiver-detection results.
- Lane connectivity.
- Crosslink or platform configuration, if applicable.
- Hardware-debug or protocol-analyser evidence.

## 5.9 Linkwidth.Start — Enterprise Validation Scenario

**Expected behaviour:**

A host Root Port and PCIe switch Upstream Port should begin the expected initial TS1 exchange.

**Observed behaviour:**

The LTSSM remains in `Configuration.Linkwidth.Start` and returns to Detect.

**Validation investigation:**

1. Confirm which lanes detected receivers during Detect.
2. Identify the active Downstream and Upstream Lanes.
3. Check the Root Port's TS1 Link Number and Lane Number fields.
4. Check the switch's initial PAD/PAD transmission.
5. Check whether the switch recognized the required Downstream TS1 sequences.
6. Check whether the switch began transmitting the selected Link Number.
7. Check whether the Root Port received the required consecutive matching TS1 sequences.
8. Investigate lane-specific errors and physical connectivity.
9. Collect supported protocol-analyser or hardware-debug evidence.

The timeout is evidence that the expected Linkwidth.Start transition conditions were not completed. It is not, by itself, root-cause evidence.

## 5.10 Linkwidth.Start — Transition Summary

| Port / Condition | Behaviour or Result |
|---|---|
| Downstream Port during initial Link-up | Sends TS1 with selected Link Number and PAD Lane Number on active Downstream Lanes |
| Upstream Port at entry | Sends its own TS1 with PAD/PAD on active Upstream Lanes |
| Upstream Port receives two consecutive qualifying TS1s with non-PAD Link and PAD Lane | Selects the Link Number and transmits it with PAD Lane Number on qualifying lanes |
| Downstream Port receives required prior PAD/PAD condition and then two consecutive matching non-PAD Link/PAD Lane TS1s | Enters Linkwidth.Accept |
| Crosslink not supported | Preliminary PAD/PAD receiving condition is considered satisfied |
| Applicable 24 ms timeout | Returns to Detect |

---

# 6. CONFIGURATION.LINKWIDTH.ACCEPT

## 6.1 Purpose

`Configuration.Linkwidth.Accept` identifies a valid group of physical lanes that can form a configured PCIe Link.

It also assigns Lane Numbers to the selected lanes.

The Downstream Port and Upstream Port follow different lane-selection and numbering rules.

Therefore, their behaviours must be understood separately.

Our normal example continues to be:

**Root Port (Downstream Port) ↔ Switch Upstream Port (Upstream Port).**

Assume one initial x4 Link is being established.

## 6.2 Downstream Lanes — Root Port Behaviour

### 6.2.1 What Information Does the Root Port Receive?

During Linkwidth.Start, the Root Port received TS1 Ordered Sets from the Switch Upstream Port.

In our four-lane example, the switch is transmitting:

`Link Number = N`

`Lane Number = PAD`

The Root Port evaluates the received TS1 Ordered Sets on its Downstream Lanes.

### 6.2.2 How Does the Root Port Identify Lanes for Link N?

The Downstream Port checks whether a configured Link can be formed with at least one group of lanes that received:

1. Two consecutive TS1 Ordered Sets.
2. The same received Link Number in both Ordered Sets.
3. A non-PAD Link Number.
4. A Link Number matching one that the Downstream Lanes transmitted.

For our single-Link example, the matching Link Number is N.

Consider the following example:

| Root Port Physical Lane | Received Link Number | Same Matching Link? |
|---|---|---|
| Lane 0 | N | Yes |
| Lane 1 | N | Yes |
| Lane 2 | N | Yes |
| Lane 3 | N | Yes |

Assume all four lanes receive the required two consecutive qualifying TS1 Ordered Sets.

The Root Port can identify these four lanes as a group that can form Link N, provided its configuration and lane capabilities permit that Link width.

The identification is based on the received training-sequence information, not merely on:

- The number of physical connections.
- The number of receivers detected during Detect.
- The number of lanes present on the connector.

### 6.2.3 Root Port — Assigning Lane Numbers

Once a configured Link can be formed, the Downstream Port transmits TS1 Ordered Sets with the same Link Number and assigns unique non-PAD Lane Numbers to the selected lanes.

The assigned Lane Numbers must:

1. Range from 0 to n-1.
2. Be assigned sequentially.
3. Be assigned to the same grouping of lanes receiving the same Link Number.

For our illustrative x4 Link:

| Physical Lane | Link Number Transmitted | Lane Number Transmitted |
|---|---|---|
| Lane 0 | N | 0 |
| Lane 1 | N | 1 |
| Lane 2 | N | 2 |
| Lane 3 | N | 3 |

The Lane Numbers are unique and sequential.

The Downstream Lanes that are not receiving TS1 Ordered Sets must not disrupt the initial sequential numbering of the widest possible Link.

Any remaining lanes that are not included in the configured Link must transmit TS1 Ordered Sets with:

`Link Number = PAD`

`Lane Number = PAD`

### 6.2.4 What Happens After Lane Number Assignment?

Once the Downstream Port has identified the configured Link and assigned the Lane Numbers, its next substate is:

`Configuration.Linkwidth.Accept → Configuration.Lanenum.Wait`

This is an important transition.

The Root Port has assigned its Lane Numbers, but the two link partners have not yet completed their Lane Number agreement.

### 6.2.5 Recommended Delay When Lane Errors Occur

The PCIe specification provides a recommendation to avoid prematurely configuring a narrower Link when some lanes may still be usable.

If a possible multi-lane Link has received an error in a TS1 Ordered Set or lost 128b/130b Block Alignment on a subset of lanes, the Port is recommended to delay the evaluation by:

- Two or more additional TS1 Ordered Sets when using 8b/10b encoding.
- 34 or more additional TS1 Ordered Sets when using 128b/130b encoding.

This additional delay must not exceed 1 ms.

The purpose is to avoid prematurely configuring a smaller Link than may be possible.

This is a specification recommendation, not a universal mandatory fixed delay in every situation.

## 6.3 Upstream Lanes — Switch Behaviour

### 6.3.1 What Information Does the Switch Upstream Port Receive?

The Switch Upstream Port receives TS1 Ordered Sets transmitted by the Root Port.

After the Root Port performs its lane-number assignment, these TS1 Ordered Sets carry:

- A non-PAD Link Number.
- A non-PAD Lane Number.

For our example, the Root Port begins transmitting:

| Root Port Physical Lane | Link Number | Lane Number |
|---|---|---|
| Lane 0 | N | 0 |
| Lane 1 | N | 1 |
| Lane 2 | N | 2 |
| Lane 3 | N | 3 |

The switch examines the received training information independently on its Upstream Lanes.

### 6.3.2 How Does the Upstream Port Identify a Valid Link?

A configured Link can be formed using lanes that:

1. Previously transmitted a non-PAD Link Number from the Upstream Port.
2. Receive two consecutive TS1 Ordered Sets.
3. Have the same non-PAD Link Number in the received TS1 Ordered Sets.
4. Receive any non-PAD Lane Number.

For our example, the non-PAD Link Number is N.

The switch must also select a valid grouping of lanes according to its Upstream Port lane-grouping rules.

### 6.3.3 Upstream Port — Transmitting Its Own TS1

After selecting an appropriate lane group, the Upstream Port transmits its own TS1 Ordered Sets.

It uses:

- The same non-PAD Link Number.
- Lane Numbers that, if possible, match the received Lane Numbers.
- Different Lane Numbers if necessary to accommodate Lane Reversal.

For a normal non-reversed lane ordering:

| Switch Physical Lane | Link Number Transmitted | Lane Number Transmitted |
|---|---|---|
| Lane 0 | N | 0 |
| Lane 1 | N | 1 |
| Lane 2 | N | 2 |
| Lane 3 | N | 3 |

This is an illustrative normal-order example.

The Upstream Port does not simply echo the Root Port's TS1 Ordered Sets. It independently generates its own TS1 Ordered Sets using the selected Link and Lane Numbers.

### 6.3.4 Continuous Lane Grouping Requirement

The Upstream Port has a specific lane-grouping requirement.

The newly assigned Lane Numbers must:

1. Range from 0 to m-1.
2. Be assigned sequentially.
3. Apply only to some continuous grouping of lanes that are receiving non-PAD Lane Numbers.
4. Include either Lane 0 or Lane n-1, where n-1 is the largest received Lane Number.
5. Have m-1 equal to or smaller than n-1.

Here:

- n-1 is the largest received Lane Number.
- m-1 is the largest newly assigned Lane Number.

A lane that is not receiving any TS1 Ordered Sets always disrupts a continuous grouping and must not be included in that grouping.

Remaining lanes must transmit TS1 Ordered Sets with:

`Link Number = PAD`

`Lane Number = PAD`

The purpose is to prevent the Upstream Port from forming a Link using an invalid, disconnected lane grouping.

### 6.3.5 Upstream Port — Recommended Delay When Lane Errors Occur

The same evaluation-delay recommendation applies when a possible multi-lane Link receives TS1 errors or loses 128b/130b Block Alignment on a subset of lanes.

Recommended additional waiting:

- Two or more additional TS1 Ordered Sets with 8b/10b encoding.
- 34 or more additional TS1 Ordered Sets with 128b/130b encoding.

The delay must not exceed 1 ms.

This allows a possible wider Link to remain under consideration before a narrower Link is selected.

## 6.4 Lane Reversal

### 6.4.1 What Is Lane Reversal?

Lane Reversal is an optional PCIe capability that accommodates reversed Lane Number ordering within a multi-lane Link.

Consider a four-lane example.

The Downstream Port transmits Lane Numbers in its normal sequence:

`0, 1, 2, 3`

The Upstream Port may receive them in reverse order relative to its preferred ordering:

`3, 2, 1, 0`

This is a Lane Reversal condition.

It is not the same as polarity inversion.

- Polarity inversion swaps P and N conductors within one differential lane.
- Lane Reversal changes the ordering of Lane Numbers across a multi-lane Link.

### 6.4.2 How Does the Upstream Port Handle Lane Reversal?

If the Upstream Port supports optional Lane Reversal, it can transmit the same received Lane Numbers back to the Downstream Port.

If it does not support this behaviour, it transmits the opposite Lane Numbers.

The Downstream Port then has the option to accommodate the reversed Lane ordering, if it supports Lane Reversal, or exit Configuration.

Optional Upstream Port Lane Reversal is important for a Link where the Upstream Port's default sequential Lane Numbering (0 to n-1) receives reversed Downstream Lane Numbering (n-1 to 0), and the Downstream Port itself does not support Lane Reversal.

### 6.4.3 Example of Upstream Lane 0 Not Receiving TS1

The PCIe specification includes an example involving an x8 Upstream Port that does not receive TS1 Ordered Sets on its Upstream Lane 0.

If the Upstream Port supports only x8 or x1 Links and supports Lane Reversal:

- It can assign Lane Number 0 only to the received Lane 7.
- The remaining seven lanes transmit TS1 Ordered Sets with Link and Lane Numbers set to PAD.

If it cannot support Lane Reversal:

- No Link can be formed under that x8-or-x1-only restriction.
- The Upstream Port eventually reaches the 2 ms timeout and returns to Detect.

This illustrates why Lane Reversal support can affect whether a valid Link can be formed.

## 6.5 Downstream and Upstream Lane Grouping — Comparison

| Area | Downstream Port | Upstream Port |
|---|---|---|
| Received TS1 qualification | Same non-PAD Link Number matching one transmitted by the Downstream Lanes | Same non-PAD Link Number and any non-PAD Lane Number |
| Lane assignment | Unique sequential Lane Numbers from 0 to n-1 | Sequential Lane Numbers from 0 to m-1 |
| Lane grouping | Same grouping of lanes receiving the matching Link Number | Continuous grouping of lanes receiving non-PAD Lane Numbers |
| Lane Reversal | Can optionally accommodate reversed Lane Numbers | Can assign matching received numbers where possible, or different numbers where necessary |
| Remaining lanes | PAD/PAD | PAD/PAD |
| Normal next substate | Lanenum.Wait | Lanenum.Wait |

The Downstream and Upstream Port rules are not identical.

## 6.6 Multiple Links — Specification Awareness

A PCIe Port may optionally support configuring multiple Links from its available lanes.

For example:

An x8 Downstream Port can connect to two separate x4 Upstream Ports.

The Downstream Port can transmit two different Link Numbers:

- Link Number N on four lanes.
- Link Number N+1 on the other four lanes.

The two Upstream Ports can select their respective Link Numbers.

The Downstream Port then chooses one set of lanes to configure as its current Link and leaves the remaining set for a secondary LTSSM.

This configuration can be handled by a secondary LTSSM during the later Configuration process.

Another example is an x16 Downstream Port connected to an x12 Upstream Port.

The resulting configuration may be an x8 Link, with the remaining four lanes using PAD/PAD, depending on the available Link configuration.

This shows why a Link cannot be determined merely from the physical number of lanes.

For our current classroom example, we continue to assume one normal x4 Link.

## 6.7 Configuration.Linkwidth.Accept — Timeout and Failure Behaviour

For Downstream and Upstream lane processing, the LTSSM returns to Detect when:

1. The 2 ms timeout expires.
2. No Link can be configured.
3. All lanes receive two consecutive TS1 Ordered Sets with Link Number and Lane Number set to PAD.

**Transition:**

`Configuration.Linkwidth.Accept → Detect`

The next entry into Detect begins with Detect.Quiet.

The timeout alone does not identify the root cause.

Possible investigation areas include:

- Downstream Port lane selection.
- Upstream Port lane selection.
- Link Number recognition.
- Lane Number assignment.
- Continuous lane-grouping requirements.
- Lane Reversal support.
- TS1 Ordered Set errors.
- Block alignment loss.
- Physical lane connectivity.
- Platform and firmware configuration.

## 6.8 Storage Validation Engineer Perspective

During enterprise NVMe SSD validation, the expected PCIe Link width is an important part of device qualification.

For example, an NVMe SSD may be expected to operate through a PCIe switch using an x4 Link.

Detect and Polling may complete successfully, but Configuration might establish a narrower Link.

An x4 SSD operating at x2 may still enumerate and function, but its achieved bandwidth capability is lower than expected.

Therefore, successful device enumeration alone does not prove that the expected PCIe Link width has been established.

### 6.8.1 Expected Behaviour

The expected Root Port-to-switch Link should establish the intended Link width.

For the simplified example:

`Expected Link width = x4`

### 6.8.2 Observed Behaviour

Possible symptom:

- NVMe SSD enumerates successfully.
- PCIe Link is operational.
- Negotiated Link width is x2 instead of x4.

### 6.8.3 Validation Investigation

1. Confirm the negotiated Link width.
2. Inspect Root Port lane configuration.
3. Inspect switch Upstream Port lane configuration.
4. Identify which lanes participated in the configured Link.
5. Examine TS1 Ordered Set Link and Lane Number fields, if protocol traces are available.
6. Check the continuous lane-grouping conditions.
7. Investigate Lane Reversal and lane wiring.
8. Check lane connectivity, connector, riser and backplane routing.
9. Investigate signal-integrity and receiver errors.
10. Determine whether the width reduction is due to device capability, platform configuration, physical connectivity or a hardware issue.
11. Reproduce the failure and collect evidence.
12. Verify the fix and run regression testing.

Do not automatically declare the SSD defective because the expected Link width was not achieved.

The validation engineer must isolate the responsible layer before assigning the root cause.

## 6.9 Linkwidth.Accept — Transition Summary

| Port Role / Condition | Behaviour | Next State |
|---|---|---|
| Downstream Port forms a valid Link from qualifying received TS1s | Transmits the selected Link Number and assigns unique sequential Lane Numbers | Configuration.Lanenum.Wait |
| Upstream Port forms a valid Link using qualifying lanes with non-PAD Link and Lane Numbers | Transmits the same Link Number and matching or appropriately reversed Lane Numbers | Configuration.Lanenum.Wait |
| No valid Link can be formed | Configuration attempt fails | Detect |
| All lanes receive two consecutive PAD/PAD TS1s | Configuration attempt fails | Detect |
| 2 ms timeout expires | Configuration attempt times out | Detect |



