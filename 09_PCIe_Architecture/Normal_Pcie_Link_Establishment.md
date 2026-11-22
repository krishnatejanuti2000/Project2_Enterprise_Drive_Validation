# PCIe LTSSM — Link Training and Status State Machine

## What Is LTSSM?

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

---

# Detect State

The purpose of the Detect state is to detect the presence of the expected far-end receiver termination.

During normal PCIe Link initialization, the LTSSM initially enters the major Detect state, which contains two substates:

- Detect.Quiet
- Detect.Active

## Detect.Quiet

Detect.Quiet is the first substate of the Detect state.

When the LTSSM enters this substate, the transmitter is placed in Electrical Idle. The transmitter does not transmit normal differential data signalling.

PCIe uses differential signalling, with two conductors:

- P — Positive.
- N — Negative.

During Electrical Idle, the differential voltage between P and N is approximately zero because both conductors are at approximately the same electrical potential.

The PCIe hardware prepares the electrical conditions required for Receiver Detection. The operating frequency is prepared for 2.5 GT/s. The applicable receiver electrical requirements must be met within 1 ms of entering this substate.

On entry:

`LinkUp = 0b`

This means the PCIe Link is not yet indicated as operational.

The LTSSM remains in Detect.Quiet until either of these conditions occurs:

- The 12 ms timeout expires.
- Electrical Idle is broken on any Lane according to the applicable electrical criteria.

When either condition is satisfied:

`Detect.Quiet → Detect.Active`

## Detect.Active

In Detect.Active, the transmitter's Receiver Detection circuitry performs an electrical Receiver Detection procedure on all relevant unconfigured Lanes that can form one or more Links.

The purpose is to determine whether the expected receiver termination is present at the far end of each relevant Lane.

There are three possible outcomes.

### Case — Receiver detected on all relevant Lanes

The LTSSM transitions:

`Detect.Active → Polling.Active`

### Case — No receiver detected on any relevant Lane

The LTSSM transitions:

`Detect.Active → Detect.Quiet`

### Case — Partial Receiver Detection

Partial detection occurs when at least one, but not all, relevant Lanes detect a receiver.

The confirmation procedure is:

1. Wait for 12 ms.
2. Repeat Receiver Detection on all relevant unconfigured Lanes.
3. Compare the detected Lane sets from both attempts.

There are two possible outcomes.

**Same Lane set detected during both attempts**

The LTSSM proceeds to Polling.Active on the consistently detected Lanes.

**Different Lane sets detected during both attempts**

The LTSSM returns to Detect.Quiet.

The Lanes that fail to detect a receiver must either be associated with a new LTSSM (if the optional feature is supported) or transition to Electrical Idle.

The comparison is based on the identities of the detected Lanes, not merely the number of successful detections.

Successful Receiver Detection does not guarantee that the final PCIe Link width will equal the number of detected Lanes.

Receiver Detection establishes electrical receiver presence. The Link must still undergo Polling and Configuration before reaching L0.

## Detect State — Complete Transition Summary

```text
                     Detect.Quiet
                         |
            12 ms timeout or Electrical Idle exit
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

After successful Receiver Detection, the LTSSM transitions from Detect.Active to Polling.Active.

---

# Polling State

The purpose of Polling is to establish the initial synchronization and training conditions required for PCIe Link initialization.

Its purpose is to:

- Establish initial receiver synchronization.
- Exchange TS1 Training Ordered Sets.
- Establish Bit Lock.
- Establish Symbol Lock.
- Identify and correct Lane Polarity Inversion, where necessary.
- Establish the training conditions required to proceed to Polling.Configuration.

Polling has four substates:

- **Polling.Active**
  - Initial training stage where the link partners exchange TS1 Ordered Sets and establish synchronization and training conditions.

- **Polling.Compliance**
  - Used for PCIe electrical compliance testing and specified interoperability testing.

- **Polling.Configuration**
  - The link partners exchange TS2 Ordered Sets and establish the conditions required to enter the main Configuration state.

- **Polling.Speed**
  - A defined PCIe 5.0 substate, but it is not reached in the normal PCIe 5.0 initial Link-up flow.

---

# Polling.Active

## Purpose

Polling.Active begins the initial exchange of TS1 Ordered Sets between the two PCIe link partners.

The purpose is to establish the initial synchronization and training conditions required for further PCIe Link initialization.

The receiver examines the incoming TS1 Ordered Sets to establish whether it can reliably recover and recognize the transmitted serial data.

Polling.Active performs the following operations:

- Initial TS1 Ordered Set exchange.
- Receiver Bit Lock.
- Symbol Lock.
- Lane Polarity Inversion detection and correction, where necessary.
- Verification of the required transmitter and receiver training conditions.

## Entry into Polling.Active

The LTSSM enters Polling.Active from Detect.Active after successful Receiver Detection.

The participating Lanes are the Lanes that detected a receiver during Detect.

Receiver Detection confirms the presence of the expected electrical receiver termination, but it does not establish that the receiver can correctly recover and recognize the incoming serial data.

Polling.Active begins the actual training-sequence exchange.

## Initial TS1 Ordered Set Transmission

During Polling.Active, both link partners independently transmit and receive TS1 Ordered Sets.

The initial TS1 Ordered Sets contain:

| TS1 Field | Initial Value |
|---|---|
| Link Number | PAD |
| Lane Number | PAD |

PAD is a placeholder used during the early training process, before Link Numbers and Lane Numbers have been established.

The final association of physical Lanes into a logical PCIe Link is determined during the subsequent Configuration process.

TS1 Ordered Sets are training information. They are not NVMe commands or user-data traffic.

## Initial Link Data Rate

During initial Link training, the Link operates at 2.5 GT/s.

This remains as the initial training rate even when both link partners support higher PCIe generations.

The TS1 Ordered Set contains a Data Rate Identifier that advertises the data rates supported by the Port.

Important:

- The Data Rate Identifier advertises supported data rates.
- Advertising a higher data rate does not mean that the Link is currently operating at that rate.
- The initial training process still takes place at 2.5 GT/s.

For PCIe Gen3, Gen4 and Gen5 devices, the initial 2.5 GT/s training process uses 8b/10b encoding.

## Bit Lock

### What Problem Does Bit Lock Solve?

The transmitter sends a serial stream of bits over the PCIe Lane.

The receiver initially receives an electrical waveform rather than already-separated binary values.

The receiver must recover the timing information required to sample the incoming signal correctly.

This operation is performed by the Clock and Data Recovery (CDR) circuit.

CDR recovers timing information from the incoming serial signal so that the receiver can determine the appropriate sampling points.

Bit Lock means that the receiver has recovered the timing required to sample the incoming bit stream correctly.

Without suitable bit timing, the receiver cannot reliably recover the transmitted bits.

Bit Lock does not mean that all Link-training operations are complete.

## Symbol Lock

Bit Lock and Symbol Lock perform different functions.

- Bit Lock establishes the timing required to recover individual bits.
- Symbol Lock identifies the boundaries of encoded symbols in the recovered bit stream.

During initial PCIe training at 2.5 GT/s, 8b/10b encoding is used.

In 8b/10b encoding, each 8-bit data or control value is represented by a 10-bit transmitted code group.

Therefore, after recovering the incoming bits, the receiver must also identify where each encoded symbol begins and ends.

### COM Symbol

The COM symbol is the 8b/10b control character K28.5.

It provides a recognizable alignment pattern that the receiver can use to establish the correct symbol boundary.

Remember:

- Bit Lock: Recover the timing of incoming bits.
- Symbol Lock: Identify the boundaries of encoded symbols.

## Lane Polarity Inversion

### What Is Polarity Inversion?

PCIe uses differential signalling.

Each PCIe Lane has a transmit differential pair and a receive differential pair.

Each differential pair contains two conductors:

- Positive (P).
- Negative (N).

Under normal wiring:

- TX+ is connected to RX+.
- TX− is connected to RX−.

If the P and N conductors of a differential pair are swapped, the polarity of the received differential signal is reversed.

This condition is called Lane Polarity Inversion.

### How Does the Receiver Detect Polarity Inversion?

During Polling, the receiver examines specific symbols in the received TS1 and TS2 Ordered Sets.

For the initial 8b/10b training representation, the PCIe specification defines recognizable normal-polarity and inverted-polarity symbol patterns.

For TS1:

- Normal polarity: Symbols 6–15 contain the expected D10.2 pattern.
- Inverted polarity: Symbols 6–15 are received as D21.5.

For TS2:

- Normal polarity: Symbols 6–15 contain the expected D5.2 pattern.
- Inverted polarity: Symbols 6–15 are received as D26.5.

These patterns allow the receiver to identify whether the incoming signal has inverted polarity.

### What Happens When Polarity Inversion Is Detected?

When the receiver detects polarity inversion, it must invert the received data internally to compensate for the inversion.

The transmitter does not change its transmitted data to perform this correction.

Polarity detection and correction are performed independently for each PCIe receiver Lane.

Both directions of a PCIe Link handle their incoming signals independently.

For example:

- Port A transmits to Port B.
- Port B's receiver detects and corrects any polarity inversion in the incoming signal.
- Port B transmits to Port A, and Port A's receiver independently detects and corrects any polarity inversion in its incoming signal.

### When Must Polarity Correction Be Completed?

The PCIe 5.0 specification requires polarity correction, if necessary, before the receiver exits Polling.Configuration.

Detection and correction may begin earlier, including during Polling.Active.

Therefore, polarity correction does not have to be completed during Polling.Active.

Polarity inversion is not automatically a hardware failure because the receiver can compensate for a supported inversion.

## Normal Transition: Polling.Active to Polling.Configuration

The normal transition from Polling.Active to Polling.Configuration requires two independent conditions.

### Condition — Transmitter Requirement

The local transmitter must transmit at least 1,024 TS1 Ordered Sets.

### Condition — Receiver Requirement

The local receiver must receive eight consecutive qualifying training sequences on every Lane that detected a receiver during Detect.

The qualifying training sequences include the following categories.

**Category — Qualifying TS1**

- Link Number = PAD.
- Lane Number = PAD.
- Compliance Receive = 0b (deasserted).

**Category — Qualifying TS1**

- Link Number = PAD.
- Lane Number = PAD.
- Loopback = 1b (asserted).

**Category — Qualifying TS2**

- Link Number = PAD.
- Lane Number = PAD.

The eight consecutive qualifying training sequences may be a mixture of these qualifying TS1 and TS2 Ordered Sets.

They do not have to be eight TS1 Ordered Sets or eight TS2 Ordered Sets exclusively.

Every participating Lane must independently satisfy the required eight-consecutive-sequence condition.

Both transmitter and receiver requirements must be satisfied for the local Port's normal transition.

When the two conditions are satisfied:

`Polling.Active → Polling.Configuration`

## Independent LTSSM Operation at Both Ports

Each PCIe Port has its own LTSSM and independently evaluates its training conditions.

For example, consider two connected Ports:

- Port A has its own transmitter (A_TX) and receiver (A_RX).
- Port B has its own transmitter (B_TX) and receiver (B_RX).

For Port A's normal transition:

1. A_TX must transmit at least 1,024 TS1 Ordered Sets.
2. A_RX must receive eight consecutive qualifying training sequences on every relevant Lane.

For Port B's normal transition:

1. B_TX must transmit at least 1,024 TS1 Ordered Sets.
2. B_RX must receive eight consecutive qualifying training sequences on every relevant Lane.

The two Ports evaluate their respective conditions independently.

Therefore, one Port can enter Polling.Configuration before the other Port.

## Why Can TS2 Count During Polling.Active?

Although Polling.Active initially involves TS1 transmission, the receiving Port can also count qualifying TS2 Ordered Sets toward its eight-consecutive-sequence requirement.

This is possible because the two link partners have independent LTSSMs.

For example:

1. Port A satisfies its own normal transition conditions and enters Polling.Configuration.
2. Port A's transmitter begins sending TS2 Ordered Sets.
3. Port B may still be in Polling.Active.
4. Port B's receiver can recognize these qualifying TS2 Ordered Sets and include them in its eight-sequence count.

The same situation can occur in the opposite direction if Port B enters Polling.Configuration first.

Therefore, TS2 is included as a qualifying training sequence to accommodate the independent progress of the two link partners.

## Polling.Active — Normal Transition Summary

The normal transition requires:

- At least 1,024 TS1 Ordered Sets transmitted by the local transmitter.
- Eight consecutive qualifying training sequences received on every relevant Lane by the local receiver.

Both conditions must be satisfied.

When they are satisfied:

`Polling.Active → Polling.Configuration`

The normal transition can occur before the 24 ms timeout expires.

## Polling.Active — 24 ms Timeout and Fallback Conditions

### Purpose of the 24 ms Timeout

When the LTSSM enters Polling.Active, a 24 ms timeout is started.

During this period, the link partners perform the initial TS1 training exchange and attempt to satisfy the normal transition requirements.

The normal transition requires:

- The local transmitter must transmit at least 1,024 TS1 Ordered Sets.
- The local receiver must receive eight consecutive qualifying training sequences on every Lane that detected a receiver during Detect.

If both normal transition conditions are satisfied before the 24 ms timeout expires:

`Polling.Active → Polling.Configuration`

However, if the normal transition conditions are not satisfied within the 24 ms timeout, the LTSSM evaluates the specified fallback conditions to determine its next state.

The timeout is a decision point. It does not automatically indicate that the SSD is defective or that the Link must return to Detect.

There are three possible fallback outcomes:

- Outcome A — Continue to Polling.Configuration.
- Outcome B — Enter Polling.Compliance.
- Outcome C — Return to Detect.Quiet.

### Outcome A — Continue to Polling.Configuration

The LTSSM can transition to Polling.Configuration after the 24 ms timeout if all three following conditions are satisfied.

**Receiver Requirement**

At least one Lane that detected a receiver during Detect must have received eight consecutive qualifying training sequences, or their complement.

The qualifying training sequences are the same categories defined in the normal Polling.Active transition:

- TS1 with Link Number = PAD, Lane Number = PAD and Compliance Receive = 0b.
- TS1 with Link Number = PAD, Lane Number = PAD and Loopback = 1b.
- TS2 with Link Number = PAD and Lane Number = PAD.

The eight consecutive qualifying training sequences may be a mixture of these qualifying TS1 and TS2 Ordered Sets.

Unlike the normal transition, this fallback condition requires only at least one relevant Lane to satisfy the eight-consecutive-sequence requirement.

**Transmitter Requirement**

At least 1,024 TS1 Ordered Sets must have been transmitted after receiving one TS1 or TS2 Ordered Set.

The local receiver must first receive one TS1 or TS2 Ordered Set.

After that reception, the local transmitter must transmit at least 1,024 TS1 Ordered Sets.

For example, consider Port A and Port B:

1. Port A's receiver (A_RX) receives one TS1 or TS2 Ordered Set from Port B's transmitter (B_TX).
2. After that reception, Port A's transmitter (A_TX) transmits at least 1,024 TS1 Ordered Sets to Port B's receiver (B_RX).

This satisfies the transmitter requirement for Port A.

**Electrical Idle Exit Requirement**

At least a predetermined set of Lanes that detected receivers during Detect must have detected an exit from Electrical Idle at least once on every Lane in that set since entering Polling.Active.

When a Port enters Polling.Active, its 24 ms timer starts.

During this period:

1. The transmitter exits Electrical Idle and begins transmitting active training signals.
2. The receiving side monitors the incoming Lanes.
3. Each relevant Lane can detect an Electrical Idle exit.
4. The LTSSM records whether the required Lanes have detected this event.

When the 24 ms timeout expires, the LTSSM checks whether every Lane in the predetermined set has detected an Electrical Idle exit at least once since entering Polling.Active.

The Electrical Idle exit event happens during the Polling.Active training period. The LTSSM evaluates the recorded result at the timeout decision point.

The predetermined Lane set is implementation-specific.

It must not automatically be assumed to be identical to the complete set of Lanes that initially detected receivers during Detect.

**Additional Requirement**

Any Lane that receives the eight consecutive TS1 or TS2 Ordered Sets should have detected an exit from Electrical Idle at least once since entering Polling.Active.

#### Why Does Outcome A Exist?

The purpose of this fallback is to prevent a defective receiver or transmitter on one or more Lanes from holding up an otherwise valid Link.

If the required fallback conditions are satisfied, the LTSSM allows further Link training to continue in Polling.Configuration.

When all three conditions are satisfied:

`Polling.Active → Polling.Configuration`

### Outcome B — Enter Polling.Compliance

If Outcome A does not apply, the LTSSM enters Polling.Compliance if either of the following conditions is satisfied.

There are two alternative conditions:

- B1 — Electrical Idle Exit Condition.
- B2 — Compliance Request Through TS1.

**Condition B1 — Electrical Idle Exit Condition**

Not all Lanes in the implementation-defined predetermined Lane set have detected an exit from Electrical Idle since entering Polling.Active.

For example, suppose the predetermined Lane set contains Lane 0 and Lane 1.

| Lane | Electrical Idle Exit Detected? |
|---|---|
| Lane 0 | Yes |
| Lane 1 | No |

Here, not all Lanes in the predetermined set have detected an Electrical Idle exit.

Therefore, B1 is satisfied.

This condition is the opposite of the Electrical Idle exit requirement in Outcome A — Condition 3.

**Condition B2 — Compliance Request Through TS1**

A Lane that detected a receiver during Detect must receive eight consecutive TS1 Ordered Sets, or their complement, with the following field values:

| TS1 Field | Required Value |
|---|---|
| Link Number | PAD |
| Lane Number | PAD |
| Compliance Receive | 1b |
| Loopback | 0b |

The receiver examines the incoming TS1 Ordered Sets for this specific field combination.

For B2, all four field requirements must be satisfied.

The receiver must receive eight consecutive qualifying TS1 Ordered Sets on at least one Lane that detected a receiver during Detect.

This is a separate compliance-entry check.

It must not be confused with the three qualifying training-sequence categories used for normal Polling.Active progression.

The B2 pattern selects the PCIe compliance-testing path.

**Example**

Suppose A_RX0 receives eight consecutive TS1 Ordered Sets with:

| TS1 Field | Received Value |
|---|---|
| Link Number | PAD |
| Lane Number | PAD |
| Compliance Receive | 1b |
| Loopback | 0b |

All eight sequences contain the required field values.

Therefore, B2 is satisfied.

If Outcome A does not apply and either B1 or B2 is satisfied:

`Polling.Active → Polling.Compliance`

**Important Specification Note**

If a passive test load is applied on all Lanes, the device enters Polling.Compliance.

### Outcome C — Return to Detect.Quiet

If neither Outcome A nor Outcome B applies, the LTSSM returns to Detect.Quiet.

Transition:

`Polling.Active → Detect.Quiet`

When the LTSSM enters Detect.Quiet:

1. The transmitter is placed in Electrical Idle.
2. The LTSSM waits for the applicable transition condition.
3. The 12 ms timeout or a qualifying Electrical Idle exit causes a transition to Detect.Active.
4. Receiver Detection is performed again on the relevant Lanes.

The Link therefore restarts the receiver-detection process.

Returning to Detect.Quiet does not automatically prove that the SSD is defective.

The 24 ms timeout indicates that the required training and fallback conditions were not completed for the other outcomes.

The root cause must be established through further validation and debugging.

## Important Difference Between Normal Transition and Timeout Fallback

The normal transition and timeout fallback use different receiver requirements.

| Requirement | Normal Transition | Timeout Outcome A |
|---|---|---|
| Receiver requirement | Eight consecutive qualifying sequences on every Lane that detected a receiver during Detect | Eight consecutive qualifying sequences on at least one Lane that detected a receiver during Detect |
| Transmitter requirement | At least 1,024 TS1 Ordered Sets transmitted | At least 1,024 TS1 Ordered Sets transmitted after receiving one TS1 or TS2 Ordered Set |
| Electrical Idle exit requirement | Not one of the two stated normal transition conditions | Every Lane in the implementation-defined predetermined set must have detected an Electrical Idle exit since entering Polling.Active |

Do not confuse the normal transition requirements with the timeout fallback requirements.

## Polling.Active — Complete 24 ms Timeout Transition Summary

| Current State | Condition | Next State |
|---|---|---|
| Polling.Active | Normal transmitter and receiver training conditions satisfied before timeout | Polling.Configuration |
| Polling.Active | 24 ms timeout; all three Outcome A conditions satisfied | Polling.Configuration |
| Polling.Active | 24 ms timeout; Outcome A does not apply and B1 or B2 is satisfied | Polling.Compliance |
| Polling.Active | 24 ms timeout; neither Outcome A nor Outcome B applies | Detect.Quiet |

## Polling.Active — Final Understanding

The 24 ms timeout provides a decision point when normal initial Link training has not completed.

The LTSSM evaluates the specified fallback conditions to decide whether to:

- Continue Link training in Polling.Configuration.
- Enter Polling.Compliance for electrical compliance testing.
- Return to Detect.Quiet and restart receiver detection.

The timeout alone does not identify the root cause of a Link-training failure.

---

# Polling.Compliance

## Purpose

Polling.Compliance is a special-purpose substate of the PCIe Polling state, intended for PCIe electrical compliance testing.

Its purpose is to evaluate the electrical behaviour of the PCIe Transmitter and the interconnect present in the device-under-test setup.

It also supports a simplified interoperability-testing mechanism using predefined PCIe compliance patterns.

The test environment may use equipment such as:

- Oscilloscope.
- Pattern Generator.
- Bit Error Rate Tester (BERT).
- PCIe Compliance Fixture or Load Board.

Polling.Compliance is not a required stage of normal NVMe SSD startup.

An unexpected entry into this substate must be investigated, but it does not automatically indicate an SSD hardware failure.

## Entry into Polling.Compliance

The LTSSM enters Polling.Compliance from Polling.Active through the specified compliance-entry mechanisms.

### Entry Through the Enter Compliance Register

The Enter Compliance bit is bit 4 of the Link Control 2 register.

If this bit is set before the LTSSM enters Polling.Active:

1. The LTSSM enters Polling.Active.
2. It recognizes that the Enter Compliance bit is already asserted.
3. It immediately transitions to Polling.Compliance.
4. It does not transmit TS1 Ordered Sets before entering the compliance substate.

If the Enter Compliance bit is asserted while Polling.Active is already in progress, the applicable compliance-entry rules must be followed.

### Entry Through the 24 ms Polling.Active Timeout — Outcome B

When the normal Polling.Active transition conditions are not satisfied before the 24 ms timeout, the LTSSM evaluates the specified fallback conditions.

If Outcome A does not apply, the LTSSM enters Polling.Compliance when either of the following conditions is satisfied.

**B1 — Electrical Idle Exit Condition**

Not all Lanes belonging to the implementation-defined predetermined Lane set have detected an exit from Electrical Idle at least once since entering Polling.Active.

This indicates that the required Electrical Idle exit condition has not been observed on every Lane in the predetermined set.

**B2 — Compliance Request Through TS1**

At least one Lane that detected a Receiver during Detect must receive eight consecutive TS1 Ordered Sets, or their complement, with the following field values:

| TS1 Field | Required Value |
|---|---|
| Link Number | PAD |
| Lane Number | PAD |
| Compliance Receive | 1b |
| Loopback | 0b |

This field combination represents the specified Compliance Receive request.

When Outcome A does not apply and either B1 or B2 is satisfied:

`Polling.Active → Polling.Compliance`

### Entry Due to Passive Test Load

When a passive test load is applied on all Lanes, the device enters Polling.Compliance through the specified Polling.Active fallback behaviour.

## Internal Operations of Polling.Compliance

Once the LTSSM enters Polling.Compliance, the Transmitter performs the electrical compliance-testing operations.

### Transmit Margin Configuration

On entry, the Transmit Margin field of the local Link Control 2 register is sampled.

The selected voltage level must become effective at the Transmitter package pins within 192 ns of entering this substate.

The selected voltage level remains effective throughout Polling.Compliance.

The register contains the configured selection; the transmitter applies the corresponding electrical output setting.

The Transmit Margin field is distinct from the Enter Compliance bit, which requests entry into the substate.

### Compliance Data Rate and Transmitter Settings

The compliance transmission data rate and de-emphasis or preset settings depend on the entry mechanism.

**Port capable of transmitting only at 2.5 GT/s**

- Compliance-pattern data rate: 2.5 GT/s.
- De-emphasis: -3.5 dB.

**Entry Due to Compliance Receive TS1 Ordered Sets**

- The compliance-pattern data rate is the highest common data rate indicated by the transmitted and received Data Rate Identifiers in the relevant eight consecutive TS1 Ordered Sets.
- The received TS1 Selectable De-emphasis information is used for the applicable 5.0 GT/s operation.
- At 8.0 GT/s or higher, the Transmitter Preset information in eight consecutive EQ TS1 Ordered Sets on the corresponding Lane is used, provided the encoding is not Reserved.
- At 8.0 GT/s, use of the Receiver Preset Hint in those EQ TS1 Ordered Sets is optional.
- A Lane without the required eight consecutive EQ TS1 Ordered Sets, or with a Reserved preset encoding, may use any supported Transmitter Preset in an implementation-specific manner.

**Entry Due to the Enter Compliance Bit in Link Control 2**

- The compliance-pattern data rate is selected by the Target Link Speed field in Link Control 2.
- At 5.0 GT/s, the Compliance Preset/De-emphasis field (bits 15:12) determines the de-emphasis selection.
- At 8.0 GT/s or higher, the same field provides the Transmitter Preset encoding, provided the encoding is not Reserved.

**Default Compliance-Setting Sequence**

When the preceding selection branches do not apply, the Port follows the PCIe-defined compliance-setting sequence according to its maximum supported data rate and the number of entries into Polling.Compliance using this entry criterion.

The sequence repeats on subsequent entries.

| Setting | Compliance Data Rate | De-emphasis / Preset |
|---|---:|---|
| #1 | 2.5 GT/s | -3.5 dB |
| #2 | 5.0 GT/s | -3.5 dB |
| #3 | 5.0 GT/s | -6 dB |
| #4–#14 | 8.0 GT/s | Transmitter Presets 0000b through 1010b in increasing order |
| #15–#25 | 16.0 GT/s | Transmitter Presets 0000b through 1010b in increasing order |
| #26–#34 | 16.0 GT/s | Transmitter Preset 0100b |
| #35–#45 | 32.0 GT/s | Transmitter Presets 0000b through 1010b in increasing order |
| #46–#54 | 32.0 GT/s | Transmitter Preset 0100b |

All Ports are permitted to reset the sequence to Setting #1 in Polling.Configuration. The specification requires this in certain cases, including Ports supporting 16.0 GT/s or higher. 

If a compliance-pattern data rate above 2.5 GT/s is selected, the transmitter must enter Electrical Idle and change to the selected rate before sending the pattern.

Where TS1 Ordered Sets were transmitted in Polling.Active before entering Polling.Compliance, the transmitter sends one or two consecutive EIOS Ordered Sets as specified before entering Electrical Idle.

If no TS1 Ordered Sets were transmitted before entering Polling.Compliance, the transmitter enters Electrical Idle without sending EIOS.

During Electrical Idle:

- The operating data rate is changed to the selected compliance rate.
- The new rate is allowed to stabilize.
- At 5.0 GT/s, the de-emphasis level is selected according to the `select_deemphasis` variable.
- At 8.0 GT/s or higher, the transmitter uses the selected preset.

The Electrical Idle period must be greater than 1 ms and must not exceed 2 ms.

### Compliance Pattern Transmission

Depending on the entry mechanism and selected compliance setting, the Transmitter sends one of the following patterns.

**Standard Compliance Pattern**

Used for ordinary electrical compliance testing.

It allows the electrical behaviour and signal quality of the transmitter to be evaluated.

**Modified Compliance Pattern**

Used for the specified interoperability-testing mechanism.

It supports receiver pattern-lock indication and receiver error reporting.

**Jitter Measurement Pattern**

Used for transmitter jitter measurement on specified Lanes.

The other applicable Lanes may continue transmitting the ordinary Compliance Pattern, depending on the selected compliance setting.

The internal bit-by-bit pattern definitions are not required for this learning scope.

## Compliance Test Equipment

### Oscilloscope

An oscilloscope captures and measures the actual electrical waveform produced by the transmitter.

It is used to examine:

- Transmitter voltage and amplitude.
- Eye diagram characteristics.
- Jitter.
- Signal quality and electrical compliance.

### Pattern Generator

A pattern generator produces known data sequences for testing the PCIe receiver or supporting interoperability testing.

### BERT — Bit Error Rate Tester

A BERT generates controlled test signals and/or checks received patterns to determine the number of bit errors.

It is particularly important for PCIe receiver testing.

A receiver test may operate as follows:

1. The BERT generates a calibrated, stressed signal.
2. The signal enters the DUT Receiver.
3. The DUT is configured for the required receiver test, commonly using Loopback.
4. The received data is returned through the DUT Transmitter.
5. The BERT compares the returned data against the expected pattern and calculates BER.

Bit Error Rate:

`BER = Number of erroneous bits / Total number of received bits`

### PCIe Compliance Fixture / Load Board

Provides the specified electrical connection, termination and load for the compliance test environment.

### PCIe Protocol Analyzer

A PCIe Protocol Analyzer, such as a Teledyne LeCroy analyzer, captures and decodes PCIe protocol activity.

It is used to examine:

- TS1 and TS2 Ordered Sets.
- Compliance-related training fields.
- LTSSM-related protocol activity.
- Link-training failures.
- Protocol errors and abnormal behaviour.

A Protocol Analyzer does not replace an oscilloscope for analog waveform or jitter measurements.

The distinction is:

- Oscilloscope: Examines the actual electrical waveform.
- BERT: Generates and checks test patterns and measures bit errors.
- Protocol Analyzer: Captures and decodes PCIe protocol activity.

## Exiting Polling.Compliance

The exit conditions depend on the mechanism used to enter Polling.Compliance.

There are three important exit paths.

### Modified Compliance Pattern Exit

This path applies when the Port enters Polling.Compliance through the specified Compliance Receive TS1 mechanism, or when both Enter Compliance and Enter Modified Compliance are asserted in Link Control 2.

**Case — Enter Compliance is 0b**

When the Enter Compliance bit is 0b, the next state is Detect if directed.

This is a directed return to the Detect state rather than an automatic transition to Polling.Active.

The receiver-detection process must be performed again.

**Case — Enter Compliance was 1b when entering Polling.Compliance**

The LTSSM can transition to Polling.Active when either of the following conditions occurs:

- The Enter Compliance bit changes from 1b to 0b after entry into Polling.Compliance.
- The Port is an Upstream Port, the Enter Compliance bit is still 1b, and an EIOS is received on any Lane.

When the Upstream Port receives the EIOS under this condition, the Enter Compliance bit is reset to 0b.

Before transitioning to Polling.Active, the Transmitter performs the required return sequence when it was transmitting above 2.5 GT/s or when Enter Compliance was set to 1b on entry.

The return sequence includes:

- Transmitting eight consecutive EIOS Ordered Sets.
- Entering Electrical Idle.
- Changing the operating data rate to 2.5 GT/s.
- Allowing the new data rate to stabilize.
- Setting the de-emphasis level to -3.5 dB as specified.

The Electrical Idle period must be greater than 1 ms and must not exceed 2 ms.

After completing the required sequence:

`Polling.Compliance → Polling.Active`

### Register-Directed Standard Compliance Pattern Exit

This path applies when the Port entered Polling.Compliance because Enter Compliance was set to 1b and Enter Modified Compliance was set to 0b.

The Transmitter sends the Compliance Pattern on all Lanes that detected a Receiver during Detect, using the previously selected data rate and de-emphasis or preset settings.

The LTSSM transitions to Polling.Active when either of the following conditions occurs.

**Condition — Enter Compliance is Cleared**

- Enter Compliance was 1b when the Port entered Polling.Compliance.
- Software changes the bit from 1b to 0b.
- The specified exit condition is satisfied.

**Condition — Upstream Port Receives EIOS**

- The Port is an Upstream Port.
- Enter Compliance is still 1b.
- The Receiver detects an EIOS on any Lane.
- The Enter Compliance bit is reset to 0b.

Before transitioning to Polling.Active, the Transmitter:

- Sends eight consecutive EIOS Ordered Sets.
- Enters Electrical Idle.
- Changes the operating data rate to 2.5 GT/s.
- Allows the new data rate to stabilize.

The Electrical Idle period must be greater than 1 ms and must not exceed 2 ms.

After completing the sequence:

`Polling.Compliance → Polling.Active`

The purpose of sending multiple EIOS Ordered Sets is to provide sufficient robustness for the other Port to detect at least one EIOS and exit its compliance condition.

### Default Compliance Pattern Sequence Exit

This path applies when the Port is using the default compliance-setting sequence.

The Transmitter sends the applicable Standard Compliance Pattern or Jitter Measurement Pattern according to the current compliance setting.

The exit condition is:

**The Receiver detects an exit from Electrical Idle on any Lane that detected a Receiver during Detect.**

The exit sequence is:

- The Receiver detects the Electrical Idle exit.
- The LTSSM satisfies the transition condition.
- If the Transmitter is operating at a data rate other than 2.5 GT/s, it sends eight consecutive EIOS Ordered Sets.
- The Transmitter enters Electrical Idle.
- The operating data rate is changed to 2.5 GT/s.
- The new data rate is allowed to stabilize.

The Electrical Idle period must be greater than 1 ms and must not exceed 2 ms.

After completing the required return sequence:

`Polling.Compliance → Polling.Active`

If the Transmitter is already operating at 2.5 GT/s, the additional higher-rate return sequence is not required.

## What Happens After Exiting Polling.Compliance?

This is an important distinction in PCIe LTSSM operation.

The Port does not directly transition from Polling.Compliance to Polling.Configuration.

It has two possible exit destinations depending on the applicable exit condition.

### Return to Polling.Active

For the normal compliance exit paths:

`Polling.Compliance → Polling.Active`

After returning to Polling.Active:

- The Port resumes the TS1 Ordered Set training process.
- The Port must satisfy the applicable Polling.Active transmitter and receiver conditions.
- The normal transition requires at least 1,024 TS1 Ordered Sets transmitted by the local Transmitter.
- The local Receiver must receive eight consecutive qualifying training sequences on every Lane that detected a Receiver during Detect.

When the normal requirements are satisfied:

`Polling.Active → Polling.Configuration`

Therefore, the complete progression is:

```text
Polling.Compliance
        |
        | Applicable compliance exit condition
        v
   Polling.Active
        |
        | Required TS1 training conditions satisfied
        v
Polling.Configuration
```

### Directed Return to Detect

Under the specified Modified Compliance exit condition, the LTSSM can transition directly to Detect when directed.

In this case:

`Polling.Compliance → Detect`

The Port restarts the receiver-detection process through the Detect state.

The progression is:

```text
Polling.Compliance
        |
        | Directed transition to Detect
        v
       Detect
        |
        | Receiver Detection
        v
  Polling.Active
```

The Port must complete the subsequent Polling and Configuration procedures before reaching normal Link operation.

## Polling.Compliance — Complete Entry and Exit Summary

| Entry Condition | Exit Condition | Next State |
|---|---|---|
| Enter Compliance bit set before Polling.Active | Applicable register-directed compliance exit | Polling.Active |
| 24 ms timeout, Outcome A does not apply, B1 satisfied | Applicable compliance-pattern exit | Polling.Active |
| 24 ms timeout, Outcome A does not apply, B2 satisfied | Modified Compliance exit condition | Polling.Active or directed Detect |
| Passive test load on all Lanes | Applicable default compliance-pattern exit | Polling.Active |
| Modified Compliance path with Enter Compliance = 0b | Detect transition when directed | Detect |

Important:

- Polling.Compliance does not directly transition to Polling.Configuration.
- Polling.Active is the usual exit destination after compliance testing.
- Detect is a possible directed exit destination under the specified Modified Compliance condition.
- If the Port returns to Polling.Active, it must satisfy the Polling.Active transition requirements before proceeding to Polling.Configuration.
- If the Port returns to Detect, Receiver Detection must be performed again.
- Returning to Polling.Active or Detect does not mean that the Link has reached L0.

## Enterprise NVMe SSD Validation Relevance

If an NVMe SSD unexpectedly enters Polling.Compliance, the validation engineer must first determine why the LTSSM selected this substate.

The investigation should include:

- Whether compliance testing was intentionally requested by software or firmware.
- Enter Compliance and Enter Modified Compliance register settings.
- Target Link Speed, Compliance Preset/De-emphasis and Transmit Margin.
- Received TS1 Compliance Receive and Loopback fields.
- Electrical Idle behaviour and Receiver Detection.
- Root Port and SSD Endpoint behaviour.
- Passive test load and physical test environment.
- Protocol traces and electrical measurements.

A Polling.Active timeout or unexpected Polling.Compliance entry alone does not establish the root cause of a Link-training failure.

## Polling.Compliance — Essential Understanding

The important engineering capability is to explain:

- Why Polling.Compliance exists.
- How the LTSSM enters this substate.
- What transmitter operations take place during compliance testing.
- Why the compliance exit condition depends on the entry mechanism.
- When the Port returns to Polling.Active.
- When a directed transition to Detect occurs.
- Why the Port must complete Polling and Configuration again before reaching L0.

---
# Polling.Configuration

## Purpose

`Polling.Configuration` continues the initial PCIe Link-training process after Polling.Active.

Its purpose is to establish the additional training conditions required before entering the main Configuration state.

During Polling.Configuration, the two PCIe link partners exchange TS2 Ordered Sets.

The receiver examines the incoming TS2 Ordered Sets to determine whether the required consecutive training sequences have been received.

The transmitter also has to satisfy the specified TS2 transmission requirement.

Polling.Configuration performs the following operations:

- Continues initial Link training using TS2 Ordered Sets.
- Performs the required receiver polarity correction, if necessary.
- Resets the Transmit Margin field to the normal operating-range selection.
- Updates the Polling.Compliance sequence setting when required.
- Establishes the required receiver and transmitter TS2 conditions.
- Determines whether the Port can proceed to the main Configuration state.

Successful completion of Polling.Configuration does not mean that the final PCIe Link has already been established.

Link-width negotiation and Lane Number assignment take place in the subsequent Configuration substates.

## Entry into Polling.Configuration

The LTSSM enters Polling.Configuration from Polling.Active when the applicable transition requirements are satisfied.

There are two possible entry paths.

### Normal Entry from Polling.Active

The normal Polling.Active transition requires:

- The local transmitter must transmit at least 1,024 TS1 Ordered Sets.
- The local receiver must receive eight consecutive qualifying training sequences on every Lane that detected a receiver during Detect.

When both conditions are satisfied before the 24 ms timeout:

`Polling.Active → Polling.Configuration`

### Entry Through the Polling.Active Timeout — Outcome A

If the normal transition conditions are not satisfied before the 24 ms timeout expires, the LTSSM evaluates the specified fallback conditions.

The LTSSM can still enter Polling.Configuration when all three Outcome A conditions are satisfied:

- At least one Lane that detected a receiver during Detect must receive eight consecutive qualifying training sequences, or their complement.
- At least 1,024 TS1 Ordered Sets must be transmitted after receiving one TS1 or TS2 Ordered Set.
- The implementation-defined predetermined set of Lanes must satisfy the specified Electrical Idle exit requirement.

Any Lane that receives the eight consecutive qualifying training sequences must also have detected an Electrical Idle exit at least once since entering Polling.Active.

When all three conditions are satisfied:

`Polling.Active → Polling.Configuration`

The two entry paths have different requirements in Polling.Active, but both lead to the same Polling.Configuration substate.

## Entry Actions

When the LTSSM enters Polling.Configuration, the following entry actions apply:

- The receiver must invert polarity if necessary.
- The Transmit Margin field of the Link Control 2 register must be reset to `000b`.
- The transmitter's Polling.Compliance sequence setting must be updated if required by the specification.

### Receiver Polarity Correction

The receiver must correct Lane polarity inversion if necessary.

Consider two connected PCIe Ports:

```text
Port A                              Port B

A_TX  --------------------------->  B_RX
A_RX  <---------------------------  B_TX
```

Suppose the signal transmitted by A_TX reaches B_RX with inverted polarity.

B_RX must identify the inversion and compensate for it internally.

The receiver performs this correction. The transmitter does not need to modify its transmitted data to compensate for the inversion.

Both directions operate independently.

- If A_RX detects polarity inversion in the signal received from B_TX, A_RX performs the correction.
- If B_RX detects polarity inversion in the signal received from A_TX, B_RX performs the correction.

Polarity detection and correction are supported independently for each receiver Lane.

**When must polarity correction be completed?**

Polarity inversion may already have been detected and corrected during Polling.Active.

However, the PCIe specification requires polarity correction, if necessary, to be complete before the receiver exits Polling.Configuration.

Therefore, polarity correction does not necessarily have to be performed again when entering Polling.Configuration.

The important requirement is that the receiver must complete the correction before leaving this substate.

### Reset the Transmit Margin Field

The Transmit Margin field is present in the Link Control 2 register.

It controls the transmitter's non-de-emphasized voltage level at the transmitter package pins.

During electrical compliance testing, a particular Transmit Margin selection may be used to evaluate transmitter electrical behaviour.

When the Port enters Polling.Configuration, the Transmit Margin field must be reset to:

`Transmit Margin = 000b`

`000b` represents the normal operating-range voltage selection.

For example:

- A Port enters Polling.Compliance for electrical testing.
- A non-default Transmit Margin value may have been selected for that testing.
- The Port subsequently enters Polling.Configuration.
- The Transmit Margin field must be reset to `000b`.

This restores the normal operating-range voltage selection instead of retaining a special compliance-testing selection.

**Important:**

- This action changes the transmitter voltage-level selection.
- It does not change the PCIe data rate.
- It does not assign Link Numbers or Lane Numbers.
- It does not perform equalization.

The reset is required on every entry into Polling.Configuration, including a direct transition from Polling.Active.

### Update the Polling.Compliance Sequence Setting

The third entry action concerns the transmitter's future PCIe compliance-testing sequence.

It is important not to confuse this sequence with the TS1 and TS2 Ordered Set training sequences.

PCIe defines a separate sequence of compliance-test settings.

Each setting identifies a particular combination of:

- Compliance-pattern data rate.
- Transmitter de-emphasis or preset.

For example, the default compliance-setting sequence contains settings such as:

| Setting | Compliance Data Rate | Transmitter Setting |
|---|---:|---|
| #1 | 2.5 GT/s | -3.5 dB |
| #2 | 5.0 GT/s | -3.5 dB |
| #3 | 5.0 GT/s | -6 dB |
| #4–#14 | 8.0 GT/s | Different transmitter presets |

When the default compliance-setting sequence applies, successive entries into Polling.Compliance progress through the defined settings.

The Port therefore maintains a position in this compliance-test sequence.

The PCIe specification requires the sequence to be set to Setting #1 in Polling.Configuration when either of the following conditions applies:

- The Port supports 16.0 GT/s or higher data rates.
- The Port's Receivers do not meet the ZRX-DC specification for 2.5 GT/s when operating at 8.0 GT/s or higher data rates.

All Ports are permitted to reset the sequence to Setting #1, even when neither mandatory condition applies.

**Example:**

Suppose a Port has progressed to Setting #4 in its default compliance-test sequence.

When it enters Polling.Configuration, the applicable requirement may reset the sequence to Setting #1.

The next applicable entry into Polling.Compliance using the default sequence will then start from Setting #1.

**Important:**

- This action does not reset TS1 or TS2 training.
- The Port does not transmit compliance patterns as part of this entry action.
- It updates the selection position for a subsequent compliance-testing entry.

## TS2 Ordered Set Transmission

After completing the entry actions, the Port begins transmitting TS2 Ordered Sets.

During Polling.Configuration, the transmitter sends TS2 Ordered Sets on all Lanes that detected a receiver during Detect.

The participating physical Lanes are determined using the receiver-detection results obtained during Detect.

Consider an example with four physical Lanes.

Suppose Port A detected receivers on Lane 0, Lane 1 and Lane 2, but did not detect a receiver on Lane 3.

A_TX sends TS2 Ordered Sets on the three Lanes that detected receivers.

| Port A Physical Lane | Receiver Detected During Detect? | TS2 Transmission |
|---|---|---|
| Lane 0 | Yes | Yes |
| Lane 1 | Yes | Yes |
| Lane 2 | Yes | Yes |
| Lane 3 | No | No |

Port B performs its own independent operation.

B_TX sends TS2 Ordered Sets on the Lanes that detected receivers during Port B's own Detect process.

Port A and Port B do not automatically use identical receiver-detected Lane sets.

### Link Number and Lane Number in TS2

During Polling.Configuration, both the Link Number and Lane Number fields remain set to PAD.

| TS2 Field | Value |
|---|---|
| Link Number | PAD |
| Lane Number | PAD |

PAD is a placeholder used during the early Link-training process.

At this stage:

- The hardware already knows which physical Lanes detected receivers during Detect.
- The Link Number has not yet been assigned in the TS2 Ordered Sets.
- The Lane Number has not yet been assigned in the TS2 Ordered Sets.

The physical receiver-detection results and logical Lane Number assignments are two separate things.

For example, the hardware can identify that physical Lane 0 detected a receiver without having assigned that Lane its logical PCIe Lane Number.

The Link Number and Lane Number are established during the subsequent Configuration process.

### Data Rate Identifier in TS2

The TS2 Ordered Set contains a Data Rate Identifier.

The Data Rate Identifier advertises all data rates supported by the transmitting Port, including rates that the Port does not intend to use.

For example, a Port may support:

- 2.5 GT/s
- 5.0 GT/s
- 8.0 GT/s
- 16.0 GT/s
- 32.0 GT/s

Its TS2 Ordered Sets advertise these supported data rates.

However, advertising support for 32.0 GT/s does not mean that the transmitter is currently operating at 32.0 GT/s.

Under the PCIe 5.0 initial Link-establishment baseline, the initial training rate remains 2.5 GT/s.

Important:

- The Data Rate Identifier advertises supported data rates.
- Advertising a higher data rate does not mean that the transmitter is currently operating at that rate.
- Initial Link training and TS2 transmission take place at 2.5 GT/s.
- A subsequent higher-speed change is performed through the specified PCIe speed-change process.

## Normal Transition: Polling.Configuration to Configuration

The normal transition from Polling.Configuration to the main Configuration state requires two independent conditions.

Both conditions must be satisfied.

### Condition 1 — Receiver Requirement

The receiver must receive:

**Eight consecutive TS2 Ordered Sets with Link Number = PAD and Lane Number = PAD.**

These TS2 Ordered Sets must be received on any Lane or Lanes that detected a receiver during Detect.

This is different from the normal Polling.Active receiver requirement.

In Polling.Active:

- Every Lane that detected a receiver during Detect must independently satisfy the eight-consecutive-qualifying-training-sequence requirement.

In Polling.Configuration:

- At least one Lane that detected a receiver during Detect must receive eight consecutive qualifying TS2 Ordered Sets.
- All receiver-detected Lanes do not have to satisfy this condition.

**At least one receiver-detected Lane is sufficient.**

#### Example — Four Receiver-Detected Lanes

Suppose A_RX detected receivers on all four physical Lanes during Detect.

| Port A Receiver Lane | Eight Consecutive Qualifying TS2 Received? |
|---|---|
| Lane 0 | Yes |
| Lane 1 | No |
| Lane 2 | No |
| Lane 3 | No |

The receiver requirement is satisfied because Lane 0 received eight consecutive qualifying TS2 Ordered Sets.

Lanes 1, 2 and 3 do not have to satisfy the same eight-sequence condition.

If all four Lanes receive eight consecutive qualifying TS2 Ordered Sets, the requirement is also satisfied, but that is not mandatory.

**Remember:**

The receiver requirement is based on at least one Lane from the set of Lanes that detected receivers during Detect.

### Condition 2 — Transmitter Requirement

The transmitter must transmit at least 16 TS2 Ordered Sets after its local receiver has received one TS2 Ordered Set.

Consider two connected PCIe Ports:

```text
Port A                              Port B

A_TX  --------------------------->  B_RX
A_RX  <---------------------------  B_TX
```

The operation for Port A is as follows.

**A_RX Receives One TS2**

B_TX transmits TS2 Ordered Sets towards A_RX.

A_RX receives one TS2 Ordered Set.

This reception establishes the starting event for Port A's transmitter requirement.

**A_TX Transmits at Least 16 TS2 Ordered Sets**

After A_RX receives one TS2 Ordered Set, A_TX must transmit at least 16 TS2 Ordered Sets towards B_RX.

```text
A_RX receives one TS2 from B_TX
               |
               v
A_TX transmits at least 16 TS2
               |
               v
      Transmitter requirement
           is satisfied
```

These are separate receiver and transmitter operations.

The transmitter requirement is not a requirement to transmit 16 additional TS2 Ordered Sets for every TS2 received.

It must also not be confused with the eight-consecutive-TS2 receiver requirement.

### Independent LTSSM Operation at Both Ports

Each PCIe Port has its own LTSSM and independently evaluates its transition requirements.

For example:

**Port A:**

- A_RX receives one TS2 Ordered Set from B_TX.
- A_TX subsequently transmits at least 16 TS2 Ordered Sets.
- A_RX also satisfies the eight-consecutive-TS2 receiver requirement on at least one receiver-detected Lane.
- Port A can transition to Configuration.Linkwidth.Start when both conditions are satisfied.

**Port B:**

- B_RX receives one TS2 Ordered Set from A_TX.
- B_TX subsequently transmits at least 16 TS2 Ordered Sets.
- B_RX also satisfies the eight-consecutive-TS2 receiver requirement on at least one receiver-detected Lane.
- Port B can transition to Configuration.Linkwidth.Start when both conditions are satisfied.

The two Ports evaluate their conditions independently.

Therefore, one Port can enter Configuration.Linkwidth.Start before the other Port.

### Both Conditions Must Be Satisfied

| Condition | Requirement |
|---|---|
| Receiver requirement | Eight consecutive TS2 Ordered Sets with PAD/PAD received on at least one Lane that detected a receiver during Detect |
| Transmitter requirement | At least 16 TS2 Ordered Sets transmitted after the local receiver receives one TS2 Ordered Set |

Both conditions must be satisfied for the normal transition.

When they are satisfied:

`Polling.Configuration → Configuration.Linkwidth.Start`

Configuration.Linkwidth.Start is the first substate of the main Configuration state.

The normal transition can occur as soon as both conditions are satisfied. The Port does not have to wait for the complete 48 ms timeout.

## Polling.Configuration — 48 ms Timeout and Failure Behaviour

### Purpose of the 48 ms Timeout

When a Port enters Polling.Configuration, a 48 ms timeout period applies.

This timeout is separate from the 24 ms timeout used in Polling.Active.

During this period, the link partners continue transmitting and receiving TS2 Ordered Sets and attempt to satisfy both normal transition conditions.

If both conditions are satisfied before the 48 ms timeout expires:

`Polling.Configuration → Configuration.Linkwidth.Start`

The Port does not have to wait for the complete 48 ms period.

However, if both normal transition requirements are not satisfied within the 48 ms timeout, the LTSSM returns to Detect.

**Transition:**

`Polling.Configuration → Detect`

The next entry into Detect begins with Detect.Quiet.

The Port subsequently restarts the receiver-detection process.

### No Three-Way Timeout Fallback

Unlike Polling.Active, Polling.Configuration does not have three alternative fallback outcomes.

In Polling.Active, the 24 ms timeout evaluates three possible outcomes:

- Outcome A — Continue to Polling.Configuration.
- Outcome B — Enter Polling.Compliance.
- Outcome C — Return to Detect.Quiet.

In Polling.Configuration, the 48 ms timeout has one specified timeout outcome when the normal transition requirements are not satisfied:

`Polling.Configuration → Detect`

There is no equivalent three-way fallback decision.

### Polling.Configuration Timeout — Complete Sequence

Suppose Port A fails to satisfy both normal transition conditions within 48 ms.

The progression is:

```text
Polling.Configuration
          |
          | 48 ms timeout expires
          | without satisfying both conditions
          v
      Detect.Quiet
          |
          v
      Detect.Active
          |
          | Receiver Detection
          v
      Polling.Active
          |
          | TS1 training requirements
          v
  Polling.Configuration
```

The Port restarts the receiver-detection and Link-training process.

Each Port has its own LTSSM. The Port experiencing the timeout follows its own specified transition; both link partners must not be assumed to transition simultaneously.

Returning to Detect does not automatically prove that the SSD or its link partner is defective.

The actual root cause must be established through further validation and debugging.

## Enterprise NVMe SSD Validation Relevance

If an NVMe SSD repeatedly enters Polling.Configuration but fails to progress to Configuration.Linkwidth.Start, the validation engineer must determine why the normal TS2 transition requirements are not being satisfied.

The investigation should include:

### TS2 Ordered Set Transmission

- Verify that the transmitter sends TS2 Ordered Sets on all Lanes that detected receivers during Detect.
- Check that the Link Number and Lane Number fields contain PAD.
- Check the Data Rate Identifier information.
- Confirm that initial training is operating at 2.5 GT/s.

### Receiver Behaviour

- Verify that the receiver recognizes TS2 Ordered Sets.
- Check whether eight consecutive qualifying TS2 Ordered Sets are received on at least one receiver-detected Lane.
- Investigate Lane-specific reception problems.
- Check whether polarity correction has been completed where necessary.

### Transmitter Behaviour

- Verify that the local receiver has received a TS2 Ordered Set.
- Verify that the local transmitter subsequently transmits at least 16 TS2 Ordered Sets.
- Confirm that the transmitter requirement is satisfied independently of the receiver requirement.

### Link Partner and Physical Connection

- Compare Root Port and Endpoint training behaviour.
- Investigate physical Lane connectivity.
- Examine relevant Electrical Idle and signal-reception behaviour.
- Collect available PCIe protocol-analyser traces and hardware-debug information.

### Understanding a Polling.Configuration Timeout

If a Port returns to Detect after the 48 ms timeout, it indicates that the required TS2 training conditions were not completed within the specified period.

The timeout alone does not identify the root cause.

The validation engineer must use the available protocol and hardware evidence to determine whether the issue is related to:

- TS2 transmission.
- Receiver recognition.
- Lane-specific behaviour.
- The transmitter's TS2 count.
- Link partner training behaviour.
- Physical connectivity or platform configuration.

## Polling.Configuration — Complete Transition Summary

| Current State | Condition | Next State |
|---|---|---|
| Polling.Active | Normal transmitter and receiver requirements satisfied | Polling.Configuration |
| Polling.Active | 24 ms timeout; Outcome A conditions satisfied | Polling.Configuration |
| Polling.Configuration | Eight consecutive qualifying TS2 received on at least one receiver-detected Lane AND transmitter sends at least 16 TS2 after receiving one TS2 | Configuration.Linkwidth.Start |
| Polling.Configuration | 48 ms timeout expires without satisfying both normal conditions | Detect |

## Polling.Configuration — Essential Understanding

The important engineering capability is to explain:

- Why Polling.Configuration follows Polling.Active.
- What the three entry actions accomplish.
- Why polarity correction must be complete before leaving Polling.Configuration.
- Why the Transmit Margin field is reset to `000b`.
- What the Polling.Compliance sequence setting controls.
- Which physical Lanes transmit TS2 Ordered Sets.
- Why Link Number and Lane Number remain PAD in TS2.
- What the Data Rate Identifier advertises.
- Why the actual initial training rate remains 2.5 GT/s.
- Why at least one receiver-detected Lane is sufficient for the eight-consecutive-TS2 receiver requirement.
- Why the transmitter must send at least 16 TS2 Ordered Sets after receiving one TS2 Ordered Set.
- Why both normal transition conditions must be satisfied.
- Why the 48 ms timeout returns the Port to Detect instead of using a three-way fallback mechanism.
- How to investigate an NVMe SSD that repeatedly fails to progress beyond Polling.Configuration.

---
---

# Polling.Speed

## Purpose

`Polling.Speed` is a defined LTSSM substate associated with changing the PCIe Link data rate during the Polling stage.

In earlier PCIe Link-training specifications, it was used to move the Link from its initial 2.5 GT/s training rate to the highest common data rate supported by both link partners before proceeding to the main Configuration state.

## Historical Operations

The earlier Polling.Speed procedure involved:

- The Transmitter entering Electrical Idle.
- Changing the operating data rate to the highest common supported data rate.
- Transmitting at least 1,024 consecutive TS1 Ordered Sets on all Lanes at the selected rate.
- Allowing sufficient time for the receiver to establish Bit Lock at the new data rate.
- Transitioning to the main Configuration state after completing the required TS1 transmission.

## Polling.Speed in PCIe 5.0

In PCIe 5.0, Polling.Speed is unreachable during normal Link initialization.

The reason is that PCIe 5.0 establishes the Link at 2.5 GT/s and changes to higher data rates through the Recovery state after reaching L0.

The sequence is:

- Both link partners initially establish the Link at 2.5 GT/s.
- The Link completes Configuration and reaches L0.
- Both Ports' supported data rates are established through the Configuration process.
- A higher-speed transition is subsequently initiated through Recovery.

Therefore, Polling.Speed is not part of the normal PCIe 5.0 initial Link-up sequence.

## Essential Understanding

- Polling.Speed represents an earlier approach to changing the Link data rate during initial Link training.
- Its historical operation involved Electrical Idle, data-rate selection and TS1 transmission at the selected rate.
- PCIe 5.0 does not use Polling.Speed for normal initial Link establishment.
- Higher-speed transitions in PCIe 5.0 are performed through Recovery after reaching L0.

---
---

# Configuration State

## Purpose

The Configuration state is responsible for forming the physical Lanes of a PCIe Port into one or more logical PCIe Links.

During Configuration, the two link partners perform:

- Link-width negotiation.
- Link and Lane identification.
- Lane Number assignment.
- Link formation.
- Lane-to-Lane de-skew.
- Configuration of additional Link parameters.
- Preparation for normal Link operation.

The Lanes of a Port are associated into a logical Link through the Configuration process.

Successful entry into Configuration does not mean that the Link is already operational.

The Link must complete the required Configuration procedures before reaching L0.

## Configuration Substates

The Configuration state contains the following substates:

- Configuration.Linkwidth.Start
- Configuration.Linkwidth.Accept
- Configuration.Lanenum.Accept
- Configuration.Lanenum.Wait
- Configuration.Complete
- Configuration.Idle

`Configuration.Linkwidth.Start` is the first substate of the main Configuration state.

---

# Configuration.Linkwidth.Start

## Purpose

`Configuration.Linkwidth.Start` begins the Link-width negotiation process between the two PCIe link partners.

Its purpose is to establish the candidate logical Link grouping using the physical Lanes that successfully detected Receivers during Detect.

During this substate, the Downstream and Upstream Ports exchange TS1 Ordered Sets to identify the Link grouping that can be formed.

The substate establishes the initial Link Number selection and determines which individual physical Lanes have satisfied the required TS1 reception conditions.

The major responsibilities include:

- Identifying the active physical Lanes.
- Initiating Link identification from the Downstream Port.
- Receiving and evaluating Link identification information at the Upstream Port.
- Selecting a Link Number at the Upstream Port.
- Responding with the selected Link Number on individually qualified Lanes.
- Allowing the Downstream Port to recognize and qualify the Upstream response.
- Determining whether the Ports can proceed to Configuration.Linkwidth.Accept.
- Handling alternative transitions such as Disabled and Loopback.
- Handling timeout and optional crosslink behavior.

## Scope of Current Learning

There are two different situations in which a Port can enter Configuration.Linkwidth.Start:

- Initial Link establishment.
- Reconfiguration or Link-width upconfiguration of an existing Link.

The current learning scope is restricted to:

**Initial Link establishment — LinkUp = 0b**

In this situation:

- The PCIe Link has not yet been established.
- The LTSSM has entered Configuration through the normal Detect → Polling → Configuration sequence.
- Link-width negotiation is being performed for the first time.
- The physical Lanes detected during Detect are the candidates for the initial Link configuration.

The LinkUp value is a Link status indication. It is not a software instruction used to initiate initial Link training.

The separate LinkUp = 1b behavior for an already-configured Link will be studied later.

## Working Topology

The learning example uses one PCIe Link between a Root Port and a PCIe Switch.

```text
           Root Complex
                |
                |
           Root Port
         Downstream Port
                |
          PCIe Link
                |
         Four Physical Lanes
                |
       ---------------------
       Lane 0
       Lane 1
       Lane 2
       Lane 3
       ---------------------
                |
        Switch Upstream Port
                |
           PCIe Switch
```

Assumptions for the current example:

- The Root Port is the Downstream Port.
- The Switch Upstream Port is the Upstream Port.
- Four physical Lanes are available.
- Receiver Detection succeeded on all four Lanes at both ends.
- The intended Link is one x4 Link.
- The Link is undergoing initial establishment.
- LinkUp = 0b.
- Optional crosslink behavior is not supported in this normal example.
- No higher-layer instruction to enter Disabled or Loopback is present.

The two Ports have independent LTSSMs.

The Configuration process of this Root Port-to-Switch Upstream Port Link does not configure the Switch's other PCIe Links.

For example, a Switch Downstream Port connected to an NVMe SSD has a separate PCIe Link and its own independent LTSSM.

## **Understanding Link Number and Lane Number**

The TS1 and TS2 Ordered Sets contain fields used to establish the logical identity of physical Lanes.

Two important fields are:

### **Link Number**

The Link Number identifies the logical PCIe Link to which a group of physical Lanes belongs.

For example:

- A group of four physical Lanes may form one x4 Link.
- Another group of four physical Lanes may form a separate x4 Link.

Different Link Numbers can be used to identify different groups of Lanes that are capable of forming unique Links.

### **Lane Number**

The Lane Number identifies the logical position of a physical Lane within the configured PCIe Link.

For example, a successfully configured x4 Link eventually has four logical Lane Numbers:

- Lane Number 0
- Lane Number 1
- Lane Number 2
- Lane Number 3

The logical Lane Numbers have not yet been assigned during Configuration.Linkwidth.Start.

The Lane Number field remains PAD during the initial Linkwidth.Start TS1 exchanges.

The detailed assignment process is performed in subsequent Configuration substates.

### **Meaning of PAD**

PAD is a placeholder used during Link initialization.

It indicates that the corresponding Link or Lane identification field has not yet been assigned in the training sequence.

During Configuration.Linkwidth.Start:

- The Downstream Port begins advertising a non-PAD Link Number.
- The Upstream Port initially transmits PAD for both Link Number and Lane Number.
- The Lane Number field remains PAD during the normal Linkwidth.Start negotiation.

## **Entry into Configuration.Linkwidth.Start**

The normal initial entry path is:

```
Detect
   |
   v
Polling
   |
   v
Polling.Configuration
   |
   | Required TS2 receiver and transmitter conditions satisfied
   |
   v
Configuration.Linkwidth.Start
```

When the LTSSM enters this substate from Polling:

**Any Lane that detected a Receiver during Detect is considered an active Lane.**

This rule applies independently at the Downstream and Upstream Ports.

### **Meaning of an Active Lane**

An active Lane is a physical Lane that is considered for the initial Link configuration because it successfully detected a Receiver during Detect.

Receiver Detection establishes the presence of the expected far-end receiver termination.

However, an active Lane is not necessarily a successfully configured logical Lane.

At this point:

- Receiver Detection has identified the physical Lanes with receiver terminations.
- Polling has established the required initial training conditions.
- Configuration.Linkwidth.Start begins the logical Link identification process.
- The final configured Link width has not yet been established.
- The final logical Lane Numbers have not yet been assigned.

### **Example — Four Receiver-Detected Lanes**

Suppose the Root Port detects Receivers on all four physical Lanes.

| Physical Lane | Receiver Detection | Status on Entry |
| ------------- | ------------------ | --------------- |
| Lane 0        | Successful         | Active          |
| Lane 1        | Successful         | Active          |
| Lane 2        | Successful         | Active          |
| Lane 3        | Successful         | Active          |

All four Lanes are considered active candidates for the intended x4 Link.

**Important distinction:**

- Physical Lane availability.
- Successful Receiver Detection.
- Active Lane status.
- Logical Link configuration.

These are related but different concepts.

A physical x4 connection does not automatically guarantee that all four Lanes will become part of the final configured x4 Link.

### **Recovery Entry — Deferred**

When the LTSSM enters Configuration.Linkwidth.Start from Recovery, the active-Lane rule is different.

The Lanes that were part of the previously configured Link during the previous Configuration.Complete are considered active.

This belongs to the existing-Link reconfiguration path and is outside the current initial LinkUp = 0b learning scope.

# **Initial TS1 Transmission by the Downstream and Upstream Ports**

Both Ports enter Configuration.Linkwidth.Start, but their initial TS1 transmission fields are different.

The Downstream Port initiates Link identification.

The Upstream Port initially transmits PAD for both Link Number and Lane Number.

Both Ports independently transmit and receive TS1 Ordered Sets while their respective LTSSMs evaluate the incoming training information.

## **Downstream Port — Root Port**

The Root Port is the Downstream Port in the current example.

When it enters Configuration.Linkwidth.Start with LinkUp = 0b, it transmits TS1 Ordered Sets on all active Downstream Lanes with:

| TS1 Field   | Value                        |
| ----------- | ---------------------------- |
| Link Number | Selected non-PAD Link Number |
| Lane Number | PAD                          |

Assume the selected Link Number is `N`.

Because the example contains one intended x4 Link, all four active Downstream Lanes transmit the same Link Number.

### **Example — Downstream TS1 Transmission**

| Physical Lane | Link Number | Lane Number |
| ------------- | ----------- | ----------- |
| Lane 0        | N           | PAD         |
| Lane 1        | N           | PAD         |
| Lane 2        | N           | PAD         |
| Lane 3        | N           | PAD         |

The Root Port is proposing the Link identity that should be used by the physical Lanes forming this logical Link.

The Lane Number remains PAD because logical Lane Number assignment has not yet occurred.

### **Multiple Link Support**

A Downstream Port may support multiple possible Link groupings.

For example, consider eight Downstream Lanes that can form:

- One x8 Link when connected to one component.
- Two independent x4 Links when connected to two different components.

An example Link Number assignment is:

| Physical Lane Group | Link Number | Lane Number |
| ------------------- | ----------- | ----------- |
| First four Lanes    | N           | PAD         |
| Next four Lanes     | N+1         | PAD         |

Different Link Numbers are used only for groups of Lanes capable of forming unique Links.

This allows the Downstream Port to advertise its possible logical Link groupings.

For the current x4 example, all four Lanes use the same Link Number `N`.

### **Data Rate Identifier**

The TS1 Ordered Sets contain a Data Rate Identifier.

The Data Rate Identifier must advertise all data rates supported by the Port, including rates that the Port does not intend to use.

For example, a PCIe Gen5 Port may advertise support for:

- 2.5 GT/s
- 5.0 GT/s
- 8.0 GT/s
- 16.0 GT/s
- 32.0 GT/s

However, advertising support for 32.0 GT/s does not mean that the Link is currently operating at 32.0 GT/s.

During initial PCIe 5.0 Link establishment, the actual training rate remains 2.5 GT/s.

## **Upstream Port — Switch**

The Switch Upstream Port is the Upstream Port in our example.

When it enters Configuration.Linkwidth.Start from Polling, its transmitter initially sends TS1 Ordered Sets on all active Upstream Lanes with:

| TS1 Field   | Initial Value |
| ----------- | ------------- |
| Link Number | PAD           |
| Lane Number | PAD           |

### **Example — Upstream Initial TS1 Transmission**

| Physical Lane | Link Number | Lane Number |
| ------------- | ----------- | ----------- |
| Lane 0        | PAD         | PAD         |
| Lane 1        | PAD         | PAD         |
| Lane 2        | PAD         | PAD         |
| Lane 3        | PAD         | PAD         |

At this point, the Upstream Port has not yet selected the Link Number for the logical Link.

Its receiving circuitry simultaneously examines the TS1 Ordered Sets arriving from the Downstream Port.

The Upstream Port will select the Link Number based on the qualifying TS1 information it receives.

# **Upstream Port — TS1 Reception and Lane Qualification**

## **Purpose**

The Upstream Port must determine which of its receiving Lanes have received valid Link identification information from the Downstream Port.

It examines the incoming TS1 Ordered Sets independently on each receiving Lane.

The receiver does not combine TS1 observations from different Lanes to satisfy the two-consecutive-TS1 requirement.

## **Required TS1 Conditions**

The Upstream Port must receive two consecutive TS1 Ordered Sets on a receiving Lane with:

| TS1 Field   | Required Value |
| ----------- | -------------- |
| Link Number | Non-PAD        |
| Lane Number | PAD            |

The two qualifying TS1 Ordered Sets must be received consecutively on the same Lane.

At least one Lane must satisfy this condition for the normal Upstream Port transition.

### **Example — Per-Lane TS1 Reception**

Assume the Root Port is transmitting Link Number `N` and Lane Number PAD.

The Switch Upstream Port receives the following TS1 Ordered Sets:

| Upstream RX Lane | TS1 #1    | TS1 #2    | Qualification |
| ---------------- | --------- | --------- | ------------- |
| Lane 0           | N / PAD   | N / PAD   | Qualified     |
| Lane 1           | N / PAD   | PAD / PAD | Not qualified |
| Lane 2           | N / PAD   | N / PAD   | Qualified     |
| Lane 3           | PAD / PAD | PAD / PAD | Not qualified |

In this example:

- Lane 0 received two consecutive qualifying TS1 Ordered Sets.
- Lane 2 independently received two consecutive qualifying TS1 Ordered Sets.
- Lane 1 did not receive two consecutive qualifying TS1 Ordered Sets.
- Lane 3 received only PAD/PAD TS1 Ordered Sets.

The Upstream Port's receiver evaluates each Lane independently.

### **Important — One Lane Is Sufficient**

The Upstream Port does not require every active receiving Lane to satisfy the two-consecutive-TS1 condition.

If any one receiving Lane satisfies the condition, the normal Link Number selection procedure can proceed.

For example:

- Lane 0 qualifies.
- Lane 1 does not qualify.
- Lane 2 does not qualify.
- Lane 3 does not qualify.

The normal transition condition is still satisfied because Lane 0 received two consecutive qualifying TS1 Ordered Sets.

### **Why Two Consecutive TS1 Ordered Sets?**

A single TS1 Ordered Set is one observation of the Link identification information.

The receiver must recognize the required information in two consecutive TS1 Ordered Sets before the specified transition condition is satisfied.

The two-consecutive-sequence requirement is part of the PCIe LTSSM transition rules.

A qualifying TS1 received on Lane 0 cannot be combined with another qualifying TS1 received on Lane 2 to create a valid two-sequence pair.

Each receiving Lane must satisfy the condition independently.

# **Upstream Port — Link Number Selection and Transmitter Response**

## **Link Number Selection**

After any receiving Lane receives two consecutive qualifying TS1 Ordered Sets with:

- Link Number different from PAD.
- Lane Number equal to PAD.

The Upstream Port selects a single Link Number.

For the current example, assume the Root Port transmitted Link Number `N`.

The Upstream Port therefore selects:

`Selected Link Number = N`

This is a Port-level Link Number selection.

The Upstream Port does not select a different Link Number for Lane 0 and Lane 2.

It selects one Link Number for the Link being configured.

## **Upstream Transmitter Response**

After selecting the Link Number, the Upstream Port transmits TS1 Ordered Sets with:

| TS1 Field   | Transmitted Value            |
| ----------- | ---------------------------- |
| Link Number | Selected non-PAD Link Number |
| Lane Number | PAD                          |

However, it does not transmit the selected Link Number on every active Lane automatically.

The transmission rule is evaluated independently for each Lane.

The Upstream transmitter sends the selected Link Number with Lane Number PAD on every Lane satisfying both conditions:

1. The Lane detected a Receiver during Detect.
2. The same Lane received two consecutive qualifying TS1 Ordered Sets with a non-PAD Link Number and Lane Number PAD.

Any remaining Lanes that detected a Receiver during Detect continue transmitting TS1 Ordered Sets with both Link Number and Lane Number set to PAD.

### **Example — Upstream Transmitter Response**

Assume:

- All four Upstream Lanes detected Receivers during Detect.
- RX Lane 0 received two consecutive qualifying TS1 Ordered Sets.
- RX Lane 2 received two consecutive qualifying TS1 Ordered Sets.
- RX Lane 1 and RX Lane 3 did not qualify.
- The selected Link Number is `N`.

The Upstream transmitter responds as follows.

| Upstream Lane | Receiver Qualification | TX Link Number | TX Lane Number |
| ------------- | ---------------------- | -------------- | -------------- |
| Lane 0        | Qualified              | N              | PAD            |
| Lane 1        | Not qualified          | PAD            | PAD            |
| Lane 2        | Qualified              | N              | PAD            |
| Lane 3        | Not qualified          | PAD            | PAD            |

Only Lane 0 and Lane 2 transmit the selected Link Number in this example.

Lane 1 and Lane 3 continue transmitting PAD/PAD.

### **Important — Port-Level Transition Versus Per-Lane Qualification**

The Upstream Port has one LTSSM for this Link.

Individual Lanes are evaluated separately for their TS1 reception conditions, but they do not each have an independent Configuration.Linkwidth.Start state machine.

Once the Upstream Port satisfies the normal transition condition and performs the required Link Number response:

`Configuration.Linkwidth.Start → Configuration.Linkwidth.Accept`

The entire Upstream Port's LTSSM transitions to the next substate.

It does not wait for every remaining active Lane to qualify.

Therefore:

- Lane 0 and Lane 2 can qualify.
- Lane 1 and Lane 3 can still be transmitting PAD/PAD.
- The Upstream Port's LTSSM can nevertheless transition to Configuration.Linkwidth.Accept.

The remaining Lanes do not stay behind in Configuration.Linkwidth.Start with separate 24 ms timers.

They are subsequently handled by the Port's Linkwidth.Accept behavior.

### **Important Distinction**

A Lane can be:

- Active because it detected a Receiver during Detect.
- Not yet qualified for the selected-Link-Number response.
- Not necessarily part of the final configured Link.

Active Lane status and final configured Link membership are different concepts.

The final Link width is not determined merely by the number of physical Lanes that detected Receivers during Detect.

# **Downstream Port — TS1 Reception and Qualification**

## **Purpose**

After the Upstream Port responds with its selected Link Number, the Downstream Port must receive and recognize that response.

The Root Port is the Downstream Port in the current example.

Its receiver examines the TS1 Ordered Sets transmitted by the Switch Upstream Port.

## **Required TS1 Conditions**

The Downstream Port must receive two consecutive TS1 Ordered Sets on the same applicable receiving Lane with:

| TS1 Field   | Required Value |
| ----------- | -------------- |
| Link Number | Non-PAD        |
| Lane Number | PAD            |

There is one additional requirement concerning the initial PAD/PAD observation.

### **Initial PAD/PAD Observation Condition**

If any Downstream Lanes first received one or more TS1 Ordered Sets with both:

- Link Number = PAD.
- Lane Number = PAD.

Then the Downstream Port transitions to Configuration.Linkwidth.Accept immediately after any of those same Downstream Lanes receives two consecutive TS1 Ordered Sets with:

- A non-PAD Link Number.
- A Link Number matching one of the Link Numbers transmitted by the Downstream Port.
- Lane Number = PAD.

The required two consecutive TS1 Ordered Sets must be received on the same applicable Lane.

### **Crosslink Configuration Not Supported**

The specification provides an important simplification.

If crosslink configuration is not supported, the condition of first receiving a TS1 Ordered Set with both Link Number and Lane Number set to PAD is always considered true.

Therefore, in the current normal example, we do not need to separately establish that initial PAD/PAD reception condition.

The normal Downstream receiving requirement is satisfied when any applicable receiving Lane receives two consecutive TS1 Ordered Sets with:

- A non-PAD Link Number matching one of the Downstream Port's transmitted Link Numbers.
- Lane Number = PAD.

## **Example — Downstream Reception**

Assume the Switch Upstream Port has selected Link Number `N`.

It transmits:

| Upstream TX Lane | Link Number | Lane Number |
| ---------------- | ----------- | ----------- |
| Lane 0           | N           | PAD         |
| Lane 1           | PAD         | PAD         |
| Lane 2           | N           | PAD         |
| Lane 3           | PAD         | PAD         |

Assume straightforward physical Lane correspondence for this illustration.

The Root Port receives:

| Downstream RX Lane | TS1 #1    | TS1 #2    | Qualification |
| ------------------ | --------- | --------- | ------------- |
| Lane 0             | N / PAD   | N / PAD   | Qualified     |
| Lane 1             | PAD / PAD | PAD / PAD | Not qualified |
| Lane 2             | N / PAD   | N / PAD   | Qualified     |
| Lane 3             | PAD / PAD | PAD / PAD | Not qualified |

Lane 0 and Lane 2 independently satisfy the normal TS1 reception condition.

The Downstream Port does not require all four receiving Lanes to qualify.

One qualifying receiving Lane is sufficient for the normal transition when crosslink configuration is not supported.

### **Link Number Matching Requirement**

The Downstream Port must verify that the received non-PAD Link Number matches one of the Link Numbers it transmitted.

For example:

- Downstream transmitted Link Number `N`.
- Downstream receives Link Number `N` with Lane Number PAD.
- The received Link Number matches the transmitted Link Number.

The received TS1 information therefore satisfies the Link Number matching requirement.

A different non-PAD Link Number that does not match one of the transmitted Link Numbers does not satisfy this condition.

## **Downstream Normal Transition**

When the applicable receiving Lane satisfies the two-consecutive-TS1 condition:

`Configuration.Linkwidth.Start → Configuration.Linkwidth.Accept`

The transition can occur without waiting for all active Lanes to qualify.

# **Independent LTSSM Operation at Both Ports**

The Downstream and Upstream Ports have independent LTSSMs.

Each Port independently evaluates:

- Its own transmitted TS1 Ordered Sets.
- Its own received TS1 Ordered Sets.
- Its own Lane qualification conditions.
- Its own normal transition condition.
- Its own timeout.

Consider the two connected Ports.

```
       Root Port                         PCIe Switch
     Downstream Port                   Upstream Port

         RP_TX  ---------------------->  SW_RX

         RP_RX  <----------------------  SW_TX
```

The Root Port's receiving and transmitting operations are independent of the Switch's receiving and transmitting operations.

### **Upstream Port Transition**

The Upstream Port evaluates the TS1 Ordered Sets received from the Root Port.

When its normal condition is satisfied, it selects a Link Number, performs its qualifying-Lane response and transitions to Configuration.Linkwidth.Accept.

### **Downstream Port Transition**

The Downstream Port evaluates the TS1 Ordered Sets received from the Switch.

When its normal condition is satisfied, it recognizes the matching Link Number and transitions to Configuration.Linkwidth.Accept.

### **Transitions Need Not Be Simultaneous**

One Port may satisfy its transition condition before the other Port.

For example:

- The Upstream Port may receive its qualifying TS1 pair first.
- The Upstream Port may select the Link Number and enter Configuration.Linkwidth.Accept.
- The Downstream Port may still be evaluating its received TS1 information.
- The Downstream Port transitions independently when its own condition is fulfilled.

There is no requirement that both Ports transition at exactly the same instant.

# **Configuration.Linkwidth.Start — 24 ms Timeout**

## **Purpose**

The 24 ms timeout provides a limit for the Port to make the required progress through Configuration.Linkwidth.Start.

The normal transition should occur when the required TS1 conditions are satisfied.

If the required normal transition condition is not satisfied within the 24 ms timeout, the Port has a timeout exit to Detect.

The 24 ms timeout applies independently to the Downstream and Upstream Ports.

## **Downstream Port — Timeout**

The Root Port is the Downstream Port.

Its normal transition requires the appropriate receiving Lane to receive two consecutive TS1 Ordered Sets containing:

- A non-PAD Link Number matching one of the transmitted Link Numbers.
- Lane Number = PAD.

The initial PAD/PAD observation requirement is considered satisfied automatically when crosslink configuration is not supported.

If the required normal transition condition is not fulfilled within 24 ms:

`Configuration.Linkwidth.Start → Detect`

### **Example — Downstream Timeout**

Assume the Root Port transmits:

`TS1: Link Number = N, Lane Number = PAD`

However, it continuously receives TS1 Ordered Sets from the Upstream Port with:

`TS1: Link Number = PAD, Lane Number = PAD`

The Root Port never receives two consecutive qualifying TS1 Ordered Sets with the required matching non-PAD Link Number.

When the 24 ms timeout expires:

`Configuration.Linkwidth.Start → Detect`

The Port returns to the Detect state machine.

The timeout does not mean that the Link has been successfully configured at a narrower width.

## **Upstream Port — Timeout**

The Switch Upstream Port initially transmits:

`TS1: Link Number = PAD, Lane Number = PAD`

Its normal transition requires at least one receiving Lane to receive two consecutive TS1 Ordered Sets with:

- Link Number different from PAD.
- Lane Number = PAD.

If no receiving Lane satisfies this condition within 24 ms:

`Configuration.Linkwidth.Start → Detect`

### **Example — Upstream Timeout**

Assume the Upstream Port continuously receives:

`TS1: Link Number = PAD, Lane Number = PAD`

It never receives two consecutive qualifying TS1 Ordered Sets with a non-PAD Link Number and Lane Number PAD.

The Upstream Port cannot perform the normal Link Number selection procedure.

When the 24 ms timeout expires:

`Configuration.Linkwidth.Start → Detect`

## **Independent Timeout Behavior**

Both Ports have independent 24 ms timeout behavior.

| Port       | Normal Transition Condition                                                                                                    | Timeout Destination |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------ | ------------------- |
| Downstream | Receives two consecutive TS1 with a matching non-PAD Link Number and Lane Number PAD, subject to the initial PAD/PAD condition | Detect              |
| Upstream   | Receives two consecutive TS1 with a non-PAD Link Number and Lane Number PAD on any one receiving Lane                          | Detect              |

The timeout of one Port does not require the other Port to transition at precisely the same instant.

### **Important — 24 ms Is Not a Per-Lane Timeout**

The 24 ms timeout is associated with the Port's progress through Configuration.Linkwidth.Start.

It is not a separate 24 ms allowance for each physical Lane.

For example:

- Upstream Lane 0 qualifies.
- Upstream Lane 2 qualifies.
- Upstream Lane 1 does not qualify.
- Upstream Lane 3 does not qualify.

The Upstream Port can satisfy its normal transition condition and move to Configuration.Linkwidth.Accept.

It does not wait for Lane 1 and Lane 3 to qualify within separate 24 ms periods.

The remaining Lanes do not have separate LTSSMs that remain in Configuration.Linkwidth.Start.

### **Timeout Does Not Identify the Root Cause**

A timeout indicates that the required transition condition was not satisfied within the specified period.

It does not independently identify whether the issue is caused by:

- Physical Lane connectivity.
- Receiver behavior.
- Incorrect TS1 transmission.
- Incorrect Link Number information.
- Lane-specific reception errors.
- Link partner behavior.
- Platform or hardware configuration.

The root cause must be established through further investigation.

# **Configuration.Linkwidth.Start — Alternative Transition to Disabled**

## **Purpose**

The Disabled alternative transition provides a mechanism to deliberately prevent a PCIe Link from becoming operational.

A Port may successfully complete:

- Receiver Detection.
- Initial TS1 training.
- Polling.Configuration.

However, successful completion of these stages does not require the Link to become operational if the higher layer instructs the Port to disable the Link.

The Disabled transition is an intentional control operation.

It must not be confused with a Link-training failure or a 24 ms timeout.

## **Why Can Disabled Be Entered During Configuration.Linkwidth.Start?**

Configuration.Linkwidth.Start is the stage where the Ports begin forming the logical PCIe Link.

The higher layer can direct the Physical Layer not to proceed with normal Link establishment.

For example, platform software or firmware may intentionally disable a Root Port.

The physical connection may exist and Receiver Detection may have succeeded, but the platform does not want that PCIe Link to become operational.

The LTSSM must follow the higher-layer instruction.

Therefore, Disabled is an alternative exit from Configuration.Linkwidth.Start.

## **Downstream Port — Higher-Layer-Directed Disabled Transition**

The Root Port is the Downstream Port in our example.

When instructed by a higher layer to disable the Link, the Downstream Port must assert the Disable Link bit in:

- TS1 Ordered Sets.
- TS2 Ordered Sets.

It must assert this bit on all Lanes that detected a Receiver during Detect.

The next state is:

`Configuration.Linkwidth.Start → Disabled`

### **Example — Four Receiver-Detected Lanes**

| Physical Lane | Receiver Detected During Detect | Disable Link in TS1 | Disable Link in TS2 |
| ------------- | ------------------------------- | ------------------- | ------------------- |
| Lane 0        | Yes                             | Asserted            | Asserted            |
| Lane 1        | Yes                             | Asserted            | Asserted            |
| Lane 2        | Yes                             | Asserted            | Asserted            |
| Lane 3        | Yes                             | Asserted            | Asserted            |

All four receiver-detected Downstream Lanes carry the Disable Link indication.

### **Understanding the Higher-Layer Instruction**

The higher-layer instruction is a local control decision.

It is not itself a TS1 Ordered Set received from the other Port.

For example, software can request that a Root Port disable its Link through the Link Disable field in the Link Control Register.

The Port's hardware then performs the required LTSSM operation.

The distinction is:

- Higher-layer instruction: Local control request to disable the Link.
- Disable Link bit in TS1 and TS2: Control indication transmitted over the PCIe Link.

### **Important**

The Downstream Port does not enter Disabled because:

- A Lane failed Receiver Detection.
- The Link Number was not received.
- The normal Link-width negotiation failed.
- The 24 ms timeout expired.

It enters Disabled because the higher layer deliberately instructed it to disable the Link.

## **Upstream Port — Receiving Disable Link**

The Switch Upstream Port must recognize the Disable Link indication transmitted by the Downstream Port.

The Upstream Port receives the Root Port's TS1 Ordered Sets.

During normal non-crosslink operation, its transition condition is:

**Any Lane that is transmitting TS1 Ordered Sets receives two consecutive TS1 Ordered Sets with the Disable Link bit asserted.**

The Upstream Port then transitions to:

`Configuration.Linkwidth.Start → Disabled`

### **Example — One Lane Receives Disable Link**

Assume the Upstream Port is transmitting TS1 Ordered Sets on all four active Lanes.

Its Lane 0 receiver receives:

| Received Ordered Set | Disable Link Bit |
| -------------------- | ---------------- |
| TS1 #1               | Asserted         |
| TS1 #2               | Asserted         |

Lane 0 has received two consecutive TS1 Ordered Sets with Disable Link asserted.

This is sufficient for the normal non-crosslink Upstream Port transition.

The Upstream Port's LTSSM transitions to Disabled.

It does not require all four active Lanes to receive the Disable Link indication in the normal non-crosslink case.

### **Important — Two Consecutive TS1 Ordered Sets**

The Upstream Port's normal received Disable Link condition requires two consecutive TS1 Ordered Sets.

The receiver must not count a TS2 Ordered Set as one of the two required TS1 Ordered Sets.

### **Crosslink Exception — Deferred**

When optional crosslink support is used, the Upstream Port has a stricter Disable Link reception condition.

All Lanes that are both transmitting and receiving TS1 Ordered Sets must receive the Disable Link bit asserted in two consecutive TS1 Ordered Sets.

This optional crosslink-specific behavior will be studied separately.

# **Configuration.Linkwidth.Start — Normal and Completed Alternative Paths**

## **Normal Link Establishment**

The normal Link-establishment procedure is:

```
        Configuration.Linkwidth.Start
                    |
                    |
        Downstream sends TS1:
             Link N / PAD
                    |
                    |
          Upstream initially sends:
               PAD / PAD
                    |
                    |
          Upstream RX evaluates
          TS1 independently per Lane
                    |
                    |
          Any Lane receives two
          consecutive qualifying TS1
                    |
                    |
          Upstream selects Link N
                    |
                    |
          Upstream TX sends:
          N / PAD on qualified Lanes
          PAD / PAD on remaining
          receiver-detected Lanes
                    |
                    |
          Upstream enters
          Configuration.Linkwidth.Accept
```

The Downstream Port independently receives and evaluates the Upstream Port's response.

When its own required condition is satisfied:

`Configuration.Linkwidth.Start → Configuration.Linkwidth.Accept`

## **Alternative Transition — Downstream Directed Disable**

```
Configuration.Linkwidth.Start
             |
             |
Higher layer directs Link disable
             |
             |
Downstream asserts Disable Link
in TS1 and TS2 on all receiver-
detected Lanes from Detect
             |
             v
          Disabled
```

## **Alternative Transition — Upstream Received Disable**

```
Configuration.Linkwidth.Start
             |
             |
Upstream receives two consecutive
TS1 Ordered Sets with Disable Link
asserted on a qualifying Lane
             |
             v
          Disabled
```

## **Timeout Exit**

For either Port:

```
Configuration.Linkwidth.Start
             |
             |
Required normal transition condition
not fulfilled within 24 ms
             |
             v
            Detect
```

# **Configuration.Linkwidth.Start — Enterprise NVMe SSD Validation Relevance**

An Enterprise NVMe SSD Validation Engineer must be able to identify and investigate Link-width negotiation failures at both the Root Port and the SSD Endpoint.

The validation engineer must understand the independent transmission and reception behavior at both ends of the PCIe Link.

## **TS1 Transmission Verification**

Verify the initial TS1 transmission fields.

### **Downstream Port**

- Verify that the Downstream Port transmits a selected non-PAD Link Number.
- Verify that Lane Number is PAD.
- Verify that the appropriate active Downstream Lanes transmit TS1.
- Verify the Data Rate Identifier advertises all supported data rates.

### **Upstream Port**

- Verify that the Upstream Port initially transmits Link Number PAD and Lane Number PAD.
- Verify that the appropriate active Upstream Lanes transmit TS1.
- Verify that its transmitter changes to the selected Link Number only on Lanes satisfying the specified receiver qualification conditions.

## **Receiver Qualification**

Examine TS1 reception independently for every physical Lane.

Verify:

- The Link Number field.
- The Lane Number field.
- Whether the Link Number is PAD or non-PAD.
- Whether the Lane Number is PAD.
- Whether two consecutive qualifying TS1 Ordered Sets are received on the same Lane.
- Whether the received Link Number matches one transmitted by the Downstream Port.

Do not combine qualifying TS1 observations from different Lanes.

## **Partial Lane Qualification**

Investigate scenarios where:

- All physical Lanes detected Receivers during Detect.
- Only a subset of Lanes receives two consecutive qualifying TS1 Ordered Sets.
- The Upstream Port responds with a selected Link Number on the qualified Lanes.
- Remaining receiver-detected Lanes continue transmitting PAD/PAD.
- The Port transitions to Configuration.Linkwidth.Accept without waiting for all active Lanes to qualify.

The final configured Link width must be evaluated in subsequent Configuration substates.

## **Timeout Investigation**

If a Port returns to Detect after the 24 ms timeout, collect evidence for:

- Downstream TS1 transmission.
- Upstream TS1 transmission.
- Received TS1 fields on every Lane.
- Missing or incorrect Link Number responses.
- Lane-specific reception behavior.
- The independent transition behavior of both Ports.

The timeout alone does not identify the root cause.

## **Disabled Transition Investigation**

When a Link enters Disabled during initial Configuration, determine whether:

- A higher layer intentionally directed the Downstream Port to disable the Link.
- The Downstream Port asserted Disable Link in its TS1 and TS2 Ordered Sets.
- The Upstream Port received two consecutive TS1 Ordered Sets with Disable Link asserted.
- The Disabled transition was intentional or unexpected.

A protocol trace can help distinguish a deliberate disable request from a Link-training timeout.

# **Configuration.Linkwidth.Start — Essential Understanding**

The important engineering capability is to explain:

- Why Configuration.Linkwidth.Start exists.
- Why initial Link establishment uses LinkUp = 0b.
- Which physical Lanes become active when entering from Polling.
- Why active Lane status does not guarantee final Link membership.
- Why the Downstream Port initially sends a non-PAD Link Number and Lane Number PAD.
- Why the Upstream Port initially sends PAD/PAD.
- Why the Downstream and Upstream Ports have different responsibilities.
- Why Link Number and Lane Number are separate concepts.
- Why the Upstream Port evaluates TS1 reception independently on each Lane.
- Why two consecutive qualifying TS1 Ordered Sets must be received on the same Lane.
- Why one qualifying Lane is sufficient for the normal Upstream Port transition.
- How the Upstream Port selects a single Link Number.
- Why only individually qualified Upstream Lanes transmit the selected Link Number.
- Why the remaining receiver-detected Lanes continue transmitting PAD/PAD.
- Why the entire Port LTSSM transitions to Configuration.Linkwidth.Accept without waiting for every active Lane to qualify.
- How the Downstream Port independently receives and qualifies the Upstream response.
- Why the Downstream Port checks that the received Link Number matches one it transmitted.
- Why the Downstream Port has an additional conditional initial PAD/PAD reception requirement.
- Why that initial PAD/PAD condition is always considered true when crosslink configuration is not supported.
- Why each Port has an independent 24 ms timeout.
- Why the timeout is not a separate waiting period for every physical Lane.
- Why a timeout leads to Detect rather than successful Link configuration.
- Why the Disabled alternative transition exists.
- How a higher-layer instruction directs the Downstream Port to disable the Link.
- How the Disable Link indication is transmitted in TS1 and TS2.
- How the Upstream Port recognizes the Disable Link indication through two consecutive TS1 Ordered Sets.
- Why Disabled is an intentional control transition rather than automatically a training failure.



