# Changelog

## [Unreleased]

### Added
- A-08: mechanical key backup (supports DS-02 = Moderate).
- Convention line in 01: the item definition cites no TARA-produced identifier.
- 02 §4: per-factor attack-potential scales (Annex G) and documented scale
  limitation (access-means dominant, window of opportunity not separately weighted).
- 02 §5: matrix reading rules (monotonicity, feasibility-as-likelihood, critical corner).
- CSG-02 (authorize key-management operations: pairing F-03 / revocation F-06),
  with CSR-SYS-02 / CSR-SW-02 / CSR-HW-02. Covers TS-03 now and will cover
  TS-04 / TS-05 / TS-06. CSR-HW-02 notes the dependency on closing the I-06b
  debug path (cf. A-07 / TS-08..11).

### Changed
- Damage scenarios re-rated on the 4 axes (S/F/O/P) at worst-credible-case.
  Retained impacts: DS-01 Major, DS-02 Moderate, DS-03 Major, DS-04 Major.
- Risk matrix: Major × Low changed from Medium to Low. Rationale: a
  low-feasibility attack is improbable; a Major impact that is improbable is a
  low risk. Restores discrimination between Medium-feasibility (CAN) and
  Low-feasibility (debug) lines.
- 02_tara.csv: TS-02 rated (attack_path, attack_feasibility = High,
  feasibility_rationale) and treated (Accept).
- 02_tara.csv: TS-03 rated (feasibility Medium, risk Medium) and treated
  (Reduce, cs_goal = CSG-02).
- 04_traceability-matrix.csv: TS-02 (Accept) and TS-03 → CSG-02 chains added.

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
  replay chain (TS-01 → CSG-01 → CSR SYS/SW/HW), traceability matrix.