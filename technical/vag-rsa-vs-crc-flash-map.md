# VAG flash-integrity map — RSA-signed vs CRC-only (Oct-2024 FlashDaten)

Scanned **19,289 FRFs** across Audi / VW / Seat / Skoda / Bentley, collapsed to **504
distinct functional module types** (the 6-digit part code; 279 are shared across brands).
One representative FRF per type, classified by whether its ODX carries a
`SIG_SHA1-RSA1024` per-block signature (RSA, unforgeable) or only a CRC/checksum.

## Headline

| Integrity | Count | Share | Flashability |
|---|---|---|---|
| **RSA-signed** | **52** | ~11 % | needs the ECU bootloader to NOT verify, or bench/exploit |
| **CRC / no-marker** | **417** | ~83 % | **OBD-flashable — recompute checksum + flash** |
| unclassified (parse err) | 35 | ~7 % | bootloader-only / multi-part / special FRFs |

**The vast majority of VAG modules are CRC-only — i.e. modified-firmware flash is tractable
over the OBD port.** RSA is the exception, reserved for a predictable core.

## The RSA-signed core (the ~11 % that are walled)

Grouped by function — it's exactly the security / high-value / safety / EV cluster:

- **Gateways & immobiliser core:** 907468 (LEAR gw), 907530/566/590 (Continental MQB gw),
  907063 (BCM1), 907064 (Kessy/BCM2), 907547 — the anti-theft spine.
- **Infotainment (modern MIB2/MIB3):** 035503, 035869/872/875/885/887/888. (Older/simpler
  radios — 035008/284/285/286/335/456/466/729/730 — are CRC.)
- **Digital clusters / displays:** 919158 (C7 display), 920770/862/863/962/963 (Virtual
  Cockpit).
- **Driver-assist sensors / radar / airbag-gen:** 1EB959392/592/593/596/597/793.
- **Matrix / laser headlights:** 941472, 941572; onboard-supply/lighting 2Q937087/090/185/
  187/189/190.
- **EV high-voltage:** 915115/215/225/250/253 (battery mgmt), 971675 (charger, Taycan).
- **Misc high-value:** 927755 (TCU), 980946 (camera/night-vision), 907660/669/777/338/427/
  156/180, Bentley 423051 / 501605.

Everything NOT in that list — comfort, body, doors, seats, most HVAC-adjacent, engine/trans,
chassis/ABS, TPMS, older radios — is **CRC-only**.

## What it means for your repair

- **Seats (959760) → CRC**, confirmed twice (the C7 `.sgo` directly, and a Q7 `4M0` FRF
  here). OBD-flashable: programming session → SA2 → download patched block → recompute the
  checksum. No bench.
- **HVAC (820043) → RSA** — the one genuine wall of your three. Bench (serial-ISP), or the
  make-or-break test: does the HVAC bootloader actually verify the ODX signature, or only
  ODIS? If only ODIS, a DIY UDS flasher pushes the CRC-valid patched block over OBD too.

## Important nuance — ODX-side vs ECU-side

"RSA" here means the **ODX (ODIS-side) signature** is present — the gate *ODIS* enforces.
Whether the **ECU bootloader** re-verifies it is a separate question. Your LEAR gateway app
(907468) is flagged RSA by its ODX, yet its block1 code contains **no RSA/SHA verifier** —
suggesting the ECU may not enforce it, so a DIY flasher that skips the ODIS RSA step could
still work. The CRC modules have *no* signature at all, so they're cleanly OBD-flashable;
the RSA ones are "flash if the ECU doesn't check."

## Caveats

- 35 types failed to parse (bootloader-only `_BTLD`, multi-part `PART1`, `VWSAFE`/`TMALL`
  special containers) — unclassified, not necessarily RSA.
- `.sgo`-distributed modules (some seats/amps) aren't in this FRF pass; checked directly
  where relevant (seat `.sgo` = no RSA).
- Full data: `vag_rsa_map.csv` (code, module, integrity, platforms, odx_bytes, rep_frf).
