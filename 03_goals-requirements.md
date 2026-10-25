# 03 – Goals & Requirements

**Scope:** cybersecurity goals and requirements derived from `02_tara.csv`
**Status:** fictional case study – generic passenger vehicle, no real OEM
**Coverage:** TS-01 is treated by *Reduce* with two goals — CSG-01 (counter freshness) and CSG-04 (command binding) — plus a documented residual (RES-01). The key-management family TS-03..TS-06 is treated by *Reduce* with CSG-02 (authorization) and CSG-03 (accountability). The physical-access family TS-07..TS-11 is treated by *Reduce* with CSG-05 (protect the debug interfaces), TS-11 jointly with CSG-06 (secure boot). TS-12 (owner tracking, High) is treated by *Reduce* with CSG-07 (non-correlatable radio emission). TS-02 is treated by *Accept* and recorded under "Risk acceptance" (ACC-01). All 12 threat scenarios are now rated and treated.

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

Accept a radio command only if its rolling-code counter is strictly greater than the last accepted counter, within the F-05 look-ahead window. The stored counter is updated only after the authentication code is verified. The underlying principle — freshness, as opposed to origin authentication — is the same one SecOC (AUTOSAR Secure Onboard Communication) applies on in-vehicle buses; SecOC itself is not transposable here, as it assumes a bus and an AUTOSAR stack, where this link is a battery-powered fob on a one-way UHF channel. No specific product is mandated at this stage. Where the counter is also protected against eavesdropping (CSG-07), its on-air integrity is carried by the authentication tag of the frame defined there — CSG-01 consumes that tag, it does not add a second mechanism.

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

**Source risk:** TS-07 (fob secret read via I-06a), TS-08 (all BCM secrets read via I-06b), TS-09 (a BCM key record overwritten via I-06b), TS-10 (rolling-code counter rolled back via I-06b) and TS-11 (firmware flashed via I-06b, jointly with CSG-06) — risk Low each — treatment = Reduce.

**Goal:** The debug interface shall not allow a secret to be read from, or the non-volatile memory to be written to, on the key fob (I-06a) or the BCM (I-06b).

**Initial state vs goal.** A-07 (BCM) and A-12 (fob) state that the debug paths are open — that is the item as currently defined, and it is what makes TS-07..TS-11 feasible on this path. This goal introduces a control the item does not provide: it closes those paths. This is the same assumption-vs-requirement pattern as A-10 / CSG-04.

**Rationale.** These lines are Major × Low = Low: invasive physical access, one device at a time, no fleet-wide master key (`01` §1). On their own risk value alone a retention would be defensible. They are reduced for a different reason — a **dependency**: closing the debug path is a precondition of CSG-02 (`02_tara-method.md` §9). If the debug stays open, the Security Access secret is read out (TS-08 applied to that secret) and CSG-02 falls. The same hardware control therefore protects the authorization secret and reduces the physical-access family — the `02_tara-method.md` §5 rule that a measure can be justified by a dependency rather than by its own risk value. The control is **hardware**: the debug bypasses the BCM software, so software cannot block it. TS-11 (firmware flash) is reduced here on the injection path and additionally by CSG-06 on the execution path.

### Requirements

| req_id | Level | Requirement | Verification |
|---|---|---|---|
| CSR-HW-05 | Hardware | Readout protection / debug lock shall be enabled at production on the key fob (I-06a) and the BCM (I-06b), so that the debug interface neither discloses stored secrets nor permits writing the non-volatile memory. A secret required to resist this shall reside in a secure element (SHE/HSM). | Review + test |

**Discharges:** the debug-path dependency on CSR-HW-02 (Security Access secret) — once CSG-05 is implemented, the "depends on A-07" caveat is satisfied by construction.

**Traceability:** TS-07, TS-08, TS-09, TS-10, TS-11 → CSG-05 → {CSR-HW-05} → verification. Carried in `04_traceability-matrix.csv`.

## CSG-06 — Firmware authenticity (secure boot)

**Source risk:** TS-11 (firmware flashed via the BCM debug interface) — risk Low — treatment = Reduce, jointly with CSG-05. **Discharges A-11.**

**Goal:** The BCM shall execute only firmware whose authenticity and integrity have been verified.

**Initial state vs goal.** A-11 states that no firmware authenticity protection is present — that is the item as analysed. This goal introduces the control that removes A-11, the same assumption-vs-requirement pattern as A-07 / CSG-05 and A-10 / CSG-04.

**Rationale.** TS-11 does not trick the RKE logic, it **replaces** it: a backdoored firmware voids CSG-01, CSG-02, CSG-03 and CSG-04 at once (`02_tara-method.md` §9). The firmware cannot protect itself, so the control sits **below** it, on two complementary layers: CSG-05 (debug-lock) closes the injection path I-06b, and CSG-06 (secure boot) rejects a modified firmware **whatever the installation path**. Secure boot = a cryptographic signature of the firmware, verified at start-up by an immutable boot stage anchored in a hardware root of trust.

### Requirements

| req_id | Level | Requirement | Verification |
|---|---|---|---|
| CSR-SW-06 | Software | At start-up, the boot stage shall verify the cryptographic signature of the BCM firmware before executing it, and refuse to run firmware whose signature is invalid or absent. | Test (boot an unsigned firmware, verify refusal) |
| CSR-HW-06 | Hardware | The signature-verification public key (root of trust) and the boot verifier shall reside in immutable storage (ROM / locked region / secure element), not in rewritable Flash, so they cannot be replaced together with the firmware. | Review + test |

**Traceability:** TS-11 → CSG-06 → {CSR-SW-06, CSR-HW-06} → verification (TS-11 also → CSG-05). Carried in `04_traceability-matrix.csv`.

## CSG-07 — Non-correlatable radio emission

**Source risk:** TS-12 (Key ID eavesdropping / owner tracking) — risk **High** — treatment = Reduce. Treats DS-03; mandated by GDPR art. 25 (`01` §6).

**Goal:** The RKE system shall not expose on I-01 any observable data that lets an observer correlate two transmissions of the same key fob.

**Initial state vs goal.** `01` §2 describes the I-01 frame as carrying the Key ID, the counter, the authentication code and the command, the Key ID and counter in clear, and A-06 states the Key ID is stable. This goal changes the frame format. Same assumption-vs-requirement pattern as A-10 / CSG-04 and A-07 / CSG-05.

**Rationale.** TS-12 is the second **High** line of the study (Major × High), so it cannot be retained like the Low debug lines. The Key ID is not the only linking field: the rolling-code counter is transmitted in clear and is **monotonic** — successive values (4471, 4472, 4473) identify one fob as surely as a fixed identifier. Rotating the identifier while leaving the counter in clear moves the linking field, it does not remove it. The goal is therefore stated over **every observable field**, not over "the identifier".

### Cybersecurity concept

The frame shall provide, together: **confidentiality** of the counter (so it is not readable → tracking) and **integrity** of the counter and command (so they cannot be forged or bit-flipped → replay / command-flip). These are the two properties of **authenticated encryption**: the payload `{counter, command}` is encrypted and carries an authentication tag; nothing else travels in clear that could link two frames; the tag is verified **before** the payload is decrypted.

**Nonce and counter — reconciliation.** An authenticated-encryption construction needs a nonce. The nonce must not itself be a linking field: reused as the rolling-code counter (the natural reflex) it would travel in clear and monotonic, reopening TS-12. The counter is therefore **not reused as the public nonce**; it is kept for freshness (CSG-01) **inside the authenticated payload**, recovered only after the tag is verified. The nonce is a separate, non-correlatable value. Its generation and length — a random nonce sized against the birthday bound, or a separate non-observable source — are a frame-budget trade-off recorded as an open point (`OPEN-POINTS.md`).

**No primitive is mandated** at this stage: the required properties and the family (authenticated encryption, truncated tag, bounded verification) are specified; the construction and primitive belong to the design phase.

**Consistency with CSG-01.** Because the counter is no longer readable before verification, the BCM must verify the tag, recover the counter, then apply the freshness check and update the stored value — the order CSR-SW-01 already requires. Two goals from two distinct risks (replay, tracking) converge on one frame construction: CSG-01 / CSG-04 consume its **tag** (integrity), CSG-07 consumes its **encryption** (confidentiality).

**Effect on F-04.** With a rotating on-air identifier, the identifier emitted on I-01 and the Key ID listed by F-04 are no longer the same object: F-04 lists the **stored** identity, visible over diagnostics only. This conditions the "reference baseline for F-04" open point.

### Requirements

| req_id | Level | Requirement | Verification |
|---|---|---|---|
| CSR-SYS-07 | System | No field observable on I-01 — identifier, nonce, ciphertext or tag — shall be constant across transmissions of one fob or carry an order relation between them, so that two transmissions of the same fob cannot be correlated. | Test — see verification note |
| CSR-SW-07 | Software | The BCM shall identify the emitting key by trial over its stored secrets (one tag verification per stored key per frame) and recover the rolling-code counter from the authenticated payload only after the tag is verified; the number of frame verifications accepted per unit of time shall be bounded, so that the truncated tag's forgery probability stays negligible over the vehicle lifetime and the per-frame trial over stored secrets cannot raise the quiescent current of a parked vehicle. | Test / review (incl. invalid-frame flooding; acceptance rate and current draw) |

**Verification note (CSR-SYS-07).** Capture at least 50 successive frames from one fob and 50 from a second, decode the fields, and verify for every field (nonce included) that it is neither constant nor ordered across one fob's frames, and that the two fobs' values are not separable into disjoint ranges. A criterion limited to "the observable identifier differs between two frames" is **not** sufficient — it passes while a monotonic counter or nonce stays correlatable.

### Documented limit — physical-layer fingerprinting

CSR-SYS-07 bounds correlation at frame level. It does not remove correlation at **physical level** (crystal frequency offset, turn-on transient, modulation imperfections), which a determined observer can fingerprint with suitable equipment. No frame-format requirement reaches this; it is an RF design constraint on the fob, recorded as a lead in `OPEN-POINTS.md` so the goal is not over-claimed.

**Traceability:** TS-12 → CSG-07 → {CSR-SYS-07, CSR-SW-07} → verification. Carried in `04_traceability-matrix.csv`.

## Risk acceptance (Retain)

Risks treated by acceptance produce no goal and no requirement: no cybersecurity control is built. Each is recorded here with its justification, so the decision is traceable rather than silent.

### ACC-01 — TS-02 (denial jamming)

**Source risk:** TS-02 (denial jamming of the lock command) — risk High — treatment = Accept (Retain).

**Justification:** The RKE function is inherent to the radio medium: any receiver can be denied its signal, so channel jamming cannot be prevented by any RKE cybersecurity control — the attack removes the channel rather than forging a message (see `02_tara-method.md` §7). The threat belongs to the radio-robustness regime (RED / UN R10), not to ISO/SAE 21434. The only in-vehicle mitigation (lock-confirmation feedback) is out of the item boundary and is detective, not preventive. The High rating is retained as a known residual, outside the reach of RKE cybersecurity controls.

**TS-01 vs TS-02.** Both rest on 433 MHz jamming, and they are treated differently for one reason: in TS-01 a frame *is* eventually received and processed by the BCM, so cryptographic controls bite on it — partially (CSG-01 + CSG-04, remainder RES-01). In TS-02 no frame ever reaches the BCM: there is nothing to verify, so no cybersecurity control of the item applies. Consistently with RES-01, any detective measure splits the same way — RF-level detection would be inside the boundary (receiver C-02), while the lock-confirmation feedback invoked here is entirely outside it, executed by the lighting / comfort functions (`01` §4).

**Traceability:** TS-02 → ACC-01 (no CSG, no CSR). Carried in `04_traceability-matrix.csv`.