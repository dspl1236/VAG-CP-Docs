# OBD Approach: Tools, UDS Protocol, CAN IDs, and Hardware

*Technical research for the VAG-CP-Docs project. All information from publicly available sources.*

---

## Your Tool Inventory — What Each Can Do

### VNCI 6154A — Your Primary Tool

The VNCI 6154A is a clone of the VAS 6154A that runs original ODIS drivers without modification. Community testing confirms:

> *"ODIS S and E recognize SVCI 6154A as VAS6154A without any problems. Tested ODIS S 7.2.1 GFF, online component protection, parameterisation and full SVM."*

**What your VNCI 6154A can do for your situation:**
- Full ODIS-S / ODIS-E operation — it's functionally identical to OEM hardware for your 2013 A6
- Supports ODIS-S up to V25.03 (with firmware update)
- CAN FD and DoIP support (not needed for your 2013 car, but future-proofed)
- **Online CP removal via GRP session** — this is your fastest path to fixing the original J255
- Full UDS communication with all modules
- CAN trace function (extremely useful for protocol research)

**ODIS version check:** Your VNCI should tell you which ODIS-S version is installed. If it's 25.x, you're on the GRP system. If 23.x or earlier, you're on the legacy GEKO system.

### Mongoose VW/Audi ISO Cable

The Mongoose is a J2534 PassThru device. It can communicate with the car using standard J2534 API, which means:
- Works with VCDS (basic diagnostics, coding, basic settings)
- Works with python-udsoncan and similar open-source tools via J2534 interface
- Cannot be used with ODIS directly (ODIS requires the proprietary 6154/5054 protocol, not J2534)
- **Excellent for research** — J2534 PassThru means you can write Python/C code to talk directly to the bus

### ESP32 — Your Research Platform

The ESP32 has a built-in TWAI (Two-Wire Automotive Interface) controller, which is CAN-compatible. Combined with a CAN transceiver (SN65HVD230 for 3.3V or MCP2551 for 5V), it becomes a full CAN bus research tool:
- Direct CAN bus access via OBD-II connector
- Can implement UDS protocol in MicroPython or C
- WiFi/BLE for wireless logging and remote control
- Low cost (~$5 for ESP32-S3 + $2 for CAN transceiver)
- Large community: `esp32_uds_obd_demo`, `ESP32 CanLite` projects exist

---

## The VAG CAN Architecture — What's on the Bus

This is crucial to understand before building anything:

On a VW/Audi, the only thing reachable from the OBD CAN bus is the CAN gateway. You cannot talk to any other module directly. The bus is silent unless you send a request.

This means:
- OBD-II connector (T16) connects to J533 gateway only
- J533 then routes your request to the appropriate sub-bus (Convenience CAN, Powertrain CAN, etc.)
- J533 acts as a router/proxy for all module communication
- The gateway uses UDS addressing to forward requests to sub-bus modules

### The VAG Diagnostic CAN Architecture

```
OBD-II Port (T16)
      │
      ▼
Diagnostic CAN (500kbps HS-CAN)
      │
      ▼
J533 Gateway  ◄──── The ONLY module directly accessible
      │
      ├──► Convenience CAN (100kbps) ─► J255, J136, J521, J393, J519...
      ├──► Powertrain CAN (500kbps) ──► J623, J217, J104...
      ├──► Extended CAN (500kbps) ────► J428 ACC, J769 LCA...
      ├──► FlexRay bus ───────────────► J197 Air suspension...
      └──► MOST bus (optical) ────────► J794 MMI, Radio...
```

J533 translates your UDS request from the diagnostic CAN and forwards it to the right sub-bus module. This is why replacing J533 is so destructive — it's the only router.

---

## The Complete VAG UDS CAN ID Table

Extracted from ODIS by the community (ConnorHowell/vag-uds-ids on GitHub). These are the CAN request/response IDs for UDS communication with each module on your A6 C7:

| Module | ODIS Addr | Request CAN ID | Response CAN ID |
|---|---|---|---|
| **J533 Gateway** | 19 | **0x710** | **0x77A** |
| J255 Climatronic | 08 | **0x746** | **0x7B0** |
| J136 Seat Mem Driver | 36 | **0x74C** | **0x7B6** |
| J521 Seat Mem Passenger | 06 | **0x74D** | **0x7B7** |
| J518 KESSY | 25 | 0x732 | 0x79C |
| J285 Instrument Cluster | 17 | 0x714 | 0x77E |
| J393 Central Comfort | 46 | 0x70D | 0x777 |
| J519 Vehicle Elect. | 09 | 0x70E | 0x778 |
| J527 Steering Column | 16 | 0x70C | 0x776 |
| J794 Info Electronics 1 | 5F | 0x773 | 0x7DD |
| J623 Engine Control | 01 | 0x7E0 | 0x7E8 |
| J217 Transmission | 02 | 0x7E1 | 0x7E9 |
| J104 ABS/ESP | 03 | 0x713 | 0x77D |
| J234 Airbag | 15 | 0x715 | 0x77F |
| J428 ACC Radar | 13 | 0x757 | 0x7C1 |

**Your three CP-affected modules:**
- J255 Climatronic: Request `0x746`, Response `0x7B0`
- J136 Driver seat: Request `0x74C`, Response `0x7B6`
- J521 Passenger seat: Request `0x74D`, Response `0x7B7`

---

## UDS Protocol Layer — How ODIS Talks to Modules

Everything runs over **ISO 14229 UDS** (Unified Diagnostic Services) over **ISO-TP** (ISO 15765-2) framing on CAN.

### Session Flow to Reach J255 via Gateway

The gateway routes requests based on the target address. For J255 Climatronic:

```
Tester → J533 (0x710): [02] [10] [03]  ← Enter Extended Diagnostic Session
J533 → Tester (0x77A): [06] [50] [03] [00] [32] [01] [F4]  ← Confirmed

Tester → J533 (0x710): [03] [22] [F1 11]  ← Read ECU ID from J533
J533 → Tester (0x77A): [positive response with gateway identity]

# To talk to J255, use functional addressing or physical address 0x746:
Tester → J255 (0x746): [02] [10] [03]  ← Extended session request TO J255
J255 → Tester (0x7B0): [06] [50] [03] ...]  ← J255 confirms session
```

### Key UDS Service IDs Relevant to CP

| SID | Name | Description |
|---|---|---|
| `0x10` | DiagnosticSessionControl | Enter Default/Extended/Programming session |
| `0x11` | ECUReset | Soft/hard reset a module |
| `0x14` | ClearDiagnosticInformation | Clear DTCs |
| `0x19` | ReadDTCInformation | Read fault codes |
| `0x22` | ReadDataByIdentifier | Read a specific data value by DID |
| `0x27` | SecurityAccess | Seed/Key authentication to unlock write access |
| `0x2E` | WriteDataByIdentifier | Write a specific data value |
| `0x2F` | InputOutputControlByIdentifier | Control actuators (flap motors etc.) |
| `0x31` | RoutineControl | Execute routines (like flap calibration, CP removal) |
| `0x34/35/36/37` | Download/Upload/Transfer | Flash firmware |
| `0x3E` | TesterPresent | Keep session alive (must send every ~2s) |

### The Security Access Dance (UDS 0x27)

Before ODIS can do anything privileged (write adaptations, run routines, CP removal), it must pass security access:

```
# Request seed (level 0x01 = standard, 0x03 = extended, etc.)
Tester → Module: [03] [27] [01]           ← "Give me a seed at level 1"
Module → Tester: [06] [67] [01] [AA] [BB] [CC] [DD]  ← Seed returned

# Tester calculates key from seed using manufacturer algorithm
key = vag_seed_key_algorithm(seed=0xAABBCCDD, level=1)

# Send calculated key
Tester → Module: [06] [27] [02] [KK] [KK] [KK] [KK]  ← "Here's my key"
Module → Tester: [02] [67] [02]           ← Access granted (or 7F 27 35 = denied)
```

**The VAG seed-key algorithm** for lower security levels (coding, adaptation) is publicly known in the community and has been implemented in open source tools. The higher security levels used for CP operations are what require the GEKO/GRP server involvement.

### What Happens During CP Removal (Protocol Level)

Based on community analysis of ODIS traffic, CP removal is a `0x31 RoutineControl` call:

```
# Rough sequence (documented from ODIS captures):
1. Enter extended session (0x10 03) on J533
2. SecurityAccess exchange (0x27) with J533 at CP security level
3. J533 builds CP request payload: VIN + module serial + part number + technician ID
4. ODIS sends this payload to GEKO/GRP server over HTTPS
5. Server returns signed authorization token
6. ODIS sends RoutineControl (0x31 01 [routine_id] [token]) to J533
7. J533 forwards to target module (e.g. J255 via 0x746)
8. J255 verifies token signature, removes CP restriction
9. J255 writes new CP state to EEPROM
10. J255 sends positive response (0x71 01 [routine_id])
```

The routine identifier for CP removal is not publicly documented, but can be identified by:
- Capturing ODIS traffic with Wireshark during a real CP removal
- Examining the ODIS PostSetup database files (XML/ODX format) which contain routine definitions

---

## What You Can Do via OBD Right Now

### Tier 1: With VCDS/ODIS-S (No Online Account Needed)

These work immediately with your VNCI 6154A:

**Diagnostics:**
- Full module scan across all buses
- Read CP-related DTCs on J255, J136, J521
- Read gateway constellation data (what modules J533 expects)
- Read module part numbers and serial numbers

**J255 Climatronic (your immediate problem):**
- `08 → Basic Settings → Adapt flap end stops` — run this after CP is cleared to fix your flap issues
- Read measuring blocks to see exact flap positions vs targets
- Read adaptation channel 81 (vehicle data binding)

**Read VIN binding from modules:**
```
In VCDS: 08 → Measuring Blocks → Group 29 (Coding)
Shows the current coding including CP state flag
```

### Tier 2: With ODIS-S + GRP Session (Online Account, ~€20-50)

This solves your J255 CP immediately:
1. Connect VNCI 6154A to OBD port
2. Launch ODIS-S 25.x
3. Identify vehicle
4. Module 08 → Guided Functions → Component Protection removal
5. ODIS connects to GRP server, clears CP
6. Run basic settings → flap calibration
7. Done in 20 minutes

### Tier 3: Python + python-udsoncan + Mongoose J2534

Open source UDS exploration. The Mongoose J2534 cable works with `python-udsoncan` via the J2534 connection backend:

```python
import udsoncan
import isotp
from udsoncan.connections import J2534Connection
from udsoncan.client import Client

# Connect to J255 Climatronic via J2534/Mongoose
conn = J2534Connection(
    windll='C:\\...\\mongoose.dll',
    rxid=0x7B0,  # J255 response ID
    txid=0x746   # J255 request ID
)

config = {
    'request_timeout': 2,
    'data_identifiers': {
        0xF190: udsoncan.AsciiCodec(17),  # VIN
        0xF18C: udsoncan.AsciiCodec(20),  # ECU Serial
    }
}

with Client(conn, config=config) as client:
    client.change_session(udsoncan.services.DiagnosticSessionControl.Session.extendedDiagnosticSession)
    
    # Read VIN from J255 (will show which VIN it's bound to)
    result = client.read_data_by_identifier(0xF190)
    print(f"J255 is bound to VIN: {result.service_data.values[0xF190]}")
    
    # Read ECU identification
    result = client.read_data_by_identifier(0xF18C)
    print(f"J255 serial: {result.service_data.values[0xF18C]}")
```

This alone tells you what VIN your J255 thinks it belongs to — useful diagnostic information.

### Tier 4: ESP32 CAN Sniffer/Research Tool

Build a dedicated hardware tool for protocol research:

**Hardware:**
```
ESP32-S3 DevKit
    + SN65HVD230 CAN transceiver (3.3V, perfect for ESP32)
    + OBD-II breakout connector or cable
    + Optional: SD card module for logging
    + Optional: small OLED for status display
```

**Wiring:**
```
ESP32         SN65HVD230        OBD-II Pin
3.3V    ──►   VCC               Pin 16 (Battery+, via regulator)
GND     ──►   GND               Pin 4/5
GPIO5   ──►   TX (D)            -
GPIO4   ──►   RX (R)            -
                CANH ──────────► Pin 6 (CAN High)
                CANL ──────────► Pin 14 (CAN Low)
```

**MicroPython UDS scanner:**
```python
from machine import CAN
import struct

# Initialize CAN at 500kbps (VAG diagnostic CAN speed)
can = CAN(0, mode=CAN.NORMAL, baudrate=500000, 
          tx=5, rx=4)

# Send extended diagnostic session request to J255 Climatronic
# ISO-TP single frame: [length] [SID] [subfunction]
def send_uds(can, txid, data):
    # Single frame ISO-TP
    frame = bytearray(8)
    frame[0] = len(data)  # ISO-TP single frame header
    for i, b in enumerate(data):
        frame[i+1] = b
    # Pad with 0xCC
    for i in range(len(data)+1, 8):
        frame[i] = 0xCC
    can.send(frame, txid)

def read_response(can, rxid, timeout=200):
    msg = can.recv(timeout=timeout)
    if msg and msg[0] == rxid:
        return msg[3]  # data bytes
    return None

# Enter extended session on J255
send_uds(can, 0x746, [0x10, 0x03])
response = read_response(can, 0x7B0)
print(f"Session response: {response}")

# Read VIN binding from J255 (DID 0xF190)
send_uds(can, 0x746, [0x22, 0xF1, 0x90])
response = read_response(can, 0x7B0)
# Parse VIN from response
```

**What this ESP32 tool can do:**
- Sniff all CAN traffic (in monitor mode) to capture ODIS communications
- Send raw UDS requests to any module
- Log all CP-related traffic during an ODIS session
- Explore undocumented DIDs by scanning 0x0000-0xFFFF
- Read module identification without security access
- Perform TesterPresent (0x3E) to keep sessions alive

---

## The CP Protocol Research Path

Here's how to actually understand the CP protocol using your tools:

### Step 1: Capture a Real CP Session

If you manage to do an online CP removal (via GRP session), simultaneously run a packet capture:

1. VNCI 6154A connected to car
2. Wireshark running on your PC with the VAS6154A's USB interface in promiscuous mode
   - The VNCI communicates over its own USB protocol with ODIS
   - Some users capture the HTTP traffic to GRP server using Fiddler/Charles proxy
3. Log: what UDS messages ODIS sends before, during, and after the CP removal
4. This reveals the exact routine ID and data format

### Step 2: Scan the J533 and J255 for Undocumented DIDs

Using python-udsoncan or your ESP32:

```python
# Brute-force DID scan — legal on your own vehicle
def scan_dids(client, start=0x0100, end=0x0200):
    found = []
    for did in range(start, end):
        try:
            result = client.read_data_by_identifier(did)
            print(f"DID 0x{did:04X}: {result.service_data.values[did].hex()}")
            found.append(did)
        except udsoncan.exceptions.NegativeResponseException as e:
            if e.code != 0x31:  # 0x31 = requestOutOfRange (not supported)
                print(f"DID 0x{did:04X}: error {e.code}")
    return found
```

This finds DIDs that contain CP state, constellation data, and security-relevant information without needing write access.

### Step 3: Security Access Level Enumeration

```python
# Try all security access levels to see which ones J533 supports
for level in range(1, 0xFF, 2):  # Odd numbers = seed request
    try:
        seed = client.request_seed(level)
        print(f"Security level 0x{level:02X} supported, seed: {seed.hex()}")
    except Exception as e:
        pass  # Level not supported
```

The level that ODIS uses for CP operations will respond with a seed. Once you know the level, you can analyze what algorithm converts the seed to a key.

### Step 4: Analyze ODIS PostSetup Files

Your ODIS-S installation contains the vehicle database (PostSetup). Within the PostSetup files (typically in `C:\ProgramData\ODIS-S\...`):

- `.odx` files — contain UDS service definitions in XML format
- `.pdx` files — packed ODX containers
- `.sgo` firmware files — ECU flash packages

The ODX files define the routine identifiers, security levels, and data identifiers for each module. Extracting these reveals the CP routine ID without a network capture. Look in the ODX file for J255 (address 08, `EV_AirCondi*`) for routines containing "ComponentProtection" or "Komponentenschutz".

---

## Immediate Action Plan

**This week:**
1. Check your ODIS-S version — open ODIS-S, go to Help → About
2. If 25.x: find a GRP portal (vw-geko.com or similar), buy a single session (~€20-30)
3. Run CP removal on J255 with your VNCI 6154A
4. Immediately after: run Basic Settings → Adapt flap end stops
5. Problem solved for daily driving

**Research track (parallel):**
1. Install python-udsoncan: `pip install udsoncan python-can`
2. Connect Mongoose cable, test basic session with J255
3. Read what VIN J255 currently thinks it's bound to
4. Begin DID scan of J533 to understand constellation data structure
5. If you can capture a GRP session, use Wireshark to log the routine

**Hardware track:**
1. Order ESP32-S3 + SN65HVD230 transceiver (~$10 total)
2. Wire up the OBD-II interface
3. Write the MicroPython CAN scanner
4. Use it for passive sniffing during ODIS sessions — this is gold for protocol research

---

## Key GitHub Projects to Study

| Project | URL | Relevance |
|---|---|---|
| ConnorHowell/vag-uds-ids | github.com/ConnorHowell/vag-uds-ids | Complete VAG CAN ID table (extracted from ODIS) |
| pylessard/python-udsoncan | github.com/pylessard/python-udsoncan | Python UDS library — works with your Mongoose |
| aep/vag_reverse_engineering | github.com/aep/vag_reverse_engineering | Live VAG protocol reverse engineering notes |
| rlourette/esp32_uds_obd_demo | github.com/rlourette/esp32_uds_obd_demo | ESP32 UDS starter code |
| mdabrowski1990/uds | github.com/mdabrowski1990/uds | Full UDS implementation, multi-transport |

---

## Sources
- ConnorHowell/vag-uds-ids — VAG UDS CAN ID database extracted from ODIS
- aep/vag_reverse_engineering — Live VAG protocol research log
- STIC SVCI/VNCI product pages — VNCI 6154A capability confirmation
- MHH Auto forum — ODIS session community testing
- pylessard/python-udsoncan documentation
- Medium: "ECU Reverse Engineering & Bypassing UDS SecurityAccess (0x27)"
- CSS Electronics — UDS protocol tutorial
