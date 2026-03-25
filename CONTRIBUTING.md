# Contributing to VAG-CP-Docs

Thank you for helping document the impact of Volkswagen Group's Component Protection system. Every contribution — whether a personal story, a technical detail, a legal reference, or a press link — strengthens the case for right-to-repair reform.

---

## Ways to contribute

### 1. Submit a consumer case
If you or someone you know has been affected by Component Protection, your story is valuable evidence. See `cases/template.md` for the format. You can:
- Open a GitHub Issue using the "Case Submission" template
- Submit a Pull Request adding a new file to the `cases/` folder
- Email [maintainer contact TBD] if you prefer not to use GitHub

**You do not need to include your real name.** Anonymous and pseudonymous submissions are welcome.

### 2. Add technical documentation
If you have publicly documented information about how Component Protection works — module lists, platform coverage, ODIS behavior, error codes — please add it to the `technical/` folder. 

**Important:** Only include information from public sources (manufacturer documentation, published TSBs, forum posts, academic research). Do not include instructions for circumventing the system.

### 3. Add legal/regulatory content
Help us track relevant legislation, FTC filings, DMCA rulings, court decisions, and regulatory comments. Add files to the `legal/` folder with clear sourcing.

### 4. Add press coverage
Found a relevant article, video, or forum thread? Add it to `media/press-coverage.md` with the title, source, date, and URL.

### 5. Translate content
Component Protection affects VW Group owners worldwide. Translations of key documents are very welcome. Create a subfolder like `technical/de/` or `cases/uk/` as appropriate.

---

## Case submission guidelines

When submitting a case, please include as much of the following as you're comfortable sharing:

- **Vehicle:** Year, make, model (e.g., 2013 Audi A6 C7)
- **Country/Region**
- **Part replaced:** What module triggered Component Protection
- **Source of replacement part:** Dealer new / salvage / aftermarket
- **CP removal outcome:** Were you able to get it removed? At what cost? By whom?
- **Impact:** What happened if you couldn't get it removed?
- **Spontaneous activation:** Did CP trigger without any parts change?
- **Date of incident** (approximate is fine)
- **Source documentation:** Forum links, dealer invoices (redacted), TSB references

---

## General guidelines

- All content should be factual and sourced where possible
- Respectful, professional tone — this is an advocacy resource, not a complaint forum
- No personal identifying information about third parties without consent
- No instructions for circumventing security systems
- All contributions are released under CC0 (public domain)

**VIN handling:** Filenames may contain partial VIN-derived identifiers (e.g., the last six characters of a VIN) to support research reproducibility and cross-referencing of related documents. Within document text, VINs should be redacted to prevent identification of individual vehicles (e.g., `WAUGGAFC*********` or similar masking).

---

## Opening a Pull Request

1. Fork the repository
2. Create a branch: `git checkout -b add-case-2013-a6` or `git checkout -b add-oregon-law`
3. Add your content
4. Open a Pull Request with a clear description

If you're not comfortable with GitHub, open an Issue instead and a maintainer will help.

---

## Code of conduct

This project is committed to being a welcoming space for vehicle owners, independent repair professionals, journalists, and advocates. Harassment of any kind will not be tolerated.
