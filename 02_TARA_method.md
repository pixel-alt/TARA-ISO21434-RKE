# 02 – TARA Method

**Scope:** supporting document for `02_tara.csv`
**Status:** fictional case study – generic passenger vehicle, no real OEM

This file holds what the CSV references but does not spell out: the asset register, the damage-scenario definitions, the rating scales, the risk matrix, and the justified exclusions. The CSV is the single source of truth for the threat scenarios; this file is the single source of truth for everything they point to.

## 1. Asset register

Assets are the items of value whose compromise leads to a damage scenario. An asset carries only the cybersecurity properties whose loss produces a retained damage — not C, I and A by default.

| asset_id | Asset | Properties | Property rationale |
|---|---|---|---|
| AS-01 | Key secret K (key fob) | C | Disclosure lets an attacker clone the fob. |
| AS-02 | Key secrets stored in the BCM (all paired keys), including their association to each Key ID | C, I | Disclosure clones any key (C); overwriting a stored record (Key ID + secret) makes an attacker key accepted (I). |
| AS-03 | Stored rolling-code counter (BCM) | I | Rolling it back re-validates a previously captured code (I). |
| AS-04 | Paired-key list | I | Adding an attacker key or revoking the owner's keys changes who can operate the vehicle (I). |
| AS-05 | Lock/unlock and trunk release function | I, A | Unauthorized actuation (I); loss of the radio channel denies the function (A). |
| AS-06 | Firmware (BCM) | I | A backdoored firmware bypasses the RKE logic (I). Confidentiality is not retained — see exclusions. |
| AS-07 | Key ID (fob identifier, sent in clear on I-01) | C | The value is observable by design; the property protects against the disclosure of private information enabled by correlating this persistent identifier across uses (presence, absence, habits), not the secrecy of the ID value. See DS-03. |

## 2. Damage scenarios

A distinct damage scenario is justified only when the consequence **for the affected party** differs from the others — not when the attack path differs. Several threat scenarios may therefore share one DS.

Impact is rated on the four ISO/SAE 21434 axes (Safety, Financial, Operational, Privacy), each on the scale Severe / Major / Moderate / Negligible. Each axis is rated at the **worst credible case**, not the average case. The **retained impact** is the maximum across the four axes.

| ds_id | Damage scenario | S | F | O | P | Retained | Rationale per axis |
|---|---|---|---|---|---|---|---|
| DS-01 | Parked vehicle actively opened (doors or trunk) by an unauthorized person; contents may be stolen (engine start blocked by the immobilizer, out of scope) | Neg | Major | Neg | Mod | **Major** | F: worst credible case — a valuable item (laptop, bag) is stolen = significant loss; the vehicle itself cannot be driven away (immobilizer). P: a document in the cabin (registration → home address) may be read once = one-off disclosure, not durable profiling; and this harm stems from the contents left in the vehicle, not from the RKE function itself. S/O not affected. |
| DS-02 | Owner can no longer lock or unlock the vehicle with the key fob | Neg | Mod | Mod | Neg | **Moderate** | O: a mechanical key exists (01, §4) → mechanical fallback available, so Moderate, not "locked out until workshop" (Major). The mechanical key covers access; engine start remains blocked by the immobilizer (out of scope). F: key reprogramming / workshop cost = limited loss. No data exposed (P Neg) — this is what distinguishes DS-02 from DS-01. |
| DS-03 | Owner's key fob is recognizable at each use; repeated eavesdropping lets an observer infer presence, absence and travel habits (GDPR: persistent identifier) | Neg | Neg | Neg | Major | **Major** | P: repeated eavesdropping of the persistent Key ID (A-06) lets a determined observer infer habits. **Major and not Severe**: the radio range (~10–50 m, 01 §5) limits tracking to a local observer (home, workplace), with no large-scale persistent tracking. The P is legitimately high because the clear-text Key ID is an emission of the RKE function (asset AS-07). |
| DS-04 | Vehicle left unlocked without the owner's knowledge: the owner believes the vehicle is secured while it remains accessible with no defeat of the electronic lock; contents may then be stolen | Neg | Major | Neg | Mod | **Major** | S: Negligible — the earlier rating (assault during sleep) is dropped, as ISO 21434 rates Safety via vehicle behaviour (ISO 26262 link), not via a downstream third-party crime. F: theft of contents, same worst credible case as DS-01 (the same damage cannot be rated differently). O: the vehicle functions normally. P: same logic as DS-01. |

**DS-01 vs DS-04:** the difference is the state of the vehicle, which changes the consequence for the owner, not merely the attack path. In DS-01 the vehicle was locked and its security was defeated; in DS-04 the vehicle was never locked and no defeat of the electronic lock took place — the attacker exploits a door left unlocked, and the owner is unaware the vehicle is unsecured. The two also call for different treatments (anti-replay / anti-pairing control for DS-01; lock-confirmation feedback, out of boundary, for DS-04), which is why they are kept separate even though both reach a retained impact of Major.

**Aggregation rule:** ISO/SAE 21434 does not mandate collapsing the four axes into a single value. This study retains the **maximum** axis as the single `impact_rating` reported in the CSV, a common convention. The per-axis ratings above are kept so the choice stays auditable.

**No Severe case:** Safety stays Negligible on every DS — the RKE touches access only, never vehicle behaviour (engine start is blocked by the immobilizer, out of scope). This caps the item at Major and is why no DS reaches Severe.

**`impact_category` rule:** the CSV `impact_category` names the axis carrying the retained (maximum) impact. On a tie, it names the axis that reflects the scenario's nature — for DS-02, where F and O both reach Moderate, Operational is chosen (the scenario is a denial of the lock/unlock function).

## 3. Impact rating scale

Impact is rated per the four ISO/SAE 21434 categories (Safety, Financial, Operational, Privacy). The Safety axis is defined for completeness but is not exercised in this study (all DS rate Safety Negligible — see §2).

| Rating | Safety | Financial | Operational | Privacy |
|---|---|---|---|---|
| Severe | Life-threatening or fatal injury | Loss comparable to the vehicle value or its full contents | Vehicle permanently unusable | Persistent, re-identifiable tracking of the owner |
| Major | Severe injury (survival probable) | Significant loss (valuable contents stolen) | Owner locked out until workshop intervention | Habits inferable over time by a determined observer |
| Moderate | Light to moderate injury | Limited loss (minor contents) | Temporary loss of RKE, mechanical fallback available | One-off observation, no lasting profile |
| Negligible | No injury | No meaningful loss | Minor inconvenience | No meaningful privacy effect |

## 4. Attack feasibility scale

Feasibility follows the ISO/SAE 21434 attack-potential approach: the harder the attack, the lower the feasibility, the lower the risk. Each CSV line records its reasoning in `feasibility_rationale`.

| Level | Meaning | Typical marker in this study |
|---|---|---|
| High | Low effort, common equipment, no prior physical breach | Radio capture/replay from range (TS-01), key ID eavesdropping (TS-12) |
| Medium | Moderate effort or limited physical access | Exterior harness connector (TS-04, TS-06); OBD access inside the cabin (TS-03, TS-05) |
| Low | Specialist effort, invasive physical access | Opening the BCM and soldering onto the debug interface (TS-08 to TS-11) |

This ordering is the reason the debug-access lines (TS-08 to TS-11) must not receive the same feasibility as the harness lines (TS-04, TS-06): they demand opening the BCM, where the harness path only needs an exterior connector.

## 5. Risk matrix

Risk = f(impact, feasibility). Feasibility stands for likelihood: the harder the attack (Low), the less probable, the lower the risk.

| Impact \ Feasibility | High | Medium | Low |
|---|---|---|---|
| Severe | Very High | High | Medium |
| Major | High | Medium | **Low** |
| Moderate | Medium | Medium | Low |
| Negligible | Low | Low | Low |

Note: Major × Low = Low (not Medium) because a Low-feasibility attack is improbable; a Major impact that is improbable remains a low risk. The matrix is monotone (risk never decreases when impact or feasibility rises).

## 6. Justified exclusions

Excluding an attack is defensible only when the reason is stated. An omitted threat looks ignored; an excluded one looks decided.

| Excluded element | Reason |
|---|---|
| I-02 (LIN, receiver → BCM) | Excluded from this v0.1. The analysis prioritises the external RF attack surface and the diagnostic CAN paths. Injection on the internal LIN interface is not assessed separately in this version; its distinct attack potential, and exactly where the BCM performs frame validation relative to the LIN, would require further investigation. |
| Actuator command (I-03) | Power output, not a message. Driving the actuator directly is mechanical break-in, outside the cyber perimeter; no cybersecurity control applies. |
| Key-fob counter modification | Pointless: an attacker who holds the fob (I-06a) can already open the vehicle by pressing the button. |
| Firmware confidentiality | No fleet-wide master key (see `01`, §1): a firmware dump reveals no secret usable against other vehicles. Security rests on the key, not on the secrecy of the algorithm (Kerckhoffs). Firmware integrity is retained (AS-06 / TS-11). |

## 7. Limits (analysed or bounded, not excluded)

These are not exclusions: the threat is real and kept in scope, but the point worth stating is where its treatment lies or why the analysis stops at the RKE boundary.

- **Denial jamming of I-01.** Jamming the lock command so the BCM receives no frame cannot be prevented by any RKE cybersecurity control — the attack removes the channel rather than forging a message (RED / UN R10 territory, not ISO 21434). It is kept in scope through its consequence DS-04 (TS-02); its only mitigation, lock-confirmation feedback, is out of boundary. This is distinct from RollJam capture-jamming (TS-01), where the replayed frame *is* processed and the control is cryptographic (counter + window).
- **CAN-wide denial (e.g. bus-off attack).** Forcing the body CAN into a bus-off error state would deny communication on the whole bus. This is a distinct mechanism from radio jamming, not a transposition of it. It is not assessed in this v0.1 because it targets the network as a whole, not the RKE function specifically; it would belong to a vehicle-network-level analysis.

## 8. Traceability chain

Each CSV line is meant to be read as one chain:

```
asset → cybersecurity property → threat scenario → attack path → damage scenario → impact → feasibility → risk → cybersecurity goal
```

Goals and requirements derive from the risks that need treatment, and live in `03_goals-requirements.md`.