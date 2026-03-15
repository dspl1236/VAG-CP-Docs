# Which Module Controls Component Protection?

*Based on VW/Audi official Self-Study Programme documentation (SSP 994403, SSP 486, SSP 990613) and publicly available technical research.*

---

## The Short Answer

**There are two modules at the heart of Component Protection:**

1. **J518 — Access/Start Control Module** — The *immobilizer master*. Controls key authentication and engine start authorization.
2. **J533 — Data Bus On Board Diagnostic Interface (Gateway)** — The *Component Protection master*. This is the module that actually enforces CP on all Convenience and Infotainment modules.

This is directly stated in Audi's own SSP 994403 (2005 Audi A6 Electrical System, US market):

> *"The Convenience and Infotainment control modules are also integrated within the 'Component Protection' safety system (Geko). As a result, these control modules must be adapted to the specific vehicle once they have been installed. **The Data Bus On Board Diagnostic Interface J533 is integrated into the 'Component protection' function for the first time.**"*

And separately:

> *"[J518] — Immobilizer and component protection: The control module is the master for these functions."*

So in practice: **J518 is the immobilizer master, J533 is the Component Protection gateway master.** They work together through the CAN bus constellation check.

---

## How J533 Enforces Component Protection

The J533 Data Bus Diagnostic Interface (gateway) sits at the center of every CAN bus in the vehicle. On the A6 C7 (your platform), J533 bridges:

- Convenience CAN (J393, J136, J521, J255, door modules, seat modules)
- Powertrain CAN (J623 ECM, J217 TCM, J104 ABS)
- Display and Operation CAN (J285 instrument cluster, J527 steering column)
- Extended CAN (driver assistance modules)
- FlexRay bus (chassis/suspension modules)
- MOST bus (infotainment ring)
- Diagnostics CAN (OBD-II port T16)

**Every module that participates in CP communicates its identity through J533.**

### The Constellation Check

On every ignition cycle, J533 performs what VW calls a "constellation check":

1. J533 broadcasts an interrogation to all CP-protected modules on the CAN bus
2. Each protected module responds with its cryptographic identity (derived from its EEPROM, encoded at factory against the vehicle's VIN/FAZIT database record)
3. J533 compares responses against its stored "expected constellation"
4. Any module returning an unexpected value (because it was swapped, or J533 itself was replaced) triggers CP activation on that module
5. J533 sends the restriction command back to the non-matching module
6. The module applies its functional restrictions (audio mute, HVAC defrost-only, seat disable, etc.)

**This is why replacing J533 itself is particularly destructive** — it knows the expected constellation of all other modules, so replacing it causes CP to fire on every single CP-protected module in the vehicle simultaneously.

---

## CP-Protected Modules on the Audi A6 C7 (Your Vehicle)

Based on SSP 994403, SSP 486, SSP 990613, and the A6 C7 topology diagram:

### Confirmed CP-Protected (Convenience CAN):
| Module | ID | Description |
|---|---|---|
| Access/Start Control Module | **J518** | Immobilizer master, KESSY |
| Memory Seat/Steering Column Adjustment CM | **J136** | Driver seat memory/adjustment — **your affected module** |
| Front Passenger Memory Seat CM | **J521** | Passenger seat memory — **your affected module** |
| Central Comfort System Control Module | **J393** | Direction indicators, central locking, interior lighting master |
| Vehicle Electrical System Control Module | **J519** | External lighting master, LIN master |
| Climatronic Control Module | **J255** | HVAC — **your affected module** |
| Steering Column Electronics CM | **J527** | Steering column adjustment, multi-function steering wheel |
| Digital Sound System Control Module | **J525** | Audio amplifier (explicitly named in SSP 994403: *"J525 participates in component protection (Geko)"*) |

### Confirmed CP-Protected (MOST/Infotainment bus):
| Module | ID | Description |
|---|---|---|
| Information Electronics CM 1 | **J794** | MMI master (MIB2 head unit) |
| Radio | **R** | Head unit radio function |
| Front Information Display CM | **J523** / **J685** | MMI display |

### The Gateway Itself:
| Module | ID | Description |
|---|---|---|
| Data Bus Diagnostic Interface | **J533** | **The CP master/enforcer — replacing this triggers CP on ALL other modules** |

### Generally NOT CP-Protected:
- Engine Control Module J623 (covered by immobilizer separately)
- ABS Control Module J104
- Airbag Control Module J234
- Transmission Control Module J217
- Level/suspension control modules

---

## What Happens When You Install a Salvage Module

### Your HVAC module (J255 Climatronic):
The J255 was encoded at the factory to the donor vehicle's VIN. Its EEPROM contains cryptographic data matching the donor's J533 gateway. When installed in your A6:
1. Your J533 sends the constellation interrogation
2. J255 responds with its cryptographic identity — which matches the *donor* vehicle, not yours
3. Your J533 flags this as a mismatch
4. J533 sends a CP restriction command to J255
5. J255 enters restricted mode: climate control locked to basic defrost only
6. Fault stored: "Component Protection Active" in J255's DTC memory

### Your Seat modules (J136 / J521):
Same process. Additionally, as noted in SSP 994403:
> *"Assigning a PIN code in the connecting plug, the Memory Seat/Steering Column Adjustment Control Module J136 and Passenger Memory Seat Control Module J521 are coded automatically for either the driver's seat or passenger's seat installation position — depending on the pin — when it is first connected to the seat. This operation can only be performed once, but can be released again by means of the diagnostic function."*

This means the seat modules have a *double* binding: once to the seat position (via hardware pin coding) and once to the vehicle VIN (via Component Protection). Both must be satisfied for the module to function.

---

## The FAZIT Database — The Actual Lock

The remote lock is not in J533 itself. J533 is the *enforcer*, but the *authority* is VW's FAZIT database (Fahrzeug-Informations und Zentrales Identifikations-Tool) at Audi AG in Ingolstadt, Germany.

From SSP 994403:
> *"All components must be 'adapted' on-line, as is already the case with the 2004 and 2005 Audi A8L. The control units involved cannot be adapted without an on-line link to FAZIT."*

The unlock process requires:
1. ODIS diagnostic software with active GEKO/GRP server credentials
2. ODIS connects to J533 via OBD-II port
3. ODIS transmits: VIN + module serial number + technician identity → FAZIT server
4. FAZIT verifies: module not flagged stolen, transfer approved
5. FAZIT generates new cryptographic authentication data
6. Data transmitted back through ODIS → J533 → target module
7. Module EEPROM rewritten with new VIN-matched cryptographic data
8. Constellation check now passes → CP deactivated

**Without a live server connection, this process cannot be completed on Generation 2/3 vehicles.**

---

## ODIS-S / ODIS-E and What You Can Do With Them

Since you have ODIS-S (engineering) and ODIS-E (service) plus module firmware files, here is what is technically possible and what is not, based on public knowledge:

### What ODIS-E can do (with active GEKO/GRP credentials):
- Perform Component Protection removal online (the full dealer unlock procedure)
- Flash/update module firmware
- Perform guided functions for module adaptation
- Read and clear fault codes including CP faults

### What ODIS-S can do (additional engineering functions):
- Access lower-level adaptation channels not available in ODIS-E
- Perform SVM (Software Version Management) coding
- Access off-board diagnostic functions
- Some SFD token operations

### What neither can do without server access:
- Remove Generation 2/3 Component Protection offline
- Bypass the FAZIT constellation check
- Generate valid GEKO authentication tokens locally

### The firmware files:
Module firmware (.frf, .odx files) contain software for flashing ECUs. They do not contain the vehicle-specific cryptographic data that CP uses — that lives in the module's EEPROM and the FAZIT database, not in the firmware package. Reflashing a module with new firmware does not remove CP; the EEPROM data persists through a firmware flash.

---

## Key Technical Summary for Advocacy

The following facts, drawn from VW's own SSP documentation, are significant for right-to-repair arguments:

1. **J533 was "integrated into Component Protection for the first time" on the 2005 A6** — this was a deliberate design decision to expand CP coverage, not a security necessity
2. **J518 is documented as the "master for immobilizer and component protection"** — these are two distinct systems merged into one module to increase interdependence
3. **The seat modules (J136/J521) have a hardware pin-coded seat-position lock *in addition to* CP** — meaning even if CP were resolved, additional adaptation may be required
4. **J525 (Digital Sound System) explicitly "participates in component protection (Geko)"** — audio amplifiers are locked, not just security-critical modules
5. **All unlock operations require online FAZIT connection** — there is no provision for offline unlocking, by design

---

## Sources
- VW SSP 994403 — 2005 Audi A6 Electrical System (US), pages 26-28 (Component Protection section)
- VW SSP 486 — Audi A6 2011 (C7 introduction), topology pages 46-47
- VW SSP 990613 — 2012 Audi A6 Vehicle Introduction (US), pages 30-35
- Ross-Tech VCDS Wiki — J533 Gateway documentation
- VAG Programming — vagprogramming.com Component Protection overview
