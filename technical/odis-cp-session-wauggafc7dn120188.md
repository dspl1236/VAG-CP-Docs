# ODIS CP Session — WAUGGA**********8 — Feb 27, 2024

**Source:** `NO_ORDER_WAUGGA**********8_2024-02-27_22-59-28_dprot.htm`  
**Session:** February 27, 2024, 22:59 UTC  
**Tool:** ODIS-S Patch 23.0.1, VAS 6154A, VCI via USB  
**ODIS user:** Dealer 17483, importer 219 (authorized dealer)  
**Tester ID:** AGJ1E

---

## What happened in this session

This is a complete Component Protection removal session performed on VIN
`WAUGGA**********8` (2013 Audi A6 C7, 3.0T TFSI CGWB, built Neckarsulm).

**12 modules were successfully CP-cleared:**

| Module | System | Protocol |
|--------|--------|----------|
| J136 | Memory Seat/Steering Column | KWP2000 |
| J518 | Access/Start Control (Kessy) | KWP2000 |
| J255 | Climatronic | UDS |
| J519 | Vehicle Electrical System | KWP2000 |
| J234 | Airbag | UDS |
| J285 | Instrument Cluster | UDS |
| J525 | Digital Sound System | KWP2000 |
| R- | Radio | KWP2000 |
| J794 | Information Electronics 1 (MMI) | UDS |
| J854 | Left Front Seat Belt Tensioner | KWP2000 |
| J855 | Right Front Seat Belt Tensioner | KWP2000 |
| J533 | Gateway (master) | UDS |

---

## IKA key blob — J136 (only unmasked blob)

DID `0x00BE` written via `DiagnServi_WriteDataByIdentGenerServi`:

```
E6 2B 41 D1 1C 44 AF 20 21 77 FB 1F 27 4B 0A C2
D1 5B D2 62 E4 FD 27 AB 61 D1 23 C2 F1 5A 2C 93 26 00
```

- **Length:** 34 bytes (272 bits) — matches confirmed DID 0x00BE structure
- **Last byte:** `00` — possible padding or version indicator
- **Module:** J136 — `SVF6000` variant, FAZIT `CU5-SIB 21.03.11 0002 0968`

All other modules had their IKA keys masked as `******` in the ODIS log.
The J136 blob was logged verbatim because it went through a direct
`WriteDataByIdentifier` call rather than the masked `TrainICA` KWP service.

---

## Constellation write — J533

DID `0x04A3` written to J533 Gateway:

```
Before: FD A1 E9 0C FE 62 64 8D 00 00
After:  FD A1 E8 0C FE 62 60 0D 00 00
```

- 10 bytes, encoded component presence bitmap
- Bytes 4–5 changed: `64 8D` → `60 0D` (bit changes marking modules as enrolled)

---

## J518 identification (read before sending to GEKO)

The ODIS function `j518_4g_96_Algem_3_0613_21_Komponentenschutz_C6_00021`
read the following from J518 before contacting GEKO:

| DID | Value |
|-----|-------|
| F190 VIN | `WAUGGA**********8` |
| F17C FAZIT (J518) | `HLH-W41 / 21.02.13 / 1003 / 1126` |
| F191 HW number | `4H0907064CR` |

And from J533 Gateway:
| Field | Value |
|-------|-------|
| FAZIT | `LAK-000 / 22.02.13 / 2009 / 7162` |

**These 5 values are what ODIS sent to GEKO to request the token.**  
The key fob transponder data was NOT explicitly logged — it was passed
internally via the `TrainICA` / `TrainGVA` KWP calls (masked with `******`).

---

## CP procedure flow (reconstructed)

```
1. Read J518 FAZIT + HW number
2. Read J533 FAZIT
3. Send to GEKO: VIN + J518_FAZIT + J533_FAZIT + (transponder data via J518)
4. GEKO returns: IKA key blob for each component (masked in log)
5. For KWP modules: TrainICA(blob) → TrainGVA(blob) → stage 3 confirm
6. For UDS modules: WriteDataByIdentifier(0x00BE, blob) via gateway function
7. J533 constellation (0x04A3) updated to reflect newly enrolled modules
```

---

## What inputs drive the IKA key

From this log we can confirm GEKO received:
- **VIN:** `WAUGGA**********8`
- **J518 FAZIT:** `HLH-W4121.02.1310031126` (part number + date + codes)
- **J518 HW:** `4H0907064CR`
- **J533 FAZIT:** `LAK-00022.02.1320097162`
- **Target component FAZIT:** (varies per module — read before each token request)
- **Transponder data:** via J518 `TrainICA` (key fob challenge-response, not logged)

The transponder data from the key fob is the unknown input. If the IKA key
derivation is deterministic given these inputs + a fixed GEKO signing key,
then the entire CP removal is reproducible offline once:
1. The J533 verification key is extracted from MCU firmware, OR
2. Two examples with different VINs are compared to isolate the derivation

---

## Next steps

1. **Read DID 0x00BE from J136 on the live car** — confirm the blob written
   here is still present (it should be, unless J136 was swapped again)
2. **Compare with a second car** — a second CP log with different VIN/modules
   would let us test whether the IKA key is VIN-bound or module-bound
3. **J533 firmware extraction** — the verification key that accepted this blob
   is in the Renesas D70F3433 MCU flash
