# 05 – Test specification: CSR-SYS-01

**Status:** specification only — **not executed**. No bench available for this study. No result, measurement or verdict is reported anywhere in this repository.
**Traces to:** CSR-SYS-01, CSR-SW-01, and CSR-SYS-04 / CSR-SW-04 (§8) — `04_traceability-matrix.csv`
**Scope note:** the file is named after the primary requirement; it also carries the assumption-qualification step (§4, step 0) and the CSG-04 case (§8), both belonging to the same chain.

## 1. Chain under test

| Element | ID | Statement |
|---|---|---|
| Threat scenario | TS-01 | Replay of a captured authentication code that the vehicle never received |
| Damage scenario | DS-01 | Parked vehicle opened by an unauthorized person |
| Assumption | A-10 | The command field is not covered by the authentication code |
| Goal (bounds) | CSG-01 | Reject a code whose counter is not strictly greater than the last accepted counter |
| Goal (closes) | CSG-04 | The RKE system shall not act on a radio command whose command field is not covered by the authentication code computed by the key fob |
| Requirements | CSR-SYS-01, CSR-SW-01, CSR-SYS-04, CSR-SW-04 | Reject a stale counter; update the stored counter only after the code is verified; cover the command field |
| Residual | RES-01 | What survives both goals — see `03` |

## 2. Objective

Show that a code the vehicle never received is **rejected once a legitimate press has advanced the stored counter**, and characterise the window during which it is still accepted (RES-01).

This test verifies the **threat**, not only the goal: replaying a code the vehicle has already accepted would pass trivially and prove nothing about TS-01.

## 3. Setup

| Item | Detail |
|---|---|
| Device under test | Key fob (C-01) + BCM RF receiver (C-02/C-03) on bench |
| Coupling | **Conducted**: jammer signal and replayed frames injected into the receiver antenna port through a coupler/attenuator. No radiated emission |
| Transmit | Transmit-capable SDR (HackRF One class) for jamming and replay |
| Receive | RTL-SDR for capture |
| Analysis | Universal Radio Hacker (demodulation, frame extraction, command-field edit) |
| Instrumentation | Bench read of the stored counter; actuator current monitored to observe any lock/unlock movement |

**Why conducted coupling.** Two reasons, in this order. First, regulatory: the use of jamming devices is restricted in France, and spectrum enforcement sits with the ANFR — conducted injection emits nothing and raises no spectrum-use question. Second, repeatability: the jamming level becomes reproducible to the dB, which a radiated setup is not. A shielded enclosure is required only for whole-vehicle validation, where conducted injection is not possible.

## 4. Procedure

| Step | Action | Expected |
|---|---|---|
| 0 | **Assumption qualification** — characterisation, not a conformity verdict. Capture a lock frame and replay it with only the command field changed | **Accepted** → A-10 holds and the sequence below applies as written. **Rejected** → A-10 is invalid, and TS-01 must be requalified before this test runs |
| 1 | Read the initial stored counter | Baseline value recorded |
| 2 | Owner press #1 (lock) — jam the band and capture the frame | Vehicle does not react; stored counter unchanged |
| 3 | Owner press #2 (lock) — jam and capture again, then replay the frame captured at step 2 | Vehicle **locks**; stored counter advances to the step-2 value |
| 4 | Replay the frame captured at step 3, command field switched to unlock | Vehicle **unlocks** — this is RES-01, the documented residual, not a failure |
| 5 | **Control case.** Repeat steps 1–3, then insert one legitimate press (received normally by the vehicle), then replay the frame captured at step 3 with its command field switched to unlock | Vehicle **rejects** the command — no actuator movement |
| 6 | **Positive control.** One legitimate fob press immediately after step 5 | Vehicle **actuates** — the output path is alive |
| 7 | Read the BCM diagnostic trace, where one exists | A negative entry for the command rejected at step 5 |

**Note on steps 4 and 5.** The command field is switched to unlock so that acceptance is observable on the actuator: a replayed lock on an already-locked vehicle produces no movement either way, which would make the step-5 criterion meaningless. The rejection at step 5 is attributable to the counter only because step 0 confirmed A-10.

Step 0 decides whether the line exists at all. Steps 1–4 characterise the residual. Steps 5 and 6 verify the requirement.

## 5. Pass criterion

Steps 5 and 6 both as expected:

- step 5: the replayed code produces **no** actuator movement on the lock/unlock output, **and**
- step 6: the legitimate press immediately after it **does** actuate.

Together they distinguish "command rejected" from "command accepted but actuator inoperative", with no dependency on a diagnostic trace. Step 3 must also execute normally (the vehicle locks), otherwise the setup is not exercising the scenario.

Step 7 is **supporting evidence, not part of the criterion**: no requirement in this repository mandates a diagnostic trace for rejected radio commands (CSR-SW-03 covers key-management operations only). The gap is recorded in `OPEN-POINTS.md`.

## 6. Repetitions

5 runs of the full sequence. **5/5 required on steps 5 and 6.** Any single acceptance at step 5, or any failure to actuate at step 6, fails the requirement.

## 7. Known limitations

- **Step 4 succeeding is the expected result**, not a defect: it documents RES-01. This specification characterises the attack window; it does not demonstrate its elimination.
- Bench only. Whole-vehicle behaviour (range, antenna placement, ambient interference) is not covered.
- The jamming level is set for reliable suppression at the receiver and is not swept. A sensitivity study of the suppression threshold is out of scope.
- The stored counter is read with bench instrumentation. No function of `01` §3 exposes a counter read, so this is not a service of the item; a diagnostic read of the counter is an open point.
- Counter values read this way assume the read is trustworthy. Under A-07 that is not guaranteed on a unit whose debug path is open; the bench unit is assumed uncompromised.

## 8. Verification of CSG-04 — separate test case

**Applies to** an implementation where CSR-SYS-04 and CSR-SW-04 are in place. It is **not executable on the item as defined**, since A-10 states the command field is not covered.

Same manipulation as step 0: capture a frame and replay it with only the command field changed.

**Pass criterion:** the frame is **rejected**, and a legitimate press immediately after it actuates (positive control, as in §5).

**Why this is a distinct case and not step 0.** The same manipulation has opposite expected results depending on what it tests. On the item as defined, acceptance confirms A-10 and the scenario exists. On an implementation carrying CSR-SYS-04, acceptance is a non-conformity. A pass criterion that depends on what the operator believes is being tested is not a pass criterion.

**Status:** Planned. 5 runs, 5/5 required.