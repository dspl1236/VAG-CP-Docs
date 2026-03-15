# Component Protection: Technical Overview

*This document summarizes publicly available information about how VW Group's Component Protection system works. Sources include manufacturer Self-Study Programmes, NHTSA TSBs, Ross-Tech documentation, and community research.*

---

## What it is

Component Protection (German: *Komponentenschutz*) is a software-based anti-theft and parts-pairing system used across all Volkswagen Group brands. It cryptographically binds electronic modules to a specific vehicle by registering their serial numbers against the vehicle's VIN in VW's central FAZIT database (Fahrzeug-Informations und Zentrales Identifikations-Tool), physically located at Audi AG in Ingolstadt, Germany.

It is distinct from the **immobilizer** (which prevents engine starting without a valid key transponder). Component Protection restricts the *functionality* of peripheral modules — a vehicle with CP-locked infotainment will still start and drive, but audio is permanently muted; a CP-locked instrument cluster displays continuous warning lights; a CP-locked climate control reverts to basic mode.

---

## How the lock works

The **J533 Data Bus Diagnostic Interface (CAN Gateway)** is the master control module. At factory build:

1. Each protected module's serial number and part number are registered in the FAZIT database against the vehicle's VIN
2. The module's EEPROM is written with encrypted data tying it cryptographically to the specific Gateway
3. On every ignition cycle, the Gateway interrogates all protected modules in a "constellation check" — comparing their cryptographic identity with values stored from previous cycles
4. If any module returns an unexpected value (because it was swapped, or the Gateway was replaced, or a voltage event corrupted stored data), Component Protection activates on that module

**Activation is immediate and cannot be cleared through standard fault-code erasure (VCDS, OBD-II scanners, etc.)** Only a valid online adaptation through VW's servers removes the lock.

---

## Activation triggers (documented)

Component Protection is documented to activate in the following scenarios:

| Trigger | Source |
|---|---|
| Module replaced with used/salvage part | VW SSP 294 (2002) |
| Module swapped between two vehicles "on a trial basis" | VW SSP 294 (2002) |
| Battery disconnection / low voltage event | Multiple owner reports, VW TSBs |
| Gateway (J533) replacement | VW ODIS documentation |
| Spontaneous activation without any modification | NHTSA TSB 91-18-13TT (2018) |
| ECU flash / software update error | Community reports |

---

## Functional restrictions when active

| Module | Effect of CP Activation |
|---|---|
| MMI / Infotainment head unit | Audio permanently muted; display functional |
| Instrument cluster | "SAFE CP" displayed; all warning lights active |
| Climatronic (climate control) | Reverts to basic defrost mode; full HVAC disabled |
| Seat memory module | All memory/adjustment functions disabled |
| KESSY (keyless entry/start) | Remote functions disabled |
| Adaptive cruise control | "ACC not available" displayed |
| Steering column module | Column adjustment may be disabled |

---

## Three generations + SFD overlay

See `generations.md` for full detail. Summary:

| Generation | Years | Workaround |
|---|---|---|
| Gen 1 | 2003–2008 | EEPROM manipulation possible with skill |
| Gen 2 | 2008–2015 | Limited third-party tool support (AVDI/ABRITES) |
| Gen 3 | 2016–present | **No offline workaround. Server dependency absolute.** |
| SFD (overlay) | 2020–present | Time-limited tokens required for any diagnostic write |
| SFD2 (overlay) | 2024–present | Cryptographically signed commands required for all parameter changes |

---

## The unlock process

To remove Component Protection, a technician must:

1. Connect OEM-level diagnostic hardware (VAS 6150x or equivalent) to the vehicle's OBD-II port
2. Launch ODIS (Offboard Diagnostic Information System) — the proprietary VW diagnostic platform
3. Authenticate with personal GEKO/GRP credentials (two-factor authentication required)
4. Navigate to the relevant module's adaptation channels
5. Initiate a "Component Protection removal" guided function
6. ODIS transmits: vehicle VIN, module serial numbers, technician identity, timestamp → FAZIT server
7. FAZIT checks: Is this module flagged as stolen? (FAZ4990E = stolen, blocks unlock) Is the transfer legitimate?
8. If approved, FAZIT generates new authentication data → transmitted back through ODIS → module re-married to new VIN
9. Full audit log created at VW with technician identity

**The entire process takes 10–20 minutes of actual work.** Dealers routinely charge 1–2 hours of labor.

---

## Key error codes

| Code | Meaning |
|---|---|
| FAZ4990E | Module flagged as stolen — unlock blocked |
| FAZ2946E | Used part not permitted — replace with new (post-Oct 2021 EU rule) |
| SAFE CP | Displayed on instrument cluster when CP is active |
| Component Protection Active | Generic display message on affected modules |

---

## Sources

- VW Self-Study Programme SSP 294 (August 2002) — original CP documentation
- NHTSA TSB 91-18-13TT (November 2018) — spontaneous activation acknowledgment
- Ross-Tech VCDS Wiki — SFD documentation
- VW erWin portal — FAZIT/GeKo authorization documentation
- VAG Programming (vagprogramming.com) — independent technical summary
- MHH Auto forums — community technical research
- Ross-Tech Forums — community documentation
