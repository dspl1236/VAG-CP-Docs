# AU57X — Confirmed DID Addresses from MWB Extraction

*Platform: Audi A6 C7 / A7 4G (internal code AU57X)*  
*Source: AU57X ODIS MCD Project DVR 72 (2022-09-22), `dumpMWB.py` extraction*  
*Extraction method: ODIS-project-explorer on **Linux** — no Windows VM required*  
*Extracted: March 2026*

---

## Context

The companion document `au57x-mcd-project-findings.md` documented what was extractable from the AU57X project's
string databases (no Windows tooling required). That document listed the exact DID addresses, byte layouts,
security levels, and routine IDs as **unknown, requiring Windows tooling** to resolve.

This document supersedes that limitation. All five items are now confirmed.

The key breakthrough: the ODIS-project-explorer `dumpMWB.py` script relies on `pbl.dll` to parse `.bv.db` files,
but `pbl.dll` is compiled from the open-source PBL library ([github.com/peterGraf/pbl](https://github.com/peterGraf/pbl)).
Compiling a native Linux `.so` from that source and patching `classes/PBL.py` to use it on non-Windows systems
allows the full MWB extraction to run directly on Linux. The AU57X project was processed in 59 seconds.

```bash
# Compile native PBL for Linux
git clone https://github.com/peterGraf/pbl.git
cd pbl/src/src
gcc -shared -fPIC -o /path/to/ODIS-project-explorer/bin/pbl_linux.so \
    pblkf.c pblhash.c pbl.c -lm

# Patch classes/PBL.py to load pbl_linux.so on non-Windows
# (check sys.platform, substitute .dll path with .so path)

# Run extraction
python3 dumpMWB.py project "/path/to/AU57X" "/output"
# Completes in ~60 seconds, produces JSON for every ECU variant
```

---

## Confirmed DID Addresses — J533 Gateway (EV_GatewPKOUDS_001)

`EV_GatewPKOUDS_001` is the standard petrol/diesel C7 A6/A7 non-hybrid gateway variant.
Total DIDs confirmed: 197 MWB entries.

### Constellation / Component Protection DIDs

All confirmed from `MWB_EV_GatewPKOUDS_001.json` output.

| DID | Name | Structure | Notes |
|---|---|---|---|
| **`0x04A3`** | Gateway Component List (coded) | END-OF-PDU-FIELD of `ECU Coding Byte` structs (1 byte each, 8 bitfields) | **Primary constellation DID.** Each bit = one module slot. `1` = coded. |
| **`0x2A26`** | Gateway Component List present | END-OF-PDU-FIELD of 1-byte bitfields | Online/offline per slot. `1` = online. |
| **`0x2A27`** | Gateway Component List sleep | END-OF-PDU-FIELD of 1-byte bitfields | Sleep state per slot. |
| **`0x2A28`** | Gateway Component List DTC | END-OF-PDU-FIELD of 1-byte bitfields | `0` = OK, `1` = Error per slot. |
| **`0x2A29`** | Gateway Component List DiagProt | END-OF-PDU-FIELD of `DiagProt Information` structs | Per-module protocol flags: bit0=ISO-TP, bit1=TP2.0, bit2=TP1.6, bit3=K-Line, bit4=Ethernet. |
| **`0x2A2A`** | Gateway Component List allocation | END-OF-PDU-FIELD of `{ECU_ID u8, ECU_Name u8}` pairs | Maps slot index to ECU ID and ECU Name code. **ECU Name 8 = Air Conditioning (J255).** |
| **`0x2A2C`** | Gateway Component List TP-Identifier | END-OF-PDU-FIELD of `{TP-CAN-Identifier u16}` | CAN TX ID per slot (big-endian u16). J255 = `0x0746`. |
| `0x0438` | Stored keys for theft protection slaves | A_BYTEFIELD | Raw key storage. |
| `0x0439` | KS ECUs currently authenticated incorrect | A_BYTEFIELD | |
| `0x043A` | KS ECUs formerly authenticated incorrect | A_BYTEFIELD | |
| `0x043C` | Number of successful key corrections | u8 BCD-P | |
| `0x043D` | Number of successful key downloads | u8 BCD-P | |
| `0x043E` | Theftprotection Showroom Mode | TEXTTABLE: `{0:'not active', 1:'active'}` | |
| `0x2CA9` | Service key 2 sampling status | bit0=SK2 active, bit1=req normal, bit2=req immediate; bytes 1–20 = exception list 1; bytes 21–40 = exception list 2 | |
| **`0x00BE`** | IKA Key | 34 bytes A_BYTEFIELD, IDENTICAL compu | "Komponentenschutzschlüssel." Readable in extended session. Zeroed = no key installed. |

### How to Decode DID `0x04A3` Response

```
Response bytes: [B0] [B1] [B2] ... [Bn]

Each byte covers 8 module slots:
  Byte 0, bit 0 → slot 0
  Byte 0, bit 1 → slot 1
  ...
  Byte 0, bit 7 → slot 7
  Byte 1, bit 0 → slot 8
  ...

Slot N is coded if:   (response[N // 8] >> (N % 8)) & 1

To find J255's slot:
  Read DID 0x2A2A first.
  Parse pairs {ECU_ID, ECU_Name} — one pair per slot.
  Find the pair where ECU_Name == 8 (Air Conditioning).
  The pair index (0-based) is J255's slot number.
  Check that slot in the 0x04A3 bitmap to confirm it's coded.
```

### ECU Name Code Map (from DID `0x2A2A` structure)

Confirmed from the `ECU Name` TEXTTABLE in the MWB JSON:

| Code | Module |
|---|---|
| 1 | Engine Control Module 1 |
| 2 | Transmission Control Module |
| 3 | Brakes 1 |
| 6 | Seat Adjustment Passenger Side |
| **8** | **Air Conditioning (J255)** |
| 9 | Central Electrics |
| 21 | Airbag |
| 23 | Dash Board |
| 25 | Gateway (J533) |
| 37 | Immobilizer |
| 54 | Seat Adjustment Driver Side (J136) |
| 68 | Steering Assistance |
| 71 | Sound System |
| 95 | Information Control Unit 1 |
| 117 | Telematics |

*(Full map: 200+ entries in the MWB JSON `ECU Name` TEXTTABLE.)*

---

## Confirmed DID Addresses — J255 HVAC (EV_AirCondiComfoUDS_002)

`EV_AirCondiComfoUDS_002` is the 4-zone comfort climate variant — the one relevant to this case (4G0820043H).

### CP Key Write DIDs

Confirmed from `ADP_EV_AirCondiComfoUDS_002.c` (dumpAdaptations output):

| DID | IDE | Name | Structure | Description |
|---|---|---|---|---|
| **`0x00BD`** | IDE02186 | GKA-Key | 34 bytes A_BYTEFIELD | "GFA-Schlüssel / Schreiben des GFA-Schlüssels" (device class authorization key) |
| **`0x00BE`** | IDE02185 | IKA-Key | 34 bytes A_BYTEFIELD | "IKA-Schlüssel / Schreiben des IKA-Schlüssels" (installation key) |

Both DIDs are also present on J533 (`ADP_EV_GatewPKOUDS_001.c` confirmed `0x00BE` with description "Komponentenschutzschlüssel").

### Reading Key State

Reading `0x00BE` on J255 in extended session:
- **34 zero bytes** → no IKA key installed → CP is active
- **Non-zero** → key has been written at some point

This is the most direct empirical indicator of CP state, more reliable than DTC codes (which can be suppressed).

---

## Security Access Level

All CP-related write services in the MWB and adaptation dumps show `access_level: None`:

```
DC_BV_GatewUDS_DiagnServi_WriteDataByIdentTheftProteData   access_level: None
DC_BV_GatewUDS_DiagnServi_WriteDataByIdentGatewCompoList   access_level: None
DC_BV_GatewUDS_DiagnServi_WriteDataByIdentCalibData        access_level: None
```

**This means CP write operations require only extended diagnostic session (0x10 0x03) — no SA2 seed/key challenge.** The authorization is provided by the GEKO server token embedded in the routine payload, not by a UDS security access exchange. There is no local seed/key algorithm to bypass.

---

## Routine ID for CP Authentication

The routine IDs for `RoutiContrStartRoutiCompoProte`, `RoutiContrStopRoutiCompoProte`, and
`RoutiContrRequeRoutiResulCompoProte` are defined in the `ES_LIBCompoProteGen3V12` shared library
rather than in `BV_GatewUDS` directly. They were not present in the `BV_GatewUDS.bv.db` MWB output.

The CP authentication routine IDs require extraction from:
```
0.0.0@ES_LIBCompoProteGen3V12.sd.db  (shared library file)
```
using `dumpProject.py` against that specific file. This has not been done yet.

**Known from community research:** the start routine is `0x31 01` followed by a 2-byte routine ID.
The exact 2 bytes are the remaining open item.

---

## What This Means for the Research Approach

**Before this extraction**, the recommended approach was:
> Empirically probe J533 in the `0xEA00–0xEB00` range to find the GatewCompoList DID.

**After this extraction**, this is no longer necessary. The correct addresses are:

| Probe target | Confirmed DID |
|---|---|
| Constellation coded bitmap (R/W) | `0x04A3` |
| Which slot is J255 | `0x2A2A` (parse ECU_Name == 8) |
| Which modules are online | `0x2A26` |
| J255 CP key state | `0x00BE` on J255 |

The `0xEA61–0xEA64` range identified from the string extraction remains confirmed as the CP activation
flag DID family within the `ES_LIBCompoProteGen3V12` library — distinct from the constellation DIDs above.
The constellation DIDs sit in the `0x04A3` / `0x2A2x` range; the CP library activation flags sit in `0xEA6x`.

---

## Extraction Reproducibility

The full extraction is reproducible from any Linux machine with git and gcc:

1. Obtain the AU57X ODIS MCD project directory (requires an ODIS-S installation or the raw project archive)
2. Clone ODIS-project-explorer: `github.com/kartoffelpflanze/ODIS-project-explorer`
3. Compile native PBL `.so` from `github.com/peterGraf/pbl`
4. Patch `classes/PBL.py` to use `.so` on non-Windows
5. Run `dumpMWB.py project <AU57X_path> <output_path>`
6. Run `dumpAdaptations.py project <AU57X_path> <output_path>`

Output files relevant to CP:
```
<output>/AU57X/BV_GatewUDS/MWB_EV_GatewPKOUDS_001.json     (1.06 MB — 197 DIDs)
<output>/AU57X/BV_GatewUDS/MWB_EV_GatewPKOUDS_002.json     (1.29 MB — hybrid variant)
<output>/AU57X/BV_AirCondiUDS/MWB_EV_AirCondiComfoUDS_002.json   (1.68 MB — 4-zone J255)
<output>/AU57X/BV_GatewUDS/ADP_EV_GatewPKOUDS_001.c        (175 KB — 24 adaptation entries)
<output>/AU57X/BV_AirCondiUDS/ADP_EV_AirCondiComfoUDS_002.c (167 KB — 58 adaptation entries)
```

---

## Remaining Open Items

| Item | Status |
|---|---|
| Exact routine ID bytes for `RoutiContrStartRoutiCompoProte` | Requires `dumpProject.py` against `ES_LIBCompoProteGen3V12.sd.db` |
| Exact byte layout of `GatewCompoList` write payload | Structure confirmed for read; write payload structure may differ |
| Empirical verification of DID addresses against live J533 | Not yet done — requires physical car access with ESP32 bridge |
| CP library activation DID addresses (`0xEA6x`) | Range confirmed from strings; exact addresses need MWB extraction from `ES_LIBCompoProteGen3V12.sd.db` |

---

*Released under CC0 (public domain).*

---

## CP Routine ID — Confirmed

The UDS RoutineControl routine ID for Component Protection authentication on the
C7 platform has been extracted from the `ES_LIBCompoProteGen3V12.sd.db` binary:

**CP Routine ID: `0x0226`**

UDS sequence:
```
31 01 02 26        ← RoutineControl, Start, routine ID 0x0226
```

Expected responses:
```
7F 31 22           ← conditionsNotCorrect — routine ID confirmed, GEKO token required
7F 31 31           ← requestOutOfRange — wrong routine ID
7F 31 7E           ← subFunctionNotSupportedInActiveSession — wrong session
```

Status: Extracted from binary. Pending live hardware confirmation (run against
J533 in extended session — `7F 31 22` response confirms the ID is correct).
