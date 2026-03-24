# Asa (عصا موسى) — Forensic Data Integrity Screener

**Date:** 2026-03-24
**Status:** Design approved
**Target location:** `C:\Models\Asa\asa.html`
**Target publication:** Research Synthesis Methods / BMJ Evidence-Based Medicine

---

## 1. Purpose

Asa is a browser-based forensic data integrity screener for meta-analysis. It applies 4 statistical forensic methods — GRIM, SPRITE, statcheck, and Terminal Digit Analysis — as a sequential pipeline (the "Gauntlet") to detect fabricated, manipulated, or impossible reported statistics in study-level data.

**The gap it fills:** Publication bias detection (Egger, Begg, P-curve, etc.) asks "are studies missing?" Reproducibility auditing asks "can I replicate the pooled result?" Asa asks a different question: **"Is this study's data physically possible?"** No browser-based tool currently combines these forensic methods. This is world-first.

**Metaphor:** The Staff of Moses (Quran 7:117, 20:69). Pharaoh's magicians created illusions — ropes that appeared to be serpents. Moses' staff became a real serpent that swallowed the fakes. Asa does the same: studies enter as serpents, the Staff's 4 fangs examine each, illusions are consumed, truth survives.

---

## 2. Architecture

- **Single HTML file** (~5-8K lines at MVP, up to 15K at maturity)
- No backend, no build step, no dependencies — fully offline
- Module pattern: sections wrapped in `<div class="module-*" data-module="N">`
- localStorage for session persistence (key prefix: `asa_`)
- Seeded PRNG (xoshiro128**) for any randomization
- Dark mode default (arena aesthetic), light mode toggle

---

## 3. Input

### 3.1 Primary: CSV / Paste

Upload or paste a CSV with study-level summary statistics. Column mapping is flexible (auto-detect common aliases):

| Field | Required by | Aliases |
|-------|------------|---------|
| study | all | study, author, name, label, studlab |
| n_t | GRIM, SPRITE | n_t, n1, n_treatment, n_exp |
| n_c | GRIM, SPRITE | n_c, n2, n_control |
| mean_t | GRIM, SPRITE | mean_t, m1, mean_treatment |
| mean_c | GRIM, SPRITE | mean_c, m2, mean_control |
| sd_t | SPRITE | sd_t, s1, sd_treatment |
| sd_c | SPRITE | sd_c, s2, sd_control |
| items | GRIM, SPRITE | items, n_items, scale_items (default: 1) |
| scale_min | SPRITE | scale_min, min_val (default: 1) |
| scale_max | SPRITE | scale_max, max_val (required for SPRITE) |
| data_type | GRIM | data_type: "discrete" or "continuous" (default: "discrete") |
| test_stat | statcheck | test_stat, test_statistic (NOT bare "t"/"F" — ambiguous) |
| df | statcheck | df, df1 |
| df2 | statcheck (F-test) | df2 |
| p_reported | statcheck | p_value, p_reported (NOT bare "p" — ambiguous) |
| stat_type | statcheck | stat_type (t/F/chi2/z) |
| tailed | statcheck | tailed: 1 or 2 (default: 2) |

**Column alias priority:** When auto-detecting, prefer longer/more-specific aliases over short ambiguous ones. E.g., `p_value` matches before `p`. Always show the detected mapping and allow manual override before running.

**Input validation:** n must be positive integer (n >= 2). n=1 triggers SPRITE skip (SD undefined). Negative/zero/non-numeric values → row-level error, not crash.

Partial data is accepted — Asa runs whichever fangs have sufficient fields and marks the rest as SKIPPED. GRIM is automatically SKIPPED for rows where data_type = "continuous".

### 3.2 Secondary: Manual Entry

Row-by-row table with +Add Row / Delete Row. Same fields as CSV. Useful for spot-checking a single study.

### 3.3 Demo Datasets

Two built-in datasets loaded via buttons:

1. **Tutorial** — 12 synthetic studies designed to demonstrate each fang:
   - 4 with GRIM violations (impossible means given n)
   - 3 with SPRITE violations (impossible SD given n + mean)
   - 2 with statcheck mismatches (reported p != recomputed p)
   - 1 with TDA anomaly (non-uniform terminal digits)
   - 2 fully clean
   - Each annotated with inline explanation of what's wrong

2. **Case Study** — Real retraction data from the Fujii anesthesia fraud case (172 retracted papers). Carlisle (2012, *Anaesthesia*, doi:10.1111/j.1365-2044.2012.07128.x) demonstrated that Fujii's reported baseline distributions were statistically impossible. For Asa's demo, we use a subset of ~15 Fujii studies where GRIM and statcheck violations are independently documentable from the published tables (means impossible given n, p-values inconsistent with reported test statistics). Note: Carlisle's original method was baseline distribution comparison (Monte Carlo), which is distinct from GRIM/TDA — Asa tests the same data using different forensic methods.

---

## 4. The Gauntlet (4 Fangs)

Studies pass left-to-right through 4 sequential forensic gates. Each gate is a "fang" of the serpent.

### Fang 1: GRIM (Granularity-Related Inconsistency of Means)

**Reference:** Brown & Heathers (2017), doi:10.1177/1948550616673876

**Precondition:** GRIM only applies to discrete/granular data (integers, Likert items, counts). If `data_type = "continuous"`, GRIM is SKIPPED for that row. The UI must clearly communicate this: "GRIM tests whether a mean is mathematically possible given the sample size. It only applies to discrete data (counts, Likert scales, yes/no). For continuous measures (blood pressure, weight, lab values), GRIM does not apply."

**Logic:** For a sample of n observations on a scale with `items` items, the mean must be a multiple of `1 / (n * items)`. For a single-item scale (items=1) with n=10, only means ending in .0, .1, ..., .9 are possible. For a 5-item composite (items=5) with n=10, the effective denominator is 50, allowing finer granularity.

**Rounding strategy:** Following `scrutiny::grim_test()`, test under both rounding-up and rounding-down of the last decimal place. A mean passes if it is consistent under EITHER rounding direction. Detect decimal places from the string representation of the input (preserving trailing zeros — "2.50" = 2 decimals, "2.5" = 1 decimal). When input comes from CSV (where trailing zeros may be lost), allow the user to specify decimal precision per column.

**Implementation:**
```
function grimCheck(mean, n, items, decimals, rounding) {
  // Effective denominator accounts for multi-item scales
  const effectiveN = n * items;
  const scale = Math.pow(10, decimals);
  // Sum must be integer (or close, accounting for rounding)
  const sum = mean * effectiveN;
  const remainder = Math.abs(sum - Math.round(sum));
  // Test under both rounding directions
  if (rounding === 'up_or_down') {
    const granularity = 0.5 / scale;
    const sumUp = (mean + granularity) * effectiveN;
    const sumDown = (mean - granularity) * effectiveN;
    return isCloseToInteger(sum) || isCloseToInteger(sumUp) || isCloseToInteger(sumDown);
  }
  return remainder < 0.5 / scale;
}
```

**Inputs needed:** n, mean, items (default 1), decimal precision, data_type guard
**Output:** PASS / FAIL + explanation (e.g., "Mean 2.45 × n=10 = 24.5, not integer → IMPOSSIBLE")

### Fang 2: SPRITE (Sample Parameter Reconstruction via Iterative TEchniques)

**Reference:** Heathers et al. (2018), doi:10.7287/peerj.preprints.26968v1

**Logic:** Given n, mean, and SD, SPRITE attempts to reconstruct a valid dataset. If no dataset of size n can produce both the reported mean and SD simultaneously, the combination is flagged as impossible.

**Implementation:** Monte Carlo reconstruction approach:
1. Generate candidate datasets of size n that produce the target mean
2. Check if any can also match the target SD (within rounding tolerance)
3. If k iterations fail to find a match, flag as FAIL
4. Use seeded PRNG (xoshiro128**) for deterministic results

**Inputs needed:** n, mean, SD, items (default 1), scale_min, scale_max (REQUIRED — no default, user must specify or SPRITE is SKIPPED)
**Output:** PASS / FAIL + closest achievable SD + reconstruction details
**Config:** max_iterations (default 10000), rounding tolerance (0.5 units of last decimal)
**Precondition:** Same discrete-data guard as GRIM. Also requires scale_min and scale_max to bound the search space. If either is missing, SPRITE is SKIPPED with message: "SPRITE requires scale bounds (e.g., Likert 1-7). Set scale_min and scale_max in settings or input."

### Fang 3: statcheck

**Reference:** Epskamp & Nuijten (2016), doi:10.3758/s13428-015-0664-2

**Logic:** Recompute the p-value from the reported test statistic and degrees of freedom. Flag if the recomputed p doesn't match the reported p (within rounding tolerance), or if the reported p is on the wrong side of a significance threshold.

**Implementation:**
```
function statcheckVerify(statType, statValue, df1, df2, pReported, tailed) {
  tailed = tailed || 2;
  let pComputed;
  switch(statType) {
    case 't':
      pComputed = tailed === 1
        ? (1 - tCDF(Math.abs(statValue), df1))
        : 2 * (1 - tCDF(Math.abs(statValue), df1));
      break;
    case 'F': pComputed = 1 - fCDF(statValue, df1, df2); break;
    case 'chi2': pComputed = 1 - chi2CDF(statValue, df1); break;
    case 'z':
      pComputed = tailed === 1
        ? (1 - normalCDF(Math.abs(statValue)))
        : 2 * (1 - normalCDF(Math.abs(statValue)));
      break;
  }
  // Rounding-aware comparison: recompute at stat ± 0.5 units of last decimal
  const match = roundingAwareMatch(pComputed, pReported, statValue, statType, df1, df2, tailed);
  // Check standard thresholds: 0.05, 0.01, 0.001, 0.10
  const decisionError = checkThresholds(pComputed, pReported, [0.10, 0.05, 0.01, 0.001]);
  return { pComputed, match, decisionError };
}
```

**Rounding-aware matching:** Following the R `statcheck` approach, recompute p at the test statistic rounded to the reported number of decimal places, then check if the reported p could be the correctly rounded result of pComputed. This handles the common case where t(48)=2.04, p=.047 is correct under one rounding but appears as p=.046 under another.

**Inequality handling:** If p_reported is entered as "<0.05" or ">0.10" (string prefix), statcheck verifies the inequality direction rather than exact match. E.g., reported "p < .05" passes if pComputed < 0.05, fails if pComputed >= 0.05.

Requires pure-JS implementations of t, F, chi-squared, and normal CDFs (regularized incomplete beta / gamma functions). These exist in the user's other tools (PairwisePro has t/chi2/normal CDFs, IPD-Meta-Pro has the full suite). Port and verify.

**Inputs needed:** test statistic type, test statistic value, df (df2 for F), reported p-value (exact or inequality), tailed (1 or 2, default 2)
**Output:** PASS / WARN (rounding) / FAIL (mismatch) / DECISION_ERROR (crosses a standard threshold)

### Fang 4: Terminal Digit Analysis (TDA)

**Reference:** Mosimann et al. (2002), doi:10.1080/08989620212969

**Logic:** The last digit of legitimately reported statistics should be approximately uniformly distributed (0-9). Fabricated data often shows non-uniform terminal digits because humans are bad at generating random-looking numbers.

**Scope: DATASET-LEVEL ONLY.** TDA does not contribute to per-study verdicts. It requires a sufficient pool of terminal digits to have statistical power (minimum ~50 digits for a meaningful chi-squared test with 9 df). A typical study row has 6-10 numeric fields — far too few for per-study testing. Instead, TDA runs across the entire dataset and reports a global assessment displayed in the Serpent's Eye as a separate panel (not a column in the per-study heatmap).

**Implementation:**
1. Extract terminal digits from all numeric values across ALL studies in the dataset
2. Compute observed frequency of digits 0-9
3. Chi-squared goodness-of-fit test against uniform distribution (df=9)
4. Report global result: PASS (p > 0.05) / WARN (0.01 < p <= 0.05) / FAIL (p <= 0.01)
5. Show per-digit bar chart with expected vs observed frequencies
6. If dataset has < 50 total digits, TDA is SKIPPED with explanation

**Inputs needed:** Any numeric values (means, SDs, ns, test stats, p-values) — collected from ALL rows
**Output:** Chi-squared statistic, p-value, per-digit frequencies, bar chart visualization
**Note:** TDA is suggestive, not conclusive. A significant result could indicate fabrication OR systematic rounding conventions in a field. The UI must display this caveat.

---

## 5. Verdict Logic

Each study gets a per-fang result for Fangs 1-3: PASS / WARN / FAIL / SKIPPED. TDA (Fang 4) is dataset-level only and does not contribute to per-study verdicts.

**Fang certainty tiers:**
- **Tier A (Conclusive):** GRIM, SPRITE — a FAIL means the data is mathematically impossible. No ambiguity.
- **Tier B (Strong):** statcheck DECISION_ERROR — reported p crosses a significance threshold vs recomputed p. Likely error or manipulation.
- **Tier C (Moderate):** statcheck FAIL (mismatch but no threshold crossing) — could be typo, rounding, or transcription error.

**Aggregate per-study verdict:**
- **PASS** all applicable fangs → Green. "Truth confirmed."
- **FAIL** any Tier A fang (GRIM or SPRITE) → Red (SWALLOWED). A single conclusive impossibility is sufficient. "Illusion detected."
- **FAIL** statcheck with DECISION_ERROR → Amber-Red (SWALLOWED if combined with any other WARN/FAIL). "Critical inconsistency."
- **FAIL** statcheck only (no decision error) → Amber (SUSPICIOUS). Survives with flag. "Requires investigation — possible typo or rounding."
- **All fangs SKIPPED** → Grey. Not assessed.

**Dataset-level verdict (TDA):**
- Displayed as a separate panel, not per-study
- PASS / WARN / FAIL based on chi-squared p-value
- Contextualizes the per-study results: "This dataset's terminal digit distribution is [consistent with / suspicious of / inconsistent with] authentic data."

**Important caveat (displayed in UI):** Forensic flags are evidence for investigation, not proof of fraud. GRIM/SPRITE failures prove mathematical impossibility — but the cause could be transcription error, not fabrication. statcheck mismatches may be typos. Always investigate flagged studies before drawing conclusions.

---

## 6. Output

### 6.1 The Gauntlet View (Tab 2)

Animated sequential examination:
- Studies listed on the left as rows
- 4 fang columns in the center
- Each study "flows" through the fangs left-to-right
- PASS: green checkmark, study continues
- FAIL: study animates downward into a "consumed" area with explanation tooltip
- After all studies processed: summary counts shown

### 6.2 Serpent's Eye Heatmap (Tab 3)

Matrix overview:
- Rows: studies (sorted by severity — worst at top)
- Columns: 4 fangs
- Cells: colored PASS (green) / WARN (amber) / FAIL (red) / SKIPPED (grey)
- Click any cell → expands forensic detail for that study × fang combination
- Row highlight on hover, column sorting

### 6.3 Forensic Report Cards (Tab 4)

Per-study expandable cards:
- Study name + overall verdict badge
- Per-fang section with:
  - What was tested
  - What was found
  - Why it's flagged (or why it passed)
  - Mathematical detail (e.g., "Mean 2.45 with n=10: sum = 24.5, not integer → IMPOSSIBLE")

### 6.4 Export (Tab 5)

- **Cleaned CSV:** Original data + verdict columns (grim_result, sprite_result, statcheck_result, tda_result, overall_verdict)
- **Flagged-only CSV:** Just the suspicious/swallowed studies
- **TruthCert receipt:** JSON with input hash (SHA-256), tests run, per-study verdicts, timestamp, Asa version
- **Report (HTML):** Printable forensic report for all studies

---

## 7. UI Structure

### Tab 1: Data Entry
- Textarea for CSV paste (with column auto-detection preview)
- File upload button (.csv)
- Manual entry table (+Add Row / Delete Row)
- Demo dataset buttons: [Tutorial] [Case Study: Fujii]
- Column mapping preview (shows detected → expected field mapping)
- [Run Gauntlet] button

### Tab 2: The Gauntlet
- Animated view (runs once, replayable)
- Progress bar showing current study being examined
- Left panel: incoming studies
- Center: 4 fang gates with pass/fail indicators
- Right panel: surviving studies
- Bottom panel: consumed studies with reasons
- Summary stats: X passed, Y suspicious, Z swallowed

### Tab 3: Serpent's Eye
- Heatmap matrix (studies × fangs)
- Color legend
- Sort controls (by study name, by severity, by specific fang)
- Filter controls (show only failures, show all)
- Click-to-expand detail

### Tab 4: Forensic Reports
- Accordion-style per-study cards
- Expand all / collapse all
- Search/filter by study name
- Print-friendly layout

### Tab 5: Export
- Download buttons for each format
- TruthCert receipt preview
- Hash verification display

### Global
- Dark mode default, light mode toggle
- Quranic verse header: Quran 7:117 — "So the truth was established, and what they did was made void" (فَوَقَعَ الْحَقُّ وَبَطَلَ مَا كَانُوا يَعْمَلُونَ)
- Version display, settings gear

---

## 8. Statistical Distribution Functions

Asa needs pure-JS implementations of:

1. **Normal CDF** — for z-test p-values and TDA chi-squared
2. **t-distribution CDF** — for t-test p-values (statcheck)
3. **F-distribution CDF** — for F-test p-values (statcheck)
4. **Chi-squared CDF** — for chi-squared test p-values (statcheck, TDA)

These require:
- **Regularized incomplete beta function** (I_x(a,b)) — for t and F CDFs
- **Regularized lower incomplete gamma function** (P(a,x)) — for chi-squared CDF
- **Error function (erf)** — for normal CDF

All exist in the user's portfolio (PairwisePro has t/chi2/normal CDFs, IPD-Meta-Pro has the full suite). Port and verify.

---

## 9. Testing Strategy

### Unit tests (embedded or Selenium)
- GRIM: known-pass cases (n=20, mean=3.15 → PASS), known-fail cases (n=10, mean=2.45 → FAIL)
- SPRITE: known-possible (n=10, mean=3.5, SD=1.2, scale 1-5 → PASS), known-impossible combinations
- statcheck: recompute known t/F/chi2 → p-value pairs, verify match within tolerance
- TDA: uniform digits → PASS, fabricated pattern (excess 0s and 5s) → FAIL

### Integration tests
- Load tutorial dataset → verify exactly 4 GRIM fails, 3 SPRITE fails, 2 statcheck mismatches, 1 TDA anomaly
- Load Fujii dataset → verify known violations are caught
- Export cleaned CSV → verify flagged studies are marked correctly
- TruthCert receipt → verify hash matches input data

### Validation against reference implementations
- GRIM: compare against R `scrutiny::grim_test()` (test with items=1 and items>1, both rounding modes)
- SPRITE: compare against R `scrutiny::sprite()` or Python `pysprite` (known-impossible and known-possible cases)
- statcheck: compare against R `statcheck::statcheck()` (two-tailed, one-tailed, inequality inputs)
- TDA: compare against manual chi-squared calculation on known digit distributions

---

## 10. Deferred (v2.0+)

- **GRIMMER** (Anaya 2016) — analytic SD consistency check, cheaper than SPRITE. Natural Fang 1b between GRIM and SPRITE. Add as companion to GRIM for v1.1.
- **Carlisle baseline distribution analysis** — Monte Carlo comparison of observed vs expected distributions of baseline characteristics across a set of RCTs. The most powerful multi-study forensic method. Would be a 5th fang.
- **Benford's Law analysis** — first-digit distribution test (needs large datasets for power)
- **Cross-study duplication detection** — pairwise comparison matrix for identical baselines
- **PDF ingestion** — extract statistics directly from uploaded papers (leverage RCT Extractor patterns)
- **Integration with PairwisePro** — "Check integrity" button that sends data to Asa, returns verdicts
- **Batch mode** — run across multiple Cochrane reviews (like BiasForensics pipeline)
- **WebR validation tier** — cross-check against R scrutiny/statcheck packages

---

## 11. File Location

- **Source:** `C:\Models\Asa\asa.html`
- **Tests:** `C:\Models\Asa\tests\` (Selenium)
- **Demo data:** Embedded in HTML (tutorial + Fujii case study)
- **GitHub:** `mahmood726-cyber/asa` (private until publication)
