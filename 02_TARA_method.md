# 02 – TARA Method

**Scope:** supporting document for `02_tara.csv`
**Status:** fictional case study – generic passenger vehicle, no real OEM

This file holds what the CSV references but does not spell out: the asset register, the damage-scenario definitions, the rating scales, the risk matrix, the justified exclusions and the column conventions. The CSV is the single source of truth for the threat scenarios; this file is the single source of truth for everything they point to.

## 1. Asset register

Assets are the items of value whose compromise leads to a damage scenario. An asset carries only the cybersecurity properties whose loss produces a retained damage — not C, I and A by default.

| asset_id | Asset | Properties | Property rationale |
|---|---|---|---|
| AS-01 | Key secret K (key fob) | C | Disclosure lets an attacker clone the fob. |
| AS-02 | Key secrets stored in the BCM (all paired keys), including their association to each Key ID | C, I | Disclosure clones any key (C); overwriting a stored record (Key ID + secret) makes an attacker key accepted (I). |
| AS-03 | Stored rolling-code counter (BCM) | I | Rolling it back re-validates a previously captured code (I). |
| AS-04 | Paired-key list | I | Adding an attacker key or revoking the owner's keys changes who can operate the vehicle (I). |
| AS-05 | Lock/unlock and trunk release function | I, A | Unauthorized actuation (I); jamming the radio channel prevents a commanded action from executing, so the vehicle is left in a state the owner did not intend (A) — see DS-04. |
| AS-06 | Firmware (BCM) | I | A backdoored firmware bypasses the RKE logic (I). Confidentiality is not retained — see exclusions. |
| AS-07 | Key ID (fob identifier, sent in clear on I-01) | C | The value is observable by design; the property protects against the disclosure of private information enabled by correlating this persistent identifier across uses (presence, absence, habits), not the secrecy of the ID value. See DS-03. |

## 2. Damage scenarios

A distinct damage scenario is justified only when the consequence **for the affected party** differs from the others — not when the attack path differs. Several threat scenarios may therefore share one DS.

Impact is rated on the four ISO/SAE 21434 axes (Safety, Financial, Operational, Privacy), each on the scale Severe / Major / Moderate / Negligible. Each axis is rated at the **worst credible case**, not the average case. The **retained impact** is the maximum across the four axes.

| ds_id | Damage scenario | S | F | O | P | Retained | Rationale per axis |
|---|---|---|---|---|---|---|---|
| DS-01 | Parked vehicle actively opened (doors or trunk) by an unauthorized person; contents may be stolen (engine start blocked by the immobilizer, `01` A-09) | Neg | Major | Neg | Mod | **Major** | F: worst credible case — a valuable item (laptop, bag) is stolen = significant loss; the vehicle itself cannot be driven away (A-09). P: a document in the cabin (registration → home address) may be read once = one-off disclosure, no durable profile. S/O not affected. |
| DS-02 | Owner can no longer lock or unlock the vehicle with the key fob | Neg | Mod | Mod | Neg | **Moderate** | O: a mechanical key exists (`01`, A-08) → mechanical fallback available, so Moderate, not "locked out until workshop" (Major). F: key reprogramming / workshop cost = limited loss. No data exposed (P Neg) — this is what distinguishes DS-02 from DS-01. |
| DS-03 | Owner's key fob is recognizable at each use; repeated eavesdropping lets an observer infer presence, absence and travel habits (persistent identifier) | Neg | Neg | Neg | Major | **Major** | P: repeated eavesdropping of the persistent Key ID (A-06) lets a determined observer infer habits. **Major and not Severe**: the radio range (~10–50 m, `01` §5) limits tracking to a local observer (home, workplace), with no large-scale persistent tracking. The P is legitimately high because the clear-text Key ID is an emission of the RKE function (asset AS-07). |
| DS-04 | Vehicle left unlocked without the owner's knowledge: the owner believes the vehicle is secured while it remains accessible with no defeat of the electronic lock; contents may then be stolen | Neg | Major | Neg | Mod | **Major** | S: Negligible — ISO 21434 rates Safety via vehicle behaviour (ISO 26262 link), not via a downstream third-party crime. F: theft of contents, same worst credible case as DS-01 (the same damage cannot be rated differently). O: the vehicle functions normally. P: same logic as DS-01. |

**DS-01 vs DS-04:** the difference is the state of the vehicle, which changes the consequence for the owner, not merely the attack path. In DS-01 the vehicle was locked and its security was defeated; in DS-04 the vehicle was never locked and no defeat of the electronic lock took place — the attacker exploits a door left unlocked, and the owner is unaware the vehicle is unsecured. The two also call for different treatments, which is why they are kept separate even though both reach a retained impact of Major.

**Aggregation rule:** ISO/SAE 21434 does not mandate collapsing the four axes into a single value. This study retains the **maximum** axis as the single `impact_rating` reported in the CSV, a common convention. The per-axis ratings above are kept so the choice stays auditable.

**No Severe case:** Safety stays Negligible on every DS — the RKE touches access only, never vehicle behaviour (engine start is blocked by the immobilizer, `01` A-09). This caps the item at Major and is why no DS reaches Severe. If A-09 does not hold, the Financial axis reaches Severe and every theft-related DS must be re-rated.

**`impact_category` rule:** the CSV `impact_category` names the axis carrying the retained (maximum) impact. On a tie, it names the axis that reflects the scenario's nature — for DS-02, where F and O both reach Moderate, Operational is chosen (the scenario is a denial of the lock/unlock function).

**Open point:** DS-01 currently absorbs both one-off opening and persistent covert access (an attacker key that stays enrolled). A dedicated damage scenario for persistent undetected access is an open point — see `OPEN-POINTS.md`.

## 3. Impact rating scale

Impact is rated per the four ISO/SAE 21434 categories (Safety, Financial, Operational, Privacy), from the point of view of the road user. The Safety axis is defined for completeness but is not exercised in this study (all DS rate Safety Negligible — see §2).

| Rating | Safety | Financial | Operational | Privacy |
|---|---|---|---|---|
| Severe | Life-threatening or fatal injury | Loss comparable to the vehicle value or its full contents | Vehicle permanently unusable | Persistent, re-identifiable tracking of the owner |
| Major | Severe injury (survival probable) | Significant loss (valuable contents stolen) | Owner locked out until workshop intervention | Habits inferable over time by a determined observer |
| Moderate | Light to moderate injury | Limited loss: minor contents, or a repair / service cost borne by the owner | Temporary loss of RKE, mechanical fallback available | One-off observation, no lasting profile |
| Negligible | No injury | No meaningful loss | Minor inconvenience | No meaningful privacy effect |

## 4. Attack feasibility scale

Feasibility stands for the effort an attack demands: the harder the attack, the lower the feasibility, the lower the risk. Each **rated** line records its reasoning in `feasibility_rationale`; lines not yet rated carry none.

| Level | Meaning | Typical marker in this study |
|---|---|---|
| High | Low effort, common equipment, no prior physical breach | Radio capture/replay from range, key ID eavesdropping |
| Medium | Moderate effort or limited physical access | Exterior harness connector; OBD access inside the cabin |
| Low | Specialist effort, invasive physical access | Opening the BCM and soldering onto the debug interface |

This ordering is the reason the debug-access lines must not receive the same feasibility as the harness lines: they demand opening the BCM, where the harness path only needs an exterior connector.

**Per-factor scales.** The rationale cites one value per factor; each is read from these ladders (easiest → hardest).

| Factor | Scale (easiest → hardest) | Selection rule |
|---|---|---|
| Elapsed time | ≤1 day → ≤1 week → ≤1 month → ≤6 months → >6 months | time of the slowest step (prepare + execute) |
| Specialist expertise | Layman → Proficient → Expert → Multiple experts | highest level any single step requires |
| Knowledge of the item | Public → Restricted → Confidential → Strictly confidential | most sensitive information **about this item** (specification, protocol, design, service procedures) the attack needs. The public notoriety of the attack *technique* is not rated here — it belongs to Specialist expertise and Elapsed time |
| Window of opportunity | Unlimited → Easy → Moderate → Difficult | number of exploitable occasions and access conditions (not duration) |
| Equipment | Standard → Specialized → Bespoke → Multiple bespoke | availability and cost to obtain (not ease of use) |

**Scale limitation.** The level is assigned from the *access means* alone (radio without intrusion / limited physical access / invasive physical access). The five factors above are recorded as written justification and **none of them determines the level**. Consequence: TS-02 (Layman, off-the-shelf jammer) and TS-01 (Proficient, specialised SDR, narrow window) both land at High. This is an **access-means-dominant scale, not a full attack-potential scoring**. A complete 5-factor scoring on the Common Criteria CEM / ISO 18045 attack-potential tables — the approach ISO/SAE 21434 Annex G derives from — is planned for v0.3 (see `OPEN-POINTS.md`); the normative Annex G point table is not reproduced here. The simplification is assumed and documented rather than hidden.

## 5. Risk matrix

Risk = f(impact, feasibility). Feasibility stands for likelihood: the harder the attack (Low), the less probable, the lower the risk.

| Impact \ Feasibility | High | Medium | Low |
|---|---|---|---|
| Severe | Very High | High | Medium |
| Major | High | Medium | **Low** |
| Moderate | Medium | Medium | Low |
| Negligible | Low | Low | Low |

Note: Major × Low = Low (not Medium) because a Low-feasibility attack is improbable; a Major impact that is improbable remains a low risk. The matrix is monotone (risk never decreases when impact or feasibility rises).

**Reading rules (to be defensible under audit):**
- **Monotonicity:** risk never decreases when impact or feasibility rises. Re-check this after any cell revision.
- **Feasibility stands for likelihood:** Low means improbable, so a Major-impact attack that is improbable stays a low risk (Major × Low = Low).
- **Critical corner:** high impact combined with an easy attack is the top priority (Severe × High = Very High).
- **The debug-access lines.** They are expected to rate Low on feasibility, since the access means is invasive (opening the BCM, soldering onto the debug interface). Should they, the Major × Low cell places them at the bottom of the priority list — which is defensible here because no fleet-wide master key protects the RKE function (`01` §1), so a debug compromise is bounded to one vehicle. It would not be defensible on an item holding a secret shared across a model range, which is why the Security Access secret introduced in `03` is tracked as an open point.
- **A measure can be justified by a dependency rather than by its own risk value.** The key-management hardware requirement in `03` closes the debug path because all four goals depend on it (§9), independently of how the debug lines rate. The risk value prioritises treatment; it does not command a structural precondition.

*Revision (see CHANGELOG): Major × Low changed from Medium to Low; the change restores discrimination between Medium-feasibility (CAN) and Low-feasibility (debug) lines.*

## 6. Justified exclusions

Excluding an attack is defensible only when the reason is stated. An omitted threat looks ignored; an excluded one looks decided.

| Excluded element | Reason |
|---|---|
| I-02 (LIN, receiver → BCM) | Excluded in this version. The analysis prioritises the external RF attack surface and the diagnostic CAN paths. Injection on the internal LIN interface is not assessed separately; its distinct attack potential, and exactly where the BCM performs frame validation relative to the LIN, would require further investigation. |
| Actuator command (I-03) | Power output, not a message. Driving the actuator directly is mechanical break-in, outside the cyber perimeter; no cybersecurity control applies. |
| Key-fob counter modification | Pointless **towards an unauthorized opening**: an attacker who holds the fob (I-06a) can already open the vehicle by pressing the button. Forward desynchronisation, which denies the function to the owner rather than opening the vehicle, is a different damage and is an open point (see `OPEN-POINTS.md`). |
| Firmware confidentiality | No fleet-wide master key protects the RKE function (see `01`, §1): a firmware dump reveals no *key secret* usable against other vehicles. Security rests on the key, not on the secrecy of the algorithm (Kerckhoffs). Firmware integrity is retained (AS-06). **Note:** the Security Access secret introduced in `03` is a distinct asset, not covered by this exclusion; its confidentiality and its fleet-wide scope are an open point. |

## 7. Limits (analysed or bounded, not excluded)

These are not exclusions: the threat is real and kept in scope, but the point worth stating is where its treatment lies or why the analysis stops at the RKE boundary.

- **Denial jamming of I-01.** Jamming the lock command so the BCM receives no frame cannot be prevented by any RKE cybersecurity control — the attack removes the channel rather than forging a message (RED / UN R10 territory, not ISO 21434). It is kept in scope through its consequence DS-04; its only mitigation, lock-confirmation feedback, is out of boundary. This is distinct from capture-and-jam, where the replayed frame *is* processed: there the counter bounds the attack in time (see `03`, CSG-01) and binding the command to the authentication code closes the damage path (see `03`, CSG-04), with RES-01 carrying what remains.
- **CAN-wide denial (e.g. bus-off attack).** Forcing the body CAN into a bus-off error state would deny communication on the whole bus. This is a distinct mechanism from radio jamming, not a transposition of it. It is not assessed in this version because it targets the network as a whole, not the RKE function specifically; it would belong to a vehicle-network-level analysis.

## 8. Traceability chain and column conventions

Each CSV line is meant to be read as one chain, in clause order:

```
asset + cybersecurity property → damage scenario → threat scenario
  → impact (4 axes, max) → attack path → attack feasibility
  → risk value → risk treatment → cybersecurity goal
  → requirements (SYS/SW/HW) → verification
```

**`cs_goal` semantics.** In both `02_tara.csv` and `04_traceability-matrix.csv`, the column carries the identifier that receives the treatment decision: a cybersecurity goal (`CSG-xx`), a named residual (`RES-xx`) or a recorded retention (`ACC-xx`). `N/A` in this column means no treatment has been decided yet. In the requirement and verification columns, `N/A` means no requirement is built — which is the case by decision for a residual or a retention (see `03`). `goal_statement` carries the statement of whatever `cs_goal` names: the goal, the residual or the retention.

**Empty cell:** the value has not yet been produced (the line is not yet rated). This is distinct from `N/A`, whose per-column meaning is given above.

**Treatment terminology.** The options used are those of ISO/SAE 21434 — *avoiding, reducing, sharing, retaining*. This study uses `Reduce` and `Retain`. The `ACC-xx` identifiers predate this wording and denote a retained risk.

**Which controls belong to the item.** The frame authentication code and the rolling-code counter are properties of the item as defined (`01` §3), so they are in force when the initial risk is rated here. A goal that restates one of them refines its specification rather than creating a control; a goal that introduces something `01` does not provide is a treatment proper. `03` states which is which for each goal.

**Initial vs residual risk.** `risk_value` in the CSV is the **initial** risk, rated with the assumptions of `01` in force and without the controls of `03`. Residual risk is recorded in the `residual_risk` column of `04_traceability-matrix.csv` once verification has been executed. No verification has run in this version, so that column is filled only where no verification is awaited: a retained risk, whose initial value *is* its residual by decision.

Goals and requirements derive from the risks that need treatment, and live in `03_goals-requirements.md`.

## 9. Dependencies between lines

Lines are rated individually. Two lines of the table condition others, plus one asset not yet registered: a backdoored BCM firmware (TS-11) makes CSG-01, CSG-02, CSG-03 and CSG-04 inoperative at once; reading the stored secrets (TS-08) makes TS-07 superfluous; and extracting the diagnostic authorization secret — an asset tracked as an open point, not a line of this table — would open TS-03 and TS-04 without any authorized access. Closing the debug path (I-06b) is therefore a precondition common to all four goals, the dependency already noted on the key-management hardware requirement in `03`. A tree-based attack-path analysis (clause 15.6) is planned for v0.3.