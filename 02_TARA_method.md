# 03 – Goals & Requirements

**Scope:** cybersecurity goals and requirements derived from `02_tara.csv`
**Status:** fictional case study – generic passenger vehicle, no real OEM

**Coverage.** TS-01 is treated by *Reduce* with two goals: CSG-01, which refines a control the item already provides, and CSG-04, which introduces the one it does not; RES-01 carries what neither closes. TS-03 and TS-04 are treated by *Reduce* with CSG-02 (preventive, authorization) and CSG-03 (detective, accountability) — the same two goals cover both, since they differ only in the path by which the F-03 service is reached. TS-02 is treated by *Retain* and recorded under "Risk retention" (ACC-01). CSG-02 will also cover the revocation threats TS-05 / TS-06 once they are rated. The remaining lines get their treatment in a later version (see `OPEN-POINTS.md`).

A cybersecurity **goal** states *what* must be prevented (the effect), not *how*. The **concept** names the control and where it sits. The **requirements** (CSR) make the concept testable, allocated to system (SYS), software (SW) and hardware (HW). Requirement identifiers carry the number of the goal they belong to.

```
risk (TS) → goal (CSG) → concept → requirements (CSR-SYS / CSR-SW / CSR-HW) → verification
```

A risk treated by **Retain** produces no goal and no requirement — no control is built — so it is not given a CSG; it is recorded as a justified retention. Where a treatment leaves a part that no control on this item can reach, that part is recorded as a named residual (RES-xx) rather than left silent.

**Item controls vs derived controls.** Some controls are properties of the item as defined (`01` §3) and are therefore already in force when the initial risk is rated. A goal that restates one of them makes it verifiable; it does not create it. Each goal below says which case it is.

## CSG-01 — Reject a code whose counter has already been reached

**Source risk:** TS-01 — risk High — treatment = Reduce (together with CSG-04; RES-01 carries the remainder).

**Goal:** The BCM shall not unlock the vehicle or release the trunk on a radio command whose rolling-code counter is not strictly greater than the last accepted counter.

**Initial state vs goal.** The item already implements a counter validation (`01` §3, F-01), so CSG-01 does **not** create the control: it makes it verifiable — strictly greater, and the stored counter updated only after the code is verified. What it carries is specification precision, not a new barrier. The barrier that closes the damage path of TS-01 is CSG-04.

**Rationale.** A counter blocks a replay only if, between capture and replay, the stored counter has passed the captured value. Jamming prevents exactly that: the vehicle never received the frame, so its counter never moved, and the retained code stays valid whatever the width of the look-ahead window.

```
stored counter = n+1  (after the attacker replayed the first code)
attacker holds code n+2

without a legitimate press in between      with a legitimate press (n+3)
  replay n+2 : n+2 > n+1  ✓ accepted         fob emits n+3 → stored = n+3
  → bounded only in time                      replay n+2 : n+2 < n+3  ✗ REJECTED
```

So CSG-01 **bounds** TS-01: the retained code expires at the next legitimate press. What it eliminates outright is the replay of a code the vehicle *has* already accepted.

### Cybersecurity concept

Accept a radio command only if its rolling-code counter is strictly greater than the last accepted counter, within the F-05 look-ahead window. The stored counter is updated only after the authentication code is verified. The underlying principle — freshness, as opposed to origin authentication — is the same one SecOC (AUTOSAR Secure Onboard Communication) applies on in-vehicle buses; SecOC itself is not transposable here, as it assumes a bus and an AUTOSAR stack, where this link is a battery-powered fob on a one-way UHF channel. No specific product is mandated at this stage.

### Requirements

| req_id | Level | Requirement | Verification |
|---|---|---|---|
| CSR-SYS-01 | System | The RKE system shall reject a radio command whose counter is not strictly greater than the last accepted counter. | Test — see `05_test-spec-CSR-SYS-01.md` |
| CSR-SW-01 | Software | The BCM software shall verify that the received counter is strictly greater than the stored counter before executing the command, and update the stored counter only after the authentication code is verified (consistent with F-05). | Test — see `05_test-spec-CSR-SYS-01.md` |
| CSR-HW-01 | Hardware | The accepted counter shall be held in non-volatile memory (Flash/EEPROM) and protected against modification by other BCM software. | Review + test |

**Dependency note on CSR-HW-01:** protection against the debug path (I-06b) is not part of this requirement. It belongs to the debug-access threats, which are not rated in this version (see `OPEN-POINTS.md`). Under A-07 the counter remains writable through that path.

**Traceability:** TS-01 → CSG-01 → {CSR-SYS-01, CSR-SW-01, CSR-HW-01} → verification. Carried in `04_traceability-matrix.csv`.

## CSG-04 — Prevent a replayed frame from carrying a different command

**Source risk:** TS-01 — risk High — treatment = Reduce.

**Goal:** The RKE system shall not act on a radio command that differs from the one issued by the key fob whose frame is presented.

**Initial state vs goal.** A-10 states that the command field sits outside the authentication code. This goal is therefore the one control of this chain that the item does **not** already provide — the barrier proper, where CSG-01 only bounds the attack in time. The goal is stated at system level rather than BCM level because the authentication code is computed in the key fob and verified in the BCM: both components carry part of it.

**Rationale.** The frame the attacker retains is a lock command. What makes it harmful is that its command field can be changed before replay; bind the command to the authentication code and a retained frame can only repeat what the owner actually asked for. No hardware requirement is derived from this goal. Extending the authenticated payload costs a few bits in the frame, which is compatible with the lightweight-cryptography constraint of `01` §6 — unlike a bidirectional link, which that constraint rules out.

### Requirements

| req_id | Level | Requirement | Verification |
|---|---|---|---|
| CSR-SYS-04 | System | The command field shall be covered by the authentication code computed by the key fob (C-01) and verified by the BCM (C-03), so that a replayed frame cannot carry a command other than the one issued. | Test — see `05_test-spec-CSR-SYS-01.md` §8 |
| CSR-SW-04 | Software | The BCM software shall verify the authentication code over a payload that includes the command field before executing the command. | Test — see `05_test-spec-CSR-SYS-01.md` §8 |

**Traceability:** TS-01 → CSG-04 → {CSR-SYS-04, CSR-SW-04} → verification. Carried in `04_traceability-matrix.csv`.

### RES-01 — Residual risk: capture-and-delayed-replay

**What remains:** once CSG-04 is implemented, the replay of the owner's **own** command — a retained frame can only repeat what was issued — performed before the next legitimate press.

**Why the damage closes but the weakness does not.** Freshness cannot be restored on this link: the only reference is the stored counter, and jamming freezes it (one-way link, no real-time clock — `01` §6). Closing the gap would require either a receiver-generated nonce, hence a bidirectional link, or a time reference in the fob, which a coin-cell fob does not carry. What closes the *damage* is therefore not freshness but binding the command to the authentication code (CSG-04): the retained frame can then only repeat the command the owner actually issued.

The residual is what that leaves: a retained unlock produces damage only if the vehicle is re-secured between the capture and the replay by a means that does not advance the counter — the mechanical key (`01` A-08), the interior lock control or speed-based auto-lock, all outside the item boundary (`01` §4). Any radio press would advance the stored counter and invalidate the retained frame (CSG-01). The residual is therefore real but conditioned on an out-of-boundary behaviour, which is also why no requirement of this item can close it.

**Why no application-level detection is possible.** From the BCM's point of view the sequence is indistinguishable from normal use: it receives a lock (counter n+1), then later an unlock (n+2). No protocol anomaly, no counter gap, no exploitable timing deviation. Detection can therefore only exist at RF level.

**Measure considered, not retained in this version.** Monitoring the received RF level (RSSI) at the receiver, to detect the jammer rather than the missing frame. Setting a threshold in the crowded 433 MHz ISM band (TPMS, weather stations, gate remotes) requires a measurement campaign outside the scope of this study. Recorded as a lead in `OPEN-POINTS.md`. Note the split: the receiver C-02 is inside the item boundary, so the detection itself would be in scope; only the notification to the driver would be a requirement allocated outside the item.

**Status.** Residual retained and documented. Its value is not rated: it depends on CSR-SYS-04, whose verification is `Planned` (see `02` §8). Carried in `04_traceability-matrix.csv` under RES-01.

## CSG-02 — Authorize key-management operations

**Source risk:** TS-03 and TS-04 (pair an attacker key — via OBD diagnostic and via direct CAN injection on the exterior harness) — risk Medium each — treatment = Reduce. The same control covers both because it sits at the service level, independent of the arrival path; it will also cover the revocation threats TS-05 / TS-06 once rated.

**Goal:** The BCM shall not execute a key-management operation (pairing or revocation of a key) without a successful authentication of the requester.

**Initial state vs goal.** `01` §3 assumes no cybersecurity control on these services, so this goal introduces a control the item does not provide.

**Rationale.** The key-management services (F-03 pair, F-06 revoke) execute for *anyone*. The vulnerability is not the OBD port or the CAN path, it is that the *service itself* performs no authorization, so any path that reaches it succeeds (OBD = TS-03, exterior harness = TS-04). Authentication binds the operation to an authorized requester, independently of the access path.

### Cybersecurity concept

Require a successful **Security Access** (UDS service 0x27) / authenticated-diagnostics exchange before any key-management service (F-03 / F-06) is executed. The check sits at the **BCM service level**, not at the port: the BCM verifies authorization whatever the request's arrival path, so one control covers both the OBD route (through the gateway) and the exterior-harness route (bypassing the gateway, TS-04). A control placed at the OBD port would cover TS-03 only.

A 0x27 seed/key scheme relies on a secret that is typically shared across a model range. Certificate-based authentication (UDS 0x29, ISO 14229-1:2020) removes that shared secret and is the better target; its analysis, including the fleet-wide consequence of extracting a shared secret, is an open point.

### Requirements

| req_id | Level | Requirement | Verification |
|---|---|---|---|
| CSR-SYS-02 | System | The RKE system shall reject a key-management operation (pairing or revocation) requested by an unauthenticated party, irrespective of the arrival path (I-05 via the gateway or I-07 direct). | Test (unauthorized pairing attempt on bench, on both paths) |
| CSR-SW-02 | Software | The BCM software shall require a successful Security Access (UDS 0x27) authentication before executing a key-management service (F-03 pair / F-06 revoke), irrespective of the arrival path; on failed or absent authentication, the request is rejected. | Test |
| CSR-HW-02 | Hardware | The Security Access secret shall be held in protected non-volatile storage not extractable through the debug interface (readout protection, or HSM/SHE), so that I-06b does not expose it. | Review + test |

**Dependency:** CSR-HW-02 is a precondition for the whole goal. Under A-07 (debug access allows reading the BCM non-volatile memory) the Security Access secret could be read out via I-06b and the authentication defeated. CSG-02 is therefore only fully effective once the debug-readout path is closed — the precondition common to all four goals, recorded in `02` §9.

**Traceability:** TS-03 and TS-04 → CSG-02 → {CSR-SYS-02, CSR-SW-02, CSR-HW-02} → verification (TS-05, TS-06 later). Carried in `04_traceability-matrix.csv`.

## CSG-03 — Make key-management operations accountable

**Source risk:** TS-03 and TS-04 — risk Medium — treatment = Reduce (detective complement to CSG-02).

**Goal:** Every key-management operation shall leave a record attributable to its requester.

**Rationale.** All other requirements in this document are preventive. A paired attacker key produces no symptom: the vehicle behaves normally and the owner has no way of noticing. Without a record, the whole "persistent covert access" family (TS-03, TS-04, and the key-record threats) is undetectable for the life of the vehicle. CSG-03 does not prevent the operation; it makes it visible afterwards. It depends on CSG-02, which is what establishes the requester's identity.

### Requirements

| req_id | Level | Requirement | Verification |
|---|---|---|---|
| CSR-SW-03 | Software | The BCM software shall record each execution of F-03 (pair) and F-06 (revoke) with a timestamp and the authenticated requester identity, in non-volatile memory, readable through diagnostics. | Test (pair a key, read the record back) |

**Scope note:** CSR-SW-03 covers key-management operations only. It does not mandate a trace for rejected radio commands; that gap is recorded in `OPEN-POINTS.md` and is why the pass criterion of `05_test-spec-CSR-SYS-01.md` does not depend on one.

**Note:** the record is itself an asset (integrity, and personal data where it is correlatable to the owner) and should be purged when the vehicle changes hands. Adding it to the asset register, and the purge-at-resale question, are open points.

**Traceability:** TS-03 and TS-04 → CSG-03 → {CSR-SW-03} → verification. Carried in `04_traceability-matrix.csv`.

## Risk retention

Risks treated by retention produce no goal and no requirement: no cybersecurity control is built. Each is recorded here with its justification, so the decision is traceable rather than silent.

### ACC-01 — TS-02 (denial jamming)

**Source risk:** TS-02 (denial jamming of the lock command) — risk High — treatment = Retain.

**Justification.** The RKE function is inherent to the radio medium: any receiver can be denied its signal, so channel jamming cannot be prevented by any RKE cybersecurity control — the attack removes the channel rather than forging a message (see `02` §7). The threat belongs to the radio-robustness regime (RED / UN R10), not to ISO/SAE 21434. The only in-vehicle mitigation (lock-confirmation feedback) is out of the item boundary and is detective, not preventive. The High rating is retained as a known residual, outside the reach of RKE cybersecurity controls. Because nothing is built, no verification is awaited: the initial risk **is** the residual, and `04` records it as such immediately.

**TS-01 vs TS-02.** Both rest on 433 MHz jamming, and they are treated differently for one reason: in TS-01 a frame *is* eventually received and processed by the BCM, so a cryptographic control bites on it (CSG-01 bounds it, CSG-04 closes the damage path). In TS-02 no frame ever reaches the BCM: there is nothing to verify, so no cybersecurity control of the item applies. Both also end on an out-of-boundary dependency — the lock-confirmation feedback here, the non-radio re-securing of the vehicle in RES-01 — which is where the analysis of this item stops.

**Open point.** A High risk retained with no control should name the authority that accepts it, and the out-of-boundary mitigation should be recorded as a requirement allocated outside the item rather than as a bare retention. Both are recorded in `OPEN-POINTS.md`; this case study defines no governance model.

**Traceability:** TS-02 → ACC-01 (no CSG, no CSR). Carried in `04_traceability-matrix.csv`.