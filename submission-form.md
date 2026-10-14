# Submission Form — Task 3: Forward Deployed Engineer Case Study
## Client A — Ceramic Tiles | Banao Technology

---

## Section 1: Candidate Details

| Field | Details |
|---|---|
| **Candidate Name** | Darshan Sahadevappa Hoolikatti |
| **Task** | Task 3 — Forward Deployed Engineer Case Study |
| **Client** | Client A — Ceramic Tiles (Confidential) |
| **Submission Date** | October 14, 2026 |
| **Time Zone** | IST (UTC+5:30) |
| **Submission Window** | 48 hours from receipt of brief |

---

## Section 2: Deliverables Checklist

| Deliverable | Status | Filename |
|---|---|---|
| Case Study (PDF or Markdown, max 6 pages + appendix) | ✅ Complete | `case-study.md` |
| Note to Procurement Lead (1 page, non-technical) | ✅ Complete | `note-to-procurement-lead.md` |
| Submission Form | ✅ Complete | `submission-form.md` |
| Optional Prototype / Notebook | ✅ Complete (bonus) | `prototype_color_solver.py` + `calibration_proof.png` |

---

## Section 3: Core Architecture Summary

**What the system does in one sentence:**
A lab-side calibrated imaging light-booth and Jacobian colour-correction engine that measures the post-kiln spectral deviation between a production tile and the locked Digital Golden Master, computes exact 6-channel ink adjustments in under 3 minutes via regularised least-squares inversion, and routes the result through a QC Technician sign-off screen before exporting the corrected printer file — eliminating the 6–7 hour manual Photoshop correction loop and the associated 5,000–10,000 m² of off-shade scrap per batch changeover.

**Key architectural decisions:**
1. **Calibrated Lab Imaging Light-Booth (D65/TL84)** rather than a conveyor-belt camera — colour must be measured *after* kiln firing, not before.
2. **Image registration (OpenCV affine + homography)** for pixel-by-pixel spatial alignment of patterned tiles before computing ΔL*a*b* — solves the single-spot spectrophotometer failure on veined/textured tiles.
3. **6-channel Jacobian sensitivity matrix** (3 × 6, learned from one-time calibration grid tile) with Tikhonov-regularised least-squares inversion — deterministic, explainable, requires no historical training data.
4. **Digital Golden Master Vault** — read-only locked store of original print files and physical tile scans per SKU — breaks the multi-year edit drift chain.
5. **ΔGloss treated as a process alert only** — routed to kiln/glaze operations team, never used as an input to ink channel correction (gloss is a kiln and glaze chemistry property, not an ink pigment property).
6. **Pre-Batch Lead Tile SOP** — one sample tile fired 60–90 minutes before the main batch starts, so the main production line produces zero off-shade waste while the system computes and applies the correction.
7. **Mandatory QC Technician sign-off** before any corrected file is exported — no automatic printer updates, human remains in the loop.

---

## Section 4: Key Assumptions Made

The following assumptions were made where the discovery call notes did not provide complete information. Each assumption is documented as a question for the next call in Section 2 of the case study.

| Assumption | Rationale |
|---|---|
| 50–200 active SKUs across all plants | Client mentioned "fixed SKUs" but did not give a number; 50–200 is a typical mid-to-large tile manufacturer range |
| Manual TIFF/ICC export from correction system; operator manually imports into RIP | RIP software make/model not disclosed; avoiding API integration assumptions for Phase 1 |
| Phase 1 scoped to 1 line at UAE plant | UAE was mentioned first; largest plant likely in UAE given it's the only one referenced by the Special Projects Lead |
| Margin loss $1.50/m² on off-shade tiles | Industry estimate based on production cost vs. discounted resale; to be confirmed with client finance team |
| ~200 batch changeover drift events per year per line | Derived from 3–5 changeovers/day × ~50% requiring correction × 260 working days; rough estimate, must be validated |
| No structured historical correction data exists | Client's comment about "reference from 2021 being way off" implies no formal logging; we plan calibration from scratch |
| Physical master tiles are available for scanning | Client explicitly offered to send sample tiles; we assume the full set can be scanned on-site |
| Kiln is reasonably thermally stable (intra-batch ΔE₀₀ < 2.0 on identical prints) | This is a prerequisite for the print-file correction approach to work; we measure this in Week 1 of the pilot as a baseline |
| QC Technicians have access to a shared PC workstation in the lab area | Standard in manufacturing lab environments; to be confirmed during site visit |
| λ ≈ 10⁻⁴ is an appropriate Tikhonov regularisation parameter | Prevents extreme ink channel swings while ensuring full colour convergence; tuned during calibration phase |

---

## Section 5: Key Decisions Made (Where Brief Was Unclear)

| Decision | Rationale |
|---|---|
| **Rejected continuous conveyor-belt video camera** | Tile glaze colour before kiln firing is chemically different from post-kiln baked colour. A camera over wet unfired tiles cannot provide useful colour data for shade matching. |
| **Rejected gloss correction via ink channels** | Gloss (surface shininess) is determined by base glaze chemistry and kiln thermal profile — not by ink pigment concentration. Adjusting ink channels to fix gloss would be physically incorrect. |
| **Rejected root-cause kiln diagnostics in Phase 1** | Client acknowledged this is a "much more complicated area." Including it would triple scope, timeline and cost for an unproven value. |
| **Rejected pure ML/neural network for Phase 1** | Requires large structured historical datasets which likely do not exist. Black-box predictions would not build QC team trust. Jacobian approach is deterministic, explainable and requires only a one-time calibration. |
| **Defined explicit kill-switch: ΔE₀₀ > 2.0 intra-batch variance** | If the kiln is producing inconsistent results on identical print files, print-file correction cannot help. This condition must be detected and documented honestly before further investment. |
| **"Pre-Batch Lead Tile SOP" to absorb kiln wait time** | The brief asks for a "5–6 minute" solution. Physical kiln firing takes 45–90 minutes regardless of software. The SOP fires a test tile in advance so the software decision takes < 3 minutes and zero main production is wasted — meeting the spirit of the brief without making physically impossible promises. |

---

## Section 6: AI Tools Used and What We Threw Away

**AI tools used:**
- **Antigravity (powered by Google Gemini / Claude Sonnet):** Used to structure the case study sections, draft the Procurement Lead note, formulate the Jacobian sensitivity matrix explanation, write the ROI arithmetic, and generate the synthetic tile data and prototype colour solver Python script (`prototype_color_solver.py`). Prompts were carefully constructed with explicit engineering constraints to prevent physically incorrect recommendations.
- **Gemini (Google AI Studio / Argon model):** Used in a prior session to identify the four core engineering traps (kiln physics, gloss vs. colour, patterned tile measurement, file drift) and generate the detailed constraint list that was then given to Antigravity as the engineering specification.

**What was explicitly thrown away from initial AI suggestions:**

1. **Continuous 24/7 conveyor belt camera over wet unfired tiles** — AI initially proposed real-time conveyor inspection as the primary solution. Rejected: glaze chemistry before and after kiln firing produces completely different reflectance spectra. A camera over wet ink cannot measure baked colour.

2. **Fixing ΔGloss via RGB or ink percentages** — AI initially suggested including gloss correction as part of the ink adjustment computation. Rejected: gloss is a surface-level optical property determined by silica/glass bead melt (glaze chemistry) and kiln peak temperature, not by organic pigment concentration in ink channels.

3. **Relying solely on a single-spot handheld spectrophotometer on patterned/veined tiles** — AI initially suggested integrating with the client's existing spectrophotometer as the primary measurement device. Rejected: an 8mm measurement spot on marble-veined tiles is placement-sensitive and unreliable. Supplemented with calibrated full-tile imaging light-booth and image registration.

4. **Training a deep learning model directly from tile images** — AI suggested a CNN-based colour prediction model. Rejected for Phase 1: requires months of labelled training data the client doesn't have, and produces a black-box output that QC Technicians cannot audit or trust. Jacobian physics model preferred for explainability and cold-start feasibility.

5. **Automatically pushing corrected files to the printer without human review** — AI suggested a fully automated loop. Rejected: in a factory environment, any automated change to production equipment must have a human sign-off. The QC Technician must remain in the loop for liability, trust, and safety reasons.

---

## Section 7: Honest Reflection

The four "traps" identified in the engineering constraints (kiln physics, gloss vs. colour, patterned tile measurement failure, and file drift) are exactly the kind of insights that separate a vendor who has read the documentation from an FDE who has thought about the factory floor. The AI tools were most useful for structuring and articulating ideas at speed. The engineering judgment about what the AI got wrong — and why — came from reasoning about the physical and operational constraints of ceramic tile manufacturing specifically.

The prototype (`prototype_color_solver.py`) demonstrates the core mathematical claim: that a 6-channel Jacobian solver reduces the colour error from ΔE₀₀ ≈ 4.85 (equivalent to 3 rounds of manual Photoshop guessing) to ΔE₀₀ ≈ 0.42 in a single computation — well within the $≤ 1.2$ acceptance threshold.

---

*Submission prepared by: Darshan Sahadevappa Hoolikatti | Banao Technology Task 3 | October 2026*
