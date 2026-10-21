# 03 – Goals & Requirements

**Scope:** cybersecurity goals and requirements derived from `02_tara.csv`
**Status:** fictional case study – generic passenger vehicle, no real OEM
**Coverage:** TS-01 and TS-03 are treated by *Reduce* with a goal and requirements. TS-02 is treated by *Accept* and recorded under "Risk acceptance". CSG-02 also covers the other key-management threats (TS-04, TS-05, TS-06), which will reuse it once rated. The remaining rated risks will get their treatment in a later version.

A cybersecurity **goal** states *what* must be prevented (the effect), not *how*. The **concept** names the control and where it sits. The **requirements** (CSR) make the concept testable, allocated to system (SYS), software (SW) and hardware (HW).

```
risk (TS) → goal (CSG) → concept → requirements (CSR-SYS / CSR-SW / CSR-HW) → verification
```

A risk treated by **Accept (Retain)** produces no goal and no requirement — no control is built — so it is not given a CSG. It is recorded as a justified acceptance (see "Risk acceptance" below).

## CSG-01 — Reject replayed authentication codes

**Source risk:** TS-01 (RollJam) — risk High — treatment = Reduce.

**Goal:** The BCM shall not unlock the vehicle or release the trunk on an authentication code that has already been used or replayed.

**Rationale:** RollJam replays an *authentic* code. Authentication alone does not stop it — the control must verify **freshness** (reject a code already seen or out of sequence), not origin. This is the distinction between authenticating a message and verifying that it is new.

### Cybersecurity concept

Accept a radio command only if its rolling-code counter is **strictly greater** than the last accepted counter (bounded by the F-05 look-ahead window). The stored counter is updated only after the authentication code is verified. This is the freshness principle behind **SecOC** (AUTOSAR Secure Onboard Communication). A replayed frame carries an old counter, so it is rejected. No specific product is mandated at this stage.

### Requirements

| req_id | Level | Requirement | Verification |
|---|---|---|---|
| CSR-SYS-01 | System | The RKE system shall reject an authentication code already accepted (replayed). | Test (replay on bench) |
| CSR-SW-01 | Software | The BCM software shall verify that the received counter is strictly greater than the stored counter before executing the command, and update the stored counter only after the authentication code is verified (consistent with F-05). | Test (replay on bench) |
| CSR-HW-01 | Hardware | The accepted counter shall be held in non-volatile memory (Flash/EEPROM) and protected against modification from outside the RKE software (mitigates counter rollback via debug, TS-10). | Review + test |

**Traceability:** TS-01 → CSG-01 → {CSR-SYS-01, CSR-SW-01, CSR-HW-01} → verification. Carried in `04_traceability-matrix.csv`.

## CSG-02 — Authorize key-management operations

**Source risk:** TS-03 (pair an attacker key via OBD diagnostic) — risk Medium — treatment = Reduce. The same control covers the other key-management threats (TS-04 pairing via direct CAN injection; TS-05 / TS-06 revocation), which will reference CSG-02 once rated.

**Goal:** The BCM shall not execute a key-management operation (pairing or revocation of a key) without a successful authentication of the requester.

**Rationale:** The key-management services (F-03 pair, F-06 revoke) execute for *anyone* — no cybersecurity control is assumed on them (see `01_item-definition.md` §3). The vulnerability is not the OBD port or the CAN path, it is that the *service itself* performs no authorization, so any path that reaches it succeeds (OBD = TS-03, exterior harness = TS-04). Authentication binds the operation to an authorized requester, independently of the access path.

### Cybersecurity concept

Require a successful **Security Access** (UDS service 0x27) / authenticated-diagnostics exchange before any key-management service (F-03 / F-06) is executed. The check sits at the **BCM service level**, not at the port: the BCM verifies authorization whatever the request's arrival path, so one control covers both the OBD route (through the gateway) and the exterior-harness route (bypassing the gateway, TS-04). A control placed at the OBD port would cover TS-03 only.

### Requirements

| req_id | Level | Requirement | Verification |
|---|---|---|---|
| CSR-SYS-02 | System | The RKE system shall reject a key-management operation (pairing or revocation) requested by an unauthenticated party. | Test (unauthorized pairing attempt on bench) |
| CSR-SW-02 | Software | The BCM software shall require a successful Security Access (UDS 0x27) authentication before executing a key-management service (F-03 pair / F-06 revoke); on failed or absent authentication, the request is rejected. | Test |
| CSR-HW-02 | Hardware | The Security Access secret shall be held in protected non-volatile storage not extractable through the debug interface (readout protection, or HSM/SHE), so that I-06b does not expose it. | Review + test |

**Dependency:** CSR-HW-02 is a precondition for the whole goal. Under assumption A-07 (no effective debug-access protection), the Security Access secret could be read out via I-06b and the authentication defeated. CSG-02 is therefore only fully effective once the debug-readout path is closed — the same path addressed by the debug-access threats (TS-08 to TS-11).

**Traceability:** TS-03 (and later TS-04, TS-05, TS-06) → CSG-02 → {CSR-SYS-02, CSR-SW-02, CSR-HW-02} → verification. Carried in `04_traceability-matrix.csv`.

## Risk acceptance (Retain)

Risks treated by acceptance produce no goal and no requirement: no cybersecurity control is built. Each is recorded here with its justification, so the decision is traceable rather than silent.

### ACC-01 — TS-02 (denial jamming)

**Source risk:** TS-02 (denial jamming of the lock command) — risk High — treatment = Accept (Retain).

**Justification:** The RKE function is inherent to the radio medium: any receiver can be denied its signal, so channel jamming cannot be prevented by any RKE cybersecurity control — the attack removes the channel rather than forging a message (see `02_tara-method.md` §7). The threat belongs to the radio-robustness regime (RED / UN R10), not to ISO/SAE 21434. The only in-vehicle mitigation (lock-confirmation feedback) is out of the item boundary and is detective, not preventive. The High rating is retained as a known residual, outside the reach of RKE cybersecurity controls.

**Traceability:** TS-02 → ACC-01 (no CSG, no CSR). Carried in `04_traceability-matrix.csv`.