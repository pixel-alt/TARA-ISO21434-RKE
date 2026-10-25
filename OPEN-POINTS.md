# Open points

What this version does not do, and why. An omitted item looks ignored; a recorded one looks decided.

## Planned

| Point | Why not in this version | Target |
|---|---|---|
| Full 5-factor attack-potential scoring of the 12 lines, on the Common Criteria CEM / ISO 18045 tables | The current scale assigns the level from the access means alone (see `02_tara-method.md` §4). Scoring properly requires the CEM tables and a documented points-to-level mapping. TS-04 and TS-06 (Expert + Specialized) are the first candidates to drop from Medium to Low | v0.3 |
| DS-05 "persistent undetected access", and remapping of TS-03, TS-04, TS-07, TS-09 | DS-01 currently absorbs two different consequences: a one-off break-in and a valid key held for the life of the vehicle. The second is not a different attack path, it is a different damage | v0.3 |
| AS-08 — Security Access secret: asset, threat line, fleet-wide scope | Introduced by CSR-HW-02 in this version. CSG-05 now closes the debug path that would read it, but the fleet-wide scope stays open: a 0x27 secret is typically per model, not per vehicle, so the scale effect must be expressed through feasibility (Knowledge of the item shifts to Public once the secret leaks) | v0.3 |
| AS-09 — firmware signing key (introduced by CSG-06) | CSG-06 relies on an OEM firmware-signing key pair; the private key is fleet-wide and its compromise is catastrophic. Its management lives at the OEM backend, out of scope here like A-02, but must be named as an asset | v0.3 |
| UDS 0x29 (Authentication, ISO 14229-1:2020) as a certificate-based alternative to 0x27 seed/key | Follows from AS-08: it removes the shared secret rather than protecting it | v0.3 |
| TS-09b — substituting a secret under an unchanged Key ID | TS-09 overwrites the whole record, so the Key ID changes and F-04 would show it. Substituting only the secret changes nothing visible | v0.3 |
| Forward desynchronisation of the counter (write it to its maximum via I-06b), denying the RKE permanently | Same asset and interface as TS-10 but a different damage scenario (DS-02, not DS-01), so it is a distinct line | v0.3 |
| F-05 behaviour beyond the look-ahead window | The item definition describes acceptance inside the window but not the resynchronisation procedure when the fob counter has run past it. An attacker holding the fob briefly could desynchronise it with no tooling | v0.3 |
| End of life and resale: key revocation, secret erasure, log purge | `01` §5 declares these lifecycle phases but no threat or requirement follows from them. The previous owner keeping a paired fob costs the attacker nothing (ISO/SAE 21434 clause 14) | v0.3 |
| Reference baseline to make F-04 useful | F-04 lists paired keys, but without a trusted baseline recorded at delivery, neither an extra entry nor a changed Key ID is recognisable as abnormal | v0.3 |
| The key-management record as an asset (integrity; personal data) | Created by CSR-SW-03 in this version, not yet in the asset register | v0.3 |
| Tree-based attack-path analysis (clause 15.6) | `02_tara-method.md` §9 records the dependencies in prose; the lines are still rated individually | v0.3 |
| RSSI-based jamming detection at the receiver | Noted as the only possible detection for RES-01. Setting a threshold in the crowded 433 MHz ISM band requires a measurement campaign outside this study | v0.3 |
| UN R155 / CSMS mapping | Depends on the above being stable | v1.0 |

## Findings examined and rejected

| Finding | Decision |
|---|---|
| Split TS-01 into "simple replay" and "capture-and-delayed-replay" | **Rejected.** Simple replay has no realistic scenario — an attacker who replays a code the vehicle accepted has seen the vehicle react. One broadened line, two goals (CSG-01 freshness, CSG-04 command binding) and a documented residual is simpler and more honest |
| Declassify CSG-01 into a standalone robustness requirement | **Rejected.** A goal without a parent risk breaks the `risk → goal → requirement` rule this study applies elsewhere. CSG-01 keeps TS-01 as its source and is recorded as a partial reduction |
| Retain TS-07..TS-11 on their own Low risk value instead of reducing them | **Rejected.** Each is Low, so retention would be defensible in isolation; but closing the debug path is a precondition of CSG-02 (`02_tara-method.md` §9), so the dependency justifies CSG-05 regardless of the individual risk value |
| Treat TS-12 by Accept because a single capture is cheap | **Rejected.** TS-12 is Major × High = High; a High privacy risk with a technically available countermeasure (rotating identifier, no new hardware) cannot be accepted under GDPR art. 25. It is reduced by CSG-07 |
| Rate the fleet-wide impact of a shared diagnostic secret on the Financial axis (recall cost) | **Rejected.** ISO/SAE 21434 rates impact from the road user's perspective, not the manufacturer's. The scale effect belongs in feasibility, not impact |
| Treat RSSI jamming detection as a control in this version | **Rejected as a control, recorded as a lead.** The threshold cannot be justified on a fictional item without measurement |
| Add a `residual_risk` column to `02_tara.csv` | **Rejected.** Residual risk only exists once verification has been executed, and `04_traceability-matrix.csv` already carries the verification status. A column empty for two versions is noise; the convention is stated in `02_tara-method.md` §8 instead |