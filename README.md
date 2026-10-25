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
definition to four worked risk families (replay, key management, physical
debug, owner tracking), one of them carrying a documented residual.

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
- **Threat scenarios:** 12 identified across three attack surfaces — radio
  (capture-and-jam replay, denial jamming, key-ID eavesdropping),
  diagnostic/CAN (key pairing and revocation via OBD or exterior harness),
  physical debug (secret, counter and firmware access).
- **Impact:** rated on all four axes for the 4 damage scenarios.
- **Rated and treated: 12 of 12**, with **7 goals** and **14 requirements**:
  - **The replay chain — TS-01**: Major × High → High → Reduce (partial) →
    **CSG-01** (counter freshness) + **CSG-04** (command binding), with **RES-01**
    carrying the irreducible remainder, and a test specification (`05`).
  - **The key-management family — TS-03 to TS-06**: Medium → Reduce → **CSG-02**
    (authorization) + **CSG-03** (accountability).
  - **The physical-access family — TS-07 to TS-11** (debug read/write of secrets,
    key records, counter and firmware): Low → Reduce → **CSG-05** (debug-lock),
    TS-11 also **CSG-06** (secure boot, which discharges A-11).
  - **Owner tracking — TS-12** (Key ID eavesdropping): Major × High → High →
    Reduce → **CSG-07** (non-correlatable identifier, GDPR art. 25).
  - **One accepted risk — TS-02** (denial jamming) → **ACC-01**, justified.
- All 12 threat scenarios are rated and treated. Remaining work (full 5-factor
  scoring, DS-05, AS-08/AS-09) is in `OPEN-POINTS.md`.

## Roadmap

| Version | Scope |
|---|---|
| v0.1 | Item definition, 12 threat scenarios, first replay chain |
| v0.2 | A-05 and CSG-01 corrected (partial reduction + RES-01), CSG-03 accountability, feasibility definitions fixed, test specification, open points recorded |
| v0.3 | Full 5-factor scoring of all lines; DS-05 (persistent access); AS-08 (Security Access secret) and AS-09 (firmware signing key) as assets — contents of `OPEN-POINTS.md` |
| v1.0 | UN R155 / CSMS mapping |

## Limits

Fictional target and simulated environment. Not a conformity assessment.

Ratings use conventions documented in `02_tara-method.md` (e.g. impact
aggregation by maximum, access-means-dominant feasibility) that a real programme
would agree with stakeholders.

Several lines rely on explicitly stated assumptions — A-06 for tracking, A-07 and
A-12 for the debug paths, A-10 for the command-field replay, A-09 for the impact
ceiling. A-11 (firmware integrity) is now discharged by CSG-06. If the remaining
assumptions do not hold, the corresponding lines must be reassessed; A-09 in
particular would raise the Financial axis to Severe and re-rate the whole item.

No verification has been executed: every requirement is at status *Planned*, and
`05_test-spec-CSR-SYS-01.md` is a specification, not a test report.