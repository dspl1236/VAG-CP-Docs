# AU57X ODIS MCD Project — Diagnostic Database Findings

*Platform: Audi A6 C7 / A7 4G / A8 D4 (internal code AU57X)*  
*Source: AU57X ODIS MCD Project, DVR 72 — compiled 2022-09-22*  
*Extraction method: `AStringData.data.gz` decompressed (36 MB / 601K strings), grep-filtered for CP and gateway identifiers. No Windows tooling required.*

---

## What This Document Is

The ODIS MCD (Multi-Client Diagnostics) project for a given platform is the complete diagnostic definition used by ODIS when connected to a vehicle. It defines every UDS service, every DID, every routine, every adaptation channel, and every fault code for every module on the platform.

The AU57X project is the definition for the Audi A6 C7 / A7 4G / A8 D4 family. Its string databases contain human-readable identifiers for every diagnostic object — even when the actual binary encodings (hex addresses, byte layouts) require the proprietary `.bv.db` decoder to access.

This document records what was extracted from those string databases relevant to Component Protection research. It is a complement to the J533 constellation deep dive — where that document describes what happens, this one describes the confirmed service names and library structure that implement it.

---

## Project Metadata

| Field | Value |
|---|---|
| Project name | AU57X |
| DVR version | 72 |
| DVR revision label | 072102 |
| Revision date | 2022-09-22 |
| ODX generation | Gen A |
| Converter version | VW-MCD 19.0.2.0 |
| Build host | VWAGWOSOB127 |
| Converter call type | Command line (server-side build) |

The project was built on a VW internal tomcat server (`ODIS-TFD`) from a PDX source file, converted to the binary PBL runtime format used by ODIS-S clients.

---

## File Structure

The AU57X project contains approximately 280 `.bv.db` / `.sd.db` binary files (one per ECU base variant or shared library), plus:

| File | Description |
|---|---|
| `AStringData.data.gz` | All ASCII string identifiers — 36 MB decompressed |
| `AStringData.idx.gz` | Index into the string data |
| `UStringData.data.gz` | Unicode string data (UI labels) |
| `DVR-INFO.xml` | Project metadata and ODX revision list |
| `DatabaseVersionInfo.txt` | Build provenance |
| `index.xml` | Module-to-file mapping |
| `Jobs/*.jar` | Diagnostic job implementations (Java) |

The `.db` files use VW's proprietary Persistent Block Library (PBL) format. Decoding them requires `pbl.dll` from a Windows VW-MCD 19.x installation and the `dumpMWB.py` script from the ODIS-project-explorer tool. Everything in this document was extracted without that tooling, from the string databases alone.

---

## ECU Variants Present — CP-Relevant Modules

### J533 Gateway

| Identifier | Description |
|---|---|
| `BV_GatewUDS` | Base variant — all C7 gateways |
| `EV_GatewPKOUDS_001` | Standard petrol/diesel C7 (A6, A7) |
| `EV_GatewPKOUDS_002` | Hybrid/EV variant |
| `EV_GatewUDS_001` | Older gateway variant (early C7) |

Protocol path: `PR_UDSOnCAN → BV_GatewUDS → EV_GatewPKOUDS_001`

CAN IDs (from `LL_VINFO_AU57XCAN.LL_GatewUDS` link layer): TX `0x710`, RX `0x77A`

### J255 HVAC / Climatronic

| Identifier | Description |
|---|---|
| `BV_AirCondiUDS` | Base variant — all C7 HVAC |
| `EV_AirCondiBasisUDS_002/003/004` | 2-zone basic climate |
| `EV_AirCondiComfoUDS_002/003/004` | 4-zone comfort climate |

Protocol path: `PR_UDSOnCAN → BV_AirCondiUDS → EV_AirCondiComfoUDS_00x`

CAN IDs (from `LL_VINFO_AU57XCAN.LL_AirCondiUDS` link layer): TX `0x746`, RX `0x7B0`

### Shared CP Libraries

| Library identifier | Revision | Date | Description |
|---|---|---|---|
| `ES_LIBCompoProteGen3V12` | 004001 | 2022-07-12 | CP Gen 3 V12 — current |
| `ES_LIBCompoProteGen3V11` | 001002 | 2020-06-24 | CP Gen 3 V11 — legacy |
| `ES_LIBCOPKWPSubsy` | 001005 | 2020-06-24 | KWP2000 sub-bus CP identity |
| `BL_LIBTheftProte` | — | — | Theft protection / key download |

---

## J533 Diagnostic Services — Confirmed

### Constellation List (GatewCompoList)

The constellation list is the serial number table J533 uses to verify module identities at startup.

**Read:**
```
DC_BV_GatewUDS_DiagnServi_ReadDataByIdentGatewCompoList
DC_FG_AllUDSDistrSysteVW80127_DiagnServi_ReadDataByIdentGatewCompoList
DC_ES_LIBUDSDistrSysteVW80127V20_DiagnServi_ReadDataByIdentGatewCompoList
```

**Write:**
```
DC_BV_GatewUDS_DiagnServi_WriteDataByIdentGatewCompoList
DC_FG_AllUDSDistrSysteVW80127_DiagnServi_WriteDataByIdentGatewCompoList
```

**Single-job wrappers:**
```
DC_BV_GatewUDS_SinglJob_CompoListRead
DC_BV_GatewUDS_SinglJob_CompoListWrite
```

**Request/response objects:**
```
REQ_BV_GatewUDS_Req_ReadDataByIdentGatewCompoList
REQ_BV_GatewUDS_Req_WriteDataByIdentGatewCompoList
RSP_BV_GatewUDS_Resp_ReadDataByIdentGatewCompoList
```

**DOP (Data Object Parameter) for GatewCompoList:**
```
DOP_BV_GatewUDS_STRUC_GatewCompoList
DOP_BV_GatewUDS_STRUC_GatewCompoListAlloc
DOP_BV_GatewUDS_STRUC_GatewCompoListDiagP
DOP_BV_GatewUDS_STRUC_GatewCompoListDtc
DOP_BV_GatewUDS_STRUC_GatewCompoListPrese
DOP_BV_GatewUDS_STRUC_GatewCompoListSleep
DOP_BV_GatewUDS_STRUC_GatewCompoListTPIdent
DOP_BV_GatewUDS_STRUC_GatewCompoListSingl
DOP_BV_GatewUDS_STRUC_GatewCompoListSleepNew     ← added in later variants
DOP_BV_GatewUDS_STRUC_GatewCompoListECUIdent     ← serial number field (V20–V51)
DOP_BV_GatewUDS_STRUC_GatewCompoListECUNodeAddre ← node address (V52+)
DOP_BV_GatewUDS_STRUC_GatewCompoListEcuAuthe     ← auth field (V52+)
```

The `ECUIdent` / `ECUNodeAddre` field is the serial number that gets compared against the live module. This is what needs to match after a module swap.

**Table binding objects (from BL_DataLibraMIB — the MIB infotainment library that also reads constellation):**
```
DOP_BL_DataLibraMIB_STRUC_CALGatewCompoListDk1
DOP_BL_DataLibraMIB_STRUC_MWVGatewCompoListECUIdentMIBGen2
DOP_BL_DataLibraMIB_STRUC_TmpGatewCompoList
DOP_BL_DataLibraMIB_STRUC_TmpGatewCompoListAlloc
DOP_BL_DataLibraMIB_STRUC_TmpGatewCompoListDiagP
DOP_BL_DataLibraMIB_STRUC_TmpGatewCompoListDtc
DOP_BL_DataLibraMIB_STRUC_TmpGatewCompoListPrese
DOP_BL_DataLibraMIB_STRUC_TmpGatewCompoListSleep
```

This confirms that the MIB infotainment unit (MMI) also reads the constellation list from J533 — relevant for understanding the full scope of CP enrollment on the platform.

### TheftProteData / Key Download (J533)

J533 participates in the IKA key distribution:
```
DC_BV_GatewUDS_DiagnServi_WriteDataByIdentTheftProteData
REQ_BV_GatewUDS_Req_WriteDataByIdentTheftProteData
TBP_BV_GatewUDS_TAB_RecorDataIdentCalibDataWrita.TABROW_IKAKeyL
```

The IKA key is written into J533's calibration data as part of the CP authorization sequence. This is separate from the TheftProteData write to J255 — J533 gets its own IKA key copy.

### Additional Confirmed J533 Services

```
DC_BV_GatewUDS_DiagnServi_ReadDataByIdentCalibData
DC_BV_GatewUDS_DiagnServi_WriteDataByIdentCalibData
DC_BV_GatewUDS_DiagnServi_ReadDataByIdentECUIdent
DC_BV_GatewUDS_DiagnServi_ReadDataByIdentMeasuValue
DC_BV_GatewUDS_DiagnServi_InputOutpuContrByIdentActuaTestShortTermAdjus
DC_BV_GatewUDS_DiagnServi_LinkContr
```

---

## J255 Diagnostic Services — Confirmed

### TheftProteData Write — CP Key Download

This is the operation that delivers CP authorization to J255 and clears the CP active flag.

**Service:**
```
DC_BV_AirCondiUDS_DiagnServi_WriteDataByIdentTheftProteData
REQ_BV_AirCondiUDS_Req_WriteDataByIdentTheftProteData
```

**Table rows within the TheftProteData write table:**
```
TBP_BV_AirCondiUDS_TAB_RecorDataIdentTheftProteDataWrita.TABROW_TheftProteDownlGFAKey
TBP_BV_AirCondiUDS_TAB_RecorDataIdentTheftProteDataWrita.TABROW_TheftProteDownlIKAKey
```

- **GFA key** (Geräteinzelfreigabe): server-signed authorization for this specific unit
- **IKA key** (Installationsschlüssel): installation key binding the module to the target vehicle

**Additional key rows in J255 calibration write:**
```
TBP_BV_AirCondiUDS_TAB_RecorDataIdentCalibDataWrita.TABROW_GKAKey
TBP_BV_AirCondiUDS_TAB_RecorDataIdentCalibDataWrita.TABROW_IKAKey
```

- **GKA key** (Geräteklassenfreigabe): device class authorization key

### Basic Settings Routine (flap end stop calibration)

Referenced in pending actions for post-CP-removal setup:
```
DC_BV_AirCondiUDS_DiagnServi_RoutiContrStartBasicSetti
DC_BV_AirCondiUDS_DiagnServi_RoutiContrRequeRoutiResulBasicSetti

TBP_BV_AirCondiUDS_TAB_RoutiContrBasicSetti.TABROW_CalibOfStopPosit
TBP_BV_AirCondiUDS_TAB_RoutiContrBasicSetti.TABROW_AutomAdresLIN2
```

After CP removal, the flap end stop calibration should be run via VCDS address 08 → Basic Settings.

---

## CP Authentication Library — ES_LIBCompoProteGen3V12

### Core Services

| Service | UDS mapping |
|---|---|
| `DC_ES_LIBCompoProteGen3V12_DiagnServi_RoutiContrStartRoutiCompoProte` | 0x31 01 [id] — Start CP auth |
| `DC_ES_LIBCompoProteGen3V12_DiagnServi_RoutiContrStopRoutiCompoProte` | 0x31 02 [id] — Stop CP auth |
| `DC_ES_LIBCompoProteGen3V12_DiagnServi_RoutiContrRequeRoutiResulCompoProte` | 0x31 03 [id] — Request result |
| `DC_ES_LIBCompoProteGen3V12_DiagnServi_ReadDataByIdentCompoProteData` | 0x22 [did] — Read CP data |

### Confirmed CP Parameter Names and DID Range

These parameter names were found directly in the AU57X string database:

```
Param_CompoProteActiv0xEA61
Param_CompoProteActiv0xEA62
Param_CompoProteActiv0xEA63
Param_CompoProteActiv0xEA64
```

The `0xEA6x` suffix is the hex DID address embedded in the parameter name. This confirms the CP activation DID family spans **`0xEA61–0xEA64`** — these are the per-module CP activation flags within the `CompoProteData` DID.

Additional confirmed parameter names:
```
Param_CompoProte                       ← Base CP status
Param_CompoProteGener                  ← CP generation (Gen 3)
Param_CompoProteK0                     ← CP key 0
Param_CompoProteRole                   ← Role: master or slave
Param_CompoProteNoOrIncorBasicSetti    ← No or incorrect basic settings
Param_StatuOfCompoProteSlave           ← Individual slave status
Param_ListOfStatuOfCompoProteSlave2    ← Dynamic list of all slave statuses
Param_TestProgrCompoProteK0            ← Factory programming CP K0
Param_CompoProteStandSoftwMainVersi    ← Standard software main version
Param_CompoProteStandSoftwPatchNumbe   ← Standard software patch number
Param_CompoProteStandSoftwSupplVersi   ← Standard software supplier version
```

### Routine Structures

```
STRUC_RoutiContrOptioStartCompoProteAuthe     ← Start routine options
STRUC_RoutiStatuRecorStartCompoProteAuthe     ← Start routine result
STRUC_RoutiContrOptioActivOfCompoProteFunct   ← Activate CP function options
STRUC_RoutiStatuRecorActivOfCompoProteFunct   ← Activate CP function result
STRUC_DataRecorCompoProteChara               ← CP characteristics
STRUC_DataRecorStatuOfCompoProteSlave2       ← Dynamic slave status list
STRUC_StatuOfCompoProteSlave2               ← Individual slave status structure
STRUC_StatuOfCompoProteSlave2Routi          ← Slave status during routine
```

### Status/Result Fields

```
DOP_ES_LIBCompoProteGen3V12_DOP_TEXTTABLERoutiStatu      ← Routine running/done/error
DOP_ES_LIBCompoProteGen3V12_DOP_TEXTTABLERoutiResul      ← Routine result pass/fail
DOP_ES_LIBCompoProteGen3V12_DOP_TEXTTABLEStateOfAuthe    ← Authentication state
DOP_ES_LIBCompoProteGen3V12_DOP_TEXTTABLEModesOfActiv    ← Activation modes
DOP_ES_LIBCompoProteGen3V12_DOP_TEXTTABLEModesOfActivStart ← Start modes (V12 only)
DOP_ES_LIBCompoProteGen3V12_DOP_TEXTTABLEImmobParti      ← Immobilizer participation (V12 only)
DOP_ES_LIBCompoProteGen3V12_DOP_TEXTTABLEStatuRestr      ← Restriction status (V12 only)
DOP_ES_LIBCompoProteGen3V12_DOP_TEXTTABLEReceiNotReceiRequeFrame ← Frame receipt status (V12 only)
```

Fields marked "V12 only" are additions in `ES_LIBCompoProteGen3V12` not present in V11. The `StatuRestr` and `ImmobParti` fields are particularly interesting — they suggest V12 adds immobilizer interaction and granular restriction state tracking beyond V11.

---

## DIDB Fault Code Confirmation

The following DTC codes were confirmed from the AU57X DIDB diagnostic database (HSQLDB 1.8 format, 92,583 translated text rows, extracted using kartoffelpflanze's `dumpHSQLDB.py`):

### CP-Specific Fault Codes

| DTC (ISO/SAE) | VAG internal | Description | Stored in |
|---|---|---|---|
| `U110100` | `7465` | Component protection active | Affected slave module (J255, J136, J521) |
| `B1B0400` | — | Component protection master | J533 (when acting as CP master) |
| `B1B0300` | — | Component protection slave | Enrolled slave modules |

### CP-Related DIDB Channel Identifiers

These IDE (identifier) codes appear in ODIS adaptation channel lists:

| IDE | Meaning |
|---|---|
| `IDE14131` | Component protection status |
| `IDE14219` | Component protection challenge |
| `IDE14221` | FAZIT identification CP checksum |
| `IDE14245` | Component protection configuration |
| `IDE14266` | CP internal statuses |
| `IDE12084` | CP participant status |
| `IDE01979` | Stored key for CP participants |
| `IDE12153` | Start CP authentication |

### CP-Related MAS (Measurement) Identifiers

| MAS | Meaning |
|---|---|
| `MAS22057` | Number of CP slaves |
| `MAS14059` | Node address of CP participant |

---

## BL_LIBTheftProte — Theft Protection Library

The theft protection library handles the key storage and immobilizer aspects of CP. Confirmed structures relevant to CP:

```
STRUC_DataRecorTheftProteDataGFAKey          ← GFA key storage
STRUC_DataRecorTheftProteDataIKAKey          ← IKA key storage
STRUC_DataRecorTheftProteDataCompoProteChall ← CP challenge data
STRUC_DataRecorTheftProteDataCompoProteConfi ← CP configuration
STRUC_DataRecorTheftProteDataCompoProtePrope ← CP properties
STRUC_DataRecorTheftProteDataStartCompoProteAuthe ← CP auth start data
STRUC_DataRecorTheftProteDataStatuOfCompoProteSlave ← Slave CP status
STRUC_DataRecorTheftProteDataVersiCheck      ← Version check
STRUC_StatuOfCompoProteSlave                 ← Slave status
```

Text table entries confirm the authorization hierarchy:
```
DOP_BL_LIBTheftProte_DOP_TEXTTABLENotByCompoProteAdminByCompoProteAdmin
```

This two-state table ("not administered by CP admin" / "administered by CP admin") is the flag that gets toggled during CP removal. It is the persistent state stored in the module's EEPROM.

---

## ES_LIBCOPKWPSubsy — KWP2000 Sub-Bus CP Identity

For modules connected via KWP2000 sub-buses (some seat ECUs, some older modules), CP identity reads go through this library rather than direct UDS. Confirmed services:

```
DC_ES_LIBCOPKWPSubsy_DiagnServi_ReadDataByIdentVWSlaveIdentDataCOPMulti
DC_ES_LIBCOPKWPSubsy_DiagnServi_ReadDataByIdentVWSlaveFAZITIdentStrinCOP
DC_ES_LIBCOPKWPSubsy_DiagnServi_ReadDataByIdentVWSlaveSeriaNumbe COP
DC_ES_LIBCOPKWPSubsy_DiagnServi_ReadDataByIdentVWSlaveHardwNumbeCOP
DC_ES_LIBCOPKWPSubsy_DiagnServi_ReadDataByIdentVWSlaveSoftwVersiNumbeCOP
DC_ES_LIBCOPKWPSubsy_DiagnServi_WriteDataByIdentVWSlaveCodinValueCOP
```

The `VWSlaveIdentDataCOPMulti` service reads identity data from multiple KWP sub-bus slaves in a single operation — this is likely what J533 uses to batch-read module serial numbers from the seat bus.

The `TABROW_` entries in this library enumerate every possible CP-enrolled module on the C7 platform. This includes, among others: `AirCondiCompr`, `AlarmHorn`, `CeiliLightModul`, `SunRoof`, `RearSpoilAdjus`, `ElectAdjusSteerColum`, `DriveDoorContrModul`, `MultiContoSeat*` (all four seat positions), `VoltaStabi`, `SwitcModul*` (all seat switch modules).

---

## What This Enables

### Empirical DID Scanning (No Windows Required)

With the `0xEA61–0xEA64` range confirmed, a targeted probe of J533 via the ESP32 bridge becomes meaningful rather than a blind scan:

```python
import udsoncan
# Target: J533 at 0x710/0x77A
# Scan 0xEA00–0xEB00 for ReadDataByIdentifier responses
for did in range(0xEA00, 0xEB00):
    try:
        response = client.read_data_by_identifier(did)
        print(f"0x{did:04X}: {response.service_data.values[did].hex()}")
    except udsoncan.exceptions.NegativeResponseException as e:
        if e.response.code != 0x31:  # 0x31 = requestOutOfRange (DID not supported)
            print(f"0x{did:04X}: NRC {e.response.code:02X}")
```

Any DID that returns data rather than NRC `0x31` is worth investigating. The `0xEA6x` range should return CP-related data.

### MWB Extraction — Now Runs on Linux

`dumpMWB.py` was successfully run against the AU57X project on Linux without a Windows VM.
The `pbl.dll` dependency was resolved by compiling the open-source PBL library
([github.com/peterGraf/pbl](https://github.com/peterGraf/pbl)) as a native Linux `.so`
and patching `classes/PBL.py` to use it on non-Windows hosts.

The full extraction completed in 59 seconds and produced confirmed DID addresses for all
constellation and CP-related services. See `au57x-mwb-extraction-confirmed-dids.md` for
the complete findings.

---

## Limitations of This Document

- **V12 vs V11 differences.** Several V12-only fields were identified above, but a complete diff of V11 vs V12 service definitions requires full ODX decoding.
- **CP library routine IDs.** The exact 2-byte routine IDs for `RoutiContrStartRoutiCompoProte` live in `ES_LIBCompoProteGen3V12.sd.db` and have not yet been extracted.

**Previously listed as limitations — now resolved:** The hex DID addresses, byte layouts of the constellation
DIDs, and security access level are all confirmed. See  for the full
findings from the Linux-native  extraction run against this project.

---

## Sources

- AU57X ODIS MCD Project, DVR 72 (2022-09-22) — `AStringData.data.gz` extraction
- AU57X `DVR-INFO.xml` — project metadata and ODX revision list
- AU57X `DatabaseVersionInfo.txt` — build provenance
- DIDB dump from AU57X project — HSQLDB 1.8 format, 92,583 rows, extracted with `dumpHSQLDB.py`
- kartoffelpflanze/ODIS-project-explorer — tooling reference
- bri3d/VW_Flash — SA2 script reference and UDS protocol background

---

*Released under CC0 (public domain).*

---

## Update: Native Linux MWB Extraction — All DID Addresses Now Confirmed

*Added: March 2026*

The limitation noted above ("requires Windows tooling") has been resolved. The AU57X `.bv.db` files were decoded on Linux without ODIS installed by compiling the open-source PBL library natively and patching `ODIS-project-explorer` to load the resulting `.so` instead of `pbl.dll`.

### Method

```bash
# 1. Clone PBL (MIT-licensed C library, github.com/peterGraf/pbl)
git clone --depth=1 https://github.com/peterGraf/pbl.git

# 2. Compile as a Linux shared library
cd pbl/src/src
gcc -shared -fPIC -o pbl_linux.so pblkf.c pblhash.c pbl.c -lm

# 3. Patch ODIS-project-explorer/classes/PBL.py to use .so on Linux
# (two-line change: detect sys.platform != 'win32', substitute path)

# 4. Run against the AU57X project folder
python3 dumpMWB.py project /path/to/AU57X /output
# Elapsed: 00:00:59
```

No Windows VM. No VW-MCD installation. No `pbl.dll` from ODIS. The bundled `bin/pbl.dll` in ODIS-project-explorer is a Windows x64 DLL that cannot be loaded on Linux via ctypes — but the library it wraps is fully open source and compiles cleanly with a single gcc command.

Output for AU57X: 110+ JSON files across all ECU variants. The two relevant outputs:
- `BV_GatewUDS/MWB_EV_GatewPKOUDS_001.json` — 1.06 MB, 197 DIDs
- `BV_AirCondiUDS/MWB_EV_AirCondiComfoUDS_002.json` — 1.68 MB

---

## Confirmed DID Addresses — J533 (EV_GatewPKOUDS_001)

These are extracted directly from the MWB JSON, not estimated. All structures described below reflect the actual `dop` objects from the decoded `.bv.db` file.

### Constellation DIDs

| DID | Name | Structure |
|---|---|---|
| **`0x04A3`** | Gateway Component List | `END-OF-PDU-FIELD` of 1-byte bitfield records. Each byte covers 8 sequential module slots; bit N set = slot coded. Response length is dynamic (one byte per 8 modules). **Primary read/write constellation DID.** |
| **`0x2A26`** | Gateway Component List present | Same bitmap layout as `0x04A3`. Bit=1 means module is online (present on CAN bus right now). |
| **`0x2A27`** | Gateway Component List sleep indication | Same bitmap layout. Bit=1 means module is in sleep state. |
| **`0x2A28`** | Gateway Component List DTC | Same bitmap layout. Bit=1 means module has an active DTC. |
| **`0x2A29`** | Gateway Component List DiagProt | One byte per module slot (not a packed bitmap). Per-module: bit0=ISO-TP, bit1=TP2.0, bit2=TP1.6, bit3=K-Line, bit4=Ethernet. |
| **`0x2A2A`** | Gateway Component List allocation | `END-OF-PDU-FIELD` of `{ECU_ID u8, ECU_Name u8}` pairs, `byte_size=1` each. ECU Name 8 = Air Conditioning (J255). See ECU name table below. |
| **`0x2A2C`** | Gateway Component List TP-Identifier | `END-OF-PDU-FIELD` of `u16` big-endian CAN TX IDs, one per module slot. J255 appears as `0x0746`. |

### Theft Protection / Key Download DIDs

| DID | Name | Structure |
|---|---|---|
| **`0x0438`** | Stored keys for theft protection slaves | `A_BYTEFIELD`, raw |
| **`0x0439`** | KS ECUs currently authenticated incorrect | `A_BYTEFIELD`, raw |
| **`0x043A`** | KS ECUs formerly authenticated incorrect since last clearance | `A_BYTEFIELD`, raw |
| **`0x043C`** | Number of successful key corrections | `u8`, `BCD-P` encoding |
| **`0x043D`** | Number of successful key downloads | `u8`, `BCD-P` encoding |
| **`0x043E`** | Theftprotection Showroom Mode | `TEXTTABLE`: `{0: 'not active', 1: 'active'}` |
| **`0x2CA9`** | Service key 2 sampling status | Byte 0 bits: `[0]`=SK2 active, `[1]`=request normal, `[2]`=request immediate. Bytes 1–20: ECU exception list 1 (20 bytes). Bytes 21–40: ECU exception list 2 (20 bytes). |
| **`0x00BE`** | IKA Key | `A_BYTEFIELD`, 34 bytes (272 bits). Description: *"Komponentenschutzschlüssel"* (component protection key). **This is the key written by ODIS/GEKO during CP removal.** Present in both J533 and J255. |

### ECU Name Map (from DID `0x2A2A` — confirmed)

| Value | Module |
|---|---|
| 1 | Engine Control Module 1 |
| 2 | Transmission Control Module |
| 3 | Brakes 1 |
| 6 | Seat Adjustment Passenger Side |
| **8** | **Air Conditioning (J255)** |
| 9 | Central Electrics |
| 17 | Engine Control Module 2 |
| 21 | Airbag |
| 23 | Dash Board |
| 25 | Gateway (J533, self) |
| 37 | Immobilizer |
| 54 | Seat Adjustment Driver Side (J136) |
| 68 | Steering Assistance |
| 71 | Sound System |
| 95 | Information Control Unit 1 |

The full map contains 198 entries covering every possible enrolled module on the AU57X platform.

---

## Confirmed DID Addresses — J255 (EV_AirCondiComfoUDS_002, 4-zone)

From `BV_AirCondiUDS` adaptation dump (`dumpAdaptations.py`):

| DID | Name | Structure | Description |
|---|---|---|---|
| **`0x00BE`** | IKA-Key | `A_BYTEFIELD`, 34 bytes | *"IKA-Schlüssel / Schreiben des IKA-Schlüssels"* — the installation key. Written during CP removal. |
| **`0x00BD`** | GKA-Key | `A_BYTEFIELD`, 34 bytes | *"GFA-Schlüssel / Schreiben des GFA-Schlüssels"* — device class authorization key. J255-specific. |

Reading `0x00BE` in extended session: if the response is 34 zero bytes, no key has been installed — CP is definitively active. A non-zero response means a key is present.

---

## Security Access Level Finding

All CP-related write services (`WriteDataByIdentTheftProteData`, `WriteDataByIdentGatewCompoList`, `WriteDataByIdentCalibData`) show `access_level: None` in the MWB service objects. This means:

- CP write operations run in **extended diagnostic session (`0x10 0x03`) only**
- **No SA2 seed/key challenge** is required at the UDS layer
- The GEKO server token provides authorization at the application layer (not via `0x27` SecurityAccess)

This was previously unclear from string analysis alone. The binary service objects confirm it.

---

## What the `0xEA61–0xEA64` Range Actually Is

The range identified in the previous string-mining section (`0xEA61–0xEA64`) does NOT appear in the `EV_GatewPKOUDS_001` MWB output. It likely refers to CP status DIDs on the **slave modules** (J255, J136 etc.) rather than on J533 itself. The confirmed J533 CP DIDs are in the `0x04A3` and `0x2Axx` ranges as documented above.

---

## Sources

- AU57X ODIS MCD Project, DVR 72 (2022-09-22) — `BV_GatewUDS/MWB_EV_GatewPKOUDS_001.json`
- AU57X `BV_AirCondiUDS/MWB_EV_AirCondiComfoUDS_002.json`
- AU57X `BV_GatewUDS/ADP_EV_GatewPKOUDS_001.c` (dumpAdaptations output)
- AU57X `BV_AirCondiUDS/ADP_EV_AirCondiComfoUDS_002.c`
- kartoffelpflanze/ODIS-project-explorer — extraction tooling
- peterGraf/pbl — MIT-licensed PBL library (native Linux build)

