# VAG flash-integrity map — signed vs checksum-only (Oct-2024 FlashDaten)

> **Correction (2026-11-12).** An earlier version of this map under-counted signed
> modules (~11 % → **~44 %**) and described the signature as an ODIS-side gate. Both are
> wrong and are fixed here. Follow-on reverse engineering recovered the actual per-module
> signing keys and established that the flash signature is verified **by the ECU
> bootloader, not by ODIS**. See *How integrity is enforced* below.

Scanned **19,289 FRFs** across Audi / VW / Seat / Skoda / Bentley, collapsed to **504
distinct functional module types** (the 6-digit part code; 279 shared across brands). One
representative FRF per type, classified by parsing each `<SECURITY>` block's
`<SECURITY-METHOD>` name: a flash **signature** (any method beginning with `SIG`, or an
ECDSA method, with a non-empty `<FW-SIGNATURE>` — unforgeable) versus only a **checksum**
(`CRC*` / `CCITT` / `BYTESUM` / `PROGHASH` — recomputable).

## Headline

| Integrity | Count | Share | Flashability |
|---|---|---|---|
| **Signed** (RSA-1024 176 · RSA-3072 45 · ECDSA 1) | **222** | ~44 % | **walled** — the ECU verifies the signature; no valid block without the private key |
| **Checksum-only** | **245** | ~49 % | **OBD-flashable** — recompute the checksum + flash |
| unsigned supplier build (`SIG` method, empty signature) | 2 | — | signed-class module; this representative container is unsigned |
| parse error (bootloader-only / multi-part / special) | 35 | ~7 % | unclassified |

Module types split almost evenly: **~49 % carry only a recomputable checksum** (for those,
modified-firmware flash is tractable over the OBD port — recompute and flash), **~44 %
carry an unforgeable signature.** Two things the first pass got wrong:

1. **Signed modules are a far larger share than first reported (~44 %, not ~11 %).** The
   earlier number came from a narrow string match on `SIG_SHA1-RSA1024`, which missed the
   older `SIG_S`/`SIG_E` naming (**117 modules** — the single biggest miss), the 384-byte
   `SIG_SHA256-RSA3072` set, ECDSA, and the `_with_CRC32` variants.
2. **A signed module cannot be OBD-flashed with modified firmware at all** — not "unless
   the ECU doesn't verify it." The signature is an ECU-side check (below), so the only
   routes are the private key (not obtainable) or a bootloader that fails to verify (must
   be proven per module, never assumed).

## How integrity is enforced (corrected)

- **ODIS does not verify the signature — it transmits it.** The Elektrobit flash job
  (`MCD3_FlashJobUDS`) drops the `FW-SIGNATURE` into the CheckMemory request
  (`31 01 02 02 … <signature>`) and sends it to the ECU. The four ODIS job libraries
  contain no hash or signature API — a signature and a CRC occupy the *same* operand slot
  of the same ECU routine.
- **The signature is a real RSA signature in the form a bootloader checks**, recovered
  from the signatures alone: RSA-1024, **e = 3**, PKCS#1 v1.5,
  `SHA1( BE32(block#) ‖ BE32(len) ‖ data )` (the 384-byte set is SHA-256 / RSA-3072).
  Every signature tested verifies.
- **Keys are per ECU project.** Gateway, HVAC, airbag, C7 display and the cluster family
  each use a different modulus. ODIS holds none of them; the matching public key can only
  live in each ECU's bootloader. The RAM flash-driver block is signed too (it is
  authenticated before being run from RAM). The gateway binary even ships signed twice —
  a series key (`_S`) and a development key (`_E`) — and the ECU reports which key type it
  holds. None of that has any purpose unless the ECU checks.
- **Open caveat:** it is not yet *proven on a unit* that a given bootloader rejects a bad
  signature — the bootloaders sit below the flashable region and are in no FRF, so they
  haven't been read. Everything found points to enforcement and nothing points the other
  way. **Plan on enforcement;** a non-destructive on-car test to settle it per module is in
  the [LEAR/V850 signature findings](flash-signature-enforcement.md).

## The signed set

The signed modules are the security / high-value / safety / EV cluster. Grouped by
function (representative, not exhaustive — see the CSV for the full per-code list and the
detected method names):

- **Gateways & immobiliser core:** 907468 (LEAR gw — **signed and ECU-enforced**),
  907530/566/590 (Continental MQB gw), 907063 (BCM1), 907064 (Kessy/BCM2), 907547.
- **Infotainment (modern MIB2/MIB3):** 035503, 035869/872/875/887/888 (035885 is a
  signed-class project whose representative FRF is an unsigned build).
- **Clusters / displays:** 919158, **919604**, **919617** (C7/D4 — were wrongly in the
  checksum column), 920770/862/863/962/963 (Virtual Cockpit).
- **Driver-assist / radar / airbag-gen:** 1EB/1EA959392/592/593/596/597/655/793.
- **Lighting:** 941472, 941572; 2Q937087/090/185/187/189/190.
- **EV high-voltage:** 915115/215/225/250/253, 971675; 915692 (ECDSA).
- **Camera / night-vision & misc high-value:** 980945, 980946, 927755 (TCU),
  907660/669/777/338/427/156/180, Bentley 423051 / 501605.
- **The set the broad re-check moved out of "checksum"** is chiefly older `SIG_S`/`SIG_E`
  RSA-1024 modules and the 384-byte `SIG_SHA256-RSA3072` set (e.g. **8X0920930**). The CSV
  marks each with its method.

Everything else — most comfort, body, doors, **seats**, engine/trans, chassis/ABS, TPMS,
older radios — is **checksum-only**.

## What it means for a repair

- **Checksum-only module → OBD-flashable.** Programming session → SA2 → download the
  patched block → recompute the checksum at CheckMemory → reset. No bench.
- **Signed module → walled.** You need the module's private key (not obtainable) or a
  bootloader that doesn't actually verify (prove it per module with the non-destructive
  test; don't assume it). Practical route: bench serial-ISP, or run the oracle test first.
- **Worked example (owner repair, C7 A6).** Seats **959760** are checksum-only — confirmed
  by direct inspection of the `.sgo` (no signature markers at all) and by this FRF pass
  (`CRC32`) — so they are OBD-flashable. HVAC **820043** is RSA-signed with its own key →
  walled; bench or the oracle test.

## Classifying correctly (so this isn't repeated)

- **signed** = any `<SECURITY-METHOD>` starting with `SIG` — `SIG_S`/`SIG_E` (older
  128-byte RSA-1024), `SIG_SHA1-RSA1024_*`, `SIG_SHA256-RSA3072_*` (384-byte),
  `SIG_*_with_CRC32_*`, `SIG_SHA256-ECDSA_*` — **with a non-empty `<FW-SIGNATURE>`**.
- The `<SECURITY-METHOD>` carries a `TYPE="…"` attribute, and the **SA2 unlock block also
  populates `<FW-SIGNATURE>`** (with the seed/key bytecode) — so a global "has a
  FW-SIGNATURE" test is meaningless. Parse per `<SECURITY>` block and key off the method
  *name*, and match `SIG_S`/`SIG_E` exactly (they are a prefix of `SIG_SHA*`).
- A non-`_S` FRF can be an **unsigned supplier build** (`SIG` slot present, empty
  signature), e.g. `FL_4H0907468E__0204.frf` — not a signed container.
- **checksum-only** = `CRC32` / `CRC` / `CRC32ADLATUS` / `CCITT` / `BYTESUM16` /
  `IVD_PROGHASH_SHA256` with no signature.

## Caveats

- 35 types failed to parse (bootloader-only `_BTLD`, multi-part `PART1`, `VWSAFE`/`TMALL`
  special containers) — unclassified, not necessarily signed.
- Two representatives (035885, 980946) are unsigned supplier builds — the module's project
  uses signing but the picked container has an empty signature; counted on their own line,
  not as checksum-only.
- `.sgo`-distributed modules (some seats / amps) aren't in this FRF pass; checked directly
  where relevant (seat `.sgo` = no signature).
- Full data: `vag-rsa-vs-crc-flash-map.csv` — `code, module, integrity, platforms,
  odx_bytes, methods, rep_frf`. The **`methods`** column lists each representative's
  detected `SECURITY-METHOD` names, so the classification is independently checkable.
- Signing-scheme detail, the recovered moduli, and the on-car enforcement test: see the
  [LEAR/V850 flash-signature findings](flash-signature-enforcement.md).
