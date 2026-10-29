# Project 2 — Module 09: PCIe Architecture

# LTSSM, Detect and Polling

## Reference Baseline

These notes use the PCI Express Base Specification Revision 5.0,
Version 1.0, as the baseline for LTSSM states, transition rules,
training counts, and timeout values.

The emphasis is PCIe behavior relevant to enterprise NVMe SSD
drive validation.

PCIe Gen4 and Gen5 are important for our practical validation work.
Gen6 concepts will be covered separately where the behavior differs.

---

## 1. What Is LTSSM?

LTSSM stands for Link Training and Status State Machine.

It is hardware control logic associated with PCIe link operation.
It manages the initialization, training, recovery, and
power-related state transitions of a PCIe link.

A state describes the current operating condition of the link.

A state transition is the change from one state to another when
the specified transition conditions are satisfied.

For the initial PCIe link-initialization path, the LTSSM begins
in `Detect.Quiet`, a substate of the major `Detect` state.

The `Detect` state has two substates:

- `Detect.Quiet`
- `Detect.Active`

The `Polling` state has the following substates in the PCIe 5.0
specification:

- `Polling.Active`
- `Polling.Compliance`
- `Polling.Configuration`
- `Polling.Speed`

`Polling.Speed` is defined in the specification but is unreachable
in the normal PCIe 5.0 link-initialization flow. The link initially
reaches L0 at 2.5 GT/s and changes to a higher data rate through
Recovery.

The main initial link-training path is:

Detect
  ↓
Polling.Active
  ↓
Polling.Configuration
  ↓
Configuration.Linkwidth.Start
  ↓
Further Configuration substates
  ↓
L0

`Polling.Compliance` is an alternative testing path. It is not
a required step during ordinary NVMe SSD startup.

---

## 2. Detect.Quiet

### 2.1 Purpose

`Detect.Quiet` is the initial electrical-idle state in the
normal link-initialization path.

The transmitter is in Electrical Idle.

During Electrical Idle, the differential signaling is inactive.
The differential voltage is approximately zero because the two
conductors of the differential pair are at approximately the same
electrical potential.

This does not mean that the NVMe SSD or host is powered off.

The DC common-mode voltage is not required to be within its
normal operating specification while in this state.

### 2.2 Conditions for Leaving Detect.Quiet

The LTSSM remains in `Detect.Quiet` until one of these specified
conditions occurs:

1. The 12 ms timeout expires.
2. Electrical Idle is broken on any lane.

Electrical Idle is considered broken when the applicable electrical
criteria indicate that the incoming signal has left the idle condition.

An arbitrary voltage change does not automatically mean that the
electrical-idle exit criteria have been satisfied.

Transition:

`Detect.Quiet → Detect.Active`

### 2.3 Important Timeout Distinction

The 12 ms timeout in the `Detect.Quiet` to `Detect.Active`
transition is separate from the 12 ms wait used later during
partial receiver detection.

They are two different timing conditions.

### 2.4 Data Rate

For the PCIe 5.0 baseline, the initial link-training data rate
is 2.5 GT/s.

If the transmitter was operating at a different rate when entering
`Detect.Quiet`, it must change to 2.5 GT/s during the specified
Electrical Idle period.

The state also establishes the receiver electrical conditions
required for the subsequent receiver-detection procedure.

---

## 3. Detect.Active

### 3.1 Purpose

In `Detect.Active`, the transmitter performs an electrical
Receiver Detection procedure on all relevant unconfigured lanes
that can form one or more links.

The detection circuitry checks the electrical response to determine
whether the expected receiver termination is present at the far end
of a lane.

This is an electrical detection procedure.

It does not check whether the remote component can execute an NVMe
Read or Write command.

The result determines the next LTSSM behavior.

### 3.2 Case 1 — Receivers Detected on All Relevant Lanes

If a receiver is detected on all relevant unconfigured lanes,
the LTSSM transitions to Polling.

Transition:

`Detect.Active → Polling.Active`

Receiver detection has succeeded, but link training is not complete.

The link has not yet reached its normal operational state, L0.

### 3.3 Case 2 — No Receiver Detected on Any Lane

If a receiver is not detected on any relevant lane, the LTSSM
returns to `Detect.Quiet`.

Transition:

`Detect.Active → Detect.Quiet`

The receiver-detection procedure can be attempted again according
to the Detect-state rules.

This result does not, by itself, prove that the SSD is defective
or physically absent.

### 3.4 Case 3 — Partial Receiver Detection

Partial receiver detection occurs when a receiver is detected
on some, but not all, relevant unconfigured lanes.

Consider an intended PCIe x4 connection.

The first receiver-detection attempt produces this result:

| Lane | First Attempt |
|---|---|
| Lane 0 | Receiver detected |
| Lane 1 | Receiver detected |
| Lane 2 | Receiver not detected |
| Lane 3 | Receiver not detected |

This is partial detection.

The PCIe specification defines a confirmation procedure.

1. The hardware waits 12 ms.
2. The transmitter repeats Receiver Detection on all relevant
   unconfigured lanes, not just the lanes that failed the first attempt.
3. The LTSSM compares the complete set of lanes that detected
   receivers during the first and second attempts.

The comparison concerns the identities of the detected lanes,
not merely the number of successful detections.

### 3.5 Case 3A — The Same Set of Lanes Detects Receivers

Suppose the result is:

| Lane | First Attempt | Second Attempt |
|---|---|---|
| Lane 0 | Detected | Detected |
| Lane 1 | Detected | Detected |
| Lane 2 | Not detected | Not detected |
| Lane 3 | Not detected | Not detected |

The detected set is `{Lane 0, Lane 1}` during both attempts.

Because exactly the same lanes detect receivers during both
attempts, the LTSSM can transition to Polling on the detected lanes.

Transition:

`Detect.Active → Polling.Active`

Lanes that did not detect receivers are handled according to
the specification's optional multiple-link behavior.

If the optional behavior is supported, those lanes may be
associated with a separate LTSSM.

Otherwise, the lanes that cannot be associated with another LTSSM
transition to Electrical Idle. They must be re-associated when
the LTSSM in progress returns to Detect.

**Important:** Detecting receivers on two lanes does not automatically
guarantee that the final configured link will be x2.

The final link configuration is determined by subsequent training
and Configuration.

### 3.6 Case 3B — The Detected Lane Set Changes

Suppose the result is:

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

Transition:

`Detect.Active → Detect.Quiet`

The point of the retry is to verify that the partial detection
result is repeatable.

The hardware must compare the same detected lanes, rather than
merely observing that the same number of lanes succeeded.

### 3.7 Detect Transition Summary

| Receiver-Detection Result | Next State |
|---|---|
| Receivers detected on all relevant unconfigured lanes | Polling.Active |
| No receiver detected on any relevant lane | Detect.Quiet |
| Partial detection, with the same detected lane set on both attempts | Polling.Active on detected lanes |
| Partial detection, with different detected lane sets | Detect.Quiet |

The partial-detection confirmation procedure uses a 12 ms wait
before the second receiver-detection attempt.

### 3.8 Enterprise Troubleshooting Perspective

Receiver detection provides evidence about the electrical
link-initialization process.

It does not automatically establish the root cause of a failure.

For an enterprise NVMe SSD that fails to establish the expected link,
possible investigation areas include:

- Host Root Port behavior
- SSD Endpoint behavior
- PCIe lane connectivity
- Connector or slot problems
- Riser or backplane problems
- Platform configuration
- PCIe hardware or signal-integrity problems

Ordinary Linux tools can report device enumeration, negotiated link
speed and width, kernel events, and some PCIe errors.

However, they do not generally expose every internal per-lane
receiver-detection result.

A supported hardware-debug interface or protocol analyzer may
be needed to obtain additional evidence.

The validation engineer should establish what happened, collect
evidence, isolate the responsible layer, and only then attribute
the defect to the SSD or platform.

---

## 4. Introduction to Polling

### 4.1 Why Is Polling Necessary?

Detect checks whether the expected receiver termination is present
at the opposite end of the PCIe connection.

Successful receiver detection does not prove that the receiver
can synchronize with and correctly interpret the incoming serial data.

Polling establishes the initial synchronization and training
conditions required for PCIe link initialization.

During Polling, the transmitter sends predefined training patterns
called TS1 Ordered Sets.

TS1 means Training Sequence 1.

An Ordered Set is a specially structured sequence of symbols
defined by the PCIe specification.

TS1 Ordered Sets are not NVMe Read or Write commands and are
not ordinary application data. They are used for PCIe link training.

### 4.2 Bidirectional Training

Both the Root Port and the SSD Endpoint have their own
transmitter and receiver.

The Root Port's transmitter sends TS1 Ordered Sets that the SSD
Endpoint's receiver attempts to recognize.

The SSD Endpoint's transmitter independently sends training
sequences that the Root Port's receiver attempts to recognize.

The SSD does not simply echo the Root Port's training sequence.

Both link partners transmit their own training sequences according
to the PCIe LTSSM rules.

### 4.3 Polling Does Not Mean the Link Is Operational

Polling is an initial link-training stage.

Successful transition out of Polling does not mean that the link
has reached L0.

The main Configuration state must complete its procedures
before the link enters L0, the normal operational state.

### 4.4 Initial Data Rate and PCIe Generations

Under the PCIe 5.0 baseline, the link initially trains at 2.5 GT/s.

A Gen4 or Gen5 SSD does not normally begin its initial LTSSM
training directly at 16 GT/s or 32 GT/s.

The link initially reaches L0 at 2.5 GT/s.

Higher common supported data rates are advertised in the training
ordered sets and are negotiated through the applicable Recovery
and speed-change procedures after L0 is reached.

Reference rates and encoding:

| PCIe Generation | Nominal Data Rate per Lane | Encoding or Signaling |
|---|---:|---|
| Gen1 | 2.5 GT/s | 8b/10b |
| Gen2 | 5 GT/s | 8b/10b |
| Gen3 | 8 GT/s | 128b/130b |
| Gen4 | 16 GT/s | 128b/130b |
| Gen5 | 32 GT/s | 128b/130b |
| Gen6 | 64 GT/s | PAM4, FLIT mode and FEC |

The Gen6 row is included for awareness. The detailed LTSSM
transition rules in these notes use PCIe 5.0 as their reference.

The initial 2.5 GT/s training process uses the 8b/10b training
representation, including when the link's components support
higher generations.

---

## 5. Polling.Active

### 5.1 Purpose

`Polling.Active` begins the initial exchange of training ordered
sets between the Root Port and the Endpoint.

The transmitter sends TS1 Ordered Sets on the lanes that detected
a receiver during Detect.

A TS1 Ordered Set is a predefined structure containing symbols
and control fields that the receiving PCIe component knows
how to interpret.

TS1 Ordered Sets are used to establish the initial link-training
conditions required for PCIe initialization.

### 5.2 Link Number and Lane Number Fields

During `Polling.Active`, the transmitter sends TS1 Ordered Sets
with the Link Number and Lane Number fields set to PAD on all lanes
that detected a receiver during Detect.

PAD is a placeholder used during the early training process,
before the link and lane numbers have been established.

The final association of lanes into a link is determined during
the subsequent Configuration process.

### 5.3 Data Rate Identifier

The Data Rate Identifier symbol in TS1 advertises all data rates
supported by the Port, including rates the Port does not intend
to use.

Advertising a rate does not mean that the link is currently
operating at that data rate.

The current initial training rate remains 2.5 GT/s under
the PCIe 5.0 baseline.

### 5.4 Bit Lock

#### What Problem Does Bit Lock Solve?

The transmitter sends a serial stream of bits over a PCIe lane.

The receiver receives an electrical waveform rather than a row
of already-labelled binary digits.

The receiver must recover the timing needed to sample the incoming
signal correctly.

This function uses Clock and Data Recovery (CDR).

CDR recovers timing information from the incoming serial signal
so that the receiver can sample the transmitted bits at the
appropriate times.

Bit lock means the receiver has recovered the timing required
to sample the incoming bit stream correctly.

Without suitable bit timing, the receiver cannot reliably recover
the transmitted bits.

Bit lock does not mean that all link-training operations are complete.

#### Validation Relevance

If the receiver cannot establish the required bit timing,
it may not correctly recover the incoming training sequence.

As a result, the LTSSM may fail to progress through link training.

A validation engineer should investigate the available PCIe
link-training evidence instead of immediately concluding that
the NVMe SSD is defective.

### 5.5 Symbol Lock

Bit lock and symbol lock solve different problems.

Bit lock establishes the timing needed to recover individual bits.

Symbol lock identifies the boundaries of encoded symbols in the
recovered bit stream.

For the initial 2.5 GT/s PCIe training process, 8b/10b encoding
is used.

In 8b/10b encoding, an 8-bit data or control value is represented
by a 10-bit transmitted code group.

The receiver must identify where encoded symbols begin and end.

#### The COM Symbol

The COM symbol is the 8b/10b control character K28.5.

It provides a recognizable alignment pattern that the receiver
can use to establish the correct symbol boundary.

Remember:

- **Bit lock:** Recover the timing of incoming bits.
- **Symbol lock:** Identify the boundaries of encoded symbols.

For PCIe Gen3 through Gen5, 128b/130b encoding is used during
higher data-rate operation. However, understanding symbol
alignment during initial Polling remains important because the
initial training process starts at 2.5 GT/s.

Gen6 high-speed alignment and FLIT-related behavior require
separate treatment.

### 5.6 Lane Polarity Inversion

#### What Is Polarity Inversion?

PCIe uses differential signaling.

Each PCIe lane has a transmit differential pair and a receive
differential pair.

Each differential pair has positive (P) and negative (N)
signal conductors.

If the P and N conductors of a differential pair are swapped,
the polarity of the received differential signal is reversed.

This condition is called **lane polarity inversion**.

Polarity inversion is different from lane reversal.

- Polarity inversion swaps P and N within one differential lane.
- Lane reversal changes the ordering of lanes within a multi-lane link.

#### How Does the Receiver Detect Polarity Inversion?

During Polling, the receiver examines specific symbols in the
received TS1 and TS2 Ordered Sets.

For the initial 8b/10b training representation, the PCIe
specification defines recognizable normal-polarity and
inverted-polarity symbol patterns.

For TS1:

- Normal polarity: Symbols 6–15 contain the expected D10.2 pattern.
- Inverted polarity: Symbols 6–15 are received as D21.5.

For TS2:

- Normal polarity: Symbols 6–15 contain the expected D5.2 pattern.
- Inverted polarity: Symbols 6–15 are received as D26.5.

These patterns allow the receiver to distinguish normal polarity
from inverted polarity.

The receiver does not simply guess that the wires are swapped.

#### What Happens if Polarity Is Inverted?

If the receiver detects polarity inversion, it must invert the
received data to compensate for the inversion.

The transmitter must not invert its transmitted data to perform
this correction.

Support for lane polarity inversion is required independently
for each PCIe receiver lane.

#### What Happens if Polarity Is Normal?

If no polarity inversion is present, the receiver does not need
to apply polarity correction to that lane.

If the expected training patterns cannot be recognized, the
receiver cannot automatically conclude that polarity inversion
is the cause.

Other training or signal-integrity problems may need investigation.

#### When Must Polarity Correction Be Completed?

The PCIe 5.0 specification requires polarity correction, if
necessary, before the receiver exits `Polling.Configuration`.

Detection and correction may begin earlier, including in
`Polling.Active`.

Therefore, do not assume that all polarity correction must
have been completed during Polling.Active.

#### Validation Relevance

Polarity inversion is not automatically a hardware failure.

A supported inversion can be compensated for by the receiver.

The validation engineer must establish whether the link trains,
reaches the expected link speed and width, and operates reliably.

### 5.7 Normal Transition: Polling.Active to Polling.Configuration

The normal transition requires two main conditions.

#### Condition 1 — Transmitter Requirement

At least 1,024 TS1 Ordered Sets must have been transmitted.

#### Condition 2 — Receiver Requirement

Every lane that detected a receiver during Detect must receive
eight consecutive qualifying training sequences, or their
complement, according to the specified criteria.

The qualifying training sequences include:

1. TS1 with Link Number and Lane Number set to PAD and the
   Compliance Receive bit deasserted.
2. TS1 with Link Number and Lane Number set to PAD and the
   Loopback bit asserted.
3. TS2 with Link Number and Lane Number set to PAD.

PCI-SIG has clarified that, for this Polling.Active transition,
the eight qualifying training sequences may be a mixture of
qualifying TS1 and TS2 Ordered Sets.

They do not necessarily have to be eight TS1 sequences or
eight TS2 sequences exclusively.

When the required conditions are satisfied:

`Polling.Active → Polling.Configuration`

This transition can occur before the 24 ms timeout.

### 5.8 Polling.Active — 24 ms Timeout

If the normal transition conditions are not satisfied before
the 24 ms timeout expires, the LTSSM evaluates the specified
fallback conditions.

The timeout is a decision point, not an automatic declaration
that the SSD is defective.

There are three main timeout outcomes.

#### Outcome A — Continue to Polling.Configuration

The LTSSM can transition to `Polling.Configuration` if all
the following conditions are satisfied:

1. At least one lane that detected a receiver during Detect
   has received eight consecutive qualifying training sequences,
   or their complement.

2. At least 1,024 TS1 Ordered Sets have been transmitted after
   the transmitter received one TS1 or TS2 Ordered Set.

3. The implementation-defined predetermined set of lanes that
   detected receivers during Detect has detected an exit from
   Electrical Idle at least once on every lane in that set since
   entering Polling.Active.

The predetermined lane set is implementation-specific.

This fallback can allow further training to continue when
one or more lanes do not meet the normal eight-sequence
requirement.

It does not guarantee that all originally detected lanes will
form the final configured link.

#### Outcome B — Enter Polling.Compliance

If Outcome A does not apply, the LTSSM enters Polling.Compliance
if either of the following compliance conditions is satisfied.

**Condition B1 — Electrical Idle exit condition**

Not all lanes in the implementation-defined predetermined set
have detected an exit from Electrical Idle since entering
Polling.Active.

**Condition B2 — Compliance request through TS1**

A lane that detected a receiver during Detect receives eight
consecutive TS1 Ordered Sets, or their complement, with:

- Link Number = PAD
- Lane Number = PAD
- Compliance Receive bit = 1
- Loopback bit = 0

These conditions select the compliance-testing path.

#### Outcome C — Return to Detect.Quiet

If neither Outcome A nor Outcome B applies, the LTSSM returns
to Detect.Quiet.

The link begins the detection process again.

#### Important Distinction

The normal transition and timeout fallback use different conditions.

**Normal transition:**

- At least 1,024 TS1 Ordered Sets transmitted.
- Eight qualifying training sequences received on every lane
  that detected a receiver during Detect.

**Timeout fallback:**

- At least one qualifying lane.
- The required transmission count after receiving TS1 or TS2.
- The implementation-defined predetermined set of lanes has
  detected the specified exits from Electrical Idle.

Do not confuse these two sets of conditions.

### 5.9 Direct Entry to Polling.Compliance

A separate route to Polling.Compliance exists through the
Enter Compliance bit in the Link Control 2 register.

If the Enter Compliance bit is set before entry into Polling.Active,
the transition to Polling.Compliance must be immediate without
sending any TS1 Ordered Sets.

If the bit is set while Polling.Active is in progress, the
LTSSM also follows the specified compliance-entry rule.

This is a compliance-test control path, not the normal NVMe
link-initialization path.

---

## 6. Polling.Compliance

### 6.1 Purpose

Polling.Compliance is intended for PCIe electrical compliance testing.

It allows the transmitter and interconnect to be evaluated
using PCIe-defined compliance or test patterns.

The patterns and transmission settings depend on the applicable
test configuration, component capabilities and the mechanism
used to enter this state.

The test setup may use equipment such as:

- Oscilloscope
- Pattern generator
- Bit Error Rate Tester (BERT)
- Other PCIe compliance test equipment

Polling.Compliance is not a required stage of ordinary NVMe
SSD startup.

### 6.2 Entry Conditions

Polling.Compliance can be entered through specified conditions,
including:

1. The Enter Compliance bit in Link Control 2 is set before
   entry into Polling.Active.
2. The 24 ms Polling.Active timeout fallback selects
   Polling.Compliance.
3. The received TS1 Ordered Sets satisfy the specified compliance
   request pattern.
4. A compliance-test environment produces one of the specified
   compliance-entry conditions.

The precise behavior depends on the entry mechanism.

### 6.3 Behavior Inside Polling.Compliance

On entry, the transmitter samples the Transmit Margin field
from Link Control 2 as required by the specification.

The compliance transmission rate and applicable de-emphasis
or preset settings are selected according to the entry mechanism,
supported data rates, and required compliance-pattern sequence.

Depending on the mode, the transmitter sends a standard compliance
pattern or a specified modified compliance pattern.

For high-speed compliance testing, the applicable test procedure
may include preset selection, modified compliance patterns,
pattern-lock reporting and error-count reporting.

These are electrical compliance-testing functions, not normal
NVMe Read/Write operations.

### 6.4 Exiting Polling.Compliance

The exit conditions depend on why the port entered
Polling.Compliance.

#### Pattern-driven compliance path

For the compliance-pattern cycling path, the LTSSM can return
to Polling.Active when the receiver detects an exit from Electrical
Idle on any lane that detected a receiver during Detect.

If the transmitter was operating above 2.5 GT/s, the required
return-to-training-rate sequence must be performed.

The procedure includes transmitting the specified consecutive
Electrical Idle Ordered Sets, entering Electrical Idle, changing
the operating data rate to 2.5 GT/s, and allowing the rate
to stabilize.

For the PCIe 5.0 baseline, the specified Electrical Idle period
is greater than 1 ms and must not exceed 2 ms.

#### Register-directed compliance path

When compliance testing was initiated through the Enter Compliance
register mechanism, exit conditions depend on the register state
and port role.

Examples include:

- The Enter Compliance bit changes from 1 to 0.
- An Upstream Port receives an EIOS on a lane, under the
  applicable register-directed exit rule.

The applicable transition can require the transmitter to send
eight consecutive EIOSs, enter Electrical Idle, change back
to 2.5 GT/s, and allow the new operating rate to stabilize.

The complete exit sequence must be interpreted according to
the specific entry path defined by the PCIe specification.

A higher-layer instruction can also direct a specified transition
to Detect.

#### Validation Relevance

If an NVMe SSD enters Polling.Compliance unexpectedly, first
determine whether the platform intentionally requested
compliance testing.

Do not immediately treat the state as proof that the SSD has failed.

If compliance testing was not intended, investigate:

- Link Control 2 compliance settings
- Training-sequence conditions
- Electrical Idle behavior
- Host Root Port behavior
- SSD Endpoint behavior
- Available hardware-debug or protocol-analyzer evidence

---

## 7. Polling.Configuration

### 7.1 Purpose

Polling.Configuration follows Polling.Active when the applicable
normal or timeout-fallback transition requirements are satisfied.

Its purpose is to continue initial link training using TS2
Ordered Sets and establish the conditions required to transition
to the main Configuration state.

TS2 means Training Sequence 2.

### 7.2 TS2 Ordered Sets

During Polling.Configuration, the transmitter sends TS2 Ordered
Sets on all lanes that detected a receiver during Detect.

The Link Number and Lane Number fields remain set to PAD.

The Data Rate Identifier symbol advertises all data rates
supported by the Port, including rates the Port does not
intend to use.

The current initial training rate remains 2.5 GT/s under
the PCIe 5.0 baseline.

### 7.3 Polarity and Other Entry Actions

At entry to Polling.Configuration:

- The receiver must invert polarity if necessary.
- The Transmit Margin field of the Link Control 2 register
  is reset to 000b.
- The transmitter's Polling.Compliance sequence setting is
  updated if required by the specification.

Polarity correction must be complete before the receiver exits
Polling.Configuration.

Polarity inversion may have been detected and corrected earlier
during Polling.Active.

### 7.4 Normal Transition to Configuration

The normal transition from Polling.Configuration to the main
Configuration state requires two conditions.

#### Condition 1 — Receiver Requirement

Eight consecutive TS2 Ordered Sets, with Link Number and
Lane Number set to PAD, must be received on at least one lane
that detected a receiver during Detect.

The wording of this condition is different from the normal
Polling.Active transition.

Polling.Active requires qualifying training sequences on every
lane that detected a receiver during Detect.

Polling.Configuration requires the qualifying eight-sequence
TS2 condition on any lane or lanes that detected a receiver
during Detect; it does not impose the same all-lanes requirement.

#### Condition 2 — Transmitter Requirement

The transmitter must send at least 16 TS2 Ordered Sets after
receiving one TS2 Ordered Set.

These are separate requirements.

The transmitter is not required to send 16 TS2 Ordered Sets
for every received TS2 Ordered Set.

When both conditions are satisfied:

`Polling.Configuration → Configuration.Linkwidth.Start`

This is the first substate of the main Configuration state.

### 7.5 Polling.Configuration — 48 ms Timeout

The 48 ms timer applies to Polling.Configuration.

It is separate from the 24 ms timer used in Polling.Active.

The normal transition can occur as soon as the required TS2
conditions are satisfied. It does not have to wait for the
complete 48 ms timeout.

If the required transition conditions are not satisfied within
48 ms, the LTSSM returns to Detect.

Transition:

`Polling.Configuration → Detect`

The next entry into Detect starts with Detect.Quiet.

Unlike the 24 ms timeout in Polling.Active, the Polling.Configuration
48 ms timeout does not use the same three-way fallback decision.

Its specified timeout outcome is a return to Detect.

### 7.6 Validation Relevance

If an NVMe SSD repeatedly fails to leave Polling.Configuration,
the evidence indicates that the TS2 training stage did not
complete within the specified period.

The timeout alone does not identify the root cause.

The validation engineer should investigate the relevant receiver
and transmitter behavior, lane connectivity, electrical conditions,
platform configuration, and available diagnostic evidence.

---

## 8. Polling.Speed

The PCIe 5.0 Base Specification includes a Polling.Speed state.

However, it is unreachable in the normal PCIe 5.0 link-initialization
flow.

The link initially trains to L0 at 2.5 GT/s and changes to a
higher supported data rate by entering Recovery.

Therefore, Polling.Speed is not a required intermediate state
in the normal PCIe 5.0 training path described in these notes.

---

## 9. Complete Polling Transition Summary

The following table summarizes the relevant PCIe 5.0 transition
conditions.

| Current State or Substate | Condition | Next State |
|---|---|---|
| Detect.Active | Receivers detected on all relevant unconfigured lanes | Polling.Active |
| Detect.Active | No receiver detected on any lane | Detect.Quiet |
| Detect.Active | Partial detection; same detected lane set on both attempts | Polling.Active on detected lanes |
| Detect.Active | Partial detection; detected lane set differs on the two attempts | Detect.Quiet |
| Polling.Active | Normal transition conditions satisfied | Polling.Configuration |
| Polling.Active | 24 ms expires and timeout fallback conditions permit further training | Polling.Configuration |
| Polling.Active | 24 ms expires and a specified compliance condition applies | Polling.Compliance |
| Polling.Active | 24 ms expires and neither fallback applies | Detect.Quiet |
| Polling.Active | Enter Compliance is set before entry into the substate | Polling.Compliance immediately, without sending TS1 |
| Polling.Compliance | Applicable exit condition is satisfied | Polling.Active or another specified outcome, depending on entry path |
| Polling.Configuration | Normal TS2 transition conditions satisfied | Configuration.Linkwidth.Start |
| Polling.Configuration | 48 ms expires without satisfying the normal transition conditions | Detect |

The partial-detection rules in this table belong to Detect. They are
included to show the complete path from Detect through Polling.

---

## 10. Important Differences to Remember

### 10.1 Detect versus Polling

Detect checks for receiver termination.

Polling exchanges and recognizes training Ordered Sets to establish
the synchronization and training conditions required for further
link initialization.

### 10.2 Polling.Active versus Polling.Configuration

Polling.Active:

- Uses TS1 Ordered Sets.
- Establishes initial training conditions.
- Has a 24 ms timeout and specified fallback paths.
- Its normal transition requires eight qualifying training sequences
  on every lane that detected a receiver during Detect.
- Requires at least 1,024 TS1 Ordered Sets to be transmitted
  for the normal transition.

Polling.Configuration:

- Uses TS2 Ordered Sets.
- Continues initial training.
- Has a 48 ms timeout.
- Its normal transition requires eight consecutive qualifying TS2
  Ordered Sets on at least one lane that detected a receiver
  during Detect.
- Requires at least 16 TS2 Ordered Sets to be transmitted after
  receiving one TS2 Ordered Set.

### 10.3 The 24 ms and 48 ms Timers

The two timeout values belong to different LTSSM substates.

- 24 ms applies to Polling.Active.
- 48 ms applies to Polling.Configuration.

The 24 ms timeout invokes specified fallback conditions.

The 48 ms timeout returns the LTSSM to Detect if the normal
Polling.Configuration exit conditions are not satisfied.

Neither timeout automatically proves that the SSD is defective.

### 10.4 Polarity Inversion versus Lane Reversal

Polarity inversion swaps the P and N conductors within a
differential lane.

Lane reversal changes the ordering of lanes within a multi-lane link.

They are different conditions and must not be confused.

### 10.5 Polling versus L0

Polling is part of initial link training.

L0 is the normal operational state where PCIe data and control
packets can be transmitted and received.

Leaving Polling does not mean that the link has already reached L0.

The main Configuration state must still complete its procedures.

### 10.6 Advertised Data Rates versus Current Data Rate

TS1 and TS2 Ordered Sets advertise supported data rates.

Advertising a rate does not mean that the link is operating at
that rate.

Under the PCIe 5.0 baseline, the link initially trains at 2.5 GT/s
and changes to a higher supported rate through Recovery.

---

## 11. Storage Validation Engineer Perspective

During enterprise NVMe SSD validation, the objective is to
establish whether the PCIe link completes training and reaches
the expected operating state.

If training fails, the engineer should identify the first layer
for which the evidence demonstrates a problem.

A Polling timeout does not automatically prove that the SSD
is defective.

Possible investigation areas include:

- PCIe Root Port behavior
- SSD Endpoint behavior
- PCIe lane connectivity and signal integrity
- Connector, riser or backplane issues
- Platform or firmware configuration
- Compliance-test settings
- Receiver synchronization
- Training-sequence recognition
- Link speed and width behavior

Ordinary Linux tools can provide evidence of device enumeration,
negotiated link speed and width, kernel events and some PCIe errors.

However, ordinary Linux tools do not necessarily expose every
internal LTSSM transition or per-lane training result.

A validation engineer must combine available operating-system
evidence with hardware-debug or protocol-analyzer evidence
where available.

The engineering workflow is:

Observe the symptom
  ↓
Identify the first layer showing failure
  ↓
Collect available evidence
  ↓
Develop a testable hypothesis
  ↓
Reproduce the failure
  ↓
Isolate the responsible layer
  ↓
Determine root cause
  ↓
Verify the fix
  ↓
Run regression

---

## 12. End of Polling — Next LTSSM Stage

Polling is complete when the applicable training conditions
are satisfied and the LTSSM transitions from Polling.Configuration
to Configuration.Linkwidth.Start.

The next topic is:

### Configuration.Linkwidth.Start

This is the first substate of the main Configuration state.

It begins the next stage of PCIe link initialization, including
the process of negotiating link width and beginning link
and lane identification.

We will study how the Downstream Port starts the negotiation,
how Link Number and Lane Number fields are used, and how
the available lanes are organized into a PCIe link.

We must not assume that the final link width has already been
established merely because Polling has completed.

Links are formed at the conclusion of the main Configuration
process.

---

## 13. Specification References

Use the PCI Express Base Specification Revision 5.0,
Version 1.0, as the reference baseline for detailed transition
conditions and timeout values.

1. PCI Express Base Specification Revision 5.0, Version 1.0.
   Sections 4.2.5 and 4.2.6.1–4.2.6.3:
   https://studylib.net/doc/28294356/pci-express-base-spec-rev-5-version-1-0

2. PCI-SIG clarification regarding consecutive TS1/TS2
   sequences in Polling.Active:
   https://pcisig.com/section-42621-reference-pollingactive-state-described-section-42621-next-state-pollingconfiguration

3. PCI-SIG specification listing:
   https://pcisig.com/specifications

4. Intel PCIe receiver polarity-inversion explanation:
   https://edc.intel.com/content/www/us/en/design/products/platforms/details/13th-generation-core-processors-datasheet-volume-1-of-2/008/pci-express-support/
