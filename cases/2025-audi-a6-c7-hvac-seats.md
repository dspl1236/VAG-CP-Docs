# Case: 2013 Audi A6 C7 — HVAC Control and Seat Modules, Salvage Donor Vehicle

**Date of incident:** 2025–2026  
**Submitted by:** Anonymous  
**Country/Region:** United States

---

### Vehicle details
- **Year:** 2013
- **Make:** Audi
- **Model:** A6 C7 (4G)

---

### What happened

**Parts affected:**
- HVAC / Climatronic control panel
- Seat memory/adjustment module(s)

**Source of replacement parts:**
- Salvage / used — owner purchased an entire second 2013 Audi A6 as a donor vehicle specifically for parts

**Did Component Protection activate?**
- Yes, immediately on installation of both modules

**What functionality was lost:**
- HVAC control panel: climate control functionality restricted
- Seat modules: memory and adjustment functions disabled

---

### Resolution attempt

**Unlock status:** Unresolved at time of submission  
**Server status:** GEKO system in transition; third-party access unreliable  
**Dealer option:** Available but requires $300–$500+ per module in unlock fees, on top of cost of donor vehicle already purchased

**Notable:** Owner purchased an entire second Audi A6 specifically to source OEM-identical salvage parts for cost-effective repair. Despite having perfect, matching donor parts, both modules are non-functional due to Component Protection. The total cost of acquiring the donor vehicle plus dealer CP removal fees approaches or exceeds the cost of purchasing new OEM parts — entirely defeating the purpose of the repair strategy.

---

### Impact

- Two modules purchased via a complete donor vehicle are non-functional
- Owner faces dealer CP removal fees of approximately $300–$500+ per module
- GEKO server uncertainty raises the possibility that CP removal may not be available at all in the near future
- Practical HVAC functionality compromised
- Seat adjustment/memory functions disabled

---

### Why this case matters for advocacy

This case illustrates several distinct harms:

1. **The economic trap in its most extreme form:** The owner went to extraordinary lengths — purchasing an entire second vehicle — to avoid dealer parts pricing. Component Protection negated that strategy entirely.

2. **GEKO sunset urgency:** If VW's authentication servers become unavailable before the owner can afford or access the unlock procedure, these modules may be permanently locked with no recourse.

3. **No security benefit:** The parts were sourced from an identical donor vehicle by the registered owner. Component Protection provided zero anti-theft benefit in this scenario and 100% anti-repair harm.

4. **Platform-specific vulnerability:** The Audi A6 C7 (4G platform, 2012–2018) uses Generation 2 Component Protection on seat and HVAC modules, with limited third-party tool support, making dealer unlock the only practical option.

---

### Confirmed Fault Codes

The following DTC codes were confirmed from the AU57X DIDB diagnostic database (HSQLDB dump, 92,583 rows):

| DTC (ISO/SAE) | VAG internal code | Description | Module |
|---|---|---|---|
| `U110100` | `7465` | Component protection active | J255, J136, J521 (stored in affected module) |
| `B1B0400` | — | Component protection master | J533 (stored in gateway when acting as CP master) |
| `B1B0300` | — | Component protection slave | J255 etc. (stored when enrolled as CP slave) |

`U110100` / VAG `7465` is the specific DTC this vehicle is experiencing in J255. It cannot be cleared without completing the CP removal routine — the module resets the flag from its own storage on every power cycle.

---

### Documentation
- NHTSA TSB 91-18-13TT (spontaneous CP activation acknowledgment)
- VAG Programming — Component Protection on C7 platform
- Audizine forums — seat retrofit CP documentation on A6/A7 C7
- AU57X ODIS MCD project, DVR 72 (September 2022) — DIDB fault code database

---

*This submission is released under CC0 (public domain).*

---

### Technical Investigation Update (March 2026)

Active diagnostic investigation of this vehicle is in progress using a Python-based UDS probe (`j533_probe.py`) connected via an ESP32 ISO-TP BLE bridge.

**Confirmed diagnostic chain from AU57X MWB extraction:**

The following DID addresses are confirmed for this vehicle's platform (C7 A6, EV_GatewPKOUDS_001 gateway variant):

- `0x04A3` — Gateway Component List coded bitmap. Decoding this tells us which slot index J255 occupies and whether it is currently enrolled.
- `0x2A2A` — Allocation table. ECU Name value `8` = Air Conditioning (J255). Reading this identifies J255's slot number.
- `0x00BE` — IKA Key on J255. 34 zero bytes = CP active, no key installed. This is the single most direct indicator of CP state.

**Pending:** Physical probe run with ESP32 bridge connected to OBD port. Once complete this section will be updated with:
- Actual constellation table showing J255 slot and status
- IKA key state confirming CP active/inactive
- VIN mismatch confirmation between J533 and J255

**Resolution path confirmed:** CP removal requires GEKO server authentication to write a new IKA key to J255 DID `0x00BE`. Third-party option: vw-geko.com (~€30–40 per module). Dealer option: $300–500+ per module. No offline solution exists that does not involve hardware-level MCU reprogramming of J533 (CarProTool, V850 programmer).

