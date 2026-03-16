# MWB Extraction from ODIS MCD Projects on Linux

*How to decode AU57X `.bv.db` files without Windows or a VW-MCD installation.*

---

## Background

ODIS MCD project files store diagnostic object definitions in a proprietary binary format using the PBL (Persistent Block Library) key-file format. The `ODIS-project-explorer` tool by kartoffelpflanze can decode these files, but its `PBL.py` class loads a Windows DLL (`pbl.dll`) via ctypes.

PBL is open source (MIT licensed) at [github.com/peterGraf/pbl](https://github.com/peterGraf/pbl). The DLL is simply a compiled version of this library. It can be compiled as a native Linux shared library in seconds.

This was confirmed working for the AU57X project (DVR 72, September 2022) in March 2026. Runtime: 59 seconds for the full `dumpMWB` run across all 280 base variants.

---

## Prerequisites

- Python 3.10+
- gcc
- git
- The AU57X project folder (contains `.bv.db` and `.key` files)

---

## Steps

### 1. Clone ODIS-project-explorer

```bash
git clone https://github.com/kartoffelpflanze/ODIS-project-explorer.git
cd ODIS-project-explorer
```

### 2. Compile PBL as a native Linux shared library

```bash
git clone --depth=1 https://github.com/peterGraf/pbl.git /tmp/pbl_source
cd /tmp/pbl_source/src/src
gcc -shared -fPIC -o /path/to/ODIS-project-explorer/bin/pbl_linux.so \
    pblkf.c pblhash.c pbl.c -lm
```

### 3. Patch PBL.py to use the `.so` on Linux

Edit `classes/PBL.py`. Find the `__init__` method and add a platform check before loading:

```python
import sys, os
from ctypes import *

class PBL:
    def __init__(self, dll_path):
        # On Linux, use the native .so instead of the Windows .dll
        if sys.platform != 'win32':
            linux_so = os.path.join(os.path.dirname(dll_path), 'pbl_linux.so')
            if os.path.isfile(linux_so):
                dll_path = linux_so
            else:
                raise FileNotFoundError(f'pbl_linux.so not found at {linux_so}')
        
        dll = CDLL(dll_path)
        # ... rest of __init__ unchanged
```

### 4. Run dumpMWB

```bash
cd /path/to/ODIS-project-explorer

# All base variants in a project (recommended)
python3 dumpMWB.py project /path/to/AU57X /output/AU57X

# Or a specific base variant
python3 dumpMWB.py basevariant /path/to/AU57X 0.0.0@BV_GatewUDS.bv /output
```

### 5. Run dumpAdaptations (for writable DIDs)

```bash
python3 dumpAdaptations.py basevariant /path/to/AU57X 0.0.0@BV_GatewUDS.bv /output/adp
python3 dumpAdaptations.py basevariant /path/to/AU57X 0.0.0@BV_AirCondiUDS.bv /output/adp
```

---

## Output

The MWB output is one JSON file per ECU variant:

```
/output/AU57X/
  BV_GatewUDS/
    MWB_BV_GatewUDS.json              # base variant DIDs
    MWB_EV_GatewPKOUDS_001.json       # standard C7 gateway — 197 DIDs, 1.06 MB
    MWB_EV_GatewPKOUDS_002.json       # EV/hybrid variant
    MWB_EV_GatewUDS_001.json          # older variant
  BV_AirCondiUDS/
    MWB_EV_AirCondiComfoUDS_002.json  # 4-zone HVAC (your J255 variant)
    MWB_EV_AirCondiComfoUDS_003.json
    MWB_EV_AirCondiBasisUDS_002.json  # 2-zone HVAC
    ...
```

The adaptation output (`.c` text format, one per variant) contains all writable DIDs with full structure including byte size, encoding type, and German-language description strings.

---

## What to Look For

### Finding CP DIDs in the MWB JSON

Each entry has a `did` field (integer) and a nested `structure` describing the byte layout. For constellation DIDs:

```python
import json

data = json.load(open('MWB_EV_GatewPKOUDS_001.json'))
by_did = {e['did']: e for e in data}

# Gateway Component List
entry = by_did[0x04A3]  # 1219 decimal
print(entry['long_name'])           # "Gateway Component List"
print(entry['structure']['type'])   # "STRUCTURE"
# Structure contains END-OF-PDU-FIELD of 1-byte bitfield records
```

### Finding key DIDs in the adaptation output

```bash
grep -A 20 "0x00BE" ADP_EV_AirCondiComfoUDS_002.c
# Shows: 34-byte A_BYTEFIELD, description "IKA-Schlüssel"

grep -A 20 "0x00BD" ADP_EV_AirCondiComfoUDS_002.c  
# Shows: 34-byte A_BYTEFIELD, description "GFA-Schlüssel"
```

### Checking security access level

Search the `BV_GatewUDS.bv.c` project dump for the service objects:

```bash
grep "access_level" 0.0.0@BV_GatewUDS.bv.c | grep -v "None"
```

For the AU57X C7 gateway: all CP write services show `access_level: None` — extended session only, no SA2 required.

---

## Confirmed Results for AU57X (C7 A6/A7 — EV_GatewPKOUDS_001)

See [`au57x-mcd-project-findings.md`](./au57x-mcd-project-findings.md) for the complete confirmed DID address table and structure documentation extracted using this method.

---

## Notes

- This method was tested with `ODIS-project-explorer` as of early 2026. The repository may have updated since then.
- The `pbl_linux.so` compilation requires only `pblkf.c`, `pblhash.c`, and `pbl.c` from the PBL source. The other source files (collections, lists, etc.) are not needed for key-file parsing.
- The `bin/pbl.dll` included in the ODIS-project-explorer repository is a Windows x64 PE32+ DLL and cannot be loaded on Linux via ctypes. The Linux `.so` is a drop-in replacement.
- jpype1 is not required for `dumpMWB` or `dumpAdaptations` — only for `dumpDTC` and `parseMWB` which use the HSQLDB text translation database.

---

*Released under CC0 (public domain).*
