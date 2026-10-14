# Forward Deployed Engineer Case Study
## Client A — Ceramic Tiles | Shade Matching & Print-File Correction
### Prepared by: Forward Deployed Engineer | Banao Technology
**Date:** October 2026 | **Status:** Pre-Proposal (Post Discovery Call)

---

## 1. Problem Understanding

### 1.1 What the Client Said vs. What They Actually Need

The procurement team framed this as a "vision analytics" or "video analytics" problem — essentially, "put a camera on the line, compare tiles, tell us what to change in Photoshop." This framing, while reasonable from a procurement perspective, contains three critical engineering misconceptions that must be corrected before any solution is designed.

**What They Said:**
> *"Compare the master tile with the current production tile, work out the colour difference, and then suggest changes to the input file to the digital printer."*

**What They Actually Need:**
A **closed-loop colour-correction decision engine** that:
1. Measures the precise post-kiln spectral and spatial colour deviation (ΔL*, Δa*, Δb*) between the current production tile and a locked Digital Golden Master.
2. Applies a calibrated, physics-based Jacobian inversion to output the exact 6-channel ink percentage adjustments for the digital printer's RIP (Raster Image Processor) file — with no guessing.
3. Routes the computed correction through a **QC Technician sign-off screen** before any file is sent to the printer.
4. Logs every correction, creating a traceable audit trail that eliminates multi-year file drift.

**What They Must NOT Get:**
- A camera on the wet, unfired conveyor belt (physically impossible — glaze colours before kiln firing are chemically different from baked colours).
- A gloss correction via ink percentages (gloss is a function of kiln thermal profile and base glaze chemistry, not ink pigment concentration).
- A "root-cause" kiln diagnostic system (acknowledged by the client as a separate, much larger problem).

---

### 1.2 Stakeholder Map

| Stakeholder | What They Want | Success Signal |
|---|---|---|
| **Head of IT & OT Procurement** | Vendor credibility, hardware partnerships, site-visit capability | Proof of concept, structured proposal, partner references |
| **Special Projects Lead, Procurement** | Reduce 6–7 hr correction loop to minutes; eliminate 5,000–10,000 m² scrap per changeover | < 3 min software cycle; ΔE₀₀ ≤ 1.2 on 1st corrected tile |
| **Plant QC Team** (not on call — critical gap) | A tool they can use without a data science degree; human override always available | Simple sign-off UI; no black-box automation |
| **Design / Colour Team** (not on call) | Master tile colour standards are preserved and never drifted | Locked Digital Golden Master; colour deviation log per SKU |
| **Plant Operations / Line Manager** (not on call) | Minimum line downtime; changeover managed without halting the belt | Pre-Batch Lead Tile SOP; zero full-line shutdown |

> **Critical Gap:** The day-to-day users (QC Technicians, Colour Designers, Line Managers) were not on the discovery call. This means procurement is translating requirements from technical staff. Any design decision made without these users is a risk.

---

### 1.3 As-Is Process (Current State)

```
Batch Changeover → Print 1 Sample Tile → Fire in Kiln (45–90 min)
       ↓
QC Technician measures ΔL*, Δa*, Δb* with handheld spectrophotometer
       ↓
Technician VISUALLY estimates correction → Opens Photoshop
       ↓
Makes 6–7 variants (adjust Brown +5%, try again, etc.)
       ↓
Prints all 6–7 variant tiles → Fires all 6–7 in kiln (another 45–90 min)
       ↓
Lab checks each variant against Master Tile
       ↓
If close enough: approve → run full batch
If not: REPEAT the entire loop
─────────────────────────────────────────────────────────────
Total: 6–7 hours per round × often 2 rounds = up to 14 hours
Scrap: 5,000–10,000 m² of tiles produced in the wrong shade during each loop
```

**Compounding Problems Identified:**
1. **Human Colour Perception Variability:** "Different people see different variations and pick different points to change." The correction is operator-dependent and non-reproducible.
2. **Multi-Year File Drift:** Operators edit the previous run's already-edited file, not the original master. A file edited by Person A in 2021, by Person B in 2023, and by Person C in 2025 is now three layers of drift away from the original master — and nobody knows.
3. **Single-Spot Measurement Failure on Patterned Tiles:** The handheld spectrophotometer measures an ~8mm spot. On marble-veined or wood-grain tiles, a 2 cm placement shift hits a completely different vein with a completely different CIELAB (L*, a*, b*) value. This makes the measurement itself unreliable for patterned SKUs.
4. **Gloss Conflation:** The client mentioned ΔGloss as a requirement alongside ΔL*, Δa*, Δb*. Ink channel adjustments **cannot** correct gloss — gloss is determined by the base glaze chemistry and kiln thermal profile. Conflating these two leads to misguided corrections.

---

## 2. What We Don't Know Yet

The following questions are ranked by their impact on the solution architecture. For each, we state the assumption we will design to until we receive an answer.

| Rank | Question | Impact if Wrong | Assumption Until Answered |
|---|---|---|---|
| **1** | How many distinct active SKUs does the system need to handle? | Low SKU count (< 50) → simpler calibration; high SKU count (> 500) → SKU-specific Jacobian matrix management needed | Assume 50–200 active SKUs across plants |
| **2** | What digital printing machine make/model and RIP software is in use? (EFI Fiery, Caldera, Colorgate?) | RIP integration determines whether we can push corrected curves automatically or if export is manual | Assume manual TIFF/ICC export from our system, operator imports into RIP |
| **3** | Do QC Technicians have access to a computer workstation near the lab area? | Determines whether we need a ruggedised tablet UI or a desktop workstation | Assume a shared PC workstation is available in the lab |
| **4** | Can we access historical "correction log" data — even informally? (Photos of old Photoshop files, lab notes?) | Without historical data, we cannot validate the model before go-live; must do fresh calibration | Assume no historical data; all calibration done fresh on-site |
| **5** | What is the production volume (batches per line per day)? How many plants would the system serve in Phase 1? | Determines server sizing and whether a single SaaS instance is viable | Assume 3–5 changeovers/day/line, Phase 1 on 1 line in UAE plant |
| **6** | Is the client open to a site visit from our team or a hardware partner for lab booth installation? | A site visit is necessary for proper lighting calibration; remote-only significantly limits accuracy | Assume client is open to a 2-day site visit (they mentioned this explicitly) |
| **7** | Are the master tile physical samples still physically available? Can they be shipped to our lab for reference scanning? | If masters only exist as old TIFF files (possibly already drifted), we need to re-scan from physical tiles | Assume physical master tiles exist and can be scanned on-site |
| **8** | What is the kiln cycle time and typical temperature variance day-to-day? | High kiln variance (ΔE₀₀ > 2.0 on identical tiles) would mean kiln stabilisation must come before print correction | Assume kiln is reasonably stable; pilot will measure intra-batch variance as a baseline |

---

## 3. Solution

### 3.1 Options Considered

**Option A — Computer Vision Camera on the Production Line (Rejected)**
> Mount an RGB or hyperspectral camera above the conveyor belt and inspect tiles continuously.

*Why rejected:* Physically impossible before the kiln — glaze on wet/unfired tiles has chemically different reflectance than baked glaze. After the kiln, the tile has already been produced. A defect found post-kiln only confirms scrap, it cannot prevent it. Additionally, industrial conveyor-speed imaging at ΔE precision requires extremely expensive hyperspectral hardware ($80,000–$200,000 per camera) with complex lighting rig — far beyond the scope of the actual problem (which is *batch changeover*, not continuous inspection of every tile).

**Option B — Pure AI/ML Predictive Shade Model (Rejected for Phase 1)**
> Train a deep neural network on historical print-file to fired-tile colour data to predict corrections.

*Why rejected for Phase 1:* Requires large labelled historical datasets that the client likely does not have in structured form. Black-box predictions would not be trusted by QC Technicians without explainability. High risk of overfitting to one plant's kiln conditions and failing at another. Can be introduced in Phase 3 once we have 12+ months of structured data from Phase 1.

**Option C — Direct Spectrophotometer Integration with Manual Correction Lookup Table (Rejected)**
> Measure ΔL*a*b* with the client's existing spectrophotometer and provide a static lookup table of ink adjustments.

*Why rejected:* Static lookup tables cannot capture non-linear multi-channel interactions between 6 ink colours. Also, single-spot handheld measurements are unreliable on patterned tiles. A lookup table also provides no mechanism for drift compensation over time.

---

### 3.2 Recommended Solution — Physics-Calibrated Jacobian Colour Engine

**Core Architecture:** A lightweight lab-side software workstation integrated with a calibrated imaging light-booth and the client's existing spectrophotometer/glossmeter.

**What We Deliberately Will NOT Build:**
- Any camera on the wet conveyor belt.
- Gloss correction via ink channels (we flag ΔGloss as a process alert only).
- Full kiln thermal root-cause diagnostics.
- Any system that applies corrections automatically without a QC Technician sign-off.
- Mobile-app or cloud-first architecture in Phase 1 (unnecessary complexity for a single-lab deployment).

**Why This Recommendation Fits This Client:**
- Their QC team already measures ΔL*a*b* — we augment their existing skill, not replace it.
- The Jacobian approach requires only a one-time calibration print run, not months of training data.
- The output is a deterministic, explainable ink percentage change — not a black-box AI decision.
- A QC Technician reviews and approves before any file goes to the printer — building trust.
- Total Phase 1 cost ($40,000) is recovered in under 3 batch changeovers (see Section 7).

---

## 4. Solution Engineering

### 4.1 Hardware Stack

| Component | Specification | Purpose |
|---|---|---|
| **Calibrated Lab Imaging Light-Booth** | D65 + TL84 dual-illuminant LED; calibrated with X-Rite ColorChecker Passport | Standardised, glare-free illumination for tile scanning |
| **Industrial Camera** | 20MP monochrome or colour CCD (e.g., Basler ace2); mounted fixed above booth | Full-tile image capture at ~0.1mm/pixel resolution |
| **Spectrophotometer (existing)** | Client's existing unit (e.g., Konica Minolta CM-series) | Secondary spot-check; glossmeter reading for ΔGloss alert |
| **Lab Workstation** | Intel i7, 32GB RAM, 2TB SSD, Windows 10/11 | Runs the Jacobian engine, GUI, and data store |
| **Calibration Reference Tiles** | 6-channel stepped grid (printed and fired once per plant setup) | Used to compute the 3 × 6 Jacobian sensitivity matrix |

### 4.2 Software Stack

- **Image Registration Module:** OpenCV-based sub-pixel alignment of production tile image to Digital Golden Master tile image (affine + homography transform), ensuring pixel-by-pixel spatial correspondence even if the tile is placed at a slight angle.
- **Regional ΔL*a*b* Engine:** Computes colour deviation in CIELAB (L*, a*, b*) colour space (not RGB) across defined tile zones, outputting a spatial deviation map and a single representative ΔE₀₀ value.
- **Jacobian Colour Correction Solver:** The core mathematical engine (see Section 4.3).
- **QC Sign-Off UI:** Simple desktop GUI showing the deviation map, computed ink adjustments per channel, and an Approve / Reject / Override button. Logs technician ID and timestamp.
- **Digital Golden Master Vault:** Read-only locked repository of original master TIFF files and scanned physical master tile reference images, per SKU. Never edited by operators.
- **Correction Audit Log:** SQLite database recording every correction, the ΔE₀₀ before and after, the operator who approved, and the final ink values used.

### 4.3 The Mathematics — Jacobian Ink Correction Solver

**Step 1: One-Time Calibration (per plant)**

Print and fire a **6-channel stepped calibration grid tile** — a tile where each of the 6 ink channels (Cyan, Magenta, Yellow, Black, Brown, Beige) is independently varied by known percentages (-10%, 0%, +10%). After firing, measure post-kiln CIELAB (L*, a*, b*) values for each step.

This yields the **Jacobian sensitivity matrix** J (size 3 × 6):

```
    [ ∂L*/∂c1  ∂L*/∂c2  ∂L*/∂c3  ∂L*/∂c4  ∂L*/∂c5  ∂L*/∂c6 ]
J = [ ∂a*/∂c1  ∂a*/∂c2  ∂a*/∂c3  ∂a*/∂c4  ∂a*/∂c5  ∂a*/∂c6 ]
    [ ∂b*/∂c1  ∂b*/∂c2  ∂b*/∂c3  ∂b*/∂c4  ∂b*/∂c5  ∂b*/∂c6 ]
```

Where c₁ ... c₆ are the 6 ink channel percentages (Cyan, Magenta, Yellow, Black, Brown, Beige).

**Step 2: Runtime Correction (per batch changeover)**

Measure the colour deviation vector:
> **Δy = [ΔL*, Δa*, Δb*]ᵀ = y_master - y_current**

Solve for the required ink adjustment vector **Δc** using **regularised least-squares inversion** (Tikhonov regularisation with tuned parameter λ ≈ 10⁻⁴ to prevent extreme channel swings while ensuring full convergence):

> **Δc = (Jᵀ · J + λ · I)⁻¹ · Jᵀ · Δy = Jᵀ · (J · Jᵀ + λ · I)⁻¹ · Δy**

This is computed in **< 1 millisecond** on any modern laptop. The output is the exact percentage change for each of the 6 ink channels.

**Step 3: Gloss Separation**

ΔGloss is measured by the existing glossmeter and logged separately. If **ΔGloss > 2 GU** (Gloss Units), the system raises a **Process Alert** to the Kiln/Glaze Operations team: *"Gloss deviation detected. Review kiln temperature profile and base glaze batch."* The ink correction solver does not attempt to modify any ink channel in response to gloss deviation.

### 4.4 End-to-End Workflow (To-Be Process)

```
Batch Changeover Announced (T - 90 min)
       ↓
Send 1 "Lead Tile" through the printer (from locked Digital Golden Master file)
Fire it in the kiln (45–90 min) ← this happens BEFORE the main batch starts
       ↓ (T - 0 min: main batch starts printing on the line with the old file)
Lead Tile exits kiln → placed in Lab Light-Booth
System captures full-tile image (< 30 sec)
       ↓
Image Registration: align Lead Tile to Digital Golden Master image
Compute regional ΔL*, Δa*, Δb* and ΔGloss
       ↓
ΔE₀₀ ≤ 1.2?  ──── YES ──→ Approve. No change needed. Belt continues.
                             Log result. Done. (< 3 min total)
       │
      NO
       ↓
Jacobian Solver computes 6-channel ink adjustments in < 1ms
QC Technician reviews Sign-Off Screen (deviation map + suggested corrections)
Technician clicks APPROVE (or adjusts manually and approves)
       ↓
System exports corrected RIP curve / TIFF file
Operator imports into printer → restarts print head with corrected file
       ↓
Second Lead Tile fires (confirmation pass) — verifies ΔE₀₀ ≤ 1.2
       ↓
Full batch runs with corrected file
─────────────────────────────────────────────────────────────
Total software + analysis time: < 3 minutes
Main production impact: 0 m² wasted (lead tile SOP absorbs the kiln wait)
Variants required: 1 (not 6–7)
```

### 4.5 Fit With Existing Tools, People and Routine

- **Photoshop:** Completely removed from the correction loop. QC technicians are retrained to use the Sign-Off UI instead. The corrected file is generated programmatically.
- **Existing Spectrophotometer:** Retained as a secondary spot-check tool and primary glossmeter input. Not removed — used alongside the booth.
- **Digital Printer/RIP:** No changes to printer hardware or RIP software required. The corrected file is imported by the operator as they always have done, just with the right values now.
- **Lab Routine:** The "pre-batch lead tile" SOP is a formalisation of what the best operators already do informally. It adds discipline but not a new workflow concept.

---

## 5. Evaluation

### 5.1 Baseline (Current State)

| Metric | Current Value |
|---|---|
| Correction cycle time | 6–7 hours per round |
| Rounds typically needed | 1–3 rounds |
| Variants printed per round | 6–7 tiles |
| Off-shade scrap per changeover event | 5,000–10,000 m² (avg: 7,500 m²) |
| Measurement method | Visual + handheld spectrophotometer (single spot) |
| File integrity | Drifted (multi-year edits) |
| ΔE₀₀ at approval (tolerance) | Operator-dependent; often approved at ΔE > 2.0 |

### 5.2 Target Metrics

| Metric | Target | Measurement Method |
|---|---|---|
| Software cycle time | < 3 minutes | System timestamp (image capture to corrected file export) |
| Confirmed ΔE₀₀ after first correction | ≤ 1.2 | Post-kiln imaging of second lead tile |
| Variants fired per changeover | 1 (confirmation tile only) | Correction audit log |
| Main batch off-shade scrap | 0 m² | Production log cross-reference |
| ΔGloss flagging accuracy | 100% of anomalies flagged | QC review of first 20 events |

### 5.3 Test Protocol (6-Week Pilot Validation)

**Weeks 1–2:** Calibrate system. Run 10 controlled changeovers where we compare the system's suggested correction against the manual Photoshop process result (run both, pick the best). Measure ΔE₀₀ for both methods.

**Weeks 3–4:** System runs "live advisory" — QC technician sees the system's recommendation but also does their Photoshop correction. We compare and converge.

**Weeks 5–6:** System runs in full production mode. QC technician uses system output only. Measure outcomes.

**Acceptance Criterion:** On ≥ 80% of batch changeovers, the system achieves ΔE₀₀ ≤ 1.2 on the first corrected tile.

### 5.4 Kill-Switch — When We Stop

> **If intra-batch kiln variation on identical print files within a 15-minute window exceeds ΔE₀₀ > 2.0 on three consecutive measurement events, kiln thermal instability is the bottleneck.**

In this scenario, print-file correction **cannot** solve the problem — the kiln is producing inconsistent results independent of the ink file. We will:
1. Halt Phase 1 trials.
2. Document kiln variation data and present to the client's engineering team.
3. Recommend kiln thermal stabilisation as a prerequisite before resuming.
4. Propose a Phase 0 kiln variance audit (separate scope and cost).

This protects the client from investing further in print-file correction when the root cause lies upstream.

---

## 6. Build Plan and Timeline

### 6.1 Phase 1: Proof of Concept Pilot — UAE Plant (12 Weeks, $40,000)

| Week | Milestone | Dependencies from Client |
|---|---|---|
| 1 | Kickoff, requirements deep-dive with QC Team, Design Team, Line Managers | Client to schedule meetings; site access |
| 2 | Physical Golden Master tile scanning; Digital Golden Master Vault setup for 10 SKUs | Physical master tiles; storage access |
| 2–3 | Lab light-booth hardware delivery and installation; calibration with X-Rite ColorChecker | Lab space (~2m × 2m); power outlet |
| 3–4 | Calibration grid tile print + firing; Jacobian matrix computation | 1 dedicated printer slot for calibration tiles |
| 4–6 | Software development: image registration, solver, QC sign-off UI, audit log | QC Team available for UI feedback sessions |
| 6–8 | Controlled parallel testing (system vs. manual) on 10 changeovers | QC Team cooperation for dual-method runs |
| 8–10 | Live advisory mode — 20 additional changeovers | — |
| 10–12 | Full production mode; acceptance criterion validation | — |
| 12 | Phase 1 report; go/no-go decision for Phase 2 | — |

### 6.2 Phase 2: Multi-SKU Expansion — UAE Plant (Weeks 13–24, $35,000)

- Expand Golden Master Vault to all active SKUs at UAE plant.
- Refine Jacobian matrix per SKU category (matte, gloss, textured).
- Train all QC Technicians; create SOP documentation.
- Integrate ΔGloss alert into kiln operations team workflow.

### 6.3 Phase 3: Multi-Plant Rollout — India & Bangladesh (Weeks 25–52, $55,000)

- Deploy to India and Bangladesh plants.
- Centralised correction audit log across all plants.
- Begin building dataset for Phase 3 ML predictive model (optional, scope TBD).
- Total multi-plant rollout annual support: $90,000/year.

---

## 7. Cost, ROI and Pricing

### 7.1 Value to Client — The Arithmetic

| Parameter | Value | Source |
|---|---|---|
| Average off-shade scrap per changeover event | 7,500 m² | Client's statement (5,000–10,000 m²) |
| Margin loss per m² of scrapped tile | $1.50 | Industry estimate (production cost, not retail price); to be confirmed |
| Loss per changeover event | $11,250 | 7,500 × $1.50 |
| Changeover drift events per line per year | ~200 | 3–5 changeovers/day × ~50% requiring correction × 260 working days |
| **Total annual loss per line** | **$2,250,000** | $11,250 × 200 |
| Waste reduction with our system (conservative 75%) | **$1,687,500 saved/line/year** | — |
| Payback period on Phase 1 ($40,000 pilot) | **< 3 events** | $40,000 / $11,250 ≈ 3.6 events |

> **Note:** The $1.50/m² margin loss is an assumption. If the client's actual margin loss per m² is lower (e.g., $0.80/m²), the annual savings are ~$900,000 — still a strong ROI on a $40,000 pilot. We will confirm this figure in the next call.

### 7.2 Effort and Team

| Role | Phase 1 Effort | Notes |
|---|---|---|
| FDE / Project Lead | 4 weeks | Discovery, architecture, client management |
| Computer Vision Engineer | 6 weeks | Light-booth integration, image registration |
| Software Engineer | 4 weeks | Jacobian solver, QC UI, audit log |
| On-site Calibration Visit | 2 days | UAE plant; 2 engineers |

### 7.3 Cost Breakdown — Phase 1

| Item | Cost |
|---|---|
| Lab light-booth hardware (D65/TL84 LED + industrial camera) | $5,000 |
| Calibration tiles (print + firing, incl. materials) | $500 |
| Software development (12 weeks × 3 engineers, blended rate) | $28,000 |
| Site visits (2 engineers × 2 days, UAE, incl. travel) | $4,000 |
| Project management & documentation | $2,500 |
| **Phase 1 Total (Fixed Price)** | **$40,000** |

### 7.4 Proposed Pricing Structure

- **Phase 1:** $40,000 fixed price. Payment: 50% on contract signing, 50% on Phase 1 acceptance criterion achieved.
- **Phase 2:** $35,000. Same payment structure.
- **Phase 3 / Annual Support (all plants):** $90,000/year. Monthly invoicing.
- **No per-tile or per-measurement usage fees.** The system runs on the client's hardware (purchased once) and our software (licensed annually).

---

## 8. Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| **Kiln thermal instability invalidates print-file correction** | Medium | High | Kill-switch defined (Section 5.4); measure intra-batch ΔE variance in Week 1; kiln stabilisation as Phase 0 if needed |
| **QC Team resistance to new workflow** | Medium | High | Co-design UI with QC technicians in Week 4–5; manual override always available; technician sign-off is central to design |
| **Physical master tiles unavailable or degraded** | Medium | High | Can re-create reference from approved print files if tiles are available in archive; add physical tile scanner to booth |
| **RIP software integration blocked by vendor** | Low | Medium | Phase 1 uses manual file import (no API needed); RIP integration planned for Phase 2 with vendor cooperation |
| **Multi-plant kiln variation means one Jacobian doesn't generalise** | Low | Medium | Plant-specific Jacobian matrix per plant (calibration grid re-fired at each plant); not a shared model |
| **Lighting variance in light-booth affects measurement accuracy** | Low | Medium | ISO 3664:2009 compliant booth with certified illuminant; regular calibration with ColorChecker |
| **Client procurement delays hardware purchase approval** | Medium | Low | Hardware ($5,000) is a small fraction of Phase 1 cost; alternatively, we can procure and include in fixed price |

---

## Appendix A: Glossary

| Term | Definition |
|---|---|
| CIELAB (L*, a*, b*) | CIELAB colour space. L* = lightness (0=black, 100=white); a* = red-green axis; b* = yellow-blue axis |
| ΔE₀₀ | CIEDE2000 colour difference formula. ΔE₀₀ < 1.0 = imperceptible to trained eye; < 2.0 = acceptable in most industrial contexts |
| Jacobian Matrix | Sensitivity matrix mapping small changes in ink percentages to resulting changes in CIELAB (L*, a*, b*) values |
| RIP (Raster Image Processor) | Software that translates a print file into ink-channel data for the digital printer |
| Kiln | High-temperature oven (1,100–1,250°C) used to permanently cure ceramic glaze and print |
| SKU | Stock Keeping Unit — a specific tile design/size/finish combination |
| Digital Golden Master | The locked, original, never-edited print reference file for a given SKU |
| D65 | Standard daylight illuminant (6500K colour temperature) used in colour measurement |
| Lead Tile SOP | Standard Operating Procedure: fire one sample tile through the kiln before the main batch to enable pre-emptive colour correction |

---

*End of Case Study — Banao Technology FDE Submission | Task 3*
