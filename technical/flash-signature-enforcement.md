# How VAG flash signatures are enforced — the ECU checks, not ODIS (LEAR / V850 findings)

*This is the findings page the [flash-integrity map](vag-rsa-vs-crc-flash-map.md) points to for
the signing-scheme detail and the on-car enforcement test. It documents **how a signed VAG
module decides whether to accept a flash** — publicly observable behaviour on a vehicle the
owner has lawful access to, plus analysis of the signatures that ship inside publicly
distributed flash-update packages (FRFs). It contains **no private keys, no forged
signatures, and no flashing tool** — only the mechanism and the method for testing it.*

---

## Why this page exists

The flash-integrity map splits VAG modules into **checksum-only** (a recomputable `CRC*`,
so a modified image can be re-sealed and flashed over OBD) and **signed** (an unforgeable
`SIG_*` signature). That split only matters if you know **who checks the signature and how
strict the check is.** The short answer, established below:

- **ODIS does not verify the signature — it only transmits it.** The signature is checked by
  the **ECU's own bootloader.**
- The signature is a **real RSA signature in the form a bootloader checks**, so a signed
  module cannot be OBD-flashed with modified firmware unless either (a) you hold the per-ECU
  private key (not obtainable), or (b) that particular bootloader verifies *non-conformantly*
  — which must be **proven on a unit, never assumed.**

Everything here was derived from the publicly distributed flash packages and the diagnostic
behaviour of a module the contributor owns. The worked example is the Audi C7 Climatronic
(`4G0820043`, a LEAR/Continental unit on a Renesas V850) — the one genuinely signature-walled
module in the [C7 salvage case](../cases/2025-audi-a6-c7-hvac-seats.md).

---

## 1. ODIS transmits the signature; the ECU verifies it

The Elektrobit flash job that ODIS runs (`MCD3_FlashJobUDS`) does not contain any hash or
signature routine. What it does is read the `<FW-SIGNATURE>` out of the flash container and
drop it into the **CheckMemory** request sent to the module after the data is transferred:

```
31 01 02 02 … <signature>          RoutineControl(start, CheckMemory)
```

Two observations fix the architecture:

- **A signature and a CRC occupy the *same* operand slot of the *same* ECU routine.** A
  checksum-only module receives a `CRC` in that slot; a signed module receives a 128-byte
  (RSA-1024) or 384-byte (RSA-3072) signature in the same slot. ODIS treats both as opaque
  bytes to forward.
- **The four ODIS flash-job libraries contain no crypto API** — no hash, no modular
  exponentiation, no key store for these keys. ODIS *cannot* be the verifier; it has nothing
  to verify with.

The only place a public key can live, therefore, is inside each ECU's bootloader. The
corollary matters for repair: **re-sealing a modified block the way you would a checksum
module (recompute, flash) is impossible for a signed module** — there is no value you can
compute without the private key that the bootloader will accept.

## 2. The signing scheme (recovered from the signatures themselves)

The signatures that ship in the FRFs are consistent with a single, standard scheme, recovered
by analysing the signatures alone (no key extraction):

- **RSA-1024, public exponent `e = 3`, PKCS#1 v1.5**, over
  `SHA-1( BE32(block#) ‖ BE32(len) ‖ data )`, where `block#` is the block's
  `SOURCE-START-ADDRESS` and `len` its uncompressed length.
- The 384-byte set is the same construction with **SHA-256 / RSA-3072.**
- **Every stock signature tested verifies** under this construction.

Three structural facts confirm the keys are real and ECU-side, each pointless unless the ECU
checks:

1. **Keys are per ECU project.** The gateway, the Climatronic, the airbag controller, the C7
   display and the cluster family each sign under a *different* modulus. ODIS holds none of
   them.
2. **The RAM flash-driver block is signed too** — the small routine the bootloader loads into
   RAM to perform the erase/program is itself authenticated before it is allowed to run.
3. **The gateway binary ships signed twice** — once with a series key (`_S`) and once with a
   development key (`_E`) — and the ECU reports which key *type* it currently trusts. A tester
   has no use for two signatures over one image; a bootloader selecting which key to honour
   does.

### The public modulus is recoverable (and that's fine)

Because `e = 3` is small and the same key signs many packages, a module's **public** modulus
can be recovered by taking a GCD across the packages that share it — a standard, well-known
property of RSA signatures. A public modulus is a *verification* key: recovering it lets you
**check** signatures, not forge them. For the C7 Climatronic, the recovered modulus verifies
**both** stock signatures (the 724 KB application block and the 2 KB RAM driver block),
independently confirming the scheme above.

> The recovered modulus value itself is a public key and harmless, but it is held in the
> research set rather than printed here — consistent with this project's stated scope:
> mechanism and findings, not proprietary material or turnkey bypass tooling.

## 3. The one subtlety: `e = 3` and *non-conformant* verifiers

`e = 3` with PKCS#1 v1.5 is the classic setting for the **Bleichenbacher (2006) signature
forgery** — a 20-year-old, well-documented result. It does **not** break RSA and it does
**not** need the private key. Instead it exploits a verifier that is written *sloppily*:

- A **conformant** verifier reconstructs the full PKCS#1 v1.5 padding — the `00 01`, a run of
  **at least eight** `FF` bytes, the `00` separator, the DigestInfo, then the hash — and
  rejects anything that does not match **exactly, with no trailing bytes.** Against a
  conformant verifier the forgery **fails.**
- A **non-conformant** verifier that accepts a *short* `FF` run, or ignores whatever bytes
  follow the hash, can be fooled by a signature that is simply a cube root — computable by
  anyone, no key required.

So for any given signed module the real question is **not** "is it signed" (it is) but **"is
its bootloader's verify conformant."** That cannot be read off the FRF: the bootloaders sit
**below the flashable region** and are present in **no** flash package, so they have not been
dumped. Everything observed points to genuine enforcement; nothing points the other way. The
correct engineering posture is therefore: **plan on a strict verifier, and settle it per
module with the non-destructive test below** before assuming either outcome.

## 4. A non-destructive way to settle it on a bench unit

You can determine a bootloader's strictness **without risking the module** by exercising
CheckMemory against a **small, independently-erasable signed block** — on the C7 Climatronic
the 2 KB RAM driver block, which (unlike the application block) has **no erase step** in the
flash sequence, so the application is never touched. Compare how the module answers
CheckMemory across four controlled inputs:

| Input (for the small block) | If **accepted** | If **rejected** |
|---|---|---|
| stock data + stock signature | control — must accept | rig/addressing wrong, fix it |
| stock data + corrupted signature | the bootloader is **ignoring** the signature | it checks the signature at all |
| modified data + stock signature | it does not **bind** signature↔data | it binds them |
| modified data + conformant-forged signature | the verify is **non-conformant** (lenient) | the verify is **strict** |

Reading the four answers together places the bootloader in exactly one bucket —
**strict-verifying**, **lenient-verifying**, or **not-verifying** — in one bench session, with
nothing but the application-safe small block ever written. (The forged-signature input is the
Bleichenbacher cube root of §3; constructing it needs only the public modulus, and the
construction is deliberately **not** published here.)

## 5. What this means for a repair

- **Checksum-only module → OBD-flashable.** Recompute the checksum and flash over the
  diagnostic port. Worked example: the C7 [seat modules](seat-obd-flash.md).
- **Signed module → walled.** No value you can compute offline will pass a strict verifier.
  The realistic routes are, in order:
  1. The **non-destructive oracle test** above — on the chance the bootloader is lenient or
     not-verifying, which converts the repair to an OBD flash.
  2. A **bench serial-ISP** reflash (programming the microcontroller directly, below the OBD
     path) — the guaranteed route, and the reason a signed module is a *bench* job, not an
     OBD one. This is a high-skill, hardware-level intervention — disproportionate to undo a
     lock that (per the [gateway analysis](cp-master-module.md)) carries no anti-theft benefit
     for an owner reinstalling identical, self-owned parts.

That disproportion is the advocacy point: a checksum module and a signed module differ only in
how the manufacturer chose to seal the image, yet that single choice is the difference between
a ten-minute OBD repair and opening the module to reprogram its silicon.

---

## Related

- [`vag-rsa-vs-crc-flash-map.md`](vag-rsa-vs-crc-flash-map.md) — which modules are signed vs
  checksum-only across the VAG range (the data this page explains the enforcement for).
- [`seat-obd-flash.md`](seat-obd-flash.md) — the checksum-only counterpart: a C7 module that
  *is* OBD-flashable, worked end to end.
- [`c7-repairability-map.md`](c7-repairability-map.md) — which C7 modules fall on which side,
  and why.
- [`../cases/2025-audi-a6-c7-hvac-seats.md`](../cases/2025-audi-a6-c7-hvac-seats.md) — the
  owner case this work serves.

---

*Owner's-own-module research, per module, reversible, transferring no credential. Derived from
publicly distributed flash packages and the diagnostic behaviour of a contributor-owned
vehicle. No private keys, forged signatures, or flashing tools are published here. CC0 —
public domain.*
