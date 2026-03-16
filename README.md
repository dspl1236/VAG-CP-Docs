# VAG-CP-Docs

**A public documentation project tracking Volkswagen Group's "Component Protection" system and its impact on vehicle owners' right to repair.**

[![License: CC0](https://img.shields.io/badge/License-CC0%201.0-lightgrey.svg)](https://creativecommons.org/publicdomain/zero/1.0/)
[![Contributions Welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg)](CONTRIBUTING.md)

---

## What is Component Protection?

Component Protection (Komponentenschutz) is a software-based parts-locking system used across all Volkswagen Group brands — **Volkswagen, Audi, Porsche, Bentley, Lamborghini, Škoda, SEAT, and Cupra**. It cryptographically binds electronic modules to a specific vehicle using the manufacturer's FAZIT database server. When a replacement part is installed — even a perfectly functional, identical salvage part — it will not work until a Volkswagen Group dealer uses proprietary ODIS software to perform an online authentication with VW's GEKO/GRP servers.

**The result:** A $200 salvage part requires an additional $300–$500 dealer fee just to be activated, making independent repair and DIY replacement economically unviable.

### Affected modules include (but are not limited to):
- Infotainment / MMI head units
- Instrument clusters
- Climate control panels (Climatronic)
- Seat memory and adjustment modules
- Steering column electronics (KESSY)
- ABS/ESC modules
- Adaptive cruise control radar
- Gateway / CAN bus modules

### Affected vehicles:
All VW Group vehicles approximately model year 2003–present, with Generation 3 (2016+) having **no offline workaround**.

---

## Why does this matter?

### 1. The GEKO server is in transition — and vehicles may be permanently bricked

Component Protection requires a live connection to VW's backend servers (GEKO/GRP/FAZIT) to activate any replacement module. VW is migrating this infrastructure, and third-party access has been increasingly unreliable since mid-2024. **VW has made no public commitment to long-term server availability for older vehicles.** If servers go offline without an offline alternative, millions of vehicles become permanently unserviceable with used parts.

### 2. The system triggers without any parts being changed

VW's own NHTSA Technical Service Bulletin (TSB 91-18-13TT, November 2018) acknowledges Component Protection activates spontaneously — after battery disconnections, low voltage events, or completely without cause. Legitimate owners face dealer fees for a lockout they did nothing to trigger.

### 3. It destroys the used parts market and locks out independent repair

The Motor & Equipment Manufacturers Association (MEMA) estimates 70% of post-warranty repairs involve independent shops. Component Protection bypasses those shops entirely — only franchised dealers with active ODIS server credentials can perform unlocks, creating a captive repair market by software design.

### 4. It may violate consumer protection law

Oregon's SB 1596 (2024) and Colorado's HB 1121 (effective 2026) ban software-based parts pairing. The FTC has identified software locks as anticompetitive. The federal REPAIR Act would prohibit the dealer-exclusive diagnostic access that Component Protection requires. While no lawsuit has yet targeted VW's system directly, the legal landscape is closing in.

---

## Repository structure

```
VAG-CP-Docs/
├── README.md                  ← You are here
├── CONTRIBUTING.md            ← How to submit cases and documentation
├── technical/                 ← How the system works (public/documented info only)
│   ├── overview.md
│   ├── generations.md                           ← Gen 1 / Gen 2 / Gen 3 / SFD / SFD2
│   ├── geko-server.md                           ← GEKO/GRP/FAZIT infrastructure
│   ├── affected-modules.md                      ← Comprehensive module list by platform
│   ├── affected-vehicles.md                     ← Model/year matrix
│   ├── au57x-mcd-project-findings.md            ← C7 A6/A7 ODIS service names (string extraction)
│   ├── au57x-mwb-extraction-confirmed-dids.md   ← Confirmed DID addresses (Linux MWB extraction)
│   └── j533-constellation-deep-dive.md          ← J533 hardware, constellation protocol, part numbers
├── cases/                     ← Documented consumer impact stories
│   ├── README.md              ← How to submit a case
│   ├── template.md            ← Case submission template
│   └── [cases go here]
├── legal/                     ← Legislation, rulings, DMCA exemptions, FTC actions
│   ├── federal-repair-act.md
│   ├── massachusetts-rtr.md
│   ├── oregon-sb1596.md
│   ├── colorado-hb1121.md
│   ├── dmca-exemptions.md
│   ├── ftc-nixing-the-fix.md
│   └── nhtsa-tsb-91-18-13tt.md
├── media/                     ← Press coverage and links
│   └── press-coverage.md
├── advocacy/                  ← How to take action
│   ├── how-to-file-ftc-complaint.md
│   ├── how-to-contact-ifixit.md
│   ├── how-to-contact-legislators.md
│   └── organizations.md
└── contributing/
    └── style-guide.md
```

---

## The GEKO sunset — why time matters

VW is actively migrating its backend authentication infrastructure from the legacy GEKO system to GRP (Group Retail Portal). ODIS 25.x uses GRP; ODIS 23.x and earlier use GEKO. During a transition period both coexist, but since mid-2024:

- Multiple third-party ODIS providers report GEKO IMMO/CP functions intermittently or persistently offline
- VW's erWin consumer webshop closed to consumers as of **December 18, 2025**
- No official timeline for GEKO end-of-life has been published
- No offline unlock mechanism has been announced for any Generation 3 vehicle

**For 2016+ vehicles, server dependency is absolute.** When servers go dark, those modules go permanently locked.

This is the core urgency of this documentation project.

---

## How to contribute

We welcome:
- **Consumer harm cases** — Your story matters. See `cases/template.md`
- **Technical documentation** — Publicly known information about how the system works
- **Legal research** — Legislation, rulings, regulatory filings
- **Press coverage** — News articles, forum posts, videos documenting the issue
- **Translations** — This affects VW Group owners worldwide

See [CONTRIBUTING.md](CONTRIBUTING.md) for full guidelines.

---

## How to use this documentation

This repository is designed to be a resource for:

- **Vehicle owners** dealing with Component Protection right now
- **Independent repair shops** affected by dealer-exclusive access
- **Journalists** covering right to repair and automotive software
- **Advocates and legislators** building the case for reform
- **Organizations like iFixit, Repair Association, EFF, PIRG, Auto Care Association** seeking documented evidence of consumer harm

All content in this repository is released under **CC0 (public domain)**. Use it freely, without restriction, for any purpose including advocacy, journalism, and legislative testimony.

---

## Key contacts for advocacy

| Organization | Focus | Contact |
|---|---|---|
| iFixit | Right to repair advocacy | https://www.ifixit.com/News |
| The Repair Association | Legislative advocacy | https://www.repair.org |
| Auto Care Association | Industry / legislative | https://www.autocare.org |
| U.S. PIRG | Consumer advocacy | https://pirg.org/right-to-repair |
| Electronic Frontier Foundation | Digital rights / DMCA | https://www.eff.org |
| FTC Complaint Portal | File a complaint | https://reportfraud.ftc.gov |

---

## Disclaimer

This repository documents publicly available information for advocacy and educational purposes. It does not contain instructions for circumventing Component Protection or any other security system. Contributors and maintainers do not endorse any activity that may violate applicable law.

---

*Started March 2025. All contributions welcome.*
