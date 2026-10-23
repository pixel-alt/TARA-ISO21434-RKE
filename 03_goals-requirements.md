# 03 – Goals & Requirements

**Scope:** cybersecurity goals and requirements derived from `02_tara.csv`
**Status:** fictional case study – generic passenger vehicle, no real OEM
**Coverage:** TS-01 is treated by *Reduce* with two goals — CSG-01 (counter freshness, bounds the replay in time) and CSG-04 (command binding, closes the lock→unlock variant) — plus a documented residual (RES-01). The key-management family TS-03..TS-06 is treated by *Reduce* with CSG-02 (authorization) and CSG-03 (accountability). The physical-access family TS-07, TS-08, TS-09 is treated by *Reduce* with CSG-05 (protect the debug interfaces). TS-02 is treated by *Accept* and recorded under "Risk acceptance" (ACC-01). The remaining lines (TS-10, TS-11, TS-12) get their treatment in a later version (see `OPEN-POINTS.md`).

A cybersecurity **goal** states *what* must be prevented (the effect), not *how*. The **concept** names the control and where it sits. The **requirements** (CSR) make the concept testable, allocated to system (SYS), software (SW) and hardware (HW).

```
risk (TS) → goal (CSG) → concept → requirements (CSR-SYS / CSR-SW / CSR-HW) → verification
```

A risk treated by **Accept (Retain)** produces no goal and no requirement — no control is built — so it is not given a CSG. It is recorded as a justified acceptance (see "Risk acceptance" below). A risk only **partially** reduced keeps its goal(s) and records what they do not cover as a named residual (RES-xx).

## CSG-01 — Reject a code whose counter has already been reached

**Source risk:** TS-01 — risk High — treatment = **Reduce (partial)**, jointly with CSG-04; remainder documented as RES-01.

**Goal:** The BCM shall not unlock the vehicle or release the trunk on a radio command whose rolling-code counter is not strictly greater than the last accepted counter.

**Rationale:** A counter blocks a replay only if, between capture and replay, the stored counter has passed the captured value. Jamming prevents exactly that: the vehicle never received the frame, so its counter never moved, and the retained code stays valid whatever the width of the look-ahead window. This goal therefore does **not** eliminate TS-01 — it **bounds** it: the retained code expires at the next legitimate press, so the attacker's window is "owner away, vehicle unused". What the counter does eliminate is the replay of a code the vehicle *has* already accepted. The command-flip variant is closed separately by CSG-04; the remainder is RES-01.

### Cybersecurity concept

Accept a radio command only if its rolling-code counter is strictly greater than the last accepted counter, within the F-05 look-ahead window. The stored counter is updated only after the authentication code is verified. The underlying principle — freshness, as opposed to origin authentication — is the same one SecOC (AUTOSAR Secure Onboard Communication) applies on in-vehicle buses; SecOC itself is not transposable here, as it assumes a bus and an AUTOSAR stack, where this link is a battery-powered fob on a one-way UHF channel. No specific product is mandated at this stage.

### Requirements

| req_id | Level | Requirement | Verification |
|---|---|---|---|
| CSR-SYS-01 | System | The RKE system shall reject a radio command whose counter is not strictly greater than the last accepted counter. | Test — see `05_test-spec-CSR-SYS-01.md` |
| CSR-SW-01 | Software | The BCM software shall verify that the received counter is strictly greater than the stored counter before executing the command, and update the stored counter only after the authentication code is verified (consistent with F-05). | Test — see `05_test-spec-CSR-SYS-01.md` |
| CSR-HW-01 | Hardware | The accepted counter shall be held in non-volatile memory (Flash/EEPROM) and protected against modification from outside the RKE software. | Review + test |

**Traceability:** TS-01 → CSG-01 → {CSR-SYS-01, CSR-SW-01, CSR-HW-01} → verification, with CSG-04 and RES-01. Carried in `04_traceability-matrix.csv`.

## CSG-04 — Bind the command to the authentication code

**Source risk:** TS-01 — risk High — treatment = **Reduce (partial)**, jointly with CSG-01; remainder documented as RES-01.

**Goal:** The RKE system shall not act on a radio command whose command field (lock / unlock / trunk) is not covered by the authentication code computed by the key fob.

**Rationale:** The capture-and-jam attack captures the owner's *lock* presses — that is what the owner does when leaving. To open the vehicle the attacker must present an *unlock*. Under assumption A-10 (`01` §6) the command field is not covered by the authentication code, so the attacker flips the captured lock frame to unlock without invalidating the code. Binding the command into the authenticated payload removes that path: a captured lock code replays only as lock, which is useless (relocking an already-locked car). CSG-01 bounds the attack in time; CSG-04 removes the lock→unlock variant; what remains — the replay of a *genuinely captured* unlock, before the next legitimate press — is RES-01.

### Cybersecurity concept

Include the command field in the payload over which the authentication code (rolling-code MAC) is computed by the fob (C-01) and verified by the BCM (C-03), so that altering the command after emission invalidates the code. This is a design property of the frame format, independent of the counter check.

### Requirements

| req_id | Level | Requirement | Verification |
|---|---|---|---|
| CSR-SYS-04 | System | The command field shall be covered by the authentication code computed by the key fob and verified by the BCM, so that a command altered after emission is rejected. | Test (command-flip case) |
| CSR-SW-04 | Software | The BCM software shall verify the authentication code over a payload that includes the command field before executing the command. | Test (command-flip case) |

**Traceability:** TS-01 → CSG-04 → {CSR-SYS-04, CSR-SW-04} → verification. Carried in `04_traceability-matrix.csv`.

### RES-01 — Residual risk: capture-and-delayed-replay of a genuine unlock

**What remains:** the replay of a *genuinely captured unlock* code that the vehicle never received, performed before the next legitimate press. CSG-04 removes the lock→unlock flip, but it cannot help when the captured frame was already an unlock; CSG-01 only bounds this in time.

**Why no cryptographic control closes it:** freshness requires a reference the attacker cannot freeze. On a one-way, event-driven radio link the only reference is the stored counter, and jamming freezes it. Closing the gap would require either a receiver-generated nonce — hence a bidirectional link, outside the technical constraints of `01` §6 — or a time reference in the fob, which a coin-cell fob does not carry. No width of look-ahead window changes this: the retained code carries the immediately following counter value and passes even with a window of one.

**Why no application-level detection is possible:** from the BCM's point of view the sequence is indistinguishable from normal use — it receives a lock (counter n+1), then later an unlock (n+2). No protocol anomaly, no counter gap, no exploitable timing deviation. Detection can therefore only exist at RF level.

**Measure considered, not retained in this version:** monitoring the received RF level (RSSI) at the receiver, to detect the jammer rather than the missing frame. Setting a threshold in the crowded 433 MHz ISM band (TPMS, weather stations, gate remotes) requires a measurement campaign outside the scope of this study. Recorded as a lead in `OPEN-POINTS.md`. Note the split: the receiver C-02 is inside the item boundary, so the detection itself would be in scope; only the notification to the driver would be a requirement allocated outside the item.

**Decision:** risk retained as a documented residual. Carried in `04_traceability-matrix.csv` under RES-01.

## CSG-02 — Authorize key-management operations

**Source risk:** TS-03 (pair via OBD), TS-04 (pair via direct CAN injection), TS-05 (revoke via OBD), TS-06 (revoke via direct CAN injection) — risk Medium each — treatment = Reduce. One control covers all four: the vulnerability is the service, not the path.

**Goal:** The BCM shall not execute a key-management operation (pairing or revocation of a key) without a successful authentication of the requester.

**Rationale:** The key-management services (F-03 pair, F-06 revoke) execute for *anyone* — no cybersecurity control is assumed on them (see `01_item-definition.md` §3). The vulnerability is not the OBD port or the CAN path, it is that the *service itself* performs no authorization, so any path that reaches it succeeds (OBD = TS-03 / TS-05, exterior harness = TS-04 / TS-06). Authentication binds the operation to an authorized requester, independently of the access path.

### Cybersecurity concept

Require a successful **Security Access** (UDS service 0x27) / authenticated-diagnostics exchange before any key-management service (F-03 / F-06) is executed. The check sits at the **BCM service level**, not at the port: the BCM verifies authorization whatever the request's arrival path, so one control covers both the OBD route (through the gateway) and the exterior-harness route (bypassing the gateway, TS-04 / TS-06). A control placed at the OBD port would cover the OBD lines only.

### Requirements

| req_id | Level | Requirement | Verification |
|---|---|---|---|
| CSR-SYS-02 | System | The RKE system shall reject a key-management operation (pairing or revocation) requested by an unauthenticated party. | Test (unauthorized operation on bench) |
| CSR-SW-02 | Software | The BCM software shall require a successful Security Access (UDS 0x27) authentication before executing a key-management service (F-03 pair / F-06 revoke); on failed or absent authentication, the request is rejected. | Test |
| CSR-HW-02 | Hardware | The Security Access secret shall be held in protected non-volatile storage not extractable through the debug interface (readout protection, or HSM/SHE), so that I-06b does not expose it. | Review + test |

**Dependency:** CSR-HW-02 is a precondition for the whole goal. Under assumption A-07 (no effective debug-access protection), the Security Access secret could be read out via I-06b and the authentication defeated. CSG-02 is therefore only fully effective once the debug-readout path is closed — which is the object of **CSG-05**, the common precondition recorded in `02_tara-method.md` §9. CSG-05 discharges this dependency.

**Traceability:** TS-03, TS-04, TS-05, TS-06 → CSG-02 → {CSR-SYS-02, CSR-SW-02, CSR-HW-02} → verification. Carried in `04_traceability-matrix.csv`.

## CSG-03 — Make key-management operations accountable

**Source risk:** TS-03, TS-04, TS-05, TS-06 — risk Medium each — treatment = Reduce (detective complement to CSG-02).

**Goal:** Every key-management operation shall leave a record attributable to its requester.

**Rationale:** All other requirements in this document are preventive. For the pairing lines (TS-03, TS-04) a paired attacker key produces no symptom: the vehicle behaves normally and the owner has no way of noticing, so without a record the "persistent covert access" family is undetectable for the life of the vehicle. For the revocation lines (TS-05, TS-06) the denial is self-evident to the owner, so here the record provides *attribution* rather than detection — it is what allows a revoked key to be traced and restored. CSG-03 does not prevent the operation; it makes it accountable afterwards. It depends on CSG-02, which establishes the requester's identity.

### Requirements

| req_id | Level | Requirement | Verification |
|---|---|---|---|
| CSR-SW-03 | Software | The BCM software shall record each execution of F-03 (pair) and F-06 (revoke) with a timestamp and the authenticated requester identity, in non-volatile memory, readable through diagnostics. | Test (pair/revoke a key, read the record back) |

**Note:** the record is itself an asset (integrity, and personal data where it is correlatable to the owner) and should be purged when the vehicle changes hands. Adding it to the asset register, and the purge-at-resale question, are open points for v0.3 (see `OPEN-POINTS.md`).

**Traceability:** TS-03, TS-04, TS-05, TS-06 → CSG-03 → {CSR-SW-03} → verification. Carried in `04_traceability-matrix.csv`.

## CSG-05 — Protect the debug interfaces

**Source risk:** TS-07 (fob secret read via I-06a), TS-08 (all BCM secrets read via I-06b), TS-09 (a BCM key record overwritten via I-06b) — risk Low each — treatment = Reduce. CSG-05 will also cover the debug-access threats TS-10 and TS-11 once they are rated.

**Goal:** The debug interface shall not allow a secret to be read from, or the non-volatile memory to be written to, on the key fob (I-06a) or the BCM (I-06b).

**Initial state vs goal.** A-07 (BCM) and A-12 (fob) state that the debug paths are open — that is the item as currently defined, and it is what makes TS-07..TS-09 feasible. This goal introduces a control the item does not provide: it closes those paths. This is the same assumption-vs-requirement pattern as A-10 / CSG-04.

**Rationale.** These three lines are Major × Low = Low: invasive physical access, one device at a time, no fleet-wide master key (`01` §1). On their own risk value alone a retention would be defensible. They are reduced for a different reason — a **dependency**: closing the debug path is a precondition of CSG-02 (`02_tara-method.md` §9). If the debug stays open, the Security Access secret is read out (TS-08 applied to that secret) and CSG-02 falls. The same hardware control therefore protects the authorization secret and reduces the physical-access family — the `02_tara-method.md` §5 rule that a measure can be justified by a dependency rather than by its own risk value. The control is **hardware**: the debug bypasses the BCM software, so software cannot block it.

### Requirements

| req_id | Level | Requirement | Verification |
|---|---|---|---|
| CSR-HW-05 | Hardware | Readout protection / debug lock shall be enabled at production on the key fob (I-06a) and the BCM (I-06b), so that the debug interface neither discloses stored secrets nor permits writing the non-volatile memory. A secret required to resist this shall reside in a secure element (SHE/HSM). | Review + test |

**Discharges:** the debug-path dependency on CSR-HW-02 (Security Access secret) — once CSG-05 is implemented, the "depends on A-07" caveat is satisfied by construction.

**Traceability:** TS-07, TS-08, TS-09 → CSG-05 → {CSR-HW-05} → verification. Carried in `04_traceability-matrix.csv`.

## Risk acceptance (Retain)

Risks treated by acceptance produce no goal and no requirement: no cybersecurity control is built. Each is recorded here with its justification, so the decision is traceable rather than silent.

### ACC-01 — TS-02 (denial jamming)

**Source risk:** TS-02 (denial jamming of the lock command) — risk High — treatment = Accept (Retain).

**Justification:** The RKE function is inherent to the radio medium: any receiver can be denied its signal, so channel jamming cannot be prevented by any RKE cybersecurity control — the attack removes the channel rather than forging a message (see `02_tara-method.md` §7). The threat belongs to the radio-robustness regime (RED / UN R10), not to ISO/SAE 21434. The only in-vehicle mitigation (lock-confirmation feedback) is out of the item boundary and is detective, not preventive. The High rating is retained as a known residual, outside the reach of RKE cybersecurity controls.

**TS-01 vs TS-02.** Both rest on 433 MHz jamming, and they are treated differently for one reason: in TS-01 a frame *is* eventually received and processed by the BCM, so cryptographic controls bite on it — partially (CSG-01 + CSG-04, remainder RES-01). In TS-02 no frame ever reaches the BCM: there is nothing to verify, so no cybersecurity control of the item applies. Consistently with RES-01, any detective measure splits the same way — RF-level detection would be inside the boundary (receiver C-02), while the lock-confirmation feedback invoked here is entirely outside it, executed by the lighting / comfort functions (`01` §4).

**Traceability:** TS-02 → ACC-01 (no CSG, no CSR). Carried in `04_traceability-matrix.csv`.