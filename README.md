# TARA-ISO21434-RKE

A system-level **Threat Analysis and Risk Assessment (TARA)**, inspired by
ISO/SAE 21434, of a fictional **Remote Keyless Entry (RKE)** system.

> **Work in progress.** Open points are listed in `OPEN-POINTS.md`.

> Learning portfolio project — fictional target, no real OEM. Built to show a
> structured, traceable and defensible risk analysis, not an exhaustive
> manufacturer deliverable.

## Why this repo

Entry-level automotive product-security work turns on one skill: taking a
function, finding what can go wrong, and tracing each risk down to a testable
requirement. This repo does exactly that on an RKE system — from the item
definition to the full set of worked risk families (replay, key management,
physical debug, owner tracking), one of them carrying a documented residual.

## Method (inspired by ISO/SAE 21434)

```
asset + cybersecurity property → damage scenario → threat scenario
  → impact (4 axes, max) → attack path → attack feasibility
  → risk value → risk treatment → cybersecurity goal
  → requirements (SYS/SW/HW) → verification
```

- Impact rated on the four axes Safety / Financial / Operational / Privacy,
  retained value = maximum.
- Feasibility assigned from the access means; the five attack-potential factors
  (time, expertise, knowledge, window, equipment) are recorded as written
  justification. A full 5-factor scoring on the Common Criteria CEM / ISO 18045
  tables is planned for v0.3 — see `02_tara-method.md` §4.
- Risk = impact × feasibility via a documented matrix.
- The CSV is the single source of truth for the threat scenarios;
  `02_tara-method.md` defines every scale, asset and damage scenario it
  references.

## Files

| File | Content | Clause |
|---|---|---|
| `01_item-definition.md` | Item, boundary, interfaces, functions, assumptions | 9.3 |
| `02_tara.csv` | Threat scenarios, one row each (asset → risk) | 15.3–15.9 |
| `02_tara-method.md` | Asset register, damage scenarios, rating scales, risk matrix, exclusions, dependencies | 15.3–15.8 |
| `03_goals-requirements.md` | Cybersecurity goals, concepts and requirements; residual and accepted risks | 9.4–9.5 (treatment decision: 15.9) |
| `04_traceability-matrix.csv` | risk → goal → requirement → verification | 9.5 / 15.9 |
| `05_test-spec-CSR-SYS-01.md` | Test specification for the replay chain | 9.5 (verification) |
| `OPEN-POINTS.md` | What is not done and why; findings examined and rejected | — |

## Current state

- **Item definition:** complete — boundary, 7 interfaces (I-06 split into I-06a
  key fob / I-06b BCM because their access means differ), 6 functions,
  **12 assumptions**.
- **Threat scenarios:** 12 identified across four attack surfaces — radio
  (capture-and-jam replay, denial jamming, emission eavesdropping),
  diagnostic/CAN (key pairing and revocation via OBD or exterior harness),
  physical debug (secret, counter, firmware access).
- **Impact:** rated on all four axes for the 4 damage scenarios.
- **Rated and treated: 12 of 12**, with **7 goals** and **14 requirements**:
  - **The replay chain — TS-01** (replay of a code the vehicle never received):
    Major × High → High → Reduce (partial) → **CSG-01** (counter freshness, bounds
    the replay in time) + **CSG-04** (binds the command to the authentication
    code, closing the lock→unlock variant), with **RES-01** carrying the
    irreducible remainder, and a test specification (`05`).
  - **The key-management family — TS-03 to TS-06** (pair or revoke a key, each
    via OBD diagnostic and via direct CAN injection on the exterior harness):
    pairing reaches DS-01 (Major), revocation reaches DS-02 (Moderate), all →
    Medium → Reduce → **CSG-02** (authorization, service-level so it is
    path-independent) + **CSG-03** (accountability). CSR-HW-02's debug dependency
    is discharged by CSG-05.
  - **The physical-access family — TS-07 to TS-11** (read the fob secret; read /
    write the BCM memory; roll the counter back; flash a firmware — all via the
    debug interface): Major × Low → Low → Reduce → **CSG-05** (protect the debug
    interfaces, CSR-HW-05), reduced not by their own Low risk but because closing
    the debug path is a precondition of CSG-02. TS-11 (firmware flash) is reduced
    jointly by **CSG-06** (secure boot, CSR-SW-06 + CSR-HW-06), which rejects a
    modified firmware whatever the installation path and discharges A-11.
  - **Owner tracking — TS-12** (radio emission correlatable across transmissions
    — the Key ID *and* the clear-text monotonic counter): Major × High → High →
    Reduce → **CSG-07** (non-correlatable emission: the counter is moved inside an
    authenticated-encrypted payload, GDPR art. 25).
  - **One accepted risk — TS-02** (denial jamming) → **ACC-01**, justified.

## Roadmap

| Version | Scope |
|---|---|
| v0.1 | Item definition, 12 threat scenarios, first replay chain |
| v0.2 | A-05 and CSG-01 corrected (partial reduction + RES-01), CSG-03 accountability, feasibility definitions fixed, test specification, open points recorded |
| v0.2.x | All 12 scenarios rated and treated: CSG-04 (command binding), CSG-05 (debug protection), CSG-06 (secure boot), CSG-07 (non-correlatable emission over every observable field) |
| v0.3 | Full 5-factor scoring of all lines; DS-05; AS-08 (Security Access secret) and AS-09 (firmware signing key); CSG-07 construction/nonce and fob-side hardware allocation; physical-layer fingerprinting — contents of `OPEN-POINTS.md` |
| v1.0 | UN R155 / CSMS mapping |

## Limits

Fictional target and simulated environment. Not a conformity assessment.

Ratings use conventions documented in `02_tara-method.md` (e.g. impact
aggregation by maximum, access-means-dominant feasibility) that a real programme
would agree with stakeholders.

Several lines rely on explicitly stated assumptions — A-06 for tracking, A-07 and
A-12 for the debug paths, A-10 for the command-field replay, A-11 for the
firmware path, A-09 for the impact ceiling. If those do not hold, the
corresponding lines must be reassessed; A-09 in particular would raise the
Financial axis to Severe and re-rate the whole item.

CSG-07 bounds correlation at frame level only; physical-layer fingerprinting of
the fob transmitter is a documented limit, not a covered threat.

No verification has been executed: every requirement is at status *Planned*, and
`05_test-spec-CSR-SYS-01.md` is a specification, not a test report.