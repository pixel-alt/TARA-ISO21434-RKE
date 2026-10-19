# Changelog

## [Unreleased]

### Changed
- Damage scenarios re-rated on the 4 axes (S/F/O/P) at worst-credible-case.
  Retained impacts: DS-01 Major, DS-02 Moderate, DS-03 Major, DS-04 Major.
- Risk matrix: Major × Low changed from Medium to Low. Rationale: a
  low-feasibility attack is improbable; a Major impact that is improbable is a
  low risk. Restores discrimination between Medium-feasibility (CAN) and
  Low-feasibility (debug) lines.

### Added
- A-08: mechanical key backup (supports DS-02 = Moderate).
- Convention line in 01: the item definition cites no TARA-produced identifier.

### Fixed
- DS-04 wording: "no intrusion" → "no defeat of the electronic lock".
- DS-02 reference corrected to 01/A-08 (was 01/§4).
- Removed TARA-produced identifiers from 01: threat-scenario IDs (§3) and
  damage-scenario IDs (A-08 "Impact if false", GDPR constraint). Also removed
  TS IDs from 02 (§4, §6, §7). The item definition no longer references TARA
  outputs (no circular dependency).
- Version-specific wording in 02 (§6, §7) changed from "v0.1" to "this version".

## [0.1.0] — 2026-10-18
- Initial public release: item definition, 12 threat scenarios, one full
  replay chain (TS-01 → CSG-01 → CSR SYS/SW/HW), traceability matrix.g