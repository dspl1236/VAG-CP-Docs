# CP Removal Session — WAUGGAFC7DN120188 (2024-02-27)

**Vehicle:** 2013 Audi A6/A7 C7, VIN `WAUGGAFC7DN120188`, built Neckarsulm  
**Date:** February 27, 2024, 22:25–22:54 UTC  
**Tool:** ODIS-S, USB VCI, CAN/K-Line  
**GEKO status:** Online (Immobilizer/CP endpoint active — now offline as of 2025+)  
**Result:** SUCCESS — all 9 modules de-CP'd

---

## IKA Key Written to J533 (DID 0x00BE)

```
E6 2B 41 D1 1C 44 AF 20 21 77 FB 1F 27 4B 0A C2
D1 5B D2 62 E4 FD 27 AB 61 D1 23 C2 F1 5A 2C 93
26 00
```

34 bytes. Last byte `0x00` is padding. 33 bytes of key material.  
Shannon entropy: 4.83 bits/byte — consistent with AES-encrypted or RSA-signed data.

---

## Key Fob / J518 Inputs Sent to GEKO

ODIS read these values from J518 (Access/Start Control Module, KWP) before
calling GEKO. These are the authorization inputs the server used to generate
the IKA key blob:

| Source | Value |
|--------|-------|
| J518 DID 740 (key status) | `D6 A4 E4 99` |
| J518 DID 749 (immobilizer data) | `02 01 03 B9 03 00 00 00` |
| VIN (from J518) | `WAUGGAFC7DN120188` |
| J518 FAZIT | `HLH-W4121.02.1310031126` |
| J518 HW part | `4H0907064CR` |
| J533 FAZIT | `BPA-08516.02.1300670288` |
| Target module | J136 (K36 / SVF_36) |

**DID 740** is the key fob transponder status register — 4 bytes.  
**DID 749** is the immobilizer master data — 8 bytes.

The ODIS log shows the key fob placed on the transponder coil multiple times
during the session. ODIS polled DID 740 in a loop until it returned a valid
value, confirming transponder presence. DID 749 was read once, confirming
the immobilizer master data.

**Critical open question:** Are DID 740 and DID 749 static per vehicle
(fixed hardware values) or dynamic (challenge-response that changes per
session)? If static, the full input set for offline derivation is known.

---

## Redacted GEKO Tokens

The ODIS dprot log masks the actual GEKO token with `******`:

```
+Service: TrainICA
  Request: 1  ******    ← ICA token from GEKO (actual signed blob)
  Response: /ResponseID  0

+Service: TrainGVA
  Request: 1  ******    ← GVA token from GEKO
```

`TrainICA` and `TrainGVA` are KWP service calls to J518 that write the
GEKO-generated authorization tokens into the immobilizer. These tokens
are what enables J518 to then authorize the IKA key write to J533.

The sequence is:
1. GEKO receives: VIN + J518 FAZIT + key fob data + target module
2. GEKO returns: TrainICA blob + TrainGVA blob + IKA key
3. ODIS writes TrainICA/GVA to J518 (KWP)
4. ODIS writes IKA key to J533 DID 0x00BE (UDS)
5. ODIS writes IKA key to J255 DID 0x00BE (if applicable)
6. J533 updates constellation DID 0x04A3

---

## Constellation Write (DID 0x04A3)

```
Written:   FD A1 E8 0C FE 62 60 0D 00 00
Read back: FD A1 E9 0C FE 62 64 8D 00 00
```

J533 modified bytes 3 and 7 after the write — it updated its internal
timestamp/counter fields. The core bitmap (`FD A1` prefix) is unchanged.

---

## Modules Successfully De-CP'd

| Module | Description | Address | Protocol |
|--------|-------------|---------|----------|
| J533 | Data Bus OBD Interface (Gateway) | 0x19 | UDS |
| J518 | Access/Start Control Module (Kessy) | 0x05 | KWP |
| J255 | Climatronic Control Module | 0x08 | UDS |
| J519 | Vehicle Electrical System Control | 0x09 | UDS |
| J234 | Airbag Control Module | 0x15 | UDS |
| J285 | Instrument Cluster Control Module | 0x17 | UDS |
| J136 | Memory Seat/Steering Column Adj. | 0x36 | UDS |
| J525 | Digital Sound System (BOSE) | — | — |
| R | Radio | 0x56 | — |

---

## Implications for Offline Derivation

This log confirms:

1. The IKA key is **34 bytes**, written to `DID 0x00BE` on J533 and J255
2. The authorization flow goes through **J518 (KWP)** using `TrainICA`/`TrainGVA`
3. The key fob is required to read **DID 740** (4 bytes) and **DID 749** (8 bytes)
4. The GEKO server uses: VIN + J518 FAZIT/HW + fob data → generates signed tokens
5. The exact IKA key for this VIN is known: `E62B41D11C44AF202177FB1F274B0AC2D15BD262E4FD27AB61D123C2F15A2C932600`

If DID 740/749 values are static (not challenge-response), then for any car
where these values can be read offline, the full input set for offline
derivation is available. This requires:
- Knowing the GEKO signing key (in GEKO server or J533 firmware)
- Or finding a pattern across multiple VIN/key pairs

The next step is to read DID 740 and DID 749 on the same car at different
times to determine if they are static. Use simos-suite CP Tools tab or:
```
python -m cp_tools.j533_probe  # reads all CP DIDs including J518 via KWP
```
