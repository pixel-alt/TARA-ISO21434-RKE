# 03 – Goals & Requirements

**Scope:** cybersecurity goals and requirements derived from `02_tara.csv`
**Status:** fictional case study – generic passenger vehicle, no real OEM
**Coverage:** TS-01 is treated by *Reduce*, **partially**, with a documented residual (RES-01). TS-03 is treated by *Reduce* with two goals: CSG-02 (preventive) and CSG-03 (detective). TS-02 is treated by *Accept* and recorded under "Risk acceptance" (ACC-01). CSG-02 also covers the other key-management threats (TS-04, TS-05, TS-06), which will reference it once rated. The remaining lines get their treatment in a later version — see `OPEN-POINTS.md`.

A cybersecurity **goal** states *what* must be prevented (the effect), not *how*. The **concept** names the control and where it sits. The **requirements** (CSR) make the concept testable, allocated to system (SYS), software (SW) and hardware (HW).

```
risk (TS) → goal (CSG) → concept → requirements (CSR-SYS / CSR-SW / CSR-HW) → verification
```

A risk treated by **Accept (Retain)** produces no goal and no requirement — no control is built — so it is not given a CSG. It is recorded as a justified acceptance (see "Risk acceptance" below). A risk treated by *Reduce* may still leave a part no control on this item can reach; that part is recorded as a named residual (RES-xx), not silently dropped.

## CSG-01 — Reject a code whose counter has already been reached

**Source risk:** TS-01 — risk High — treatment = Reduce (partial, see RES-01 below).

**Goal:** The BCM shall not unlock the vehicle or release the trunk on a radio command whose rolling-code counter is not strictly greater than the last accepted counter.

**Rationale.** A counter blocks a replay only if, between capture and replay, the stored counter has passed the captured value. Jamming prevents exactly that: the vehicle never received the frame, so its counter never moved, and the retained code stays valid whatever the width of the look-ahead window.

```
stored counter = n+1  (after the attacker replayed the first code)
attacker holds code n+2

without a legitimate press in between      with a legitimate press (n+3)
  replay n+2 : n+2 > n+1  ✓ OPENS            fob emits n+3 → stored = n+3
  → residual, see RES-01                      replay n+2 : n+2 < n+3  ✗ REJECTED
                                              → blocked by CSG-01
```

So this goal does not eliminate TS-01 — it **bounds** it: the retained code expires at the next legitimate press, which is a real reduction of the exploitation window. What the counter does eliminate outright is the replay of a code the vehicle *has* already accepted.

### Cybersecurity concept

Accept a radio command only if its rolling-code counter is **strictly greater** than the last accepted counter, within the F-05 look-ahead window. The stored counter is updated only after the authentication code is verified. The underlying principle — freshness, as opposed to origin authentication — is the same one SecOC (AUTOSAR Secure Onboard Communication) applies on in-vehicle buses; SecOC itself is not transposable here, as it assumes a bus and an AUTOSAR stack, where this link is a battery-powered fob on a one-way UHF channel. No specific product is mandated at this stage.

### Requirements

| req_id | Level | Requirement | Verification |
|---|---|---|---|
| CSR-SYS-01 | System | The RKE system shall reject a radio command whose rolling-code counter is not strictly greater than the stored counter. | Test — see `05_test-spec-CSR-SYS-01.md` |
| CSR-SW-01 | Software | The BCM software shall verify that the received counter is strictly greater than the stored counter before executing the command, and update the stored counter only after the authentication code is verified (consistent with F-05). | Test — see `05_test-spec-CSR-SYS-01.md` |
| CSR-HW-01 | Hardware | The accepted counter shall be held in non-volatile memory (Flash/EEPROM) and protected against modification from outside the RKE software. | Review + test |

**Traceability:** TS-01 → CSG-01 → {CSR-SYS-01, CSR-SW-01, CSR-HW-01} → verification, plus RES-01. Carried in `04_traceability-matrix.csv`.

### RES-01 — Residual risk: capture-and-delayed-replay

CSG-01 does not cover the case where the captured code was never received by the vehicle. **No cryptographic control on this item does**, and the reason is structural: freshness requires a reference the attacker cannot freeze, and here the reference is the stored counter, which only advances when a frame is accepted. Jamming freezes it. Closing this would require either a receiver-generated nonce — hence a bidirectional link, outside the technical constraints of `01` — or a time reference in the fob, which a coin-cell fob does not have.

**Why no application-level detection is possible.** From the BCM's point of view the sequence is indistinguishable from normal use: it receives a lock (n+1), then later an unlock (n+2). No protocol anomaly, no counter gap, no exploitable timing deviation. Detection can therefore only exist at RF level, not at application level.

**Measure considered, not retained in this version.** Monitoring the RF level at the receiver (RSSI) would detect the *jammer* rather than the missing frame. Setting a threshold in the crowded 433 MHz ISM band (TPMS, weather stations, gate remotes, dimmers) requires a measurement campaign outside the scope of this study, so it is recorded as a lead, not claimed as a control. Note where it would sit: the RF receiver C-02 is inside the item boundary, so the *detection* would be in scope; only the *notification* to the driver would be a requirement allocated outside the item.

**Status:** residual retained and documented. Carried in `04_traceability-matrix.csv`.

## CSG-02 — Authorize key-management operations

**Source risk:** TS-03 (pair an attacker key via OBD diagnostic) — risk Medium — treatment = Reduce. The same control covers the other key-management threats (TS-04 pairing via direct CAN injection; TS-05 / TS-06 revocation), which will reference CSG-02 once rated.

**Goal:** The BCM shall not execute a key-management operation (pairing or revocation of a key) without a successful authentication of the requester.

**Rationale:** The key-management services (F-03 pair, F-06 revoke) execute for *anyone* — no cybersecurity control is assumed on them (see `01_item-definition.md` §3). The vulnerability is not the OBD port or the CAN path, it is that the *service itself* performs no authorization, so any path that reaches it succeeds (OBD = TS-03, exterior harness = TS-04). Authentication binds the operation to an authorized requester, independently of the access path.

### Cybersecurity concept

Require a successful **Security Access** (UDS service 0x27) / authenticated-diagnostics exchange before any key-management service (F-03 / F-06) is executed. The check sits at the **BCM service level**, not at the port: the BCM verifies authorization whatever the request's arrival path, so one control covers both the OBD route (through the gateway) and the exterior-harness route (bypassing the gateway, TS-04). A control placed at the OBD port would cover TS-03 only.

A 0x27 seed/key scheme relies on a secret that is typically shared across a model range. Certificate-based authentication (UDS 0x29, ISO 14229-1:2020) removes that shared secret and is the better target; its analysis — including the fleet-wide consequence of extracting a shared secret — is an open point (see `OPEN-POINTS.md`).

### Requirements

| req_id | Level | Requirement | Verification |
|---|---|---|---|
| CSR-SYS-02 | System | The RKE system shall reject a key-management operation (pairing or revocation) requested by an unauthenticated party. | Test (unauthorized pairing attempt on bench) |
| CSR-SW-02 | Software | The BCM software shall require a successful Security Access (UDS 0x27) authentication before executing a key-management service (F-03 pair / F-06 revoke); on failed or absent authentication, the request is rejected. | Test |
| CSR-HW-02 | Hardware | The Security Access secret shall be held in protected non-volatile storage not extractable through the debug interface (readout protection, or HSM/SHE), so that I-06b does not expose it. | Review + test |

**Dependency:** CSR-HW-02 is a precondition for the whole goal. Under assumption A-07 (no effective debug-access protection), the Security Access secret could be read out via I-06b and the authentication defeated. CSG-02 is therefore only fully effective once the debug-readout path is closed — the same path addressed by the debug-access threats (TS-08 to TS-11), which are not yet rated.

**Traceability:** TS-03 (and later TS-04, TS-05, TS-06) → CSG-02 → {CSR-SYS-02, CSR-SW-02, CSR-HW-02} → verification. Carried in `04_traceability-matrix.csv`.

## CSG-03 — Make key-management operations accountable

**Source risk:** TS-03 — risk Medium — treatment = Reduce (detective complement to CSG-02).

**Goal:** Every key-management operation shall leave a record attributable to its requester.

**Rationale:** All other requirements in this document are preventive. A paired attacker key produces no symptom: the vehicle behaves normally and the owner has no way of noticing. Without a record, the whole "persistent covert access" family is undetectable for the life of the vehicle. CSG-03 does not prevent the operation; it makes it visible afterwards. It depends on CSG-02, which is what establishes the requester's identity.

### Requirements

| req_id | Level | Requirement | Verification |
|---|---|---|---|
| CSR-SW-03 | Software | The BCM software shall record each execution of F-03 (pair) and F-06 (revoke) with a timestamp and the authenticated requester identity, in non-volatile memory, readable through diagnostics. | Test (pair a key, read the record back) |

**Note:** the record is itself an asset (integrity, and personal data where it is correlatable to the owner) and should be purged when the vehicle changes hands. To be added to the asset register in a later version — see `OPEN-POINTS.md`.

**Traceability:** TS-03 → CSG-03 → {CSR-SW-03} → verification. Carried in `04_traceability-matrix.csv`.

## Risk acceptance (Retain)

Risks treated by acceptance produce no goal and no requirement: no cybersecurity control is built. Each is recorded here with its justification, so the decision is traceable rather than silent.

### ACC-01 — TS-02 (denial jamming)

**Source risk:** TS-02 (denial jamming of the lock command) — risk High — treatment = Accept (Retain).

**Justification:** The RKE function is inherent to the radio medium: any receiver can be denied its signal, so channel jamming cannot be prevented by any RKE cybersecurity control — the attack removes the channel rather than forging a message (see `02_tara-method.md` §7). The threat belongs to the radio-robustness regime (RED / UN R10), not to ISO/SAE 21434. The only in-vehicle mitigation (lock-confirmation feedback) is out of the item boundary and is detective, not preventive. The High rating is retained as a known residual, outside the reach of RKE cybersecurity controls.

**TS-01 vs TS-02.** Both involve jamming, but they are not treated the same way, for one reason: in TS-01 a frame *is* eventually processed by the BCM, so a cryptographic control bites on it — partially (CSG-01 + RES-01); in TS-02 no frame ever arrives, so there is nothing for a cybersecurity control to act on. The two are also consistent on detection: RF-level detection would be inside the boundary (receiver C-02, see RES-01), whereas the lock-confirmation feedback invoked here is entirely outside it — it is executed by the lighting / comfort functions (`01` §4).

**Open point:** a High risk retained with no control should name the authority that accepts it, and the out-of-boundary mitigation should be recorded as a requirement allocated outside the item rather than as a bare acceptance. Both are open points — see `OPEN-POINTS.md`.

**Traceability:** TS-02 → ACC-01 (no CSG, no CSR). Carried in `04_traceability-matrix.csv`.