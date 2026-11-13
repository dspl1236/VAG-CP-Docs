# Seat CP limp-patch over OBD — J136 / J521 (4x0959760, V850E2)

The C7 memory-seat modules (J136 driver, J521 passenger; Renesas V850E2, part
`4x0959760`) can have their Component-Protection *limp* behaviour neutralised with a
**one-byte firmware patch that is written over the diagnostic (OBD) port** — no bench, no
N-Wire serial-ISP, no exploit. This corrects an earlier assumption that the seats, like the
HVAC, needed a bench rig; they do not.

Implementation lives in **simos-suite** (`flasher/seat_cp_patch.py`, `flasher/seat_flash.py`,
`tests/test_seat_flash.py`). This document is the *findings* behind it.

## Why it's OBD-flashable: the seat is CRC32-only, not RSA-signed

A module's flashware can only be reflashed with modified firmware over OBD if its integrity
gate is a *recomputable checksum*, not an unforgeable *signature* (see
[`vag-rsa-vs-crc-flash-map.md`](vag-rsa-vs-crc-flash-map.md)).

The `4x0959760` seat is **checksum-only**:

- Its ODX carries `SECURITY-METHOD = CRC32` + `SA2`, with **no `SIG_*` / `FW-SIGNATURE`** on
  any data block. Confirmed directly on the 4H0 `.sgo` and the 4M0 FRF, and by the
  cross-platform flash-integrity survey.
- Contrast: the HVAC (`4G0820043`) *is* `SIG_SHA1-RSA1024`-signed and ECU-enforced — the one
  genuine wall of the three salvage modules. The seat has no such wall.

## The integrity model: CheckMemory = CRC-32/ISO-HDLC over the uncompressed block

The value the seat bootloader expects at **CheckMemory (`31 01 0202`)** is the VAG ODX
`FW-CHECKSUM`, which is **CRC-32/ISO-HDLC — i.e. plain `zlib.crc32` — over the whole
*uncompressed* data block**. No VW polynomial, no padding, no segment-placement tricks.

Verified against the Oct-2024 flashdaten: on plain (ENCRYPT-COMPRESS-METHOD `0x00`) FRFs,
`zlib.crc32(block)` reproduces the declared `FW-CHECKSUM` **exactly** —

| FRF | blocks matched |
|---|---|
| `FL_4K0035335B_0533` | 6 / 6 |
| `FL_2G0035456_3133` | 2 / 2 |
| `FL_5N0035729A_0592` | 2 / 2 |
| `FL_0D7142057D_0031` | 2 / 2 |
| `FL_6C0819045A_0108` | 2 / 2 |

and `simos-suite/flasher/frf_pack.py` already recomputes the same way. An earlier attempt
reported "standard CRC variants fail" — that was computing over the *compressed* or
*zero-padded* bytes, not the uncompressed block.

Two distinctions worth keeping straight:

- This ODX `FW-CHECKSUM` (sent at CheckMemory) is **not** the Simos *internal* block-header
  CRC32 (VW poly `0x4C11DB7`, MSB-first, over areas listed in a header at offset `0x300`).
  Those are different checks on different platforms.
- The seat has **no internal checksum header at all** — startup is plain C-runtime init,
  with no CRC/SHA table in the image. So on the seat, CheckMemory is the *only* integrity
  fixup a modified image needs.

## The CP limp-gate (mechanism)

From the Ghidra decompile (`FUN_00036cda` case 7): the CP verify computes
`r7 = (memcmp(computed_AES, stored_expected) == 0)` and latches the CP auth-status byte via
`FUN_0003729c(0, r7)`. The machine tail is

```
mov 0,r7 ; cmp r0,r2 ; bnz +4 ; mov 1,r7 ; mov 0,r6 ; jarl FUN_0003729c
```

i.e. `r7` **defaults to 0 (not authenticated)** and only becomes 1 on a byte-exact match.
Flipping the default `mov 0,r7` → `mov 1,r7` makes *both* paths leave `r7 = 1` ⇒ the module
treats itself as always authenticated and never limps. One byte.

The gate is located by a 10-byte **code signature**, not a fixed offset, so it survives SW
drift across the `4x0959760` family. In the 4H0 `.sgo` it lands in the `0x038000` code block
(absolute `0x3EE66`). Exact signature + offsets are in the simos-suite implementation; this
is owner's-own-module, per-module, and reversible (write the byte back), and it transfers no
credential.

## The flash sequence (standard VAG UDS)

```
10 02                    programming session
27 … (SA2)               SecurityAccess — bytecode from the .sgo/ODX, key computed locally
  for the block holding the gate:
    31 01 FF00 …         EraseMemory
    34 … (DFI 00/00)     RequestDownload — uncompressed
    36 …                 TransferData — the PATCHED block bytes
    37                   RequestTransferExit
    31 01 0202 01 bb 00 04 <crc32>   CheckMemory — the recomputed zlib.crc32 of the block
31 01 FF01               CheckProgrammingDependencies
11 01                    ECU reset
```

Only the block holding the gate need be re-flashed (each 32 KB block is independently
erasable/CheckMemory-able).

## Open items — confirm on the car/bench (all non-destructive to probe)

1. **CAN IDs** — the convenience-bus seat-memory request/response IDs (from a live sniff of
   the car; not assumed in the tooling).
2. **DFI acceptance** — the stock seat flash is ENCRYPT-COMPRESS-METHOD `0x10` (BCB, VW's
   1A-escape codec, not yet reversed). The tooling sends *uncompressed* (DFI `00/00`); the
   on-board decompressor keys off the DFI. A clean NRC at RequestDownload for `00/00` would
   mean this bootloader insists on `0x10` transfers.
3. **CheckMemory parameter layout + address format** — the `[01][block_id][00 04][crc32]`
   framing and whether the unit addresses by physical block address or a small block index.

## Related

- [`vag-rsa-vs-crc-flash-map.md`](vag-rsa-vs-crc-flash-map.md) — which VAG modules are
  signature-walled vs checksum-only (the seat is checksum; the HVAC is the RSA exception).
- Flash signatures are **ECU-enforced** (the LEAR/V850 finding): a signed module cannot be
  OBD-flashed with modified firmware, which is why the HVAC stays bench/oracle-test and the
  seat does not.
