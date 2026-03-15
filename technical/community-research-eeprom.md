# Community Research: EEPROM, Gateway Dumps, and Offline CP Approaches

*This document compiles publicly available community research on Component Protection internals, EEPROM structures, and approaches developed by the diagnostic tool community. All information is from public sources including tool documentation, forum research, and reverse engineering publications. This is documentation for advocacy and education purposes.*

---

## Overview: What the Community Has Figured Out

The automotive diagnostic tool community — particularly the ABRITES/AVDI team, FVDI/SVCI developers, and independent researchers — has spent years reverse engineering how CP works at the EEPROM level. Their published findings reveal the actual data structures involved. This is significant for right-to-repair advocacy: it proves the system is not fundamentally unbreakable, it's just that VW has erected legal and practical barriers to the knowledge.

---

## The Core Discovery: CP Lives in the J533 Gateway EEPROM

The most important finding from community research is this:

**Component Protection data is stored in the J533 gateway's EEPROM chip.** On Generation 2 platforms (which includes the A6 C7 / 4G), the gateway uses a **95320 EEPROM** (an SPI EEPROM chip). This chip contains the "constellation" — the cryptographic binding data that ties each module to the vehicle.

From OBDII365 community research:
> *"Generation 2: All other models of A6/Q7/Allroad and A4/A5/Q5 modules. You need dump from both gateway module from the car you work on and car you get climate unit. More easy, open your old clima, and change the eeprom. Read eeprom 95320 in Gateway, and load dump eeprom dump and reset CP."*

The key part numbers for the J533 gateway on Audi A6 C7 / 4G platform include:
- `4G0907468` series (main C7 gateway family)
- `4H0907468` series (A8 D4, shares architecture)

---

## EEPROM Map: What's Known (Published Community Research)

### MMI J794 / J523 EEPROM (AT24C64A chip)

AudiEnthusiasts.com published a partial EEPROM map for the MMI infotainment module based on reverse engineering:

| Address | Content |
|---|---|
| `0x0040` | Hardware part number (ASCII) |
| `0x0040-50` | Software part number (ASCII) |
| `0x0050` | Module identifier (J794 etc.) |
| `0x0080-90` | Serial number |
| **`0x0640–0x0770`** | **Component Protection data** |
| `0x0910` | Firmware version details |
| `0x0B00–0x19C0` | Configuration data (non-human-readable) |

The researcher noted:
> *"In MMI the component protection is saved between 0x0640 and 0x0770... You can transfer/duplicate/remove this section between all different MMI 3G and MMI 3G Plus units... As of this minute there isn't an easy way to translate or generate the above code. It's not based on VIN only, it's also based on several other variables that get pulled from CAN gateway."*

This is crucial: **CP is NOT simply a VIN hash.** It incorporates variables from the gateway's own state at the time of binding, making it non-trivially reproducible without the original gateway data.

### J533 Gateway EEPROM (95320 / 25640 chips)

Community research (MHH Auto, Digital Kaos) has identified two variants of the J533 gateway EEPROM on pre-C7 platforms:
- **25640** (SPI, 64Kbit) — used on 8T0/8K0 series gateways (A4/A5/Q5 B8, A6 C6 late)
- **95320** (SPI, 32Kbit) — used on 4G0/4H0 series gateways (A6 C7, A7, A8 D4)

The gateway EEPROM contains:
- The vehicle's VIN binding data
- The "constellation" — identities of all CP-protected modules
- Security/encryption seeds
- The "confdata" (configuration data) that AVDI tools reference

---

## What ABRITES/AVDI Has Reverse Engineered (Published)

ABRITES (a Bulgarian automotive diagnostic company) has published the most detailed public documentation of CP internals through their VN002 and VN017 license documentation. Their findings:

### Generation 2 CP Architecture (Your A6 C7)

For your 2013 A6 C7, ABRITES classifies this as a **Lear Gateway** platform (as opposed to the older Temic Gateway). Their VN017 documentation reveals:

**The gateway is the master, not the immobilizer:**
> *"Component Protection Generation 2 — the master here is the gateway."*

This means for your platform, all CP binding flows through J533's EEPROM, not through J518/EZS as on older platforms.

**What VN017 can do for A6/A7/A8 2010-2017 (your platform):**
- Read/Update CP bytes of the Lear Gateway EEPROM directly
- Replace all CP-related modules if BOTH donor AND host gateway EEPROMs are present
- Replace dashboard (instrument cluster) by OBD — **no donor gateway needed**
- Replace infotainment module by OBD — **no donor gateway needed**
- Replace Climatronic (J255) by OBD

The critical insight here is the **asymmetry**: dashboard and MMI can be adapted without donor gateway data, but J255 (Climatronic) and seat modules still require it — or they did until their online CP capability was added.

**What this means for your situation:** You bought the entire donor A6, which means **you have the donor gateway (J533)**. This puts you in the best possible position for an offline approach — you have both the host and donor gateway EEPROMs available.

---

## The VVDI2 Approach (Published Manual)

Xhorse VVDI2, another widely-used diagnostic tool, published their component protection procedure in their user manual. For A6/Q7/Allroad (which includes the C7):

**Step 1 — Reset module to "virgin" state:**
Methods available:
- Read EEPROM via OBD (available for EZS-Kessy, instrument cluster, airbag, comfort module, Climatronic)
- Read module EEPROM dump directly (physical programmer)
- Read gateway EEPROM dump (works for all devices if you have the donor gateway)

**Step 2 — Load new vehicle CP data:**
For A6/Q7/Allroad: *"You can load new car J518 EEPROM"* (readable by OBD)

**Step 3 — Learn module to new car:**
> *"Load GATEWAY EEPROM dump (Load the new vehicle GATEWAY EEPROM dump)"*
> *"Read by OBDII (Requires connect to new car, will reflash GATEWAY — takes about 2 minutes)"*

The VVDI2 specifically supports:
- Climatronic (J255) CP by OBD — for A6/Q7 2004-2009
- Comfort module (J393) CP by OBD
- Seat memory (J136/J521) CP by OBD

**Important caveat noted in community research:** The VVDI2 and AVDI approaches work cleanly up to approximately 2009-2010 model year. For 2010+ (your 2013 A6), they require the VN017 license and/or online capability. This is because the C7 platform uses the Lear gateway architecture which was a significant change from the earlier Temic gateway.

---

## The "Virgin" State — Critical Warning

Community research has identified a critical pitfall that is important to document:

**If a module is reset to "virgin" state using AVDI/FVDI tools, it may become impossible to adapt it online with ODIS afterward.**

From MHH Auto forum research:
> *"I know this problem — if cluster was made virgin with AVDI, ODIS can't remove CP anymore. Also I had this problem with EZS. So it is better to use just ODIS and forget AVDI solution."*

This creates a dangerous situation: attempting an offline approach with the wrong tool can permanently brick the module's ability to be adapted by any means. **The order of operations matters enormously.**

The correct approach if using ODIS online remains:
1. Do NOT attempt to reset to virgin with third-party tools first
2. Use ODIS with active GEKO/GRP credentials directly

---

## The "EEPROM Swap" Approach (Community Folklore)

One approach circulated in the community — particularly for J255 Climatronic modules — is the physical EEPROM swap:

> *"More easy, open your old clima, and change the eeprom."* — OBDII365 community

The concept: physically remove the 95320 EEPROM from your working original J255, solder it into the replacement J255, and the replacement now carries the correct CP identity from your vehicle's constellation.

**This approach has been reported to work for some modules on some platforms but:**
- Requires precise soldering of a surface-mount chip
- Requires identifying the correct EEPROM chip (varies by module and revision)
- Does not work on modules where CP data is in the main MCU flash (not a separate EEPROM)
- Carries a risk of destroying both units
- May not work on 2013 C7 platform modules where the architecture differs from earlier generations

---

## ODIS-S Specific Capabilities

Since you have ODIS-S (engineering version), here is what it exposes beyond ODIS-E relevant to CP:

### Security Access Levels
ODIS-S exposes extended security access levels for adaptation channels. Standard ODIS-E requires the GEKO server to unlock security access for CP-relevant channels. ODIS-S in some configurations can:
- Access lower security levels (Level 1/2) without online authentication
- View (but not write) CP-related adaptation channels
- Show the raw channel values that contain CP state data
- Perform SVM (Software Version Management) operations that ODIS-E may block

### What to Look For in ODIS-S
For your affected modules, in ODIS-S the relevant adaptation paths are typically:
- **J255 (address 08 — Climatronic):** Adaptation → Component Protection channels
- **J136 (address 36 — Seat memory driver):** Adaptation → Component Protection / Security channels
- **J521 (address 06 — Seat memory passenger):** Adaptation → Component Protection channels

In ODIS-S, the "Guided Functions" for these modules should show a "Component Protection" guided function. Without GEKO credentials, this function will initiate and then fail when it tries to authenticate with the server — but the failure message and the diagnostic data visible before the failure can reveal the exact protocol being used.

### Firmware File Analysis
Your module firmware files (.frf, .odx format) contain:
- UDS (Unified Diagnostic Services) service definitions
- Security access seed/key algorithm descriptions
- Diagnostic data identifiers (DDIs) — some of which reference CP state
- Adaptation channel definitions

The UDS service 0x27 (Security Access) in your firmware files would show the security levels required for CP-related write operations. This is researchable without touching a vehicle.

---

## Ideas Worth Exploring (Community-Derived)

Based on the totality of public research, the following ideas are intellectually interesting from a documentation perspective:

### Idea 1: Gateway EEPROM Cross-Read
Since you have both the donor and host A6 — both J533 gateways are available. The fundamental requirement for an offline approach is having both EEPROMs. Tools like AVDI VN017 specifically describe this exact scenario as supported. The research question is whether the C7 Lear gateway variant is within AVDI's VN017 coverage envelope.

### Idea 2: ODIS-S Protocol Capture
With ODIS-S connected to the vehicle via a VAS adapter, a network packet capture (Wireshark) of the traffic between ODIS and the GEKO/GRP server during an attempted CP removal would reveal the exact protocol. The request payload would contain: VIN, module serial number, security seed. The response would contain the unlock token. Understanding this protocol is the key to any offline server emulation approach — which is exactly what projects like "ELSAWin Local Server" and similar have done for other VW systems.

### Idea 3: SVM Offline Emulation Precedent
There is an existing community project for offline SVM (Software Version Management) — a related VW online system. The existence of working offline SVM implementations proves the protocol is reversible. CP uses a similar client-server architecture over the same ODIS online channel.

### Idea 4: AVDI CP Online
ABRITES released an "online CP" capability for their AVDI tool — meaning they have their own server that handles the CP adaptation, independent of VW's GEKO. This is the most significant community development: **a third party has replicated the server-side functionality of GEKO.** This proves definitively that the protocol is known and implementable. Their service requires paying for their tool ecosystem, but the existence of it is extremely relevant for right-to-repair arguments — it demonstrates VW's claim that server access is technically necessary is only true if VW maintains a monopoly on the server.

---

## Implications for Right to Repair

The community research documented here establishes several important facts:

1. **The CP protocol has been reverse engineered.** ABRITES, VVDI2, and others have working implementations that bypass the GEKO server dependency — this is public knowledge.

2. **The "anti-theft" justification is technically thin.** Community-discovered approaches work via gateway EEPROM — meaning anyone who steals both the module AND the gateway from the same vehicle can adapt parts anyway. The system primarily stops legitimate owners, not sophisticated thieves.

3. **The EEPROM data is the actual lock.** CP is not a fundamentally cryptographic system that can't be broken — it's a data pairing system where the data lives in accessible EEPROM chips. The barrier is legal (DMCA) and practical (specialized tools), not mathematically unsolvable.

4. **Third-party server replication is proven possible.** ABRITES' online CP service proves the server-side protocol is known and implementable. VW's server dependency is a business choice, not a technical necessity.

---

## Sources
- AudiEnthusiasts.com — MMI EEPROM map (audienthusiasts.com/Infotainment_EEPROM.html)
- ABRITES/AVDI VN002 documentation (abrites.com)
- ABRITES VN017 documentation (abrites.com/blog/when-life-gives-you-lemons-make-component-protection)
- VVDI2 VAG User Manual — Component Protection section
- OBDII365 blog — VAG CP tool comparison
- MHH Auto forums — community technical research threads
- Keyprogtools — AVDI Component Protection Manager Gen 1/2 documentation
- Digital Kaos forums — EEPROM swap research threads
- Audizine forums — VCP/EEPROM CP transfer research
