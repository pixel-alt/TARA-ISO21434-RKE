# 01 – Item Definition

**Scope:** system-level TARA
**Status:** fictional case study – generic passenger vehicle, no real OEM

## 1. Item description

Remote Keyless Entry (RKE) system of a generic passenger vehicle.
The user presses a button on the key fob, which transmits a UHF 433 MHz
frame containing a one-time authentication code (rolling code).
The RKE logic hosted in the Body Control Module (BCM) verifies the frame
and commands the door lock actuators.

## 2. Architecture & boundary

```
┌───────── ITEM BOUNDARY ──────────┐
│ [C-01 Key fob]                   │
│     ~~ I-01 UHF 433 MHz ~~>      │
│            [C-02 RF receiver]    │
│                  │ I-02 LIN      │
│         [C-03 BCM – RKE logic]───┼── I-04 Body CAN ──[EXT-01 Central gateway]
│                  │ I-03 wired    │
│      [C-04 Door/trunk actuators] │
└──────────────────────────────────┘
                                                                   │ I-05 Diagnostic CAN
                                                           [EXT-02 OBD port]
```

**Boundary rationale:** the key fob is included because it generates the
authentication code; the analysis covers the full chain from code
generation to actuator command. Only the RKE logic of the BCM is part of
the item; other BCM functions (lighting, wipers, comfort) are outside
the boundary. The BCM hardware storing RKE secrets (non-volatile memory,
debug interface) is inside the boundary.

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
| I-02 | C-02 → C-03 | LIN | Demodulated frame | Physical (in-vehicle) |
| I-03 | C-03 → C-04 | Wired | Actuator commands | Physical (in-vehicle) |
| I-04 | C-03 ↔ EXT-01 | Body CAN | Diagnostic requests and responses | Physical (in-vehicle, see A-05) |
| I-05 | EXT-01 ↔ EXT-02 | Diagnostic CAN | Diagnostic requests and responses (UDS) | Physical (cabin) |
| I-06a | Physical access → C-01 | PCB, debug interface, memory | Potential exposure: key secret, rolling-code counter, firmware | Physical (key fob in attacker's possession) |
| I-06b | Physical access → C-03 | PCB, debug interface, non-volatile memory | Potential exposure: stored key secrets, paired-key list, counters, firmware | Physical (BCM removed or accessible) |

The BCM exposes a diagnostic endpoint on body CAN. The gateway routes
diagnostic requests from the OBD port to this endpoint (see A-01).

## 3. Functions

Functions reflect a representative RKE architecture. Diagnostic-based
key management (F-03, F-04, F-06) is assumed as part of this fictional model.
No cybersecurity control on diagnostic access (e.g. Security Access 0x27,
authorization) is assumed here: controls are derived from the TARA
(see 03_goals-requirements.md).

| ID | Function | Trigger | Actor |
|---|---|---|---|
| F-01 | Lock / unlock the doors | Key fob button (I-01) | User |
| F-02 | Release the trunk | Key fob button (I-01) | User |
| F-03 | Pair a new key | Diagnostic request (I-05 → I-04) | Workshop technician |
| F-04 | Read the list of paired keys (key IDs and status only, no secret material) | Diagnostic request (I-05 → I-04) | Workshop technician |
| F-05 | Resynchronize the rolling-code counter: accepts a frame whose counter is ahead of the stored value within a bounded look-ahead window and updates the stored counter only after the authentication code is verified | Key fob (I-01) | User |
| F-06 | Revoke a key | Diagnostic request (I-05 → I-04) | Workshop technician |

### Interface coverage check

| Interface | Functions |
|---|---|
| I-01 | F-01, F-02, F-05 |
| I-02, I-03 | Internal transport for F-01, F-02 |
| I-04, I-05 | Transport of diagnostic requests and responses for F-03, F-04, F-06 |
| I-06a, I-06b | No function – attack surface only (not used in normal operation) |

Decision: I-04 is used only to carry diagnostic requests and responses.
Other body CAN messages are excluded from this study. No RKE function is
triggered by them (e.g. speed-based auto-lock is out of scope).

## 4. Out of scope

| Element | Rationale |
|---|---|
| Passive Keyless Entry and Start (PKES) | Different technology (LF + challenge-response), separate analysis |
| Immobilizer / engine start | Different function, separate item |
| Digital key (smartphone, BLE, UWB, NFC) | Different item |
| OEM backend and factory key provisioning | Outside the vehicle, see A-02 |
| BCM software update / reprogramming | BCM-wide function, see A-04 |
| Lock confirmation (indicators, mirror folding) | Executed by lighting / comfort functions |
| Mechanical key | Non-electronic |
| Speed-based auto-lock | Not part of this item definition |

## 5. Operational environment

- **Locations:** public parking, street, private garage, workshop, dealership.
- **Vehicle states:** parked and locked, parked with owner nearby, in workshop.
- **Range:** nominal key fob range ~10–50 m.
- **Lifecycle phases:** normal use, maintenance (legitimate OBD access),
  key replacement, vehicle resale, decommissioning / end of life
  (BCM and key fobs may leave the vehicle with their secrets).

## 6. Assumptions & constraints

### Assumptions

| ID | Assumption | Rationale | Impact if false |
|---|---|---|---|
| A-01 | The gateway forwards only diagnostic requests addressed to the BCM; no other OBD frames reach body CAN | Common architecture | OBD gives direct, unfiltered access to body CAN and the BCM |
| A-02 | Key secrets are provisioned securely at production | Out of scope (backend / factory) | Secrets may be known before vehicle delivery |
| A-03 | The OBD port is located in the cabin; physical access to the cabin is required to reach it | Typical vehicle layout | Diagnostic interface reachable from outside the vehicle |
| A-04 | BCM reprogramming is protected | Out of scope (BCM-wide) | RKE logic can be replaced |
| A-05 | Body CAN wiring is not reachable from outside the vehicle | Typical design intent | Exterior harness access (e.g. headlamp connector) allows direct injection on I-04, bypassing the gateway (A-01) |

### Constraints

- **Regulatory:** UN R155 (cybersecurity), Radio Equipment Directive
  2014/53/EU (433 MHz emission), UN R10 (electromagnetic compatibility).
- **Technical:** battery-powered key fob (low power, lightweight
  cryptography), one-way radio link (key fob cannot receive).