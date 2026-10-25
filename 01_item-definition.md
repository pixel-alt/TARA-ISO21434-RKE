# 01 – Item Definition

**Scope:** system-level TARA
**Status:** fictional case study – generic passenger vehicle, no real OEM
**Convention:** the item definition (clause 9) cites no identifier produced by the TARA (threat scenarios, damage scenarios, cybersecurity goals). It may reference the TARA documents themselves (`02`, `03`), which are stable.

## 1. Item description

Remote Keyless Entry (RKE) system of a generic passenger vehicle. The user presses a button on the key fob, which transmits a UHF 433 MHz frame containing a one-time authentication code (rolling code). The RKE logic hosted in the Body Control Module (BCM) verifies the frame and commands the door lock actuators.

Each key fob is provisioned during production with a unique, randomly generated secret K. No fleet-wide master key is used. The BCM stores the secrets of all paired keys.

**Why a unique random K rather than a derived one.** A derivation K_fob = f(K_master, serial) would save provisioning effort but turns the extraction of one master key — a published class of side-channel attack on deployed rolling-code systems — into a compromise of the entire fleet. Unique random provisioning is what bounds a debug or extraction compromise to a single vehicle, and what makes the Major × Low = Low rating of the physical-access family defensible (`02` §5).

## 2. Architecture & boundary

```
┌───────── ITEM BOUNDARY ──────────┐
│ [C-01 Key fob]                   │
│     ~~ I-01 UHF 433 MHz ~~>      │
│            [C-02 RF receiver]    │
│                  │ I-02 LIN      │
│         [C-03 BCM – RKE logic]───┼── I-04 Body CAN ──[EXT-01 Central gateway]
│                  │ I-03 wired    │         ▲                     │
│      [C-04 Door/trunk actuators] │         │                     │
└──────────────────────────────────┘     I-07 Exterior harness     │
                                                           I-05 Diagnostic CAN
                                                           [EXT-02 OBD port]
```

**Boundary rationale:** the key fob is included because it generates the authentication code; the analysis covers the full chain from code generation to actuator command. Only the RKE logic of the BCM is part of the item; other BCM functions (lighting, wipers, comfort) are outside the boundary. The BCM hardware storing RKE secrets (non-volatile memory, debug interface) is inside the boundary.

### Components (inside the boundary)

| ID | Component | Role |
|---|---|---|
| C-01 | Key fob | Generates and transmits authenticated lock/unlock frames |
| C-02 | RF receiver | Demodulates UHF frames and forwards them to the BCM |
| C-03 | BCM – RKE logic | Verifies frames, manages paired keys and counters, commands actuators |
| C-04 | Door and trunk actuators | Physically lock / unlock doors and release the trunk |

### External elements

| ID | Element | Relation to the item |
|---|---|---|
| EXT-01 | Central gateway | Routes diagnostic requests from the OBD port to the BCM |
| EXT-02 | OBD port | Diagnostic access point for workshop tools |

### Interfaces

Seven interfaces are identified. I-06 is split into I-06a (key fob) and I-06b (BCM) because the two differ in access means and in what they expose, which makes their attack potential different; they are counted as one interface in the summary figures.

| ID | From → To | Medium / access means | Data | Access |
|---|---|---|---|---|
| I-01 | C-01 → C-02 | UHF 433 MHz, one-way | Key ID, counter, authentication code, command | Remote (radio range) |
| I-02 | C-02 → C-03 | LIN | Demodulated frame (message, BCM input) | Physical (in-vehicle) |
| I-03 | C-03 → C-04 | Wired, power output | Actuator drive current (power, BCM output – no message) | Physical (in-vehicle) |
| I-04 | C-03 ↔ EXT-01 | Body CAN (interior segment, BCM side of the gateway) | Diagnostic requests and responses | Physical (in-vehicle, see I-07) |
| I-05 | EXT-01 ↔ EXT-02 | Diagnostic CAN (exterior segment, OBD side of the gateway) | Diagnostic requests and responses (UDS) | Physical (cabin) |
| I-06a | Physical access → C-01 | PCB, debug interface, memory | Potential exposure: key secret, rolling-code counter, firmware | Physical (key fob in attacker's possession) |
| I-06b | Physical access → C-03 | PCB, debug interface, non-volatile memory | Potential exposure: stored key secrets, paired-key list, counters, firmware | Physical (BCM removed or accessible) |
| I-07 | Physical access → I-04 | Exterior harness / connector (e.g. headlamp) | Direct frame injection on body CAN, bypassing the gateway (A-01) | Physical (vehicle exterior) |

**I-02 vs I-03:** both are internal transport between BCM and its neighbours, but their nature is opposite. I-02 is a **message** entering the BCM (a LIN frame to read and verify); I-03 is a **power signal** leaving the BCM (drive current to the actuator, nothing to authenticate). This is why an attack on I-03 is out of cyber scope (see §4).

**I-04 vs I-05:** both are CAN segments, but the gateway (EXT-01) sits between them. I-05 is the exterior segment, from the OBD port to the gateway, where a workshop tool plugs in. I-04 is the interior segment, from the gateway to the BCM. The gateway filters between the two (A-01); interface I-07 matters precisely because it reaches I-04 directly, bypassing that filter.

The BCM exposes a diagnostic endpoint on body CAN. The gateway routes diagnostic requests from the OBD port to this endpoint (see A-01). Body CAN can also be reached directly from the vehicle exterior through the harness (I-07), without passing through the gateway.

## 3. Functions

Functions reflect a representative RKE architecture. Diagnostic-based key management (F-03, F-04, F-06) is assumed as part of this fictional model. No cybersecurity control on these key-management services (e.g. Security Access 0x27, authorization) is assumed here: controls are derived from the TARA (see `03_goals-requirements.md`). The protection of BCM reprogramming (A-04) does not extend to these services.

| ID | Function | Trigger | Actor |
|---|---|---|---|
| F-01 | Lock / unlock the doors: the BCM verifies the frame authentication code and validates the rolling-code counter before executing the requested action; invalid frames are rejected | Key fob button (I-01) | User |
| F-02 | Release the trunk: same authentication-code and rolling-code validation as F-01 before executing | Key fob button (I-01) | User |
| F-03 | Pair a new key | Diagnostic request (I-05 → I-04) | Workshop technician |
| F-04 | Read the list of paired keys (key IDs and status only, no secret material) | Diagnostic request (I-05 → I-04) | Workshop technician |
| F-05 | Resynchronize the rolling-code counter: accepts a frame whose counter is ahead of the stored value within a bounded look-ahead window and updates the stored counter only after the authentication code is verified | Key fob (I-01) | User |
| F-06 | Revoke a key | Diagnostic request (I-05 → I-04) | Workshop technician |

The authentication and rolling-code checks protect radio-triggered actions, but only the frames they process: they verify a received code, not the integrity of the secrets and counters they rely on, nor which command the signed frame carries. A captured authentic code that the vehicle never received — and therefore never consumed — remains above the stored counter and is accepted when replayed later; its command field, not covered by the code (A-10), can be altered before replay; a direct modification of the stored secrets or counter has the same effect. Each path defeats the function without breaking the checks themselves. These paths are analysed in the TARA (`02`).

### Interface coverage check

| Interface | Functions |
|---|---|
| I-01 | F-01, F-02, F-05 |
| I-02 | Internal transport (BCM input) for F-01, F-02 |
| I-03 | Internal transport (BCM output) for F-01, F-02 |
| I-04 | Interior CAN segment (gateway → BCM) carrying diagnostic requests and responses for F-03, F-04, F-06 |
| I-05 | Exterior CAN segment (OBD → gateway) carrying diagnostic requests and responses for F-03, F-04, F-06 |
| I-06a | No function – attack surface only (physical access to the key fob) |
| I-06b | No function – attack surface only (physical access to the BCM) |
| I-07 | No function – attack surface only (direct access to the F-03, F-04, F-06 endpoint, bypassing EXT-01) |

**Decision:** I-04 is used only to carry diagnostic requests and responses. Other body CAN messages are excluded from this study. No RKE function is triggered by them (e.g. speed-based auto-lock is out of scope).

## 4. Out of scope

| Element | Rationale |
|---|---|
| Passive Keyless Entry and Start (PKES) | Different technology (LF + challenge-response), separate analysis |
| Immobilizer / engine start | Different function, separate item. Its effect on impact rating is assumed in A-09 |
| Digital key (smartphone, BLE, UWB, NFC) | Different item |
| OEM backend and factory key provisioning | Outside the vehicle, see A-02 |
| BCM software update / reprogramming via diagnostic | BCM-wide function, see A-04 |
| Interior lock control (door switch, speed-based auto-lock) | Local wired command, not an RKE (radio) function |
| Lock confirmation (indicators, mirror folding) | Executed by lighting / comfort functions |
| Attacks on the mechanical key | Non-electronic; no cyber attack surface. Its existence as an access fallback is assumed in A-08 |

## 5. Operational environment

- **Locations:** public parking, street, private garage, workshop, dealership.
- **Vehicle states:** parked and locked, parked with owner nearby, in workshop.
- **Range:** nominal key fob range ~10–50 m.
- **Lifecycle phases:** normal use, maintenance (legitimate OBD access), key replacement, vehicle resale, decommissioning / end of life (BCM and key fobs may leave the vehicle with their secrets).

## 6. Assumptions & constraints

### Assumptions

| ID | Assumption | Rationale | Impact if false |
|---|---|---|---|
| A-01 | The gateway forwards only diagnostic requests addressed to the BCM; no other OBD frames reach body CAN | Common architecture | OBD gives direct, unfiltered access to body CAN and the BCM |
| A-02 | Key secrets are provisioned securely at production | Out of scope (backend / factory) | Secrets may be known before vehicle delivery |
| A-03 | The OBD port is located in the cabin; physical access to the cabin is required to reach it | Typical vehicle layout | Diagnostic interface reachable from outside the vehicle |
| A-04 | BCM reprogramming via diagnostic is protected. Physical access through the debug interface (I-06b) is not covered by this assumption and is analysed (see A-07) | Out of scope (BCM-wide) | RKE logic can be replaced through diagnostic |
| A-05 | F-05 accepts a counter ahead of the stored value within a bounded look-ahead window, to tolerate presses made out of radio range | Typical rolling-code tolerance | Resynchronisation after out-of-range presses fails. Note: the window is not what makes capture-and-replay work — a code the vehicle never received stays above the stored counter whatever the window width (see `03`, RES-01) |
| A-06 | The Key ID is stable across successive transmissions from the same fob | Explicit modelling assumption for this fictional case study | If the identifier rotates or is otherwise unlinkable, the tracking analysis must be reassessed — but not closed: the rolling-code counter, monotonic by design (§3, F-01/F-05), remains a linking field on its own, which is why CSG-07 in `03` is stated over every observable field |
| A-07 | No effective debug-access protection prevents reading or writing the BCM non-volatile memory through the debug interface (I-06b) | Explicit modelling assumption for this fictional case study | If such protection is effective on I-06b, the debug-path reads and writes (TS-08, TS-09, TS-10, TS-11) no longer hold on this path; the derived requirement is CSG-05 in `03` |
| A-08 | A mechanical key backup equips the vehicle and provides a physical access fallback if the fob is unavailable | Typical vehicle equipment | The owner is locked out until workshop intervention (higher operational impact) |
| A-09 | The immobilizer prevents engine start without a paired key, so a vehicle opened through the RKE cannot be driven away | Separate item, out of scope (§4) | Financial impact rises to Severe (vehicle value) on the theft-related damage scenarios; the whole item would have to be re-rated |
| A-10 | The authentication code does not cover the command field (lock / unlock / trunk): the command on a captured frame can be altered without invalidating the code | Explicit modelling assumption for this fictional case study | If the command field is covered, a captured lock frame cannot be replayed as unlock; the TS-01 command-flip path no longer holds (the attacker must capture a genuine unlock press). The derived requirement is CSG-04 in `03` |
| A-11 | No effective firmware authenticity or integrity protection prevents a modified BCM firmware from being installed and executed | Explicit modelling assumption for this fictional case study | If such protection is effective, a modified firmware is rejected and the firmware-flash analysis (TS-11) no longer holds, even where debug write access exists (A-07). The derived requirement is CSG-06 in `03` |
| A-12 | No effective readout protection prevents extracting the key secret K from the key fob through its debug interface (I-06a) | Explicit modelling assumption for this fictional case study | If such protection is effective, K cannot be read and TS-07 no longer holds. Fob-side counterpart of A-07; the derived requirement is CSG-05 in `03` |

### Constraints

- **Regulatory:** UN R155 (cybersecurity), Radio Equipment Directive 2014/53/EU (433 MHz emission), UN R10 (electromagnetic compatibility), GDPR (EU) 2016/679, art. 4(1) and art. 25 (the key ID broadcast in clear is a persistent identifier; data protection by design applies where an alternative exists).
- **Technical:** battery-powered key fob (low power, lightweight cryptography), one-way radio link (key fob cannot receive), no real-time clock in the fob.