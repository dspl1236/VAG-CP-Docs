# VAG-CP-Docs

**A public documentation project tracking Volkswagen Group's "Component Protection"
system and its impact on vehicle owners, independent repair shops, and the used
parts ecosystem.**

[![License: CC0](https://img.shields.io/badge/License-CC0%201.0-lightgrey.svg)](https://creativecommons.org/publicdomain/zero/1.0/)
[![Contributions Welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg)](CONTRIBUTING.md)

---

## Why this exists

A salvage yard has a $150 Audi climate control unit. It's identical to yours.
It's in perfect condition. It won't work in your car without a dealer unlocking
it with proprietary software connected to a manufacturer server. That server is
currently offline with no return date. The dealer wants $400 to fix a problem
you didn't create, using a system you had no say in, for a part that has
no electrical or mechanical reason not to work.

This happens every day, to ordinary people, across every Volkswagen Group brand.

Independent shops can't fix it. The used part market is economically broken by
it. Older vehicles are being scrapped because of it. And right now, with VW's
GEKO server offline, even dealers cannot fix it.

This repository documents how the system works, what it costs ordinary people,
and what the law says about it. Everything here is in the public domain — use
it freely for advocacy, journalism, legislation, or repair.

---

## Legal basis and nature of this research

This repository contains **documentation of publicly observable vehicle diagnostic
behavior** — protocol sequences visible on a vehicle communication bus that the
vehicle owner has lawful access to, analysis of publicly available standards and
service information, and records of documented consumer harm.

**This research is protected by:**

- **DMCA Section 1201 exemption** — The Copyright Office's triennial rulemaking
  explicitly exempts "diagnosis, maintenance, or repair" of motor vehicles from
  circumvention prohibitions. This exemption has been renewed and expanded in
  every review cycle since 2015. Documenting how diagnostic protocols work for
  repair purposes is the paradigm case the exemption was written for.

- **First Amendment** — Documentation of how a system works, including technical
  detail, is protected speech. Courts have consistently held that publishing
  information about security systems for purposes of education, research, and
  advocacy is protected even when the information is detailed.

- **Oregon SB 1596 (2024) and Colorado HB 1121 (2026)** — Both states have
  enacted laws prohibiting software-based parts pairing of the kind Component
  Protection implements. This documentation supports compliance and enforcement.

- **Fair use / public interest** — This repository does not reproduce VW's
  proprietary software or extract trade secrets. It documents the behavior of
  protocols on vehicles owned by the contributors.

**This repository does not contain:**
- VW proprietary software, firmware, or source code
- Instructions for bypassing any security system for unauthorized purposes
- Any material extracted through unauthorized access
- Vehicle-identifying information that could be used to target individual vehicles

**What "publicly observable behavior" means:** When a diagnostic tool
communicates with a module over a CAN bus in a vehicle you own, every byte
exchanged is observable by the vehicle's owner. Documenting what those bytes
mean is analogous to documenting how a light switch works by observing that
flipping it turns a light on. The observation is lawful; the documentation
is protected.

---

## Why CC0?

All content is released under **CC0 1.0 Universal (Public Domain Dedication).**

CC0 was chosen deliberately over GPL, Creative Commons Attribution, or other
licenses for specific reasons:

**For repair shops and technicians:** You can incorporate this documentation into
repair manuals, training materials, and service guides without license tracking
or attribution requirements. If this information helps you fix a car, use it.

**For journalists and advocates:** You can quote, excerpt, and reproduce without
permission requests. If this documentation supports a story about right to repair,
publish it.

**For legislators and regulators:** You can introduce this as evidence in hearings,
incorporate it into regulatory filings, and cite it in legislation without
navigating intellectual property questions.

**For other researchers:** You can build on this work, correct it, extend it to
other platforms, and publish your results without license compatibility concerns.

The goal is maximum impact for vehicle owners and the repair ecosystem. License
friction is the enemy of that goal. CC0 eliminates it entirely.

---

## What is Component Protection?

Component Protection (Komponentenschutz) is a software-based parts-locking system
used across all Volkswagen Group brands — **Volkswagen, Audi, Porsche, Bentley,
Lamborghini, Škoda, SEAT, Cupra, and MAN**. It cryptographically binds electronic
modules to a specific vehicle using the manufacturer's FAZIT database server.

When a replacement part is installed — even a perfectly functional, identical
salvage part — it will not work until a Volkswagen Group dealer uses proprietary
ODIS software to perform an online authentication with VW's GEKO/GRP servers.

**The GEKO Immobilizer/Component Protection endpoint has been offline since at
least early 2026 with no estimated return date for any VAG brand.**

The result: A $200 salvage part requires an additional $300–$500 dealer fee just
to be activated — when it can be activated at all. Independent repair is
effectively impossible. The used parts market for affected modules is broken.

### Affected modules include (but are not limited to)
- Infotainment / MMI head units
- Instrument clusters
- Climate control panels (Climatronic)
- Seat memory and adjustment modules
- Steering column electronics (KESSY)
- ABS/ESC modules
- Adaptive cruise control radar
- Gateway / CAN bus modules (J533)

### Affected vehicles
All VW Group vehicles approximately model year 2003–present, with Generation 3
(2016+) having **no offline workaround even in principle.**

---

## Why this matters — the human cost

**For the independent shop:** Component Protection routes warranty and repair work
exclusively to dealers. The Motor & Equipment Manufacturers Association (MEMA)
estimates 70% of post-warranty repairs happen at independent shops. CP systematically
excludes them from a growing share of that work — not because they lack skill or
equipment, but because they lack access to a manufacturer server.

**For the vehicle owner:** VW's own NHTSA Technical Service Bulletin (TSB 91-18-13TT,
November 2018) acknowledges Component Protection activates spontaneously after battery
disconnections, low voltage events, or without cause. Owners face dealer fees for a
lockout they didn't trigger.

**For the environment:** A functional used part that can't be activated becomes
landfill. Component Protection converts working components into waste. Every module
scrapped because it can't be unlocked is a manufacturing and environmental cost
that didn't need to happen.

**For the used parts ecosystem:** Salvage yards, remanufacturers, and parts
resellers cannot reliably sell affected modules for their primary purpose.
The economic damage radiates through the entire aftermarket supply chain.

**For long-term vehicle ownership:** When manufacturer servers eventually go
offline — as they always do — vehicles with locked modules become permanently
unserviceable for common repairs. CP transfers the lifespan decision for a vehicle
from the owner to the manufacturer's server infrastructure.

---

## Why does this matter?

### 1. The GEKO server is offline — vehicles cannot be serviced

Component Protection requires a live connection to VW's backend servers to activate
any replacement module. The GEKO Immobilizer/CP endpoint has been offline since at
least early 2026 with no ETA. **With GEKO down, even dealers cannot clear CP.**
This is not a temporary inconvenience — it is a preview of permanent server sunset.

### 2. The system triggers without any parts being changed

VW's TSB 91-18-13TT (2018) acknowledges spontaneous CP activation. Legitimate
owners face dealer fees for a lockout they didn't cause.

### 3. It destroys the used parts market and locks out independent repair

Only franchised dealers with active ODIS server credentials can perform unlocks,
creating a captive repair market by software design.

### 4. It may violate consumer protection law

Oregon's SB 1596 (2024) and Colorado's HB 1121 (effective 2026) ban software-based
parts pairing. The FTC has identified software locks as anticompetitive. The federal
REPAIR Act would prohibit the dealer-exclusive access CP requires.

---

## Repository structure

```
VAG-CP-Docs/
├── README.md                  ← You are here
├── CONTRIBUTING.md            ← How to submit cases and documentation
├── technical/                 ← How the system works (publicly observable behavior)
│   ├── overview.md                              ← Start here
│   ├── cp-system-explained.md                   ← Plain-language explanation
│   ├── community-research-eeprom.md             ← EEPROM approaches and analysis
│   ├── j533-constellation-deep-dive.md          ← J533 hardware, protocol, part numbers
│   ├── au57x-mcd-project-findings.md            ← C7 A6/A7 ODIS service names
│   ├── au57x-mwb-extraction-confirmed-dids.md   ← Confirmed DID addresses
│   ├── odis-cp-session-wauggafc7dn120188.md     ← Real ODIS session analysis (VIN redacted)
│   ├── geko-pki-analysis.md                     ← VW CA Root and GEKO TLS certificates
│   ├── odis-versions-lear-temic-chip-map.md     ← ODIS versions and gateway hardware
│   └── cp-master-module.md                      ← J533 as CP master — how enforcement works
├── cases/                     ← Documented consumer impact stories
│   └── 2025-audi-a6-c7-hvac-seats.md
├── advocacy/                  ← How to take action
│   └── how-to-take-action.md
└── media/
    └── press-coverage.md
```

---

## The GEKO sunset — why time matters

VW is migrating backend authentication from legacy GEKO to GRP (Group Retail Portal).
ODIS 25.x uses GRP; ODIS 23.x and earlier use GEKO. The GEKO Immobilizer/CP endpoint
is currently offline for all brands with no ETA.

**For 2016+ vehicles, server dependency is absolute.** When servers go dark
permanently, those modules go permanently locked.

This is the core urgency of this documentation project.

---

## How to contribute

We welcome:
- **Consumer harm cases** — Your story matters. See `cases/template.md`
- **Technical documentation** — Publicly observable behavior and public information
- **Legal research** — Legislation, rulings, regulatory filings
- **Press coverage** — News articles documenting the issue
- **Translations** — This affects VW Group owners worldwide

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

---

## How to use this documentation

This repository is a resource for:

- **Vehicle owners** dealing with Component Protection right now
- **Independent repair shops** affected by dealer-exclusive access
- **Journalists** covering right to repair and automotive software
- **Advocates and legislators** building the case for reform
- **Organizations** like iFixit, Repair Association, EFF, PIRG, Auto Care Association

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

This repository documents publicly observable vehicle diagnostic behavior and
public information for consumer protection, advocacy, and educational purposes
under the DMCA Section 1201 motor vehicle repair exemption and applicable
right-to-repair law.

It does not contain proprietary software, firmware, trade secrets, or instructions
for accessing any system without authorization. All diagnostic observations were
made on vehicles owned by the contributors using equipment they are lawfully
entitled to use.

Contributors and maintainers do not endorse any activity that violates applicable law.

---

*Documentation project for vehicle owner rights. All content CC0 — public domain.
Use freely for repair, advocacy, journalism, and legislation.*
