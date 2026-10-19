# Changelog

## [v0.1] – 2026-10-18

Initial public release.

### Added
- Item definition: boundary, interfaces, functions, assumptions (A-01 to A-07)
- 12 threat scenarios across radio, diagnostic/CAN and physical-debug surfaces
- Impact rated on 4 axes (S/F/O/P) for the 4 damage scenarios
- One risk chain fully worked end-to-end: TS-01 (RollJam) → CSG-01 → CSR-SYS/SW/HW
- Traceability matrix (TS-01 chain)

### Planned (v0.2)
- Feasibility and risk rating for the 11 remaining threat scenarios
- Risk treatment decisions, EBIOS RM view

## [v0.2] — en cours

### Changed
- Damage scenarios re-rated on the 4 axes at worst-credible-case; retained
  impacts: DS-01 Major, DS-02 Moderate (mechanical key fallback), DS-03 Major,
  DS-04 Major.
- Risk matrix: Major × Low changed from Medium to Low. Rationale: a
  low-feasibility attack is improbable; a Major impact that is improbable is a
  low risk. Restores discrimination between Medium-feasibility (CAN) and
  Low-feasibility (debug) lines.

### Added
- A-08: mechanical key backup (supports DS-02 = Moderate).