# Audi C7 (4G) — which modules an owner can restore, and why

*A companion to the [flash-integrity map](vag-rsa-vs-crc-flash-map.md) and the
[C7 salvage case](../cases/2025-audi-a6-c7-hvac-seats.md), narrowed to the modules an owner
actually faces when fitting identical, self-owned salvage parts to a 2012–2018 Audi A6/A7
(4G platform). For each module: **is it even Component-Protected, how is its flash sealed, and
what does that leave an owner able to do** — over the diagnostic (OBD) port, or only on a
bench. Mechanism only; this page names no keys, offsets, or tools.*

---

## The decision that puts a module in a bucket

Two independent questions decide an owner's options, in order:

1. **Does the module enforce CP *locally* (does it limp itself)?** A module that has no local
   CP engine cannot limp — a swap just needs coding/adaptation, not a firmware change.
2. **If it does limp, how is its flash sealed?** A **recomputable checksum** (`CRC*`) can be
   re-sealed after a one-byte limp-gate patch and flashed over OBD; an **unforgeable
   signature** (`SIG_*`, ECU-enforced — see
   [flash-signature-enforcement.md](flash-signature-enforcement.md)) cannot, and forces a
   bench job.

A quick, non-invasive way to answer (1): **search the module's firmware image for the AES
(Rijndael) S-box** — the fixed 256-byte table beginning `63 7c 77 7b f2 6b 6f c5 …`. A module
that runs a local CP challenge has this table; one that does not, does not. (Seat image: present.
Door image: absent.)

## The map

| Module (C7) | Part | Local CP? | Flash seal | Owner's route | Status |
|---|---|---|---|---|---|
| **Seat memory** (J136 / J521) | `4x0959760` | Yes (AES) | **CRC32** | **OBD limp-patch** | **Working** — see [seat-obd-flash.md](seat-obd-flash.md) |
| **Climatronic** (J255) | `4G0820043` | Yes (AES) | **RSA-1024, ECU-enforced** | Bench / oracle test | **Walled** — see [flash-signature-enforcement.md](flash-signature-enforcement.md) |
| **Doors** | `4G8959792/3/5` | **No** | n/a | Coding / adaptation only | No firmware work needed |
| **Amplifier** (Bose) | `…035223` | Gateway-mediated | n/a (no local gate) | — | **Not a patch target** (below) |
| **Transmission** (ZF 8HP) | `4G0927158` | — | **CRC** | OBD re-flashable | Checksum-only |

## The non-obvious findings

### Doors are not Component-Protected

The C7 door modules (`4G8959792/3/5`) contain **no AES engine and no CP strings** — the S-box
test above comes back empty. There is no local limp gate because there is nothing enforcing CP
inside the door. A swapped door controller needs only **coding/adaptation** to the car; it
never needs a firmware patch. This is worth stating because owners (and some forum guidance)
assume every swapped module is a CP problem — for the doors it simply is not.

### The amplifier is the wrong shape for a limp-patch

The Bose amplifier's Component Protection is **not** a local verify-and-limp like the seat or
Climatronic. The amp is an **Infineon C166/ST10**, and its CP is a **gateway-mediated
challenge–response**: the gateway sends a challenge, the amp encrypts it and **returns the
response**, and the gateway decides. Crucially:

- There is **no local "authenticated?" state byte** that the audio path reads before it will
  play. The amp answers the gateway; it does not gate *itself* on a stored CP result.
- So there is **no flip-able gate** of the seat kind — nothing in the amp's own firmware whose
  default you could change to make it behave as "authenticated."

On top of that, the specific C7 part (`4G0035223`) **is not present in the flashdaten** at all.
Between the two, the amp is **ruled out** as a seat-style OBD limp-patch target — not because
it is strongly protected, but because its CP lives in the conversation with the gateway, not in
a local decision you can reach. (The gateway side of that conversation is covered in
[cp-master-module.md](cp-master-module.md) and [gateway-firmware-analysis.md](gateway-firmware-analysis.md).)

### Transmission TCU — a correction

The C7 ZF 8HP transmission controller is **`4G0927158`**, and it is **checksum-only**
(re-flashable). An earlier note labelled the C7 TCU as RSA-signed; that confused it with
**`927755`**, which is a *different, newer* 4K0-generation module. The C7 part is on the
checksum side of the map.

## How to use this

- If your swapped C7 module is **not in the table**, run the two questions at the top: find the
  module's image in the flashdaten, test for the S-box to see whether it limps locally, and
  read its `<SECURITY-METHOD>` in the ODX (`CRC*` = checksum-only, `SIG_*` = signed) using the
  method in the [flash-integrity map](vag-rsa-vs-crc-flash-map.md).
- **Checksum-only + local CP** → the seat workflow applies (OBD limp-patch).
- **Signed + local CP** → the Climatronic situation (bench, or the non-destructive oracle test
  first).
- **No local CP** → coding/adaptation only; no firmware work.

---

## Related

- [`vag-rsa-vs-crc-flash-map.md`](vag-rsa-vs-crc-flash-map.md) — the full signed-vs-checksum
  survey across the VAG range.
- [`flash-signature-enforcement.md`](flash-signature-enforcement.md) — why a signed module is
  walled and how to test a bootloader's strictness non-destructively.
- [`seat-obd-flash.md`](seat-obd-flash.md) — the checksum-only seat, worked end to end.
- [`../cases/2025-audi-a6-c7-hvac-seats.md`](../cases/2025-audi-a6-c7-hvac-seats.md) — the
  owner case behind this map.

---

*Owner's-own-module research for a 4G A6/A7, per module, reversible, transferring no credential.
Mechanism and classification only — no keys, offsets, or tools. CC0 — public domain.*
