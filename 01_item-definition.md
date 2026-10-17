# 01 – Item Definition

**Scope:** system-level TARA
**Status:** fictional case study – generic passenger vehicle, no real OEM

## 1. Item description

Remote Keyless Entry (RKE) system of a generic passenger vehicle. The user presses a button on the key fob, which transmits a UHF 433 MHz frame containing a one-time authentication code (rolling code). The RKE logic hosted in the Body Control Module (BCM) verifies the frame and commands the door lock actuators.

Each key fob is provisioned during production with a unique, randomly generated secret K. No fleet-wide master key is used. The BCM stores the secrets of all paired keys.

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

The authentication and rolling-code checks protect radio-triggered actions. TS-01 (RollJam) explores whether a captured, authentic code can be accepted within the F-05 look-ahead window under assumption A-05. Debug-access scenarios (TS-09, TS-10) examine whether direct modification of BCM secrets or counter state can undermine the data these checks rely on, rather than breaking the checks themselves.

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
| Immobilizer / engine start | Different function, separate item |
| Digital key (smartphone, BLE, UWB, NFC) | Different item |
| OEM backend and factory key provisioning | Outside the vehicle, see A-02 |
| BCM software update / reprogramming via diagnostic | BCM-wide function, see A-04 |
| Interior lock control (door switch, speed-based auto-lock) | Local wired command, not an RKE (radio) function |
| Lock confirmation (indicators, mirror folding) | Executed by lighting / comfort functions |
| Mechanical key | Non-electronic |

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
| A-04 | BCM reprogramming via diagnostic is protected. Physical access through the debug interface (I-06b) is not covered by this assumption and is analysed | Out of scope (BCM-wide) | RKE logic can be replaced through diagnostic |
| A-05 | The look-ahead window of F-05 accepts a captured, not-yet-consumed code (enabling a RollJam-style capture-and-replay) | Typical rolling-code tolerance | Replay is ineffective; the radio path (TS-01) is closed |
| A-06 | The Key ID is stable across successive transmissions from the same fob | Explicit modelling assumption for this fictional case study | If the identifier rotates or is otherwise unlinkable, the tracking scenario (TS-12) must be reassessed |
| A-07 | No effective firmware authenticity, integrity, or debug-access protection prevents a modified firmware from being installed and executed through I-06b | Explicit modelling assumption for this fictional case study | If any such protection is effective on this path, the modified firmware is rejected and TS-11 fails |

### Constraints

- **Regulatory:** UN R155 (cybersecurity), Radio Equipment Directive 2014/53/EU (433 MHz emission), UN R10 (electromagnetic compatibility), GDPR (EU) 2016/679 (key ID broadcast in clear is a persistent identifier, see DS-03).
- **Technical:** battery-powered key fob (low power, lightweight cryptography), one-way radio link (key fob cannot receive).