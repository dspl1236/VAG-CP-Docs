# Component Protection — How the System Actually Works

*Written to accompany the Feb 27, 2024 ODIS session log for VIN WAUGGAFC7DN120188.*

---

## The short version

Component Protection is not a property of individual modules. It is a **car-wide
enrollment system managed by J533** (the CAN gateway). Every module that supports
CP is enrolled into J533's master list, and J533 validates the list on every drive
cycle. When you clear CP on one module, you are really making a change to J533's
list and then telling the affected module its new enrollment state.

---

## J533 is the master

J533 holds two critical data structures:

**DID `0x04A3` — the constellation (10 bytes)**  
A bitmap encoding which modules are enrolled and in what state. Every enrolled
module has a bit position in this field. When a module is added or removed from
CP enrollment, this field is rewritten.

From the Feb 2024 session log, DID `0x04A3` changed from:

```
FD A1 E9 0C FE 62 64 8D 00 00   (before — some modules still CP-active)
FD A1 E8 0C FE 62 60 0D 00 00   (after  — all cleared)
```

**DID `0x00BE` — the IKA key (34 bytes)**  
J533 stores one IKA (Individual Key Authentication) key that represents the
vehicle's identity. Every enrolled slave module stores the same or a
module-specific variant of this key in its own DID `0x00BE`. On each drive cycle,
J533 challenges each enrolled module — the module must present a valid key or
CP activates.

The key written to J136 (Memory Seat) in the Feb 2024 session:

```
E6 2B 41 D1 1C 44 AF 20 21 77 FB 1F 27 4B 0A C2
D1 5B D2 62 E4 FD 27 AB 61 D1 23 C2 F1 5A 2C 93 26 00
```

---

## Why the session touched every module

When ODIS runs CP removal it calls the function
`j533_4G_90_UDS_2_0613_21_KS_SGs_ermitteln_00021` — literally "determine CP
control modules." J533 returns its full enrollment list: every module currently
tracked with an active IKA key record.

ODIS then iterates through that list and clears each module in turn:

```
J533  → cleared first (the master)
J518  → Access/Start (Kessy) — KWP2000
J255  → Climatronic — UDS
J519  → Vehicle Electrical System — KWP2000
J234  → Airbag — UDS
J285  → Instrument Cluster — UDS
J525  → Digital Sound System — KWP2000
R--   → Radio — KWP2000
J794  → MMI (Information Electronics 1) — UDS
J854  → Left Front Seat Belt Tensioner — KWP2000
J855  → Right Front Seat Belt Tensioner — KWP2000
J136  → Memory Seat — KWP2000
```

Then DID `0x04A3` on J533 is rewritten to reflect the new all-cleared state.

**The reason all modules must be cleared together is consistency.**  
J533 and its slave modules must agree on enrollment state. If you clear J255 alone
without updating J533's constellation, J533 still thinks J255 is enrolled with its
old key state. On the next drive cycle J533 polls J255, the states don't match, and
CP faults return. The ODIS procedure is designed to be atomic: clear everything in
J533's list in one session, then update the constellation. All or nothing.

---

## How each module gets cleared

There are two clearing mechanisms depending on the module's protocol generation:

**KWP2000 modules (J518, J519, J525, J136, J854, J855, Radio):**  
ODIS calls `TrainICA` followed by `TrainGVA` — two KWP service calls that write
the IKA key blob to the module. The key blob itself is masked as `******` in the
ODIS log, but the service calls succeed and the module acknowledges enrollment.

**UDS modules (J255, J234, J285, J794):**  
ODIS calls `WriteDataByIdentifier` with DID `0x00BE` and the 34-byte key blob.
For most UDS modules this also goes through a gateway function and the raw bytes
are not logged. J136 is the exception — its write was logged verbatim because it
went through a different code path, giving us the only unmasked IKA blob in the
entire session.

---

## The J518 and key fob role

Before contacting GEKO, ODIS reads three things:

| Source | DID | Value |
|--------|-----|-------|
| J518 FAZIT | F17C | `HLH-W41 21.02.13 1003 1126` |
| J518 HW number | F191 | `4H0907064CR` |
| J533 FAZIT | F17C | `LAK-000 22.02.13 2009 7162` |

These — combined with the VIN and each module's own FAZIT — are what ODIS sends
to the GEKO server to request the token. The key fob transponder data is also read
from J518 via `TrainICA` (masked in the log). It is unclear whether the transponder
data is an authorization check only (proving physical possession of the car) or
whether it is mathematically incorporated into the IKA key derivation. This is an
open research question.

---

## What happened with the replacement J255

The original J255 failed physically and was removed from the car. When ODIS ran the
Feb 2024 CP removal session, the replacement (4-zone) J255 was already installed.
That unit was present in J533's constellation and was cleared in the session along
with everything else.

The original broken J255 was never in this session. It retains its factory state:
DID `0x00BE` is either all zeros (never written) or contains a stale key from
before the Feb 2024 session. Either way, J533 does not recognize it as enrolled.

When the original J255 is plugged back in, J533 sees a module that:
1. Is not in the current constellation (DID `0x04A3`)
2. Has no valid IKA key in DID `0x00BE`

J533 responds with DTC U110100 (Component Protection Active) and the module is
locked out.

---

## The fix

Two writes are needed to clear the original J255:

**Step 1 — Write the IKA key to J255:**
```
WriteDataByIdentifier(DID=0x00BE, data=<34-byte IKA blob>)
```

The blob to write depends on whether the IKA key is per-vehicle or per-module
(this is what the hardware test is designed to determine). If all modules on this
car share the same blob — as is the case if the key is purely VIN-bound — then
the J136 blob can be written directly to J255:

```
E6 2B 41 D1 1C 44 AF 20 21 77 FB 1F 27 4B 0A C2
D1 5B D2 62 E4 FD 27 AB 61 D1 23 C2 F1 5A 2C 93 26 00
```

**Step 2 — Update J533's constellation to include J255:**
```
WriteDataByIdentifier(DID=0x04A3, data=<updated 10-byte bitmap>)
```

The current constellation value `FD A1 E8 0C FE 62 60 0D 00 00` must be updated
to set the bit corresponding to J255's slot. The exact bit position can be
determined by comparing the before/after values from the Feb 2024 session.

Both writes require an extended UDS session and SA2 authentication on J533.

---

## Open questions

| Question | Why it matters | How to answer |
|----------|---------------|---------------|
| Is the IKA key per-vehicle or per-module? | Determines whether one blob works for all modules | Read DID 0x00BE from J533, J255, J136, J285 and compare |
| Does transponder data go into key derivation? | Determines whether offline generation is possible | Analysis of J533 firmware verification routine |
| What is J255's slot in the constellation bitmap? | Needed for Step 2 | Compare 0x04A3 before/after values |
| Can Step 2 be done without GEKO? | Determines if fix is fully offline | SA2 auth → direct DID write is the hypothesis |

The hardware test described in the project (read DID `0x00BE` from multiple modules
with both J255 variants installed) answers the first and most important question.
If all blobs are identical, the fix is fully determined and offline.

---

*Source: ODIS session log `NO_ORDER_WAUGGAFC7DN120188_2024-02-27_22-59-28_dprot.htm`,
Renesas SH7254R datasheet, VAG-CP-Docs research.*
