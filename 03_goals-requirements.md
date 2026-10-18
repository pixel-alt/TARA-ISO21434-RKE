# 03 – Goals & Requirements

**Scope:** cybersecurity goals and requirements derived from `02_tara.csv`
**Status:** fictional case study – generic passenger vehicle, no real OEM
**v0.1:** only the replay chain (TS-01) is developed end-to-end. The other rated risks will get their goals in a later version.

A cybersecurity **goal** states *what* must be prevented (the effect), not *how*. The **concept** names the control and where it sits. The **requirements** (CSR) make the concept testable, allocated to system (SYS), software (SW) and hardware (HW).

```
risk (TS) → goal (CSG) → concept → requirements (CSR-SYS / CSR-SW / CSR-HW) → verification
```

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