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
