# Changelog

## [Unreleased]

### Added
- A-10 (the authentication code does not cover the command field), A-11 (no
  effective firmware authenticity / integrity protection) and A-12 (no readout
  protection on the fob debug path I-06a). A-07 now covers the BCM debug-access
  path only; A-11 carries the firmware side it used to share, A-12 is the
  fob-side counterpart of A-07.
- CSG-04 (bind the command to the authentication code) with CSR-SYS-04 /
  CSR-SW-04. It closes the lock→unlock variant of TS-01 that CSG-01 does not: the
  counter (CSG-01) bounds the replay in time, command binding (CSG-04) removes
  the command-flip, and RES-01 narrows to the replay of a genuinely captured
  unlock before the next legitimate press.
- CSG-05 (protect the debug interfaces I-06a / I-06b) with CSR-HW-05 (readout
  protection / debug lock at production). Treats TS-07 to TS-11; it is justified
  by the §9 dependency (closing the debug path is the precondition of CSG-02),
  not by the Low risk of each line, and it discharges the debug-path dependency
  on CSR-HW-02.
- CSG-06 (firmware authenticity / secure boot) with CSR-SW-06 (startup signature
  check) and CSR-HW-06 (root of trust in immutable storage). Treats TS-11 jointly
  with CSG-05: CSG-05 closes the injection path I-06b, CSG-06 rejects a modified
  firmware whatever the path. Discharges A-11.
- CSG-07 (non-correlatable radio emission) with CSR-SYS-07 / CSR-SW-07. Treats
  TS-12 (the second High risk of the study). The goal is stated over every
  observable field, not over "the identifier": rotating the Key ID alone would
  have left the clear-text monotonic counter as a linking field. The counter
  travels inside an authenticated-encrypted payload (freshness kept, read after
  tag verification), a non-correlatable nonce replaces it on the wire, and the
  verification criterion is a non-correlation test over all fields. Backs the
  GDPR art. 25 argument. Construction/nonce choice and the fob-side hardware
  allocation are open points; physical-layer fingerprinting is a documented
  limit.
- `02_tara-method.md` §5: explicit reading rule that a measure can be justified
  by a dependency, not only by its own risk value (supports CSG-05).

### Changed
- TS-04, TS-05 and TS-06 rated (feasibility Medium, risk Medium) and treated
  (Reduce → CSG-02 + CSG-03), completing the key-management family TS-03..TS-06.
  Pairing (TS-03/04) reaches DS-01 (Major), revocation (TS-05/06) reaches DS-02
  (Moderate); all four land at Medium. CSG-03's rationale now notes that for
  revocation the record gives attribution rather than detection. TS-04 and TS-06
  (Expert + Specialized) are flagged as candidates to drop to Low under the full
  5-factor scoring.
- TS-07, TS-08, TS-09, TS-10 and TS-11 rated (feasibility Low, risk Low) and
  treated (Reduce → CSG-05): the physical-access family (fob secret read; BCM
  secrets read; BCM key record overwritten; counter rolled back; firmware
  flashed). TS-11 also → CSG-06.
- TS-12 rated (feasibility High, risk High) and treated (Reduce → CSG-07): owner
  tracking by eavesdropping a correlatable radio emission.
- AS-07 and DS-03 restated from "persistent Key ID" to a correlatable radio
  emission: the Key ID and the monotonic rolling-code counter are each sufficient
  on their own to link successive transmissions, so the asset and the damage
  scenario now name both. TS-12's asset, threat scenario and attack path follow.
- TS-01 treatment is now CSG-01 + CSG-04 + RES-01; its attack path and
  `assumptions` reference A-10 (command-field flip).
- `02_tara-method.md` §7 and §9 updated to the seven goals; §9 now names CSG-05
  (closes the debug injection path) and CSG-06 (secure boot) as the two layers
  that defeat the firmware-replacement dependency, and records that CSG-01 and
  CSG-07 converge on one frame construction.
- `02_tara.csv`: TS-02 `cs_goal` set to ACC-01 (was N/A), aligning denial jamming
  with the decision-id convention used for RES-01.
- `04_traceability-matrix.csv`: rows added for CSG-04, CSG-05, CSG-06, CSG-07 and
  TS-04..TS-12; the CSR-HW-02 note now reads "discharged by CSG-05".
- README: current state is 12 of 12 rated, 7 goals, 14 requirements, 12
  assumptions; roadmap updated (all scenarios rated; v0.3 scoped to full scoring,
  DS-05, AS-08/AS-09 and the CSG-07 construction/fingerprinting open points).

## [0.2.0] — 2026-10-22

### Added
- A-08: mechanical key backup (supports DS-02 = Moderate).
- A-09: immobilizer prevents engine start without a paired key. Formalises the
  assumption that caps every theft-related damage scenario at Major.
- Convention line in 01: the item definition cites no TARA-produced identifier.
- 02 §4: per-factor attack-potential scales and a documented scale limitation.
- 02 §5: matrix reading rules (monotonicity, feasibility-as-likelihood, critical
  corner), plus the reason the Low-feasibility debug lines are defensible here
  (no fleet-wide master key).
- 02 §8: initial-vs-residual risk convention.
- 02 §9: dependencies between lines (TS-11 voids the preventive goals; TS-08 makes
  TS-07 superfluous; the diagnostic secret opens TS-03/TS-04). Replaces the need
  for an attack tree in this version.
- CSG-02 (authorize key-management operations: pairing F-03 / revocation F-06),
  with CSR-SYS-02 / CSR-SW-02 / CSR-HW-02. Covers TS-03 now and will cover
  TS-04 / TS-05 / TS-06. CSR-HW-02 notes the dependency on closing the I-06b
  debug path (cf. A-07 / TS-08..11).
- CSG-03 (make key-management operations accountable) with CSR-SW-03. First
  detective requirement of the study: without a record, the persistent covert
  access of TS-03 is undetectable for the life of the vehicle.
- RES-01: named residual risk of TS-01, with the reason no cryptographic control
  and no application-level detection apply.
- 02_tara.csv: new `assumptions` column.
- 05_test-spec-CSR-SYS-01.md: test specification for the replay chain, with a
  legitimate press inserted as the control case, conducted coupling, and a pass
  criterion observable on both the actuator and the diagnostic trace.
- OPEN-POINTS.md: what is not done and why, plus findings examined and rejected.

### Changed
- A-05 rewritten. It now states the design tolerance it actually models
  (resynchronisation after out-of-range presses) instead of attributing the
  capture-and-replay to the look-ahead window, which was wrong.
- CSG-01 requalified as a **partial** reduction of TS-01, with RES-01 as the
  remainder. A counter blocks a replay only if the stored counter has passed the
  captured value; jamming prevents exactly that, so no window width helps.
  Goal, rationale and CSR-SYS-01 rewritten accordingly.
- CSG-01 concept: SecOC is situated, not claimed — the freshness principle is
  shared, the mechanism assumes a bus and an AUTOSAR stack and is not
  transposable to a one-way UHF fob.
- 02 §4: `Knowledge of the item` redefined as information about *this item*, not
  the public notoriety of the attack technique. TS-02 applied it the other way
  round, which made the lines non-comparable.
- 02 §4: scale limitation widened — window of opportunity, expertise *and*
  equipment carry no weight in the level; full 5-factor CEM / ISO 18045 scoring
  moved to v0.3.
- 02 §7: the capture-and-jam note now says the cryptographic control bites only
  partially.
- 02 §8: traceability chain reordered to follow the clause order.
- 02_tara.csv: TS-01 rewritten (threat scenario, attack path, feasibility
  rationale); treatment = Reduce (partial).
- 02_tara.csv: TS-02 rated (attack_path, attack_feasibility = High) and treated
  (Accept); TS-03 rated (feasibility Medium, risk Medium) and treated (Reduce,
  cs_goal = CSG-02;CSG-03).
- 01 §6: GDPR constraint now cites art. 4(1) and art. 25 instead of the
  regulation as a whole.
- Damage scenarios re-rated on the 4 axes (S/F/O/P) at worst-credible-case.
  Retained impacts: DS-01 Major, DS-02 Moderate, DS-03 Major, DS-04 Major.
- Risk matrix: Major × Low changed from Medium to Low. Rationale: a
  low-feasibility attack is improbable; a Major impact that is improbable is a
  low risk. Restores discrimination between Medium-feasibility (CAN) and
  Low-feasibility (debug) lines.
- Files renamed to the lower-case convention: `02_TARA.csv` → `02_tara.csv`,
  `02_TARA_method.md` → `02_tara-method.md`.

### Fixed
- Removed the orphan claim on CSR-HW-01 ("mitigates counter rollback via debug,
  TS-10"), in both 03 and 04: TS-10 is neither rated nor traced, so the
  requirement claimed a risk that does not exist in the chain.
- 04: `verification_method` on CSR-SYS-01 and CSR-SW-01 now points to
  05_test-spec-CSR-SYS-01.md (the trace was one-way).
- Null-value convention unified on `N/A` (02_tara.csv TS-02 carried `None`).
- Assumption notes moved out of `feasibility_rationale` into the new
  `assumptions` column, and added to TS-08 / TS-09 / TS-10, which rest on the
  same premise without carrying it.
- 02 §4: "each CSV line records its reasoning" → "each **rated** line".
- DS-04 wording: "no intrusion" → "no defeat of the electronic lock".
- DS-02 reference corrected to 01/A-08 (was 01/§4).
- Removed TARA-produced identifiers from 01: threat-scenario IDs (§3) and
  damage-scenario IDs (A-08 "Impact if false", GDPR constraint). The item
  definition no longer references TARA outputs (no circular dependency).
- Version-specific wording in 02 (§6, §7) changed from "v0.1" to "this version".

## [0.1.0] — 2026-10-18
- Initial public release: item definition, 12 threat scenarios, one replay chain
  (TS-01 → CSG-01 → CSR SYS/SW/HW), traceability matrix.