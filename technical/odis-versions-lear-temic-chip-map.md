# ODIS Version History, Lear vs Temic Gateway, and A6 C7 J533 Chip Mapping

*Technical research document for the VAG-CP-Docs project. All information sourced from publicly available community research, diagnostic tool documentation, and manufacturer SSPs.*

---

## Part 1: Was There Ever an ODIS Version That Did CP Offline?

### The Short Answer: No — but the architecture changed over time

CP removal has **always required an online GEKO server connection** within ODIS. There was never a version of ODIS that performed CP removal entirely offline. This is by design — the server dependency was built into the CP system from its introduction in 2002-2003.

However, the architecture of *which* server and *what* ODIS version has changed significantly:

### ODIS Version Timeline and CP Implications

| ODIS Version | Era | Online System | CP Status |
|---|---|---|---|
| VAS 5051 / VAS 5052 | Pre-2006 | FAZIT direct | CP introduced with A8 D3 only |
| ODIS 1.x / 2.x | 2008–2012 | GEKO v1 | C6 A6 Gen 1/2 CP via GEKO |
| ODIS 3.x | 2012–2015 | GEKO v2 | A6 C7 4G CP fully implemented |
| ODIS 4.x | 2015–2018 | GEKO v3 | C7 CP + MQB CP expanded |
| ODIS 5.x | 2018–2021 | GEKO v3/SFD | SFD added as overlay |
| ODIS 23.x | 2021–2024 | GEKO (final) | Last GEKO-based version |
| ODIS 25.x | 2024–present | GRP (new) | Migrated away from GEKO |

**Critical finding from community research:**
GEKO access is for ODIS 23 and earlier. ODIS-S 25 uses GRP for online operations.

This means:
- **ODIS 23.x** — Uses GEKO, which is in transition/degraded state
- **ODIS 25.x** — Uses GRP (Group Retail Portal), which is the new backend

### What This Means for Your ODIS-S

If you have ODIS-S 25.x, you are on the **GRP system**, not GEKO. GRP access is available through various portals, and community members report it working. The transition is ongoing.

If you have ODIS-S 23.x or earlier, you would need GEKO credentials — which are in an uncertain state.

**The practical implication:** Your ODIS-S version matters. Check which version you have. ODIS 25.x with GRP access may actually be *more* accessible right now than trying to find working GEKO credentials for ODIS 23.x.

### Why Offline CP Removal Was Never Possible in ODIS

The CP guided function in ODIS is structured as a remote procedure call:

1. ODIS opens the guided function for CP removal
2. ODIS collects: VIN, module serial numbers, part numbers, gateway identity
3. **ODIS sends this payload to the GEKO/GRP server via HTTPS**
4. The server queries FAZIT database: is this module stolen? (FAZ4990E blocks stolen parts)
5. The server generates a signed authorization token
6. **The token is sent back to ODIS**
7. ODIS uses the token to write new CP data to the module via UDS
8. The module verifies the token's signature and removes its CP restriction

Without step 3-6, step 7 cannot proceed. The token contains a cryptographic signature that the module verifies locally — so you can't fake it without the server's private key.

**The known GEKO error codes** (documented on vw-geko.com):
- `FAZ4990E` — Part is stolen, cannot be adapted
- `FAZ2946E` — Used part not permitted, replace with new (post-Oct 2021 EU only)
- `FAZ2781E` — Part is locked
- `FAZ9484E/9488E` — Part or gateway damaged, cannot be adapted online
- `FAZ1224E` — Problem with cluster data

---

## Part 2: Lear vs Temic Gateway — What's the Difference?

### Overview

The J533 Data Bus Diagnostic Interface (gateway) on VAG vehicles has been manufactured by two different suppliers, and this matters enormously for CP approaches:

| Type | Manufacturer | Platforms | EEPROM Chip | CP Approach |
|---|---|---|---|---|
| **Temic** | Continental/Temic | C6 A6, B7 A4, early Q7 | **25640** (64Kbit SPI) | Gen 1/2, older tools work |
| **Lear** | Lear Corporation | **C7 A6/A7, D4 A8, 7P Touareg** | **95320** (32Kbit SPI) | Gen 2/3, requires VN017 or ODIS |
| **Continental** | Continental | MQB platform (2013+) | Different architecture | Gen 3, ODIS only |

### Your Vehicle: A6 C7 (4G) Uses the Lear Gateway

Your 2013 A6 C7 uses the **Lear gateway**. This is confirmed by multiple community sources and ABRITES documentation, which specifically calls out:
- "Audi A6/A7/A8 2010-2017 — gateway LEAR"
- ABRITES VN017 explicitly supports "models with gateway LEAR: Audi A6/A7/A8 2010-2017"

The Lear gateway was introduced to replace the older Temic gateway around 2010 when the MLB platform launched. It uses different internal architecture, different EEPROM chips, and a different CP protocol — which is why older AVDI/FVDI tools that work on Temic platforms struggle with Lear platforms.

### Identifying Your J533 Gateway

The part number on your host A6's J533 gateway will be in the `4G0907468x` family. Common variants for the 2013 A6 C7:

| Part Number | Notes |
|---|---|
| `4G0907468A` | Early production |
| `4G0907468B` | Revised |
| `4G0907468C` | Common on 2013+ |
| `4G0907468D` | Late production |
| `4G0907468E` | Final C7 variant |

The variant letter matters for EEPROM chip identification — always confirm by visual inspection.

### Physical Differences

**Temic gateway (older C6/B7 platform):**
- Larger module
- Located behind center console or under driver's seat area
- Contains a **25640** (M95640 or equivalent) SPI EEPROM as the main security chip
- Also contains two **A82C250** CAN transceiver chips (not CP-relevant)

**Lear gateway (your C7 platform):**
- More compact module
- Typically located in the instrument panel area, center console, or firewall
- Contains a **95320** (M95320 or equivalent) SPI EEPROM
- The 95320 is the CP master chip — this is where constellation data lives

The community confirmed this distinction clearly:
> *"For component protection on C6 2005: there are 4 EEPROMs — A82C250 x2, 6250G0416, 95320. Which one for component protection? The 95320."* — Digital Kaos forums

---

## Part 3: A6 C7 J533 Gateway — Complete Chip Map

### The 95320 EEPROM — The CP Master Chip

**Chip designation:** ST95320 / M95320 / 95320WP  
**Package:** SO8 (8-pin Small Outline)  
**Interface:** SPI (Serial Peripheral Interface)  
**Capacity:** 32Kbit (4096 bytes / 4KB)  
**Voltage:** 2.5V–5.5V (typically 3.3V in automotive applications)  
**Operating temp:** -40°C to +85°C automotive grade

**Pin layout (SO8):**
```
     ┌────────────┐
 CS̄ ─┤ 1        8 ├─ VCC (3.3V)
  SO ─┤ 2        7 ├─ HOLD̄
  WC ─┤ 3        6 ├─ CLK
 GND ─┤ 4        5 ├─ SI
     └────────────┘
```

- **Pin 1 (CS̄):** Chip Select (active low)
- **Pin 2 (SO):** Serial Output (MISO)
- **Pin 3 (WC):** Write Control (active low, write-protect)
- **Pin 4 (GND):** Ground
- **Pin 5 (SI):** Serial Input (MOSI)
- **Pin 6 (CLK):** Serial Clock
- **Pin 7 (HOLD̄):** Hold input (active low, pauses communication)
- **Pin 8 (VCC):** Supply voltage

### How to Read the 95320 Without Desoldering

The preferred community method for reading the 95320 is **in-circuit via SOIC clip** — no desoldering required:

**Equipment needed:**
- SOIC-8 test clip (available for ~$5-10)
- CH341A programmer OR dedicated EEPROM programmer (T48, MiniPro TL866, etc.)
- 3.3V power supply (critical — some programmers output 5V which can damage automotive EEPROMs)

**Important note from community research:**
> *"The 95xxx series needs 5V supply but the automotive variant in the gateway runs at 3.3V. Use ch341a software configured as 25LC160 for reading 95320."* — MHH Auto

The CH341A programmer in "25xxx BIOS mode" with voltage modification can read the 95320. The community-developed approach:
1. Modify CH341A to output 3.3V (cut middle pin of voltage regulator, connect 5V to VCC pin 8 of EEPROM socket)
2. Configure software as **25LC160** (compatible with 95320 protocol)
3. Connect SOIC-8 clip to the chip while gateway is removed from car (NOT powered)
4. Read and verify (read twice, compare)

**Alternative: dedicated programmer**
Tools like MiniPro TL866II+, T48, or XGecu T56 directly support the 95320 chip and are the safest option.

### ABRITES Approach: Reading 95320 via OBD

ABRITES AVDI (with VN017 license) can read the Lear gateway's CP bytes directly via OBD-II port without physical access to the chip:

From ABRITES documentation:
> *"VN017 allows: Read/Update component protection bytes of Lear Gateway... Replacement of Climatronic — by OBDII... Supported models (models with gateway LEAR): Audi A6/A7/A8 2010-2017"*

This is the cleanest approach for your situation — no soldering, works via the OBD port.

### Your Specific Situation: Two Gateways Available

Since you have both the host A6 and the donor A6, you have:

| Gateway | Vehicle | Role in CP Process |
|---|---|---|
| **Host J533** (`4G0907468x`) | Your 2013 A6 | The vehicle you want the modules to work in — its CP data must be written to the modules |
| **Donor J533** (`4G0907468x`) | Donor 2013 A6 | The vehicle the modules came from — its CP data is currently in the modules |

For ABRITES VN017 (OBD approach):
1. Connect AVDI to **donor A6** → Read donor J533 CP bytes
2. Reset replacement modules to "virgin" state using donor J533 data
3. Connect AVDI to **host A6** → Read host J533 CP bytes  
4. Learn modules to host A6 using host J533 data

For manual EEPROM approach:
1. Remove J533 from **donor A6** → Read 95320 EEPROM dump (this is the "confdata")
2. Reset target module to virgin using confdata from donor gateway
3. Remove J533 from **host A6** → Read 95320 EEPROM dump
4. Learn module to host A6 using host gateway confdata

### Affected Module EEPROM Chips — Your Specific Modules

**J255 Climatronic (address 08):**
- EEPROM type: **95320** (confirmed by community for C7 generation)
- Location: Inside Climatronic unit, on main PCB
- CP data: Stored in 95320, can be reset via OBD with VVDI2 or AVDI VN017
- Key community finding: *"First time read climatronic via OBDII needs reflash module"* (VVDI2 documentation) — expect a 2-minute reflash on first OBD read

**J136 — Memory Seat/Steering Column (address 36):**
- EEPROM type: **95C160** (per ABRITES AVDI documentation for A8 platform — verify for C7)
- Alternative: May use **95320** depending on revision
- CP data: In EEPROM, addressable via OBD
- Note: Has additional PIN-code seat-position binding separate from CP

**J521 — Front Passenger Memory Seat (address 06):**
- Same architecture as J136
- EEPROM type: **95C160** or **95320** (verify by visual inspection)
- CP data: Same approach as J136

### VVDI2 Support Matrix for Your Modules

From VVDI2 documentation, confirmed supported for A6/Q7/Allroad (your platform):

| Module | ODIS Address | OBD Reset | Needs Donor Gateway |
|---|---|---|---|
| EZS-Kessy (J518) | 25 | ✅ Yes | No |
| Airbag (J234) | 15 | ✅ Yes | No |
| Instrument Cluster (J285) | 17 | ✅ Yes | No |
| Central Comfort (J393) | 46 | ✅ Yes | No |
| **Climatronic (J255)** | 08 | ✅ Yes | **Check — may need donor** |
| **Seat Memory Driver (J136)** | 36 | ✅ Yes | **Check — may need donor** |
| **Seat Memory Passenger (J521)** | 06 | ✅ Yes | **Check — may need donor** |
| Sound System (J525) | 47 | Varies | May need EEPROM dump |
| Information Electronics (J794) | 5F | ✅ Yes | No (OBD sufficient) |

The "Check" modules are the interesting case — for A6/Q7 2004-2009 generation, they work by OBD without donor gateway. For the C7 (2010+) generation (your car), ABRITES VN017 is the more reliable path, and it handles the Climatronic by OBD specifically.

---

## Part 4: Practical Roadmap for Your Situation

Given everything documented here, your optimal path from lowest risk to highest risk:

### Option A: ODIS 25.x with GRP Access (Lowest Risk, Most Reliable)
- Your ODIS-S should already support this if it's version 25.x
- Purchase GRP online session from a provider (€20-50 per session)
- Use your existing ODIS-S + VAS adapter
- Result: Clean, FAZIT-verified, no risk of bricking
- **This is what worked before GEKO became unreliable**

### Option B: ABRITES AVDI VN017 (Offline, Professional)
- Purchase AVDI hardware + VN002 license + VN017 license
- VN017 specifically designed for "Audi A6/A7/A8 2010-2017 — gateway LEAR"
- Supports Climatronic by OBD — no need for physical EEPROM access
- Requires both host and donor gateways (you have both)
- Cost: AVDI ~€1,500, VN002 ~€300, VN017 ~€200 — significant investment
- **Best offline option for your exact platform**

### Option C: VVDI2 (Partial Offline)
- Xhorse VVDI2 with VAG license
- Supports your modules including Climatronic via OBD
- Less expensive than AVDI: ~€300-400
- Coverage for C7 platform is less certain than AVDI — check current release notes
- **Worth investigating before committing to AVDI**

### Option D: Physical EEPROM Approach (Highest Skill Required)
- Read donor J533 95320 with SOIC clip + programmer
- Use FVDI/AVDI with donor gateway confdata
- Reset module to virgin, re-learn to host A6 using host J533 data
- Risk: If module is rendered "virgin" improperly, ODIS online may not fix it
- **Only recommended if you have EEPROM programming experience**

---

## Sources
- ABRITES VN017 documentation (abrites.com)
- OBDII365 blog — VVDI2 Component Protection manual
- autogmt.com — ODIS 25.x GRP transition discussion
- vw-geko.com — GEKO error code documentation
- MHH Auto forums — 95320 reading guides
- Digital Kaos forums — Temic vs Lear gateway EEPROM identification
- ABRITES VAG User Manual HTML — J136/J521/J255 EEPROM types
- diagnostics.vis4vag.com — A6 4G control unit database (J533 variants listed)
- VVDI2 VAG User Manual — Component Protection procedure
