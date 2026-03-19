# J533 Gateway Deep Dive: Constellation Process and Part Number Reference

*The most technically detailed document in this repository. Based on hardware teardown research, community reverse engineering, and publicly available documentation.*

---

## The J533 is Two Systems in One

The critical discovery that changes everything about understanding CP:

**The Lear gateway on your C7 A6 contains two completely separate storage systems:**

1. **External EEPROM (95320)** — SPI chip, 32Kbit — stores coding, adaptation, bus topology config, some CP-adjacent data
2. **Internal NEC/Renesas D70F3433(A) Microcontroller flash** — 16-bit MCU — stores the actual Component Protection constellation data

Community hardware research confirmed this explicitly:
> *"Component Protection data are stored on D70F3433(A) 16-bit Microcontroller. Cloning the ATMEL EEPROM only suppressed the CP fault — it did NOT permanently fix it."*

This is why the EEPROM-only approach from older tools is unreliable on the Lear platform. The EEPROM clone tricks the module into not displaying CP faults, but the actual cryptographic constellation lives in the MCU flash. The MCU is what actually enforces the check.

---

## The J533 Hardware Architecture (C7 Platform — Lear Gateway)

Based on community teardowns of the 8T0-series Lear gateway (same architecture as your 4G0-series):

### Primary Chip: NEC/Renesas D70F3433(A)

| Parameter | Value |
|---|---|
| Manufacturer | NEC Electronics / Renesas |
| Series | V850E/PH3 (16-bit RISC) |
| Variant | D70F3433(A) |
| Architecture | V850 core |
| Flash | Internal — contains the CP constellation data |
| Interface | V850 Fetch Bus (VFB) |
| Programmer required | CarProTool or similar V850 programmer |

**What lives in the MCU flash:**
- The CP constellation table — serial numbers of all enrolled modules
- Cryptographic binding keys for each CP-protected module
- VIN association data
- The CP state flags (active/inactive per module)
- Bus topology routing configuration
- Firmware (the gateway operating software itself)

**Why this matters:** When ODIS runs CP removal, it writes new constellation data to this MCU flash via a secure, server-authenticated UDS routine. The EEPROM is essentially config storage; the MCU is the security brain.

### Secondary Chip: External EEPROM

| Platform | Chip | Size | Interface |
|---|---|---|---|
| Temic gateway (A4 B7, A6 C6 early) | Atmel 25640AN | 64Kbit | SPI |
| Lear gateway (A6 C7, A7, A8 D4) | ST/Atmel 95320 | 32Kbit | SPI |

**What lives in the EEPROM on the Lear gateway:**
- Gateway coding (the long hex string you see in VCDS)
- Adaptation channel values
- Bus topology: which modules are expected on which sub-bus
- Energy management configuration
- Some CP-adjacent module presence data (but NOT the cryptographic constellation)
- The VCID (Vehicle Component Identification string)

**The EEPROM clone trick:** When you clone the EEPROM from a matching gateway, J533 loads configuration that matches the expected module list — so it stops flagging CP because the external config says "these modules are fine." But the MCU still has the original cryptographic data, so the security is only superficially bypassed. This is why the community notes it "suppresses but doesn't fix" CP.

### Supporting Chips (Temic/early Lear, documented)

| Chip | Function | Relevance to CP |
|---|---|---|
| TLE6250G (Infineon) | High-speed CAN transceiver | Physical CAN interface |
| TJA1041AT (NXP) | High-speed CAN with standby | Powertrain/Extended CAN |
| TJA1055T/c (NXP) | Fault-tolerant CAN | Convenience CAN |
| ATA6662 (Atmel) | LIN transceiver | LIN bus (seat modules etc.) |
| T6020AM | Watchdog microcontroller | Gateway health monitoring |

---

## The Constellation Process — Exactly How It Works

The "constellation" is VW's term for the set of module identities J533 expects to see on each ignition cycle. Here is the full process, from power-on to CP enforcement:

### Phase 1: Power-Up (Terminal 30 applied)

1. J533 MCU wakes from sleep/standby
2. MCU loads constellation table from internal flash
3. MCU initializes all CAN transceivers and bus interfaces
4. J533 begins listening on all sub-buses (Convenience CAN, Powertrain CAN, etc.)

### Phase 2: Wake Sequence (Terminal 15 — ignition ON)

1. J533 broadcasts wake-up messages on all sub-buses
2. Each control module wakes and transmits its **Node Identifier** — a broadcast that includes:
   - Module part number
   - Module serial number
   - Current software version
   - Module status flags
3. J533 collects all Node Identifiers and builds the current bus topology picture

### Phase 3: Constellation Verification

For each module enrolled in CP, J533 performs:

```
For each module M in CP_enrollment_table:
    received_serial = NodeID[M].serial_number
    expected_serial = constellation_table[M].expected_serial
    
    if received_serial != expected_serial:
        send CP_restrict_command(M)
        set DTC: "Component Protection Active" in M
    else:
        # Module OK — no action needed
        pass
```

**The comparison is serial number based, not VIN-hash based.** This is confirmed by community analysis and explains why:
- Moving the same module between two cars triggers CP (serial doesn't match new car's constellation table)
- Replacing J533 itself triggers CP on all modules (new J533 has no constellation table — all serials are "unknown")
- Battery disconnection can trigger CP if it causes J533 to reload a corrupted or reset constellation table
- Your original J255 now fails — because ODIS wrote the *donor* J255's serial into your J533's constellation table during the previous GEKO session

### Phase 4: CP Restriction Enforcement

When J533 sends the CP restriction command to a module:

```
J533 → J255 (0x746): UDS WriteDataByIdentifier or RoutineControl
  [Security level required: CP enforcement key]
  [Payload: CP_ACTIVE flag]

J255 receives this, writes CP_ACTIVE=1 to its own EEPROM
J255 enters restricted mode (defrost only, no full HVAC)
J255 stores DTC: component protection active
```

**The restriction is enforced by the module itself** — J533 only signals it. The module checks its own CP flag in its EEPROM on every power cycle. This is why:
- Clearing the DTC doesn't help (module rereads its own EEPROM and resets the flag)
- VCDS can't fix it (can't write the CP flag without security level access)
- You need a full ODIS CP removal to rewrite the flag with proper authentication

### Phase 5: Ongoing Monitoring

CP is not just checked at startup. J533 periodically re-verifies the constellation during vehicle operation:
- Typically on every ignition cycle
- Can also re-check on bus topology changes (module dropping off / coming back)
- Explains spontaneous CP activation during long battery events

---

## What ODIS Does During CP Removal (Reconstructed Protocol)

Based on community captures and ODIS behavior analysis:

```
1. ODIS opens Extended Diagnostic Session with J533 (0x710)
   → 02 10 03

2. ODIS performs SecurityAccess on J533 at CP security level (level varies by platform)
   → 02 27 [odd_level]           (Seed request)
   ← [seed_bytes]                (J533 returns random seed)
   → [key_bytes]                 (ODIS calculates key from seed)
   ← 02 67 [even_level]          (Security granted)

3. ODIS reads module identification from J255 via J533 routing
   → 03 22 F1 8C                 (Read ECU Serial Number DID)
   ← [J255 serial number]
   → 03 22 F1 90                 (Read VIN DID)  
   ← [VIN currently in J255]

4. ODIS builds CP request payload:
   {
     target_module: "J255",
     target_serial: [J255 serial],
     current_vin: [J255 current VIN],
     new_vin: [host vehicle VIN],
     gateway_serial: [J533 serial],
     technician_id: [GEKO/GRP login],
     timestamp: [current UTC]
   }

5. ODIS sends payload to GEKO/GRP server over HTTPS
   POST https://geko.vw.com/api/cp/authorize
   {payload}

6. Server queries FAZIT:
   - Is J255 serial flagged stolen? → FAZ4990E if yes
   - Is transfer authorized? → Authorization token if yes

7. Server returns signed authorization token
   {token, signature, expiry}

8. ODIS executes CP removal routine on J533:
   → 0x31 01 [CP_ROUTINE_ID] [token_payload]
   (RoutineControl, Start, routine ID, signed token)

9. J533 verifies token signature using its embedded public key
   If valid:
   → J533 sends new constellation write to J255:
     → J255 writes CP_ACTIVE=0 to its EEPROM
     → J255 writes new VIN binding to its EEPROM
     → J255 updates constellation entry in J533 MCU flash

10. J533 updates its constellation table in MCU flash:
    constellation_table[J255].expected_serial = J255_serial
    constellation_table[J255].bound_vin = new_vin

11. J255 sends positive response, stores "CP removed" in operational log
12. ODIS confirms completion to technician
```

**The token validation in step 9 is the key barrier.** The token is asymmetrically signed by the GEKO/GRP server using a private key. J533 has the corresponding public key embedded in its MCU firmware. Without the server's private key to generate a valid signature, the routine will be rejected regardless of what you send.

This is what makes the Lear platform harder than the Temic platform — on Temic, the "constellation" data was simpler and lived more fully in the EEPROM, making it accessible without cryptographic tokens.

---

## Part Numbers for Flashdaten Search

### Your J533 Gateway — 4G0907468x Family

The `4G` prefix identifies the A6/A7 C7 platform (internal code "AU57"). All variants are Lear gateway architecture.

| SW Part Number | HW Part Number | Component String | Notes |
|---|---|---|---|
| `4G0 907 468 A` | `4G0 907 468 A` | `J533--Gateway H13 xxxx` | Early production 2011 |
| `4G0 907 468 B` | `4G0 907 468 B` | `J533--Gateway H13 xxxx` | 2011-2012 |
| `4G0 907 468 C` | `4G0 907 468 C` | `J533--Gateway H13 xxxx` | 2012 common variant |
| `4G0 907 468 D` | `4G0 907 468 D` | `J533--Gateway H13 xxxx` | 2012-2013 |
| `4G0 907 468 E` | `4G0 907 468 E` | `J533--Gateway H13 xxxx` | 2013 — likely your car |
| `4G0 907 468 F` | `4G0 907 468 F` | `J533--Gateway H13 xxxx` | 2013-2014 |
| `4G0 907 468 G` | `4G0 907 468 G` | `J533--Gateway H13 xxxx` | 2014 common |
| `4G0 907 468 H` | `4G0 907 468 H` | `J533--Gateway H13 xxxx` | 2014-2015 |
| `4G0 907 468 J` | `4G0 907 468 J` | `J533--Gateway H13 xxxx` | 2015 |
| `4G0 907 468 K` | `4G0 907 468 K` | `J533--Gateway H13 xxxx` | 2015-2016 |
| `4G0 907 468 L` | `4G0 907 468 L` | `J533--Gateway H13 xxxx` | FL1 facelift intro |
| `4G0 907 468 M` | various | `J533--Gateway H13 xxxx` | Facelift |
| `4G0 907 468 AD` | `4G0 907 468 AC` | `J533--Gateway H13 0050` | Late C7 (confirmed from live scan) |

**ASAM Dataset identifiers for J533 on C7:**
- `EV_GatewPkoUDS 001013` through `001028` — various software builds
- `EV_GatewPkoUDS 002000`, `002011` — later builds
- `EV_GatewCONTIAU736 002060` — Continental-manufactured variant

**Dataset number** (appears in VCDS scan, different from part number):
- `4G0909515x` series — this is the calibration dataset number
- Example confirmed: `4G0909515H 0067`

**Flashdaten filename pattern to search:**
```
4G0907468*.sgo     ← Flash files for J533 C7
4G0907468*.frf     ← Alternative format
4G0909515*.pdx     ← Dataset/calibration files
EV_GatewPkoUDS*    ← ASAM dataset files
```

### Related C7 Gateways (Share Architecture)

| Platform | Part Number Prefix | Notes |
|---|---|---|
| A7 Sportback 4G | `4G0 907 468x` | Identical to A6 C7 |
| A8 D4 (4H) | `4H0 907 468x` | Closely related Lear gateway |
| Q7 4L (late) | `4L0 910 468x` | Older Temic, different |
| Touareg 7P | Various | Similar Lear architecture |

### J255 Climatronic — Part Numbers

The J255 for your 2013 A6 C7 depends on trim level:

| Config | Part Number | ASAM Dataset |
|---|---|---|
| 2-zone base | `4G0 820 043x` | `EV_AirCondiBasisUDS` |
| 2-zone comfort | `4G1 820 043x` | `EV_AirCondiBasisUDS` |
| 4-zone comfort | `4G0 820 043x` | `EV_AirCondiComfoUDS` |
| Digital 4-zone (donor) | `4G1 820 043x` | `EV_AirCondiComfoUDS` |

**Flashdaten search:**
```
4G0820043*.sgo
4G1820043*.sgo
EV_AirCondiBasisUDS*
EV_AirCondiComfoUDS*
```

### J136 — Memory Seat Driver / Steering Column

| Part Number | ASAM | Notes |
|---|---|---|
| `4H0 959 760x` | `EV_SeatMemoDriv*` | Standard on A6 C7 |
| `4G0 959 760x` | `EV_SeatMemoDriv*` | Alternative |

**Flashdaten search:**
```
4H0959760*.sgo
4G0959760*.sgo
EV_SeatMemo*
```

### J521 — Memory Seat Passenger

| Part Number | ASAM | Notes |
|---|---|---|
| `4H0 959 748x` | `EV_SeatMemoPasse*` | Standard |
| `4G0 959 748x` | `EV_SeatMemoPasse*` | Alternative |

---

## Flashdaten File Structure — What to Look For

Within your ODIS-S installation, the PostSetup (flashdaten) contains:

### File Types

| Extension | Content | Relevant to CP? |
|---|---|---|
| `.sgo` | Flash container (compressed) — contains .frf or .pdx | Yes — contains firmware |
| `.frf` | Flash Record File — ECU firmware binary | Yes — contains UDS service definitions |
| `.pdx` | Parameter/Dataset container | Yes — contains routine definitions |
| `.odx` | Open Diagnostic eXchange — XML format | **Most relevant** — defines all services, DIDs, routines |
| `.rod` | Read-Only Data — fault code texts | Less relevant |
| `.clb` | Calibration/label file | Less relevant |

### What to Search in ODX Files

The ODX files for J533 will contain the CP routine definition. Look for:

```xml
<!-- Search for these in EV_GatewPkoUDS*.odx or similar -->
<ROUTINE-SPEC>
  <SHORT-NAME>ComponentProtection</SHORT-NAME>   <!-- or Komponentenschutz -->
  ...
  <REQUEST-REF DOCREF="..." DOCTYPE="..." />
</ROUTINE-SPEC>

<!-- Also look for the routine identifier bytes -->
<CODED-CONST SEMANTIC="SERVICE-ID">0x31</CODED-CONST>  <!-- RoutineControl -->
<CODED-CONST SEMANTIC="TYPE">0x01</CODED-CONST>         <!-- Start routine -->
<CODED-CONST SEMANTIC="ROUTINE-ID">0xXXXX</CODED-CONST> <!-- CP routine ID -->
```

The security access levels for CP operations:
```xml
<SECURITY-ACCESS-SPEC>
  <SHORT-NAME>CP_Security</SHORT-NAME>
  <ACCESS-LEVEL>0xXX</ACCESS-LEVEL>  <!-- The level ODIS uses for CP -->
</SECURITY-ACCESS-SPEC>
```

### Where to Find These Files

In ODIS-S 25.x installation:
```
C:\ProgramData\ODIS-S\PostSetup\
    └── [brand folders: VW, AUDI, etc.]
        └── [various .sgo, .pdx, .odx files]

C:\ProgramData\ODIS-S\diagdata\
    └── [dataset folders organized by ASAM ID]
        └── EV_GatewPkoUDS\
            └── *.odx    ← This is your goldmine
```

To extract .sgo files:
```bash
# ODIS uses standard zip-based compression for .sgo containers
# Try: rename .sgo to .zip and extract
# Or use 7-zip which handles most VAG container formats
```

To read .odx files:
```python
import xml.etree.ElementTree as ET

tree = ET.parse('EV_GatewPkoUDS_001_AU57.odx')
root = tree.getroot()

# Find all routine definitions
for routine in root.iter('ROUTINE-SPEC'):
    name = routine.find('SHORT-NAME')
    if name is not None and ('protect' in name.text.lower() or 'komponent' in name.text.lower()):
        print(f"Found CP routine: {name.text}")
        # Extract the routine details...
```

---

## The Two-Chip Model — Implications for Research

Understanding that J533 uses both an EEPROM and an MCU with internal flash leads to important research directions:

### Research Path 1: OBD DID Scan of J533

Using python-udsoncan and your Mongoose cable, enumerate all Data Identifiers on J533. Some of these DIDs will return constellation-related data in readable form:

```python
# Relevant DIDs to probe on J533 (0x710/0x77A):
KNOWN_DIDS = {
    0xF190: "VIN",
    0xF18C: "ECU Serial Number", 
    0xF187: "Spare Part Number",
    0xF189: "Software Version",
    0xF197: "Vehicle Type",
    0xF1A0: "ECU date",
    # Scan 0x0000-0x9999 for unknown ones
}
```

DIDs in the range `0x0100-0x0999` often contain manufacturer-specific data including CP state. These will appear in the ODX files once you find them.

### Research Path 2: The MCU Flash Read via V850 Bus

The D70F3433 (V850E core) has a serial programming interface accessible via test points or component pads on the gateway PCB. With a V850 programmer (CarProTool supports this), you can read the MCU flash directly — bypassing all software protection. This is how the professional tools do deep CP operations on the Lear platform.

The V850 VFB (Velocity Fetch Bus) interface on the D70F3433:
- Standard 4-wire SPI-like interface
- Requires dedicated programmer (not a generic SPI reader)
- Test pins are typically accessible without desoldering the MCU

### Research Path 3: EEPROM + MCU Correlation

If you read both the EEPROM (95320) and the MCU flash (D70F3433) from both your host and donor J533, you can:
1. Diff the two EEPROMs to find what changed after the GEKO session
2. Diff the two MCU flash images to identify the constellation table structure
3. Map the EEPROM fields to their MCU flash counterparts
4. Understand the relationship between what ODIS writes where

This is the kind of research that, if documented publicly, would be a major contribution to the right-to-repair documentation project.

---

## Confirmed Service Layer — AU57X MCD Project (September 2022)

*The following section is based on direct extraction from the AU57X ODIS diagnostic project (DVR 72, revision date 2022-09-22). String databases were decompressed from `AStringData.data.gz` (36 MB decompressed, 601K strings) and filtered for CP and gateway-relevant identifiers. This moves the information in preceding sections from "reconstructed from behavior" to "confirmed from source."*

---

### Library Architecture — How CP Routines Are Structured

The CP routines do **not** live directly in the `BV_GatewUDS` base variant. They live in a shared library that is included into each relevant ECU's diagnostic project:

```
ES_LIBCompoProteGen3V12   ← CP authentication routines (revision 004001, 2022-07-12)
ES_LIBCompoProteGen3V11   ← Earlier version (revision 001002, 2020-06-24)
ES_LIBCOPKWPSubsy         ← KWP2000 subsystem CP (older platforms)
BL_LIBTheftProte          ← Theft protection / key download library
```

`ES_LIBCompoProteGen3V12` is the current library used on the C7 platform. Both V11 and V12 are present in the AU57X project for backwards compatibility across the model run.

---

### Confirmed Service Names — J533 Gateway (BV_GatewUDS)

These service identifiers were confirmed directly from the AU57X AStringData string database:

#### Constellation List (the serial number table)

| Service identifier | UDS service | Direction |
|---|---|---|
| `DC_BV_GatewUDS_DiagnServi_ReadDataByIdentGatewCompoList` | ReadDataByIdentifier (0x22) | Read constellation from J533 |
| `DC_BV_GatewUDS_DiagnServi_WriteDataByIdentGatewCompoList` | WriteDataByIdentifier (0x2E) | Write constellation to J533 |
| `DC_BV_GatewUDS_SinglJob_CompoListRead` | Single job wrapper | Read via job |
| `DC_BV_GatewUDS_SinglJob_CompoListWrite` | Single job wrapper | Write via job |

The DID address for `GatewCompoList` is **confirmed as `0x04A3`** from the Linux-native dumpMWB.py
extraction of the AU57X MCD project. Empirical probing is no longer necessary for this platform.
See `au57x-mwb-extraction-confirmed-dids.md` for the full confirmed DID map including all six
constellation sub-DIDs (`0x2A26`–`0x2A2C`).

#### Constellation Data Object Structure

The `GatewCompoList` DID returns a structured record. The following sub-fields were confirmed from AU57X:

| Field name | Description |
|---|---|
| `GatewCompoListAlloc` | Allocation status — is slot occupied |
| `GatewCompoListDiagP` | Diagnostic path to the module |
| `GatewCompoListDtc` | DTC associated with this slot |
| `GatewCompoListPrese` | Module presence flag |
| `GatewCompoListSleep` | Sleep state |
| `GatewCompoListTPIdent` | Transport Protocol identifier (CAN ID) |
| `GatewCompoListECUIdent` | ECU identity (serial number — the key field) |
| `GatewCompoListECUNodeAddre` | Node address (added in V52+ of the UDSDistr library) |
| `GatewCompoListEcuAuthe` | ECU authentication field (added in V52+) |

The `ECUIdent` / `ECUNodeAddre` fields are what get compared against the module's actual serial at each ignition cycle.

#### Other Relevant J533 Services

| Service identifier | Relevance |
|---|---|
| `DC_BV_GatewUDS_DiagnServi_WriteDataByIdentTheftProteData` | Write CP keys to gateway calibration |
| `DC_BV_GatewUDS_DiagnServi_WriteDataByIdentCalibData` | Write calibration data (contains IKA key row) |
| `TBP_BV_GatewUDS_TAB_RecorDataIdentCalibDataWrita.TABROW_IKAKeyL` | IKA key row within calibration write table |

---

### Confirmed Service Names — J255 HVAC (BV_AirCondiUDS)

#### TheftProteData — The CP Key Download

This is the write operation that actually clears CP from J255. Two key download rows confirmed:

| TABROW name | Description |
|---|---|
| `TABROW_TheftProteDownlGFAKey` | Downloads the GFA key (Geräteinzelfreigabe — server authorization key) |
| `TABROW_TheftProteDownlIKAKey` | Downloads the IKA key (installation key, ties module to vehicle) |

Both rows are within:
```
TBP_BV_AirCondiUDS_TAB_RecorDataIdentTheftProteDataWrita
```

The full service call confirmed:
```
DC_BV_AirCondiUDS_DiagnServi_WriteDataByIdentTheftProteData
```

Additional confirmed rows in calibration write for J255:
- `TABROW_GKAKey` — GKA key (Geräteklassenfreigabe — device class authorization)
- `TABROW_IKAKey` — IKA key

---

### Confirmed Service Names — CP Authentication Library (ES_LIBCompoProteGen3V12)

The actual UDS RoutineControl calls for CP authentication flow through this library:

| Service identifier | UDS mapping |
|---|---|
| `DC_ES_LIBCompoProteGen3V12_DiagnServi_RoutiContrStartRoutiCompoProte` | 0x31 01 [routine_id] — Start CP auth routine |
| `DC_ES_LIBCompoProteGen3V12_DiagnServi_RoutiContrStopRoutiCompoProte` | 0x31 02 [routine_id] — Stop CP auth routine |
| `DC_ES_LIBCompoProteGen3V12_DiagnServi_RoutiContrRequeRoutiResulCompoProte` | 0x31 03 [routine_id] — Request routine result |
| `DC_ES_LIBCompoProteGen3V12_DiagnServi_ReadDataByIdentCompoProteData` | 0x22 [did] — Read CP data |

#### Confirmed CP Activation Parameter DID Range

The following parameter names were extracted directly from AU57X, confirming the DID address range for CP activation:

```
Param_CompoProteActiv0xEA61
Param_CompoProteActiv0xEA62
Param_CompoProteActiv0xEA63
Param_CompoProteActiv0xEA64
```

This confirms the CP activation DID family sits at **`0xEA61–0xEA64`**. These are likely the per-module CP activation flags read by J533 during enforcement. The full `ReadDataByIdentCompoProteData` DID address for the CP library is within or adjacent to this range.

Additional confirmed CP parameter names:
- `Param_CompoProte` — base CP status
- `Param_CompoProteGener` — CP generation (Gen 3 = V12 library)
- `Param_CompoProteK0` — CP key 0
- `Param_CompoProteRole` — role (master / slave)
- `Param_CompoProteNoOrIncorBasicSetti` — no or incorrect basic settings flag
- `Param_StatuOfCompoProteSlave` — slave status field
- `Param_ListOfStatuOfCompoProteSlave2` — list of all slave statuses (dynamic length)
- `Param_TestProgrCompoProteK0` — test programmer CP K0 (used during factory programming)

#### Routine Option Structures

The RoutineControl Start payload for CP authentication uses these option structures:

| Structure name | Description |
|---|---|
| `STRUC_RoutiContrOptioStartCompoProteAuthe` | Options for StartRoutine CP authentication |
| `STRUC_RoutiContrOptioActivOfCompoProteFunct` | Options for routine that activates CP function |
| `STRUC_RoutiStatuRecorStartCompoProteAuthe` | Status record returned by StartRoutine |
| `STRUC_RoutiStatuRecorActivOfCompoProteFunct` | Status record for activation routine |
| `STRUC_DataRecorCompoProteChara` | CP characteristics data record |
| `STRUC_DataRecorStatuOfCompoProteSlave2` | Dynamic list of slave CP statuses |

---

### Confirmed ES_LIBCOPKWPSubsy — KWP Subsystem (per-module CP identity)

This library handles per-module CP identity reads on KWP2000 sub-buses (used for modules like seat ECUs connected via the KWP subsystem). Confirmed services:

```
DC_ES_LIBCOPKWPSubsy_DiagnServi_ReadDataByIdentVWSlaveIdentDataCOPMulti
DC_ES_LIBCOPKWPSubsy_DiagnServi_ReadDataByIdentVWSlaveFAZITIdentStrinCOP
DC_ES_LIBCOPKWPSubsy_DiagnServi_ReadDataByIdentVWSlaveSeriaNumbe COP
DC_ES_LIBCOPKWPSubsy_DiagnServi_WriteDataByIdentVWSlaveCodinValueCOP
```

This is the mechanism J533 uses to read identity information from modules on sub-buses (not directly on the main CAN), and is relevant for seat module CP.

---

### Protocol Path (Confirmed from AU57X)

The full diagnostic addressing chain confirmed from AU57X index:

```
[Protocol]PR_UDSOnCAN
  └─ [EcuBaseVariant]BV_GatewUDS
       └─ [EcuVariant]EV_GatewPKOUDS_001   ← Standard C7 gateway variant
       └─ [EcuVariant]EV_GatewPKOUDS_002   ← EV/hybrid variant
       └─ [EcuVariant]EV_GatewUDS_001      ← Older variant

[Protocol]PR_UDSOnCAN
  └─ [EcuBaseVariant]BV_AirCondiUDS
       └─ [EcuVariant]EV_AirCondiBasisUDS_002/003/004   ← 2-zone HVAC
       └─ [EcuVariant]EV_AirCondiComfoUDS_002/003/004   ← 4-zone HVAC
```

The C7 A6 non-hybrid gateway maps to `EV_GatewPKOUDS_001`.

---

### Previously Required Windows — Now Resolved on Linux

The items below were previously listed as requiring `pbl.dll` (Windows-only). They have been resolved
by running `dumpMWB.py` natively on Linux using the open-source PBL library. See
`au57x-mwb-extraction-confirmed-dids.md` for complete details.

| Item | Status |
|---|---|
| Exact hex DID addresses for `GatewCompoList` and sub-DIDs | ✓ Confirmed: `0x04A3`, `0x2A26`–`0x2A2C` |
| Exact byte layout of `GatewCompoList` response | ✓ Confirmed: END-OF-PDU-FIELD of 1-byte bitfields |
| ECU Name code for J255 | ✓ Confirmed: `8` (Air Conditioning) from `0x2A2A` TEXTTABLE |
| IKA/GKA key DID addresses on J255 | ✓ Confirmed: `0x00BE` (IKA), `0x00BD` (GKA), 34 bytes each |
| Security access level for CP writes | ✓ Confirmed: None required — extended session only, GEKO token provides auth |
| Routine ID bytes for `RoutiContrStartRoutiCompoProte` | ✗ Still open — in `ES_LIBCompoProteGen3V12.sd.db` |

---


- MHH Auto — Gateway hardware teardown thread (D70F3433 identification, EEPROM clone research)
- Ross-Tech Wiki — A6 C7 gateway part number documentation  
- Audizine — Live VCDS scans showing 4G0907468AD confirmed part number
- ABRITES VAG manual — CP procedure details
- Jim Ellis Audi Parts — OEM parts fitment data for 4G0907468C
- eBay listing data — 4G0907468G fitment (2012-2018 confirmed)
- diagnostics.vis4vag.com — A6 4G gateway variant listing (EV_GatewPkoUDS variants)

---

## Session Confirmation — Feb 27, 2024 ODIS Log

*The theoretical model described above was written from behavioral observation and
community research. The following section upgrades it to confirmed from a real
ODIS CP removal session captured on VIN `WAUGGAFC7DN120188` — a 2013 Audi A6 C7,
3.0T TFSI CGWB, built Neckarsulm.*

*Full session analysis: `technical/odis-cp-session-wauggafc7dn120188.md`*

---

### The Car-Wide Checksum Model

The most useful mental model for understanding CP is to think of J533's
constellation as a **car-wide integrity checksum across all enrolled modules.**

A traditional checksum runs over bytes of data and produces a value that only
matches if every byte is correct. The CP constellation works the same way, but
across physical modules on the CAN bus:

```
Constellation integrity = f(
    J533 serial,
    J518 serial,
    J255 serial,
    J519 serial,
    J234 serial,
    J285 serial,
    J525 serial,
    Radio serial,
    J794 serial,
    J854 serial,
    J855 serial,
    J136 serial,
    ... all enrolled modules
)
```

If any one module is swapped out, its serial number no longer matches J533's
expected value for that slot — exactly like a corrupted byte in a checksum.
The system flags the mismatch as Component Protection Active.

**This is why you can't fix CP on just one module in isolation.** The constellation
is a system-level property. The integrity check fails when any enrolled component
doesn't match. ODIS's CP removal procedure is designed around this — it clears
and re-enrolls the entire car in a single atomic session.

---

### What the Feb 2024 Session Confirms

**Confirmed: J533 is the master and the session clears everything atomically.**

The session log shows ODIS calling
`j533_4G_90_UDS_2_0613_21_KS_SGs_ermitteln_00021` — "determine CP control
modules" — on J533 first. J533 returned its full enrollment list of 12 modules.
ODIS then cleared every single one sequentially before updating the constellation.

12 modules cleared in one session:

| Module | Protocol | Result |
|--------|----------|--------|
| J533 Gateway | UDS | ✓ cleared (master first) |
| J518 Access/Start (Kessy) | KWP2000 | ✓ cleared |
| J255 Climatronic | UDS | ✓ cleared |
| J519 Vehicle Electrical System | KWP2000 | ✓ cleared |
| J234 Airbag | UDS | ✓ cleared |
| J285 Instrument Cluster | UDS | ✓ cleared |
| J525 Digital Sound System | KWP2000 | ✓ cleared |
| R-- Radio | KWP2000 | ✓ cleared |
| J794 MMI (Info Electronics 1) | UDS | ✓ cleared |
| J854 Left Front Seat Belt Tensioner | KWP2000 | ✓ cleared |
| J855 Right Front Seat Belt Tensioner | KWP2000 | ✓ cleared |
| J136 Memory Seat/Steering Column | KWP2000 | ✓ cleared |

J521 (passenger memory seat) was attempted but returned
`ECF_OPEN_LOGICAL_LINK_FAILED` — not installed on this car.

**Confirmed: DID `0x04A3` is the constellation and it changed.**

Before and after the session, J533's constellation DID changed:

```
Before:  FD A1 E9 0C FE 62 64 8D 00 00
After:   FD A1 E8 0C FE 62 60 0D 00 00
```

Four bytes changed, each representing bit-level enrollment state changes
across the module slots. This is the concrete proof that `0x04A3` is the
"checksum field" — the single value on J533 that encodes the pass/fail state
of the entire car's module complement.

**Confirmed: The IKA key for J136 (34 bytes, DID `0x00BE`).**

This is the only unmasked IKA key blob in the entire session log. All other
modules had their keys masked as `******` in the ODIS log. J136 went through
a different code path (direct `WriteDataByIdentifier` rather than the masked
`TrainICA` KWP service) and its key was logged verbatim:

```
E6 2B 41 D1 1C 44 AF 20 21 77 FB 1F 27 4B 0A C2
D1 5B D2 62 E4 FD 27 AB 61 D1 23 C2 F1 5A 2C 93 26 00
```

This 34-byte value is the individual module authorization key (IKA) — the
component-level "checksum" that ties this specific module to this specific car.

---

### The Two-Layer Checksum Architecture

The session log reveals that CP actually operates on two distinct layers, which
work together like nested checksums:

**Layer 1 — The Constellation (J533 DID `0x04A3`):**
A 10-byte bitmap on J533 encoding which modules are enrolled and in what state.
This is the car-level check. J533 holds it. It gets rewritten during any CP
operation. Think of it as the outer checksum — it covers the whole system.

**Layer 2 — The IKA Key (DID `0x00BE` on each module):**
A 34-byte cryptographic value written individually to each enrolled module.
This is the module-level check. Each module holds its own copy. It proves that
the module was authorized by GEKO for this specific VIN. Think of it as the
inner checksum — it covers the individual component.

Both layers must be consistent for CP to be cleared:

```
CP cleared iff:
    J533 constellation[module_slot] == ENROLLED
    AND
    module.DID_0x00BE == valid_IKA_key_for_this_VIN
```

Swapping a module breaks Layer 2 (new module has no valid IKA key) and
eventually Layer 1 (J533's constellation flags the mismatch). Replacing J533
itself breaks Layer 1 (new gateway has an empty constellation) and potentially
Layer 2 (if the IKA key derivation includes the J533 serial).

---

### Inputs GEKO Used to Generate the IKA Key

From the session log, before contacting GEKO, ODIS collected these values:

| Source | DID | Value |
|--------|-----|-------|
| VIN (from J518) | F190 | `WAUGGAFC7DN120188` |
| J518 FAZIT | F17C | `HLH-W41 21.02.13 1003 1126` |
| J518 HW number | F191 | `4H0907064CR` |
| J533 FAZIT | F17C | `LAK-000 22.02.13 2009 7162` |
| J136 FAZIT | F17C | `CU5-SIB 21.03.11 0002 0968` |
| Key fob transponder | TrainICA | `******` (masked) |

GEKO received these values and returned the IKA key blob. The key fob
transponder data is the one unknown input — it may be used only as an
authorization check (proving physical possession) or it may be mathematically
incorporated into the key derivation. This remains an open research question.

---

### The Replacement Module Problem — Explained by the Model

When the original J255 failed and was physically removed from the car, the Feb
2024 ODIS session enrolled the replacement (4-zone) J255 in its place. The session:

1. Read the replacement J255's FAZIT and serial number
2. Sent those to GEKO along with the VIN and other module data
3. Received an IKA key blob for that replacement unit
4. Wrote the blob to the replacement J255's DID `0x00BE`
5. Updated J533's constellation `0x04A3` to include the replacement J255's slot

The original broken J255 was out of the car during this session. It therefore:
- Was never sent to GEKO
- Never received an IKA key blob
- Is not referenced in J533's current constellation

When the original unit is plugged back in today:
- J533 checks its constellation → original J255 serial is not in the enrollment list
- J533 checks DID `0x00BE` on the original J255 → all zeros, no valid IKA key
- Both layers of the checksum fail simultaneously
- J533 asserts DTC U110100: Component Protection Active

**The fix requires updating both layers:**

Step 1 — Write a valid IKA key to the original J255's DID `0x00BE`.
Whether this is the same blob as all other modules (VIN-bound) or a unique blob
(module-serial-bound) is determined by reading DID `0x00BE` from multiple modules
and comparing. This is what the hardware test is designed to answer.

Step 2 — Update J533's constellation DID `0x04A3` to include the original J255.
The specific bit(s) that changed between the before/after values in the Feb 2024
session (`64 8D` → `60 0D` in bytes 5-6) identify the J255 slot position.

Both writes are UDS operations against J533 in extended session. SA2
authentication is required. No GEKO token is required if the IKA key is already
known — the token was only needed to generate the key the first time.

---

### Updated Research Status

| Claim | Status |
|-------|--------|
| J533 is the CP master | ✓ Confirmed — session clears J533 first |
| Constellation is atomic (all-or-nothing) | ✓ Confirmed — 12 modules cleared together |
| DID `0x04A3` is the constellation field | ✓ Confirmed — changed before/after |
| DID `0x00BE` is the IKA key, 34 bytes | ✓ Confirmed — J136 blob captured verbatim |
| CP routine ID is `0x0226` | ✓ Extracted from `ES_LIBCompoProteGen3V12.sd.db` |
| IKA key is VIN-bound vs module-bound | ✗ Open — hardware test pending |
| Transponder data in key derivation | ✗ Open — masked in log |
| J255 constellation slot bit position | ✓ Derivable — bytes 5-6 of `0x04A3` changed |
