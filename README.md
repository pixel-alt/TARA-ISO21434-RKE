# TARA-ISO21434-RKE

A system-level **Threat Analysis and Risk Assessment (TARA)**, inspired by
ISO/SAE 21434, of a fictional **Remote Keyless Entry (RKE)** system.

> Learning portfolio project — fictional target, no real OEM. Built to show a
> structured, traceable and defensible risk analysis, not an exhaustive
> manufacturer deliverable.

## Why this repo

Entry-level automotive product-security work turns on one skill: taking a
function, finding what can go wrong, and tracing each risk down to a testable
requirement. This repo does exactly that on an RKE system — from the item
definition to one fully worked risk chain (replay / RollJam), goal and
requirements included.

## Method (inspired by ISO/SAE 21434, clause 15)

```
asset → cybersecurity property (C/I/A) → damage scenario → impact (4 axes, max)
      → threat scenario → attack path → attack feasibility → risk (matrix)
      → treatment → cybersecurity goal → requirements (SYS/SW/HW) → verification
```

- Impact rated on the four axes Safety / Financial / Operational / Privacy,
  retained value = maximum.
- Feasibility rated with the attack-potential approach (time, expertise,
  knowledge, window, equipment).
- Risk = impact × feasibility via a documented matrix.
- The CSV is the single source of truth; `02_tara-method.md` defines every
  scale, asset and damage scenario it references.

## Files

| File | Content | Clause |
|---|---|---|
| `01_item-definition.md` | Item, boundary, interfaces, functions, assumptions | 9 |
| `02_tara.csv` | Threat scenarios, one row each (asset → risk) | 15 |
| `02_tara-method.md` | Asset register, damage scenarios, rating scales, risk matrix, exclusions | 15.3–15.7 |
| `03_goals-requirements.md` | Cybersecurity goal + requirements (replay chain) | 15.8–15.9 |
| `04_traceability-matrix.csv` | risk → goal → requirement → verification | 15.9 |

## Current state — v0.1

- **Item definition:** complete (boundary, 7 interfaces, 6 functions, 7 assumptions).
- **Threat scenarios:** 12 identified across three attack surfaces — radio
  (RollJam, denial jamming, key-ID eavesdropping), diagnostic/CAN (key pairing
  and revocation via OBD or exterior harness), physical debug (secret, counter
  and firmware access).
- **Impact:** rated on all four axes for the 4 damage scenarios.
- **One risk chain fully worked end-to-end — TS-01 (RollJam):**
  impact Major × feasibility High → risk High → Reduce → goal CSG-01 →
  requirements CSR-SYS/SW/HW → verification planned (bench replay).
- **The other 11 threat scenarios are identified and impact-rated, but their
  feasibility and risk are not yet rated** (planned for v0.2).

## Roadmap

| Version | Scope |
|---|---|
| v0.1 | Item definition, 12 threat scenarios, one full replay chain |
| v0.2 | Feasibility + risk for all scenarios, risk treatment, EBIOS RM view |
| v0.3 | Goals, concept and requirements for every risk to reduce; full traceability matrix |
| v1.0 | UN R155 / CSMS mapping |

## Limits

Fictional target and not a conformity assessment.··
Ratings use conventions documented in `02_tara-method.md` (e.g. impact aggregation by maximum) that a real programme would agree with stakeholders. ··
Several scenarios rely on explicitly stated assumptions (A-05 for RollJam, A-06 for tracking, A-07 for the firmware path); if those do not hold, thecorresponding lines must be reassessed.