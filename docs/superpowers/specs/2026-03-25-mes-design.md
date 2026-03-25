# Multiverse Evidence Synthesis (MES) — Design Specification

**Version:** 1.0
**Date:** 2026-03-25
**Author:** Mahmood Ahmad
**Status:** APPROVED

---

## 1. Problem Statement

Modern meta-analysis suffers from three compounding failures:

1. **The summarization fallacy** — reducing heterogeneous evidence to a single pooled number (θ̂, I², p-value) that hides more than it reveals.
2. **The model-specification lottery** — the analyst chooses one estimator, one CI method, one bias correction, and the conclusion depends on those arbitrary choices. Empirical evidence: 56.2% of 407 Cochrane MAs are Fragile or Unstable when the multiverse of defensible analytical choices is explored (Fragility Atlas, Ahmad 2026).
3. **The evidence-certainty gap** — pooling effect sizes while ignoring quality, design diversity, and causal architecture. 69.8% of "significant" Cochrane MAs have prediction intervals crossing the null (PredictionGap, Ahmad 2026).

These failures compound: hiding heterogeneity lets bad model choices go unnoticed, and ignoring evidence quality lets noise masquerade as signal.

### 1.1 Empirical Foundation

| Finding | Source | Dataset |
|---------|--------|---------|
| 56.2% of Cochrane MAs are Fragile/Unstable | Fragility Atlas | 407 reviews, 399,546 specs |
| Bias correction drives η²=0.361 of conclusion variance | Fragility Atlas | 407 reviews |
| 69.8% of "significant" MAs have PIs crossing null | PredictionGap | 403 reviews |
| 48.9% show suspected publication bias | BiasForensics | 307 reviews, 8 methods |
| 4.4% study overlap across Cochrane reviews | OverlapDetector | 501 reviews, 10,006 studies |

### 1.2 Success Criteria

1. MES processes any standard pairwise meta-analysis dataset (yi, vi pairs with optional metadata).
2. Produces a robustness-classified evidence landscape instead of a single point estimate.
3. Conditional robustness metric answers: "Does the conclusion survive when you demand better evidence?"
4. Influence decomposition answers: "Is fragility driven by model choice or evidence quality?"
5. All results certified via TruthCert provenance chain.
6. Empirically validated on 407 Cochrane reviews with R parity gate.
7. Ships as both Python package (batch/research) and HTML app (interactive/clinical).

---

## 2. Core Concept: Evidence Cartography

Traditional meta-analysis gives a GPS coordinate: "the effect is here." MES gives a topographic map — showing elevation (robustness), terrain (quality), fault lines (fragility boundaries), and fog-of-war (regions where evidence is thin).

The paradigm shift:
> Traditional meta-analysis asks: *what is the effect?*
> MES asks: *how much of the evidence landscape supports an effect, and what does that landscape look like?*

Traditional MA is a single point in the MES multiverse — one estimator, one CI method, no bias correction, all studies included.

---

## 3. Architecture: ASSESS → EXPLORE → MAP

### 3.1 Overview

```
Studies → ASSESS (annotate) → EXPLORE (multiverse) → MAP (landscape) → Certified Evidence Package
```

- **ASSESS** (pipeline): Annotate each study with quality, design, bias, and overlap metadata. No pooling, no weighting, no judgment.
- **EXPLORE** (declarative): Generate all defensible analytical specifications and execute them exhaustively. Output: result tensor.
- **MAP** (synthesis): Transform the result tensor into robustness classification, influence decomposition, fragility boundaries, and a certified evidence landscape.

### 3.2 Dual Implementation

| Component | Python Engine (`mes-core/`) | HTML Frontend (`mes-app.html`) |
|-----------|---------------------------|-------------------------------|
| Purpose | Batch validation, research pipelines | Interactive single-review analysis |
| ASSESS | Full pipeline with CSV/RDA ingestion | Manual + CSV import, interactive RoB |
| EXPLORE | `multiprocessing.Pool`, numpy vectorized | Web Workers, batched `setTimeout` |
| MAP | JSON + CSV output, auto-report generation | Plotly.js visualizations, interactive |
| Validation | R parity gate (metafor), 407-review batch | WebR in-browser validation button |
| Distribution | `pip install mes-core` / GitHub | Single HTML file, no install |

---

## 4. Phase 1: ASSESS — Evidence Dossier Construction

### 4.1 Purpose

Annotate each study with structured metadata to create the quality/design dimensions that EXPLORE will vary. ASSESS is purely descriptive — it characterizes evidence before touching it.

### 4.2 Sub-modules

#### 4.2.1 Design Classifier

- Categorizes studies: RCT | quasi-experimental | cohort | case-control | cross-sectional | Mendelian randomization
- Source: user-provided or inferred from metadata
- Output: `study.design_type`, `study.design_tier` (Tier 1=RCT, Tier 2=quasi/MR, Tier 3=observational)
- Existing code: CausalSynth design classification logic

#### 4.2.2 RoB Scorer

- RCTs → RoB 2 (5 domains, Sterne 2019)
- Non-RCTs → ROBINS-I (7 domains, Sterne 2016)
- Input: user assessments or pre-loaded from CSV
- Output: `study.rob_overall` (Low / Some Concerns / High), `study.rob_domains[]`
- Existing code: RoB Assessor (C:\Models\RoBAssessor, 25/25 tests)

#### 4.2.3 Bias Profiler

- Review-level (not study-level) assessment
- Methods: Egger regression p-value, Begg rank correlation, excess significance test
- Output: `review.bias_profile{}` — flags, not corrections
- Corrections (trim-and-fill, PET-PEESE, selection model) happen in EXPLORE as multiverse dimensions
- Existing code: BiasForensics (C:\BiasForensics, 307 reviews validated)

#### 4.2.4 Overlap Detector

- Only relevant for umbrella/multi-review contexts
- CCA matrix, shared study identification
- Output: `overlap_flag` (none / slight / moderate / high)
- Skip if single-review analysis
- Existing code: OverlapDetector (C:\OverlapDetector, 10,006 studies validated)

### 4.3 Output: StudyDossier Schema

```json
{
  "study_id": "Smith2019",
  "effect": { "yi": -0.42, "vi": 0.031, "measure": "logOR" },
  "sample": { "n1": 245, "n2": 251, "events1": 32, "events2": 51 },
  "design": { "type": "RCT", "tier": 1 },
  "rob": {
    "tool": "RoB2",
    "overall": "some_concerns",
    "domains": {
      "randomization": "low",
      "deviations": "some_concerns",
      "missing": "low",
      "measurement": "low",
      "selection": "low"
    }
  },
  // Note: ROBINS-I studies have 7 domains with different keys:
  // confounding, selection_participants, classification_interventions,
  // deviations, missing_data, measurement, selection_reported_result
  // Schema is flexible — domain keys vary by tool
  "meta": { "year": 2019, "country": "UK", "multicenter": true }
}
```

### 4.4 Design Principles

- **Annotate, don't judge**: No subjective quality weights. The dossier annotates — EXPLORE decides how to filter.
- **Don't correct for bias**: Bias profiling detects signals. Corrections happen in EXPLORE as multiverse dimensions.
- **Don't pool anything**: Zero synthesis at this stage.
- **Graceful degradation**: Missing RoB → study is "unassessed" — included in "all" filter, excluded from quality-filtered specs.

---

## 5. Phase 2: EXPLORE — Multiverse Execution Engine

### 5.1 Purpose

Treat every defensible analytical choice as a dimension in a specification space. Execute all combinations. Produce a result tensor.

### 5.2 Specification Space

#### 5.2.1 Core Dimensions (always active)

| Dim | Name | Levels | Count |
|-----|------|--------|-------|
| D1 | Estimator | FE, DL, REML, PM, SJ, ML | 6 |
| D2 | CI Method | Wald, HKSJ, t-distribution (k-1 df) | 3 |
| D3 | Bias Correction | None, Trim-and-fill (Duval-Tweedie L0), PET-PEESE (conditional), Selection model (3PSM Vevea-Hedges) | 4 |
| D4 | Quality Filter | All studies, Exclude high-RoB (Low + Some Concerns only), Low-RoB only | 3 |
| D5 | Design Filter | All designs, RCT + quasi only, RCT only | 3 |
| D6 | Sensitivity | Full dataset, Leave-one-out (×k) | k+1 |

**Base grid**: 6 × 3 × 4 × 3 × 3 = **648 specifications**
**With LOO (k=10)**: 648 × 11 = **7,128 specifications**

**LOO cap**: For k > 30, LOO is replaced with random leave-d-out sampling (d=1, 30 random draws) to bound execution time. This keeps the sensitivity dimension at ≤31 levels regardless of k.

#### 5.2.2 Optional Extension Dimensions

| Dim | Name | Levels | When |
|-----|------|--------|------|
| D7 | Effect Measure | logOR, logRR, RD, SMD | When multiple measures applicable |
| D8 | Confidence Level | 90%, 95%, 99% | Always available, opt-in |

**With all extensions**: up to ~64,000 specifications.

### 5.3 Declarative Spec Format

```python
mes_spec = MESSpec(
    estimators=["FE", "DL", "REML", "PM", "SJ", "ML"],
    ci_methods=["Wald", "HKSJ", "t-dist"],
    bias_corrections=["none", "trim-fill", "PET-PEESE", "selection-model"],
    quality_filters=["all", "exclude-high-rob", "low-rob-only"],
    design_filters=["all", "rct-quasi", "rct-only"],
    sensitivity=["full", "loo"],
    alpha=0.05
)
```

Researchers can customize: remove dimensions, add levels, restrict ranges. The spec IS the analysis plan — fully reproducible.

### 5.4 Execution Engine

#### 5.4.1 Spec Generation

```
spec_grid = cartesian_product(D1, D2, D3, D4, D5, D6, ...)
feasible_specs = [s for s in spec_grid if is_feasible(s, dossiers)]
```

#### 5.4.2 Feasibility Filtering

- **k-threshold pruning**: Quality/design filter leaves k < 2 → skip. Trim-and-fill needs k ≥ 5 → skip if insufficient. Selection model needs k ≥ 10 → skip if insufficient.
- **Logical pruning**: FE + trim-and-fill → redundant (TF re-estimates τ², irrelevant under FE). FE + PET-PEESE → valid (regression-based, keep). FE + selection model → valid (likelihood-based, keep). All-designs filter when dataset is RCT-only → collapses. LOO on k=2 after quality filter → leaves k=1, skip.
- **Convergence guards**: REML non-convergence → flag, DL fallback. PM non-convergence → flag, DL fallback. Selection model boundary → flag, report as NA. All failures logged, never silently dropped.

#### 5.4.3 Per-Spec Output

```json
{
  "spec_id": "REML_HKSJ_TF_allRoB_allDesign_full",
  "theta": -0.38,
  "se": 0.12,
  "ci_lo": -0.61,
  "ci_hi": -0.15,
  "p_value": 0.0012,
  "tau2": 0.045,
  "I2": 0.62,
  "k": 8,
  "pi_lo": -0.89,
  "pi_hi": 0.13,
  "significant": true,
  "direction": "negative"
}
```

#### 5.4.4 Result Tensor

All results stored as N-dimensional tensor with axes: [estimator, ci_method, bias_corr, quality, design, sensitivity]. Enables arbitrary slicing: "all REML results", "LOO for high-quality only", etc.

### 5.5 Performance Targets

| Scenario | Python | HTML/JS |
|----------|--------|---------|
| 648 specs, k≤30 | <1s (numpy vectorized) | <2s (Web Workers) |
| 7,128 specs (with LOO, k=10) | <5s | <10s |
| Batch 407 reviews | 5-10 min (multiprocessing) | N/A |

### 5.6 Existing Code to Extract

- **MultiverseMA** → spec generation, 6-estimator suite, specification curve
- **Fragility Atlas pipeline** → batch execution across 407 reviews, η² decomposition
- **HTML apps shared libs** → meta-dl.js (DL/REML/PM/SJ/ML/EB), stats-utils.js, effect-sizes.js
- **BiasForensics** → trim-and-fill, PET-PEESE, selection model, limit MA
- **PredictionGap** → prediction interval calculation, PI/CI ratio

---

## 6. Phase 3: MAP — Evidence Landscape & Certification

### 6.1 Purpose

Transform the result tensor into actionable insights. Answer: "How much should I trust this evidence?"

### 6.2 Sub-modules

#### 6.2.1 M1: Concordance Analysis

For each specification, classify:
- **Direction**: sign(θ)
- **Significance**: p < α
- **Clinical relevance**: |θ| > MCID (if defined by user)

Concordance metrics:
- `C_dir` = % specs agreeing on direction
- `C_sig` = % specs agreeing on direction + significance (primary metric)
- `C_full` = % specs agreeing on direction + significance + clinical relevance

#### 6.2.2 M2: Robustness Classification

Based on `C_sig`:

| Class | C_sig | Interpretation |
|-------|-------|---------------|
| **ROBUST** | ≥90% | Conclusion stable across all defensible specifications |
| **MODERATE** | 70-89% | Mostly stable, note caveats about sensitive dimensions |
| **FRAGILE** | 50-69% | Conclusion depends on analytical choices |
| **UNSTABLE** | <50% | No defensible single conclusion |

**Conditional robustness** (novel MES metric):
- `C_sig|quality` = robustness among quality-filtered specs only
- `C_sig|RCT` = robustness among RCT-only specs only
- `C_sig|no-bias-corr` = robustness excluding bias-corrected specs

This enables: *"Fragile overall (C_sig=62%), but Robust when restricted to low-RoB RCTs (C_sig|quality,RCT=94%)."* This is the actionable insight clinicians need.

#### 6.2.3 M3: Influence Decomposition

ANOVA η² for each dimension (Type III sums of squares to handle unbalanced cells from feasibility pruning):
> "What fraction of conclusion variance is explained by each analytical choice?"

Example output:
```
Bias correction:  η² = 0.36  ← dominant driver
Quality filter:   η² = 0.22  ← evidence quality matters
Estimator:        η² = 0.15
LOO sensitivity:  η² = 0.14
CI method:        η² = 0.08
Design filter:    η² = 0.05
```

Key question answered: "Is this MA fragile because of model choice or because the evidence itself is weak?"

#### 6.2.4 M4: Fragility Boundaries

Per-dimension tipping points:
- **Direction boundary**: where sign(θ) flips
- **Significance boundary**: where p crosses α
- **Clinical boundary**: where |θ| crosses MCID

Example: "Trim-and-fill flips significance in 34% of specs." "Removing Study X breaks significance in all estimators."

#### 6.2.5 M5: Evidence Landscape Synthesis

Structured MES verdict combining M1-M4:

```
MES VERDICT
├── Overall: FRAGILE (C_sig = 62.3%)
├── Conditional: ROBUST when low-RoB RCT only (C_sig = 94.1%)
├── Driver: bias correction (η² = 0.36)
├── Fragility: TF flips significance in 34% of specs
├── Prediction: PI crosses null in 71% of specs
└── Recommendation: Interpret with caution; high-quality evidence supports effect
```

#### 6.2.6 M6: TruthCert Certification

Bundle contents:
- Input data hash (SHA-256)
- MES_SPEC hash (the analysis plan)
- Full result tensor
- Evidence dossiers (from ASSESS)
- Landscape verdict + influence + boundaries
- Software version + execution timestamp
- Provenance chain (input → ASSESS → EXPLORE → MAP)

Certification rules:
- **PASS**: All specs executed, no silent failures, provenance chain complete
- **WARN**: >10% specs infeasible (data too sparse)
- **REJECT**: Broken provenance, memory-based claims, silent failures

### 6.3 Visualizations (HTML Frontend)

| Visualization | Type | Source |
|--------------|------|--------|
| Specification Curve | All specs ranked by θ, colored by significance, dimension indicators below | Adapted from MultiverseMA |
| Influence Sunburst | Nested rings: outer=dimensions, inner=levels, arc width=η² | Novel for MES |
| Decision Stability Map | Heatmap: rows=dimension levels, columns=concordance metrics | Novel for MES |
| Janus Plot | Effect size (x) vs -log10(p) (y), quadrant agreement zones | Adapted from MultiverseMA |
| Robustness Traffic Light | Large visual verdict: overall + conditional classifications | Adapted from Al-Mizan |
| Fragility Boundary Plot | Per-dimension tipping points | Novel for MES |

---

## 7. HTML Frontend: Tab Structure

| Tab | Name | Content |
|-----|------|---------|
| 1 | Data Input | CSV/JSON import, manual entry, built-in datasets. Study metadata: design, RoB, year, sample size. |
| 2 | Evidence Assessment | RoB traffic light, design classification matrix, bias profile radar, overlap heatmap. |
| 3 | Multiverse Explorer | Dimension selector (toggle estimators, CI, bias, quality). Live specification curve + Janus plot. Concordance heatmap. |
| 4 | Evidence Landscape | Robustness classification + traffic light verdict. Fragility boundaries, influence sunburst, decision stability map. Prediction intervals. |
| 5 | Report & Certify | Auto-generated methods text + R code export. TruthCert bundle (SHA-256 provenance). Print-ready report. GRADE-MES mapping. |

### 7.1 Built-in Datasets

3 exemplar datasets covering the robustness spectrum:
1. **Robust example**: A Cochrane review where MES confirms stability (C_sig ≥ 90%)
2. **Fragile example**: A review where conclusion depends on analytical choices (C_sig 50-69%)
3. **Unstable example**: A review where no defensible single conclusion exists (C_sig < 50%)

Selected from Fragility Atlas results for maximum pedagogical value.

### 7.2 Shared Libraries (extracted from existing portfolio)

- `stats-utils.js` — normalCDF, tCDF, tQuantile, betaIncomplete, chi2CDF, lnGamma
- `meta-engine.js` — DL/REML/PM/SJ/ML/FE τ² estimation + pooling
- `effect-sizes.js` — logOR, logRR, logHR, SMD (Hedges' g), RD, Fisher-z
- `truthcert.js` — SHA-256 provenance, bundle export/import
- `seeded-rng.js` — xoshiro128** deterministic PRNG

---

## 8. Publication Strategy

### 8.1 Paper 1: BMJ Flagship

**Title**: "Multiverse Evidence Synthesis: A Framework for Robust Meta-Analysis Applied to 407 Cochrane Reviews"

**Target**: BMJ (Research Article). Backup: Annals of Internal Medicine.

**Structure**:
- Introduction: The crisis (fragility, prediction gap, model dependence)
- Methods §1: MES framework (ASSESS → EXPLORE → MAP)
- Methods §2: Specification space (6 dimensions, 648+ specs)
- Methods §3: Robustness classification + conditional robustness
- Methods §4: Application to Pairwise70 dataset (407 reviews, k≥3)
- Results §1: Traditional vs MES — how many conclusions change?
- Results §2: Robustness distribution across 407 reviews
- Results §3: Influence decomposition — what drives fragility?
- Results §4: Conditional robustness — the quality signal
- Results §5: Case studies (3 exemplar reviews)
- Discussion: Implications for guideline panels, GRADE integration
- Conclusion: "Report landscapes, not point estimates"

**Headline finding**: "Traditional meta-analysis reported X% as significant. MES finds only Y% are Robust — and reveals that Z% become Robust when restricted to high-quality evidence, suggesting the signal is real but obscured by noise."

### 8.2 Paper 2: F1000 Software Paper

**Title**: "MES: An Open-Access Tool for Multiverse Evidence Synthesis in Meta-Analysis"

**Target**: F1000Research (Software Tool Article). Backup: JOSS.

**Covers**: Implementation details, browser + Python dual architecture, API reference, R parity validation, worked examples (3 built-in datasets), performance benchmarks, TruthCert certification, test suite documentation.

### 8.3 Relationship to Existing Papers

| Paper | Role in MES narrative |
|-------|----------------------|
| Fragility Atlas (BMJ) | Documents the PROBLEM — cited as primary motivation |
| BiasForensics (RSM) | Documents bias prevalence — justifies bias correction dimension |
| PredictionGap (BMJ/AIM) | Documents prediction failure — justifies PI as reporting metric |
| MultiverseMA (F1000) | Introduced browser multiverse for MA — MES extends with quality dimensions |

**Publication sequence**: Fragility Atlas → BiasForensics → PredictionGap → MultiverseMA → **MES (capstone)**

---

## 9. Validation Strategy

### 9.1 R Parity Gate

Random 20 reviews from Pairwise70 → compare MES Python results against:
- `metafor::rma()` for each estimator (DL, REML, PM, SJ, ML)
- `metafor::trimfill()` for trim-and-fill
- `metafor::regtest()` for Egger's regression
- `metafor::selmodel()` for selection models (metafor 4.0+)
- Tolerance: |Δ| < 1e-6 for θ, SE, τ² (DL, REML, PM, SJ, ML, FE, trim-and-fill, Egger). Relaxed tolerance |Δ| < 1e-3 for selection models (optimizer differences between scipy and R are expected).

### 9.2 Batch Empirical Validation

- 407 Cochrane reviews (Pairwise70, k≥3)
- Full MES pipeline: ASSESS → EXPLORE → MAP
- Compare traditional conclusion vs MES verdict
- Reuse existing validated results where possible (Fragility Atlas specs, BiasForensics profiles, PredictionGap PIs)

### 9.3 HTML Frontend Testing

- Selenium test suite: 30+ tests covering all tabs, all visualizations, all export formats
- R parity via WebR in-browser validation button
- 3 built-in datasets for manual verification

---

## 10. Project Structure

```
C:\Models\MES\
├── mes-core/                    # Python package
│   ├── assess/
│   │   ├── rob_scorer.py
│   │   ├── design_classifier.py
│   │   ├── bias_profiler.py
│   │   ├── overlap_detector.py
│   │   └── evidence_dossier.py
│   ├── explore/
│   │   ├── spec_generator.py
│   │   ├── estimators.py
│   │   ├── ci_methods.py
│   │   ├── bias_corrections.py
│   │   ├── executor.py
│   │   └── result_tensor.py
│   ├── map/
│   │   ├── concordance.py
│   │   ├── classifier.py
│   │   ├── influence.py
│   │   ├── boundaries.py
│   │   ├── landscape.py
│   │   └── certifier.py
│   ├── io/
│   │   ├── rda_reader.py
│   │   ├── csv_reader.py
│   │   └── exporter.py
│   └── tests/
│       ├── test_assess.py
│       ├── test_explore.py
│       ├── test_map.py
│       └── test_r_parity.py
├── app/
│   └── mes-app.html             # Single-file HTML frontend
├── validation/
│   ├── batch_run.py             # 407-review pipeline
│   ├── r_parity/                # R comparison scripts
│   └── results/                 # Batch output
├── paper/
│   ├── mes_bmj.md               # BMJ manuscript
│   └── mes_f1000.md             # F1000 software paper
├── data/
│   └── built-in/                # 3 exemplar datasets
└── docs/
    └── api.md                   # Python API reference
```

---

## 11. Implementation Phases

### Phase A: Python Engine (core)
- Build ASSESS + EXPLORE + MAP as Python package
- R parity gate on 20 reviews
- Batch run on 407 reviews → results for paper

### Phase B: HTML Frontend
- Single-file HTML app (mes-app.html)
- 5 tabs, 6 visualizations, built-in datasets
- Selenium test suite (30+)

### Phase C: Manuscripts
- BMJ flagship — framework + 407-review results
- F1000 software — tool documentation + validation

### Phase D: Ship
- GitHub repo, Zenodo DOI, TruthCert bundle
- Submit BMJ → F1000

---

## 12. Non-Goals (YAGNI)

- Network meta-analysis integration (future work)
- IPD-level multiverse (future work)
- Diagnostic test accuracy multiverse (future work)
- Automated RoB scoring (requires NLP/LLM — out of scope for v1)
- Real-time collaboration features
- Cloud deployment (offline-first)

---

## 13. Risk Mitigation

| Risk | Mitigation |
|------|-----------|
| Combinatorial explosion with many dimensions | Smart feasibility pruning, bounded spec count reporting |
| Reviewer says "just run metafor" | R parity gate proves equivalence; MES adds dimensions R cannot |
| Reviewer says "subjective quality weighting" | Quality is a dimension, not a weight — no subjective judgments |
| Pairwise70 RDA files lack RoB data | Graceful degradation — quality dimension collapses, other dimensions still valid |
| Performance on large k (>50 studies) | numpy vectorization, Web Workers, progress callbacks |
| Existing papers not yet published when MES submitted | MES can reference preprints/Zenodo deposits; framework stands alone |
