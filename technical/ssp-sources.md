# SSP Source Documents

This folder indexes all official Volkswagen Group Self-Study Programme (SSP) documents that contain information about Component Protection, the GEKO/FAZIT server system, and related electrical architecture for affected platforms.

SSPs are VW Group's official dealer training documents. They represent the most authoritative public source of information about how these systems work.

---

## Documents Confirmed to Contain Component Protection Content

### SSP 294 — VAS 5051 On-line Link (2002)
**The founding document for Component Protection.**
- Introduces Immobilizer 4 and Component Protection simultaneously with the Audi A8 D3 ('03)
- Describes the FAZIT central database at Audi AG Ingolstadt
- States modules "cannot be adapted without an on-line link to FAZIT"
- Explicitly states functions are restricted even when modules are "swapped between two vehicles on a trial basis"
- **Direct PDF:** http://www.volkspage.net/technik/ssp/ssp/SSP_294.pdf *(volkspage.net may be intermittently unavailable)*
- **Mirror:** Search "SSP 294 VAS 5051" on cardiagn.com or procarmanuals.com

---

### SSP 326 — Audi A6 '05 Electrics (2005)
**Component Protection section at page 30. Covers seat adjustment modules specifically.**
- Dedicated "Immobiliser and Component Protection" section
- Covers J136 (Memory Seat/Steering Column) and J521 (Passenger Memory Seat) — directly relevant to 2013 A6
- Documents seat module PIN-code position locking (separate from CP)
- **Direct PDF:** http://www.volkspage.net/technik/ssp/ssp/SSP_326.pdf
- **Mirror:** en.audiclub.eu (search SSP 326)

---

### SSP 994403 — 2005 Audi A6 Electrical System (US market)
**Most detailed English-language CP description found. Critical advocacy document.**
- Pages 26-28: Complete Component Protection section
- Explicitly states: *"The Data Bus On Board Diagnostic Interface J533 is integrated into the 'Component protection' function for the first time"*
- Explicitly states J518 is *"master for immobilizer and component protection"*
- Explicitly states J525 (Digital Sound System) *"participates in component protection (Geko)"*
- States: *"all components must be adapted on-line... cannot be adapted without an on-line link to FAZIT"*
- **Direct PDF:** https://content.datarunners.net/content/SSP/994403.pdf
- **Mirror:** vaglinks.com/Docs/SSP/

---

### SSP 486 — Audi A6 2011 / C7 Introduction
**Your vehicle's platform SSP. Contains full electrical topology.**
- Full topology diagram showing J533 as hub for all bus systems
- Lists all control modules by bus: J136, J521, J255, J393, J519, J527 all on Convenience CAN
- Shows J533 as "users of all bus systems (Gateway)"
- **PDF:** Available via vag-technique.fr (login required) and cardiagn.com

---

### SSP 990613 — 2012 Audi A6 Vehicle Introduction (US)
**US-market SSP for your exact vehicle (2013 A6 is same C7/4G platform).**
- Control module locations diagram (pages 30-31) shows J136, J521, J255, J533 positions
- Full topology diagram for C7 series
- Climate control overview showing J255 Climatronic on Convenience CAN
- **Direct PDF:** https://content.datarunners.net/content/SSP/990613.pdf

---

### SSP 481 — Audi A7 Sportback Onboard Power Supply and Networking
**Shares identical electrical architecture with A6 C7.**
- Full networking documentation for the C7 platform
- J533 gateway architecture detail
- **Available:** vag-technique.fr (login required)

---

### SSP 483 — Audi A7 Sportback Convenience Electronics
**Direct predecessor to A6 C7 convenience electronics documentation.**
- J136, J521 seat module documentation
- J393 central comfort module
- J255 Climatronic integration
- **Available:** vag-technique.fr (login required)

---

## Key SSP Download Repositories

| Source | URL | Notes |
|---|---|---|
| volkspage.net | http://www.volkspage.net/technik/ssp/ | Large collection, SSP 1-400+, may be intermittent |
| en.audiclub.eu | https://en.audiclub.eu | Good A6 C7 collection, direct downloads |
| datarunners.net | https://content.datarunners.net/content/SSP/ | US-market SSPs, reliable |
| cardiagn.com | https://cardiagn.com/vag-self-study-program-ssp/ | Large collection, July 2025 update |
| procarmanuals.com | https://procarmanuals.com/category/self-study-program/ | Free SSP PDFs |
| nininet.de | https://www.nininet.de/english/self-study-programs-(SSP).php | Complete SSP number index |
| vaglinks.com | https://www.vaglinks.com/Docs/SSP/ | US-market SSPs |

---

## How to Read SSPs for CP Information

When searching an SSP for Component Protection content, look for:
- "Component protection" or "Komponentenschutz"
- "GEKO" or "GeKo"
- "FAZIT"
- "J533" (the gateway — always involved)
- "on-line link" or "online adaptation"
- "Immobilizer 4" (CP was introduced alongside Immo 4)
- The topology diagram — modules shown on Convenience CAN are most likely CP-protected

The CP section in SSPs is almost always in the electrical/electronics chapter, often under "Access and Start Authorization" or "Immobilizer."
