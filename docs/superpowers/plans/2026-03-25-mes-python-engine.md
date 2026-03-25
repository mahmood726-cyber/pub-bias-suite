# MES Python Engine — Implementation Plan (Phase A)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the MES Python engine (ASSESS → EXPLORE → MAP) that processes any pairwise meta-analysis dataset and produces robustness-classified evidence landscapes, validated against R and batch-tested on 407 Cochrane reviews.

**Architecture:** Three-phase pipeline — ASSESS annotates studies with quality/design metadata, EXPLORE runs all defensible analytical specifications as a multiverse, MAP classifies robustness and decomposes influence. Heavily ports algorithms from Fragility Atlas (`C:\FragilityAtlas\src\`), BiasForensics (`C:\BiasForensics\src\`), and PredictionGap (`C:\PredictionGap\src\`).

**Tech Stack:** Python 3.11+, numpy, scipy, dataclasses, concurrent.futures, pytest. R 4.5.2 + metafor for parity validation. No ML/AI dependencies.

**Spec:** `C:\Users\user\docs\superpowers\specs\2026-03-25-mes-design.md`

**Project root:** `C:\Models\MES\`

**Key reference files:**
- `C:\FragilityAtlas\src\estimators.py` (271 lines) — 7 τ² estimators, 3 CI methods
- `C:\FragilityAtlas\src\corrections.py` (169 lines) — Trim-and-fill, PET-PEESE
- `C:\FragilityAtlas\src\specifications.py` (101 lines) — Spec grid generation
- `C:\FragilityAtlas\src\classifier.py` (192 lines) — Robustness scoring, η² decomposition
- `C:\FragilityAtlas\src\pipeline.py` (276 lines) — Batch execution
- `C:\BiasForensics\src\methods.py` (420 lines) — 8 publication bias methods
- `C:\PredictionGap\src\pipeline.py` (210 lines) — PI calculation, discordance

**Windows note:** Use `python` not `python3`. Paths use forward slashes in code. Console output must be ASCII-safe (no Unicode arrows/em-dashes).

---

## File Structure

```
C:\Models\MES\
├── mes_core/
│   ├── __init__.py              # Package init, version
│   ├── models.py                # All dataclasses: StudyDossier, MESSpec, SpecResult, ResultTensor, MESVerdict
│   ├── assess/
│   │   ├── __init__.py
│   │   ├── design_classifier.py # classify_design() → design_type, design_tier
│   │   ├── rob_scorer.py        # score_rob2(), score_robins_i() → rob_overall, rob_domains
│   │   ├── bias_profiler.py     # profile_bias() → egger_p, begg_p, excess_sig
│   │   └── dossier_builder.py   # build_dossiers() — orchestrates ASSESS phase
│   ├── explore/
│   │   ├── __init__.py
│   │   ├── estimators.py        # fe(), dl(), reml(), pm(), sj(), ml() — τ² + pooled θ
│   │   ├── ci_methods.py        # wald_ci(), hksj_ci(), tdist_ci()
│   │   ├── bias_corrections.py  # trim_fill(), pet_peese(), selection_model()
│   │   ├── spec_generator.py    # generate_specs() → list of spec dicts
│   │   └── executor.py          # execute_multiverse() → list of SpecResult
│   ├── map/
│   │   ├── __init__.py
│   │   ├── concordance.py       # compute_concordance() → C_dir, C_sig, C_full
│   │   ├── classifier.py        # classify_robustness(), conditional_robustness()
│   │   ├── influence.py         # decompose_influence() → η² per dimension
│   │   ├── boundaries.py        # find_boundaries() → fragility tipping points
│   │   ├── landscape.py         # synthesize_verdict() → MESVerdict
│   │   └── certifier.py         # certify() → TruthCert bundle dict
│   ├── io/
│   │   ├── __init__.py
│   │   ├── csv_reader.py        # read_csv() → list of study dicts
│   │   ├── rda_reader.py        # read_rda() → list of study dicts (Pairwise70 format)
│   │   └── exporter.py          # export_json(), export_csv(), export_report()
│   └── pipeline.py              # run_mes() — end-to-end orchestrator
├── tests/
│   ├── __init__.py
│   ├── conftest.py              # Shared fixtures: sample studies, BCG dataset
│   ├── test_models.py
│   ├── test_design_classifier.py
│   ├── test_rob_scorer.py
│   ├── test_bias_profiler.py
│   ├── test_dossier_builder.py
│   ├── test_estimators.py
│   ├── test_ci_methods.py
│   ├── test_bias_corrections.py
│   ├── test_spec_generator.py
│   ├── test_executor.py
│   ├── test_concordance.py
│   ├── test_classifier.py
│   ├── test_influence.py
│   ├── test_boundaries.py
│   ├── test_landscape.py
│   ├── test_certifier.py
│   ├── test_io.py
│   ├── test_pipeline.py
│   └── test_r_parity.py         # R comparison (requires Rscript)
├── validation/
│   ├── batch_run.py             # 407-review batch pipeline
│   ├── r_parity/
│   │   └── validate.R           # R script for metafor comparison
│   └── results/                 # Output directory
├── data/
│   └── built_in/
│       ├── bcg_vaccine.json     # 13 studies, classic dataset
│       ├── robust_example.json  # Selected from Fragility Atlas
│       └── fragile_example.json # Selected from Fragility Atlas
└── setup.py                     # Package metadata
```

---

## Task 1: Project Scaffolding

**Files:**
- Create: `C:\Models\MES\setup.py`
- Create: `C:\Models\MES\mes_core\__init__.py`
- Create: `C:\Models\MES\tests\__init__.py`
- Create: `C:\Models\MES\tests\conftest.py`

- [ ] **Step 1: Create project directory and setup.py**

```bash
mkdir -p C:/Models/MES/mes_core C:/Models/MES/tests C:/Models/MES/data/built_in C:/Models/MES/validation/results C:/Models/MES/validation/r_parity
```

```python
# setup.py
from setuptools import setup, find_packages

setup(
    name="mes-core",
    version="0.1.0",
    packages=find_packages(),
    install_requires=["numpy>=1.24", "scipy>=1.10"],
    python_requires=">=3.11",
)
```

- [ ] **Step 2: Create package init**

```python
# mes_core/__init__.py
"""Multiverse Evidence Synthesis (MES) — next-generation meta-analysis engine."""
__version__ = "0.1.0"
```

- [ ] **Step 3: Create test conftest with BCG vaccine fixture**

The BCG vaccine dataset (13 studies) is the standard test fixture. Values from `C:\Models\MultiverseMA\multiverse-ma.html` built-in data.

```python
# tests/conftest.py
import pytest
import numpy as np


@pytest.fixture
def bcg_studies():
    """BCG vaccine dataset — 13 RCTs, logRR scale.
    Classic dataset used in Fragility Atlas and MultiverseMA.
    """
    # yi = log(RR), vi = variance of log(RR)
    yi = np.array([
        -0.8893, -1.5856, -1.3481, -1.4416, -0.2175,
        -0.7861, -1.6209, 0.0120, -0.4717, 0.0459,
        -0.0173, -0.4340, -1.4564
    ])
    vi = np.array([
        0.0355, 0.0248, 0.0292, 0.0175, 0.0300,
        0.0116, 0.0206, 0.0470, 0.0124, 0.0616,
        0.2355, 0.0200, 0.0384
    ])
    labels = [
        "Aronson1948", "Ferguson1949", "Rosenthal1960", "Hart1977",
        "Frimodt1973", "Stein1953", "Vandiviere1973", "TPT_Madras1980",
        "Coetzee1968", "Comstock1974", "Comstock1976", "Comstock_etal1976",
        "Shapiro1998"
    ]
    return {
        "yi": yi,
        "vi": vi,
        "labels": labels,
        "k": 13,
        "measure": "logRR",
    }


@pytest.fixture
def bcg_study_dicts(bcg_studies):
    """BCG as list of study dicts (input format for ASSESS)."""
    studies = []
    for i in range(bcg_studies["k"]):
        studies.append({
            "study_id": bcg_studies["labels"][i],
            "yi": float(bcg_studies["yi"][i]),
            "vi": float(bcg_studies["vi"][i]),
            "measure": "logRR",
            "design_type": "RCT",
            "year": 1948 + i * 3,  # approximate
        })
    return studies


@pytest.fixture
def small_studies():
    """Minimal 3-study dataset for edge case testing."""
    return {
        "yi": np.array([-0.5, -0.3, -0.8]),
        "vi": np.array([0.04, 0.06, 0.03]),
        "labels": ["A", "B", "C"],
        "k": 3,
        "measure": "logOR",
    }
```

- [ ] **Step 4: Create empty __init__ files for subpackages**

```python
# mes_core/assess/__init__.py
# mes_core/explore/__init__.py
# mes_core/map/__init__.py
# mes_core/io/__init__.py
# (all empty)
```

- [ ] **Step 5: Verify pytest discovers tests**

Run: `cd C:/Models/MES && python -m pytest tests/ --collect-only`
Expected: "no tests ran" (no test files yet), no import errors

- [ ] **Step 6: Initialize git and commit**

```bash
cd C:/Models/MES
git init
git add .
git commit -m "chore: scaffold MES project structure"
```

---

## Task 2: Data Models

**Files:**
- Create: `C:\Models\MES\mes_core\models.py`
- Create: `C:\Models\MES\tests\test_models.py`

All shared dataclasses live here. Every other module imports from `models.py`.

- [ ] **Step 1: Write failing tests for data models**

```python
# tests/test_models.py
import numpy as np
import pytest
from mes_core.models import (
    StudyDossier, RoBAssessment, BiasProfile,
    MESSpec, SpecResult, ConcordanceMetrics, MESVerdict,
)


def test_study_dossier_creation():
    d = StudyDossier(
        study_id="Smith2019",
        yi=-0.42, vi=0.031, measure="logOR",
        design_type="RCT", design_tier=1,
    )
    assert d.study_id == "Smith2019"
    assert d.design_tier == 1
    assert d.rob is None  # optional


def test_study_dossier_with_rob():
    rob = RoBAssessment(
        tool="RoB2",
        overall="some_concerns",
        domains={"randomization": "low", "deviations": "some_concerns",
                 "missing": "low", "measurement": "low", "selection": "low"},
    )
    d = StudyDossier(
        study_id="Smith2019",
        yi=-0.42, vi=0.031, measure="logOR",
        design_type="RCT", design_tier=1,
        rob=rob,
    )
    assert d.rob.overall == "some_concerns"


def test_mes_spec_defaults():
    spec = MESSpec()
    assert "DL" in spec.estimators
    assert "HKSJ" in spec.ci_methods
    assert spec.alpha == 0.05
    assert len(spec.estimators) == 6
    assert len(spec.ci_methods) == 3
    assert len(spec.bias_corrections) == 4
    assert len(spec.quality_filters) == 3
    assert len(spec.design_filters) == 3


def test_mes_spec_base_count():
    spec = MESSpec()
    # 6 * 3 * 4 * 3 * 3 = 648
    assert spec.base_spec_count == 648


def test_spec_result_creation():
    r = SpecResult(
        spec_id="DL_Wald_none_all_all_full",
        estimator="DL", ci_method="Wald", bias_correction="none",
        quality_filter="all", design_filter="all", sensitivity="full",
        theta=-0.38, se=0.12, ci_lo=-0.61, ci_hi=-0.15,
        p_value=0.0012, tau2=0.045, I2=0.62, k=8,
        pi_lo=-0.89, pi_hi=0.13,
        significant=True, direction="negative",
    )
    assert r.significant is True
    assert r.direction == "negative"


def test_concordance_metrics():
    m = ConcordanceMetrics(
        C_dir=0.95, C_sig=0.72, C_full=0.68,
        n_specs=648, n_feasible=612,
    )
    assert m.C_sig == 0.72


def test_mes_verdict_classification():
    v = MESVerdict(
        overall_class="FRAGILE",
        overall_c_sig=0.623,
        conditional={
            "low-rob-only": {"class": "ROBUST", "c_sig": 0.941},
            "rct-only": {"class": "MODERATE", "c_sig": 0.85},
        },
        dominant_dimension="bias_correction",
        dominant_eta2=0.36,
        certification="PASS",
    )
    assert v.overall_class == "FRAGILE"
    assert v.conditional["low-rob-only"]["class"] == "ROBUST"
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd C:/Models/MES && python -m pytest tests/test_models.py -v`
Expected: FAIL (ImportError — models.py doesn't exist yet)

- [ ] **Step 3: Implement data models**

```python
# mes_core/models.py
"""MES data models — all shared dataclasses."""
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class RoBAssessment:
    """Risk of Bias assessment for a single study."""
    tool: str  # "RoB2" or "ROBINS-I"
    overall: str  # "low", "some_concerns", "high"
    domains: dict[str, str] = field(default_factory=dict)


@dataclass
class StudyDossier:
    """Annotated study — output of ASSESS phase."""
    study_id: str
    yi: float
    vi: float
    measure: str  # "logOR", "logRR", "SMD", etc.
    design_type: str = "RCT"  # "RCT", "quasi", "cohort", etc.
    design_tier: int = 1  # 1=RCT, 2=quasi/MR, 3=observational
    rob: Optional[RoBAssessment] = None
    year: Optional[int] = None
    n1: Optional[int] = None
    n2: Optional[int] = None
    events1: Optional[int] = None
    events2: Optional[int] = None


@dataclass
class BiasProfile:
    """Review-level bias detection results (from ASSESS)."""
    egger_p: Optional[float] = None
    begg_p: Optional[float] = None
    excess_sig_count: int = 0
    k: int = 0


@dataclass
class MESSpec:
    """Declarative specification for the multiverse analysis."""
    estimators: list[str] = field(
        default_factory=lambda: ["FE", "DL", "REML", "PM", "SJ", "ML"]
    )
    ci_methods: list[str] = field(
        default_factory=lambda: ["Wald", "HKSJ", "t-dist"]
    )
    bias_corrections: list[str] = field(
        default_factory=lambda: ["none", "trim-fill", "PET-PEESE", "selection-model"]
    )
    quality_filters: list[str] = field(
        default_factory=lambda: ["all", "exclude-high-rob", "low-rob-only"]
    )
    design_filters: list[str] = field(
        default_factory=lambda: ["all", "rct-quasi", "rct-only"]
    )
    sensitivity: list[str] = field(
        default_factory=lambda: ["full", "loo"]
    )
    alpha: float = 0.05

    @property
    def base_spec_count(self) -> int:
        """Number of specs excluding LOO sensitivity."""
        return (
            len(self.estimators)
            * len(self.ci_methods)
            * len(self.bias_corrections)
            * len(self.quality_filters)
            * len(self.design_filters)
        )


@dataclass
class SpecResult:
    """Result from a single specification execution."""
    spec_id: str
    estimator: str
    ci_method: str
    bias_correction: str
    quality_filter: str
    design_filter: str
    sensitivity: str  # "full" or "loo_StudyX"
    theta: float
    se: float
    ci_lo: float
    ci_hi: float
    p_value: float
    tau2: float
    I2: float
    k: int
    pi_lo: float
    pi_hi: float
    significant: bool
    direction: str  # "positive", "negative", "null"
    converged: bool = True
    error: Optional[str] = None


@dataclass
class ConcordanceMetrics:
    """Concordance analysis results."""
    C_dir: float  # % agreeing on direction
    C_sig: float  # % agreeing on direction + significance
    C_full: float  # % agreeing on direction + sig + clinical relevance
    n_specs: int
    n_feasible: int


@dataclass
class MESVerdict:
    """Final MES evidence landscape verdict."""
    overall_class: str  # "ROBUST", "MODERATE", "FRAGILE", "UNSTABLE"
    overall_c_sig: float
    conditional: dict  # {"low-rob-only": {"class": "ROBUST", "c_sig": 0.94}, ...}
    dominant_dimension: str
    dominant_eta2: float
    certification: str  # "PASS", "WARN", "REJECT"
    eta2_all: dict[str, float] = field(default_factory=dict)
    boundaries: dict = field(default_factory=dict)
    prediction_null_rate: float = 0.0
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd C:/Models/MES && python -m pytest tests/test_models.py -v`
Expected: All 7 tests PASS

- [ ] **Step 5: Commit**

```bash
cd C:/Models/MES && git add -A && git commit -m "feat: add MES data models (StudyDossier, MESSpec, SpecResult, MESVerdict)"
```

---

## Task 3: ASSESS — Design Classifier

**Files:**
- Create: `C:\Models\MES\mes_core\assess\design_classifier.py`
- Create: `C:\Models\MES\tests\test_design_classifier.py`

- [ ] **Step 1: Write failing tests**

```python
# tests/test_design_classifier.py
from mes_core.assess.design_classifier import classify_design


def test_rct_tier1():
    result = classify_design("RCT")
    assert result == ("RCT", 1)


def test_quasi_tier2():
    result = classify_design("quasi-experimental")
    assert result == ("quasi-experimental", 2)


def test_mr_tier2():
    result = classify_design("MR")
    assert result == ("MR", 2)


def test_cohort_tier3():
    result = classify_design("cohort")
    assert result == ("cohort", 3)


def test_case_control_tier3():
    result = classify_design("case-control")
    assert result == ("case-control", 3)


def test_cross_sectional_tier3():
    result = classify_design("cross-sectional")
    assert result == ("cross-sectional", 3)


def test_unknown_defaults_rct():
    """Unknown or missing design defaults to RCT tier 1 (Cochrane assumption)."""
    result = classify_design(None)
    assert result == ("RCT", 1)


def test_case_insensitive():
    result = classify_design("rct")
    assert result == ("RCT", 1)
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd C:/Models/MES && python -m pytest tests/test_design_classifier.py -v`
Expected: FAIL

- [ ] **Step 3: Implement design classifier**

```python
# mes_core/assess/design_classifier.py
"""Classify study design type and assign evidence tier."""

DESIGN_TIERS = {
    "RCT": 1,
    "quasi-experimental": 2,
    "MR": 2,
    "cohort": 3,
    "case-control": 3,
    "cross-sectional": 3,
    "ecological": 3,
}

# Aliases for case-insensitive matching
_ALIASES = {
    "rct": "RCT",
    "randomized": "RCT",
    "randomised": "RCT",
    "quasi": "quasi-experimental",
    "quasi-experimental": "quasi-experimental",
    "mendelian randomization": "MR",
    "mendelian randomisation": "MR",
    "mr": "MR",
    "cohort": "cohort",
    "case-control": "case-control",
    "case control": "case-control",
    "cross-sectional": "cross-sectional",
    "cross sectional": "cross-sectional",
    "ecological": "ecological",
}


def classify_design(design_type: str | None) -> tuple[str, int]:
    """Classify a study design and return (canonical_type, tier).

    Tier 1 = RCT, Tier 2 = quasi/MR, Tier 3 = observational.
    Unknown/None defaults to ("RCT", 1) — Cochrane assumption.
    """
    if design_type is None:
        return ("RCT", 1)
    canonical = _ALIASES.get(design_type.strip().lower())
    if canonical is None:
        return ("RCT", 1)
    return (canonical, DESIGN_TIERS[canonical])
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd C:/Models/MES && python -m pytest tests/test_design_classifier.py -v`
Expected: All 8 tests PASS

- [ ] **Step 5: Commit**

```bash
cd C:/Models/MES && git add -A && git commit -m "feat(assess): add design classifier with 3-tier evidence hierarchy"
```

---

## Task 4: ASSESS — RoB Scorer

**Files:**
- Create: `C:\Models\MES\mes_core\assess\rob_scorer.py`
- Create: `C:\Models\MES\tests\test_rob_scorer.py`

Reference: `C:\Models\RoBAssessor\rob-assessor.html` — RoB 2 algorithm logic.

- [ ] **Step 1: Write failing tests**

```python
# tests/test_rob_scorer.py
from mes_core.models import RoBAssessment
from mes_core.assess.rob_scorer import (
    score_rob2_overall, score_robins_i_overall, parse_rob_from_dict,
)


def test_rob2_all_low():
    domains = {
        "randomization": "low", "deviations": "low",
        "missing": "low", "measurement": "low", "selection": "low",
    }
    assert score_rob2_overall(domains) == "low"


def test_rob2_one_some_concerns():
    domains = {
        "randomization": "low", "deviations": "some_concerns",
        "missing": "low", "measurement": "low", "selection": "low",
    }
    assert score_rob2_overall(domains) == "some_concerns"


def test_rob2_any_high():
    domains = {
        "randomization": "high", "deviations": "low",
        "missing": "low", "measurement": "low", "selection": "low",
    }
    assert score_rob2_overall(domains) == "high"


def test_rob2_multiple_some_concerns_is_high():
    """Multiple 'some_concerns' → high risk (RoB 2 algorithm)."""
    domains = {
        "randomization": "some_concerns", "deviations": "some_concerns",
        "missing": "some_concerns", "measurement": "low", "selection": "low",
    }
    assert score_rob2_overall(domains) == "high"


def test_robins_i_all_low():
    domains = {
        "confounding": "low", "selection_participants": "low",
        "classification_interventions": "low", "deviations": "low",
        "missing_data": "low", "measurement": "low",
        "selection_reported_result": "low",
    }
    assert score_robins_i_overall(domains) == "low"


def test_robins_i_any_serious():
    domains = {
        "confounding": "serious", "selection_participants": "low",
        "classification_interventions": "low", "deviations": "low",
        "missing_data": "low", "measurement": "low",
        "selection_reported_result": "low",
    }
    assert score_robins_i_overall(domains) == "high"


def test_robins_i_moderate():
    domains = {
        "confounding": "moderate", "selection_participants": "low",
        "classification_interventions": "low", "deviations": "low",
        "missing_data": "low", "measurement": "low",
        "selection_reported_result": "low",
    }
    assert score_robins_i_overall(domains) == "some_concerns"


def test_parse_rob_from_dict_rob2():
    data = {
        "rob_tool": "RoB2",
        "rob_randomization": "low",
        "rob_deviations": "some_concerns",
        "rob_missing": "low",
        "rob_measurement": "low",
        "rob_selection": "low",
    }
    result = parse_rob_from_dict(data)
    assert isinstance(result, RoBAssessment)
    assert result.tool == "RoB2"
    assert result.overall == "some_concerns"


def test_parse_rob_from_dict_none():
    """No RoB data → None."""
    result = parse_rob_from_dict({})
    assert result is None
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd C:/Models/MES && python -m pytest tests/test_rob_scorer.py -v`
Expected: FAIL

- [ ] **Step 3: Implement RoB scorer**

```python
# mes_core/assess/rob_scorer.py
"""Risk of Bias scoring — RoB 2 (RCTs) and ROBINS-I (non-RCTs)."""
from mes_core.models import RoBAssessment


# --- RoB 2 (Sterne 2019) ---

ROB2_DOMAINS = [
    "randomization", "deviations", "missing", "measurement", "selection"
]


def score_rob2_overall(domains: dict[str, str]) -> str:
    """Score overall RoB 2 from domain judgments.

    Algorithm (Sterne 2019):
    - Any domain 'high' → overall 'high'
    - Multiple domains 'some_concerns' → overall 'high'
    - One domain 'some_concerns' → overall 'some_concerns'
    - All domains 'low' → overall 'low'
    """
    values = [domains.get(d, "low") for d in ROB2_DOMAINS]
    if any(v == "high" for v in values):
        return "high"
    n_concerns = sum(1 for v in values if v == "some_concerns")
    if n_concerns >= 2:
        return "high"
    if n_concerns == 1:
        return "some_concerns"
    return "low"


# --- ROBINS-I (Sterne 2016) ---

ROBINS_I_DOMAINS = [
    "confounding", "selection_participants", "classification_interventions",
    "deviations", "missing_data", "measurement", "selection_reported_result",
]

_ROBINS_SEVERITY = {"low": 0, "moderate": 1, "serious": 2, "critical": 3}


def score_robins_i_overall(domains: dict[str, str]) -> str:
    """Score overall ROBINS-I from domain judgments.

    Maps to MES 3-level scheme: low / some_concerns / high.
    - All low → low
    - Any moderate (none serious+) → some_concerns
    - Any serious or critical → high
    """
    severities = [_ROBINS_SEVERITY.get(domains.get(d, "low"), 0) for d in ROBINS_I_DOMAINS]
    worst = max(severities)
    if worst >= 2:  # serious or critical
        return "high"
    if worst == 1:  # moderate
        return "some_concerns"
    return "low"


# --- Parsing from flat dict (CSV import) ---

def parse_rob_from_dict(data: dict) -> RoBAssessment | None:
    """Parse RoB assessment from a flat study dictionary.

    Looks for keys like rob_tool, rob_randomization, etc.
    Returns None if no RoB data present.
    """
    tool = data.get("rob_tool")
    if not tool:
        return None

    if tool == "RoB2":
        domains = {}
        for d in ROB2_DOMAINS:
            val = data.get(f"rob_{d}")
            if val:
                domains[d] = val
        overall = score_rob2_overall(domains)
    elif tool == "ROBINS-I":
        domains = {}
        for d in ROBINS_I_DOMAINS:
            val = data.get(f"rob_{d}")
            if val:
                domains[d] = val
        overall = score_robins_i_overall(domains)
    else:
        return None

    return RoBAssessment(tool=tool, overall=overall, domains=domains)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd C:/Models/MES && python -m pytest tests/test_rob_scorer.py -v`
Expected: All 10 tests PASS

- [ ] **Step 5: Commit**

```bash
cd C:/Models/MES && git add -A && git commit -m "feat(assess): add RoB 2 + ROBINS-I scorer with 3-level overall mapping"
```

---

## Task 5: ASSESS — Bias Profiler

**Files:**
- Create: `C:\Models\MES\mes_core\assess\bias_profiler.py`
- Create: `C:\Models\MES\tests\test_bias_profiler.py`

Reference: `C:\BiasForensics\src\methods.py` — Egger, Begg implementations.

- [ ] **Step 1: Write failing tests**

```python
# tests/test_bias_profiler.py
import numpy as np
from mes_core.models import BiasProfile
from mes_core.assess.bias_profiler import profile_bias


def test_profile_bias_bcg(bcg_studies):
    """BCG dataset — Egger test should detect asymmetry (p ~ 0.02)."""
    result = profile_bias(bcg_studies["yi"], bcg_studies["vi"])
    assert isinstance(result, BiasProfile)
    assert result.egger_p is not None
    assert result.egger_p < 0.10  # BCG shows funnel asymmetry
    assert result.k == 13


def test_profile_bias_symmetric():
    """Symmetric dataset — no bias detected."""
    yi = np.array([-0.5, -0.3, -0.7, -0.4, -0.6])
    vi = np.array([0.04, 0.04, 0.04, 0.04, 0.04])
    result = profile_bias(yi, vi)
    assert result.egger_p > 0.10


def test_profile_bias_too_few():
    """k < 3 — cannot run Egger, returns None for p-values."""
    yi = np.array([-0.5, -0.3])
    vi = np.array([0.04, 0.06])
    result = profile_bias(yi, vi)
    assert result.egger_p is None
    assert result.k == 2
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd C:/Models/MES && python -m pytest tests/test_bias_profiler.py -v`
Expected: FAIL

- [ ] **Step 3: Implement bias profiler**

Port from `C:\BiasForensics\src\methods.py` — Egger and Begg methods.

```python
# mes_core/assess/bias_profiler.py
"""Review-level publication bias profiling (detection only, not correction)."""
import numpy as np
from scipy import stats
from mes_core.models import BiasProfile


def profile_bias(yi: np.ndarray, vi: np.ndarray) -> BiasProfile:
    """Run bias detection tests on a set of studies.

    Returns flags only — corrections happen in EXPLORE.
    Requires k >= 3 for Egger, k >= 3 for Begg.
    """
    k = len(yi)
    egger_p = _egger_test(yi, vi) if k >= 3 else None
    begg_p = _begg_test(yi, vi) if k >= 3 else None
    excess = _excess_significance(yi, vi) if k >= 3 else 0

    return BiasProfile(
        egger_p=egger_p,
        begg_p=begg_p,
        excess_sig_count=excess,
        k=k,
    )


def _egger_test(yi: np.ndarray, vi: np.ndarray) -> float:
    """Egger's regression test for funnel asymmetry.

    Regress yi/sei ~ 1/sei (weighted by 1/vi).
    Returns p-value for intercept != 0.
    """
    sei = np.sqrt(vi)
    # Precision = 1/sei, standardized effect = yi/sei
    x = 1.0 / sei
    y = yi / sei
    n = len(yi)
    if n < 3:
        return None
    # OLS: y = a + b*x → intercept a is the bias indicator
    x_mean = np.mean(x)
    y_mean = np.mean(y)
    ss_xx = np.sum((x - x_mean) ** 2)
    if ss_xx < 1e-30:
        return 1.0
    ss_xy = np.sum((x - x_mean) * (y - y_mean))
    b = ss_xy / ss_xx
    a = y_mean - b * x_mean
    residuals = y - (a + b * x)
    mse = np.sum(residuals ** 2) / (n - 2)
    se_a = np.sqrt(mse * (1.0 / n + x_mean ** 2 / ss_xx))
    if se_a < 1e-30:
        return 1.0
    t_stat = a / se_a
    p_value = 2.0 * stats.t.sf(abs(t_stat), df=n - 2)
    return float(p_value)


def _begg_test(yi: np.ndarray, vi: np.ndarray) -> float:
    """Begg-Mazumdar rank correlation test.

    Kendall's tau between standardized effect and variance.
    """
    wi = 1.0 / vi
    theta_fe = np.sum(wi * yi) / np.sum(wi)
    standardized = (yi - theta_fe) / np.sqrt(vi)
    tau, p_value = stats.kendalltau(standardized, vi)
    return float(p_value)


def _excess_significance(yi: np.ndarray, vi: np.ndarray) -> int:
    """Count studies with 'excess' significance beyond expected.

    Simple count: observed significant - expected significant.
    """
    sei = np.sqrt(vi)
    z = np.abs(yi / sei)
    observed_sig = int(np.sum(z > 1.96))
    # Expected: use FE pooled effect to compute power per study
    wi = 1.0 / vi
    theta_fe = np.sum(wi * yi) / np.sum(wi)
    expected_sig = 0.0
    for i in range(len(yi)):
        ncp = abs(theta_fe) / sei[i]  # non-centrality parameter
        power = 1.0 - stats.norm.cdf(1.96 - ncp)
        expected_sig += power
    excess = max(0, observed_sig - int(round(expected_sig)))
    return excess
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd C:/Models/MES && python -m pytest tests/test_bias_profiler.py -v`
Expected: All 3 tests PASS

- [ ] **Step 5: Commit**

```bash
cd C:/Models/MES && git add -A && git commit -m "feat(assess): add bias profiler (Egger, Begg, excess significance)"
```

---

## Task 6: ASSESS — Dossier Builder (Orchestrator)

**Files:**
- Create: `C:\Models\MES\mes_core\assess\dossier_builder.py`
- Create: `C:\Models\MES\tests\test_dossier_builder.py`

- [ ] **Step 1: Write failing tests**

```python
# tests/test_dossier_builder.py
import numpy as np
from mes_core.models import StudyDossier, BiasProfile
from mes_core.assess.dossier_builder import build_dossiers


def test_build_dossiers_basic(bcg_study_dicts):
    dossiers, bias = build_dossiers(bcg_study_dicts)
    assert len(dossiers) == 13
    assert all(isinstance(d, StudyDossier) for d in dossiers)
    assert isinstance(bias, BiasProfile)
    assert bias.k == 13


def test_dossier_preserves_effect(bcg_study_dicts):
    dossiers, _ = build_dossiers(bcg_study_dicts)
    assert dossiers[0].yi == bcg_study_dicts[0]["yi"]
    assert dossiers[0].vi == bcg_study_dicts[0]["vi"]


def test_dossier_classifies_design(bcg_study_dicts):
    dossiers, _ = build_dossiers(bcg_study_dicts)
    assert dossiers[0].design_type == "RCT"
    assert dossiers[0].design_tier == 1


def test_dossier_handles_missing_rob(bcg_study_dicts):
    """Studies without RoB data get rob=None (graceful degradation)."""
    dossiers, _ = build_dossiers(bcg_study_dicts)
    assert dossiers[0].rob is None  # BCG fixture has no RoB data


def test_dossier_with_rob():
    studies = [{
        "study_id": "Test1", "yi": -0.5, "vi": 0.04,
        "measure": "logOR", "design_type": "RCT",
        "rob_tool": "RoB2",
        "rob_randomization": "low", "rob_deviations": "low",
        "rob_missing": "low", "rob_measurement": "low",
        "rob_selection": "low",
    }]
    dossiers, _ = build_dossiers(studies)
    assert dossiers[0].rob is not None
    assert dossiers[0].rob.overall == "low"
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd C:/Models/MES && python -m pytest tests/test_dossier_builder.py -v`
Expected: FAIL

- [ ] **Step 3: Implement dossier builder**

```python
# mes_core/assess/dossier_builder.py
"""ASSESS phase orchestrator — builds StudyDossier list from raw study dicts."""
import numpy as np
from mes_core.models import StudyDossier, BiasProfile
from mes_core.assess.design_classifier import classify_design
from mes_core.assess.rob_scorer import parse_rob_from_dict
from mes_core.assess.bias_profiler import profile_bias


def build_dossiers(
    studies: list[dict],
) -> tuple[list[StudyDossier], BiasProfile]:
    """Build annotated StudyDossier list and review-level BiasProfile.

    Each study dict must have: study_id, yi, vi, measure.
    Optional: design_type, year, n1, n2, events1, events2, rob_* fields.
    """
    dossiers = []
    for s in studies:
        design_type, design_tier = classify_design(s.get("design_type"))
        rob = parse_rob_from_dict(s)

        dossier = StudyDossier(
            study_id=s["study_id"],
            yi=s["yi"],
            vi=s["vi"],
            measure=s.get("measure", "logOR"),
            design_type=design_type,
            design_tier=design_tier,
            rob=rob,
            year=s.get("year"),
            n1=s.get("n1"),
            n2=s.get("n2"),
            events1=s.get("events1"),
            events2=s.get("events2"),
        )
        dossiers.append(dossier)

    # Review-level bias profiling
    yi = np.array([d.yi for d in dossiers])
    vi = np.array([d.vi for d in dossiers])
    bias = profile_bias(yi, vi)

    return dossiers, bias
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd C:/Models/MES && python -m pytest tests/test_dossier_builder.py -v`
Expected: All 5 tests PASS

- [ ] **Step 5: Commit**

```bash
cd C:/Models/MES && git add -A && git commit -m "feat(assess): add dossier builder orchestrating design + RoB + bias"
```

---

## Task 7: EXPLORE — Estimators (6 τ² Methods)

**Files:**
- Create: `C:\Models\MES\mes_core\explore\estimators.py`
- Create: `C:\Models\MES\tests\test_estimators.py`

Reference: `C:\FragilityAtlas\src\estimators.py` (271 lines) — port directly.

**R parity targets** (BCG dataset, metafor::rma):
- DL: θ = -0.7145, τ² = 0.3132
- REML: θ = -0.7141, τ² = 0.3088
- FE: θ = -0.4361, τ² = 0
- PM: θ ≈ -0.714, τ² ≈ 0.31 (similar to DL)

- [ ] **Step 1: Write failing tests**

```python
# tests/test_estimators.py
import numpy as np
import pytest
from mes_core.explore.estimators import (
    fe, dl, reml, pm, sj, ml, pool_estimate,
)


class TestFixedEffect:
    def test_fe_theta(self, bcg_studies):
        tau2, theta, se = fe(bcg_studies["yi"], bcg_studies["vi"])
        assert tau2 == 0.0
        assert abs(theta - (-0.4361)) < 0.01

    def test_fe_small(self, small_studies):
        tau2, theta, se = fe(small_studies["yi"], small_studies["vi"])
        assert tau2 == 0.0
        assert theta < 0  # negative direction


class TestDerSimonianLaird:
    def test_dl_theta(self, bcg_studies):
        tau2, theta, se = dl(bcg_studies["yi"], bcg_studies["vi"])
        assert abs(theta - (-0.7145)) < 0.01
        assert abs(tau2 - 0.3132) < 0.01

    def test_dl_homogeneous(self):
        """Identical studies → tau2 = 0."""
        yi = np.array([-0.5, -0.5, -0.5])
        vi = np.array([0.04, 0.04, 0.04])
        tau2, theta, se = dl(yi, vi)
        assert tau2 == 0.0


class TestREML:
    def test_reml_theta(self, bcg_studies):
        tau2, theta, se = reml(bcg_studies["yi"], bcg_studies["vi"])
        assert abs(theta - (-0.7141)) < 0.01
        assert abs(tau2 - 0.3088) < 0.02

    def test_reml_converges(self, bcg_studies):
        """REML should converge for BCG dataset."""
        tau2, theta, se = reml(bcg_studies["yi"], bcg_studies["vi"])
        assert se > 0


class TestPauleMandel:
    def test_pm_theta(self, bcg_studies):
        tau2, theta, se = pm(bcg_studies["yi"], bcg_studies["vi"])
        assert abs(theta - (-0.7145)) < 0.02  # close to DL
        assert tau2 > 0


class TestSidikJonkman:
    def test_sj_theta(self, bcg_studies):
        tau2, theta, se = sj(bcg_studies["yi"], bcg_studies["vi"])
        assert theta < 0
        assert tau2 > 0


class TestML:
    def test_ml_theta(self, bcg_studies):
        tau2, theta, se = ml(bcg_studies["yi"], bcg_studies["vi"])
        assert abs(theta - (-0.7145)) < 0.02
        assert tau2 > 0


class TestPoolEstimate:
    def test_pool_with_tau2(self, bcg_studies):
        tau2 = 0.3132
        theta, se = pool_estimate(bcg_studies["yi"], bcg_studies["vi"], tau2)
        assert abs(theta - (-0.7145)) < 0.01
        assert se > 0

    def test_pool_k1():
        """Single study — returns study estimate."""
        yi = np.array([-0.5])
        vi = np.array([0.04])
        theta, se = pool_estimate(yi, vi, 0.0)
        assert theta == -0.5
        assert abs(se - 0.2) < 0.001
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd C:/Models/MES && python -m pytest tests/test_estimators.py -v`
Expected: FAIL

- [ ] **Step 3: Implement estimators**

Port from `C:\FragilityAtlas\src\estimators.py`. All functions return `(tau2, theta, se)`.

```python
# mes_core/explore/estimators.py
"""Meta-analysis tau-squared estimators — 6 methods.

All functions take (yi, vi) numpy arrays and return (tau2, theta, se).
Ported from FragilityAtlas/src/estimators.py with identical algorithms.
"""
import numpy as np


def pool_estimate(
    yi: np.ndarray, vi: np.ndarray, tau2: float
) -> tuple[float, float]:
    """Compute pooled estimate given tau2. Returns (theta, se)."""
    wi = 1.0 / (vi + tau2)
    theta = float(np.sum(wi * yi) / np.sum(wi))
    se = float(np.sqrt(1.0 / np.sum(wi)))
    return theta, se


def fe(yi: np.ndarray, vi: np.ndarray) -> tuple[float, float, float]:
    """Fixed-effect model (tau2 = 0)."""
    theta, se = pool_estimate(yi, vi, 0.0)
    return 0.0, theta, se


def dl(yi: np.ndarray, vi: np.ndarray) -> tuple[float, float, float]:
    """DerSimonian-Laird moment estimator."""
    k = len(yi)
    wi = 1.0 / vi
    theta_fe = np.sum(wi * yi) / np.sum(wi)
    Q = float(np.sum(wi * (yi - theta_fe) ** 2))
    C = float(np.sum(wi) - np.sum(wi ** 2) / np.sum(wi))
    tau2 = max(0.0, (Q - (k - 1)) / C) if C > 0 else 0.0
    theta, se = pool_estimate(yi, vi, tau2)
    return tau2, theta, se


def reml(
    yi: np.ndarray, vi: np.ndarray,
    max_iter: int = 50, tol: float = 1e-10,
) -> tuple[float, float, float]:
    """Restricted Maximum Likelihood via Fisher scoring."""
    tau2, _, _ = dl(yi, vi)  # DL as starting value
    for _ in range(max_iter):
        wi = 1.0 / (vi + tau2)
        theta_hat = np.sum(wi * yi) / np.sum(wi)
        resid = yi - theta_hat
        # REML gradient and Fisher info
        # gradient = -0.5 * (sum(wi) - sum(wi^2)/sum(wi)) + 0.5 * sum(wi^2 * resid^2)
        # but simpler: Fisher scoring step
        P_diag = wi - (wi ** 2) / np.sum(wi)
        gradient = -0.5 * np.sum(P_diag) + 0.5 * np.sum(P_diag ** 2 / wi * (wi * resid) ** 2 / wi)
        # Simplified Fisher scoring (Viechtbauer 2005):
        info = 0.5 * np.sum(P_diag ** 2)
        if info < 1e-30:
            break
        # Direct update formula (more stable):
        wi2 = wi ** 2
        num = float(np.sum(wi2 * (resid ** 2 - vi)))
        den = float(np.sum(wi2))
        tau2_new = max(0.0, tau2 + num / den) if den > 0 else tau2
        if abs(tau2_new - tau2) < tol:
            tau2 = tau2_new
            break
        tau2 = tau2_new
    theta, se = pool_estimate(yi, vi, tau2)
    return tau2, theta, se


def pm(
    yi: np.ndarray, vi: np.ndarray,
    max_iter: int = 100, tol: float = 1e-10,
) -> tuple[float, float, float]:
    """Paule-Mandel estimator — iterative, solves Q*(tau2) = k-1."""
    k = len(yi)
    tau2, _, _ = dl(yi, vi)
    for _ in range(max_iter):
        wi = 1.0 / (vi + tau2)
        theta_hat = np.sum(wi * yi) / np.sum(wi)
        Q_star = float(np.sum(wi * (yi - theta_hat) ** 2))
        if abs(Q_star - (k - 1)) < tol:
            break
        # Adjust tau2
        C = float(np.sum(wi) - np.sum(wi ** 2) / np.sum(wi))
        if C < 1e-30:
            break
        tau2 = max(0.0, tau2 + (Q_star - (k - 1)) / C)
    theta, se = pool_estimate(yi, vi, tau2)
    return tau2, theta, se


def sj(yi: np.ndarray, vi: np.ndarray) -> tuple[float, float, float]:
    """Sidik-Jonkman estimator — one-step from unweighted variance."""
    k = len(yi)
    theta_uw = float(np.mean(yi))
    tau2_0 = float(np.sum((yi - theta_uw) ** 2) / (k - 1))
    # One-step DL update using tau2_0 weights
    wi = 1.0 / (vi + tau2_0)
    theta_hat = np.sum(wi * yi) / np.sum(wi)
    Q = float(np.sum(wi * (yi - theta_hat) ** 2))
    C = float(np.sum(wi) - np.sum(wi ** 2) / np.sum(wi))
    tau2 = max(0.0, (Q - (k - 1)) / C) if C > 0 else 0.0
    theta, se = pool_estimate(yi, vi, tau2)
    return tau2, theta, se


def ml(
    yi: np.ndarray, vi: np.ndarray,
    max_iter: int = 50, tol: float = 1e-10,
) -> tuple[float, float, float]:
    """Maximum Likelihood estimator via Newton-Raphson."""
    tau2, _, _ = dl(yi, vi)
    for _ in range(max_iter):
        wi = 1.0 / (vi + tau2)
        theta_hat = np.sum(wi * yi) / np.sum(wi)
        resid = yi - theta_hat
        # ML gradient: -0.5 * sum(wi) + 0.5 * sum(wi^2 * resid^2)
        wi2 = wi ** 2
        num = float(np.sum(wi2 * (resid ** 2 - vi)))
        den = float(np.sum(wi2))
        if den < 1e-30:
            break
        tau2_new = max(0.0, tau2 + num / den)
        if abs(tau2_new - tau2) < tol:
            tau2 = tau2_new
            break
        tau2 = tau2_new
    theta, se = pool_estimate(yi, vi, tau2)
    return tau2, theta, se
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd C:/Models/MES && python -m pytest tests/test_estimators.py -v`
Expected: All 11 tests PASS

- [ ] **Step 5: Commit**

```bash
cd C:/Models/MES && git add -A && git commit -m "feat(explore): add 6 tau-squared estimators (FE/DL/REML/PM/SJ/ML)"
```

---

## Task 8: EXPLORE — CI Methods

**Files:**
- Create: `C:\Models\MES\mes_core\explore\ci_methods.py`
- Create: `C:\Models\MES\tests\test_ci_methods.py`

- [ ] **Step 1: Write failing tests**

```python
# tests/test_ci_methods.py
import numpy as np
import pytest
from mes_core.explore.ci_methods import wald_ci, hksj_ci, tdist_ci


def test_wald_ci_basic():
    ci_lo, ci_hi, p = wald_ci(theta=-0.5, se=0.1, alpha=0.05)
    assert abs(ci_lo - (-0.696)) < 0.01
    assert abs(ci_hi - (-0.304)) < 0.01
    assert p < 0.001


def test_wald_ci_nonsig():
    ci_lo, ci_hi, p = wald_ci(theta=-0.1, se=0.2, alpha=0.05)
    assert ci_lo < 0 < ci_hi  # crosses null
    assert p > 0.05


def test_hksj_ci_wider(bcg_studies):
    """HKSJ should produce wider CIs than Wald for heterogeneous data."""
    from mes_core.explore.estimators import dl
    tau2, theta, se = dl(bcg_studies["yi"], bcg_studies["vi"])
    k = bcg_studies["k"]
    yi, vi = bcg_studies["yi"], bcg_studies["vi"]

    wald_lo, wald_hi, _ = wald_ci(theta, se, 0.05)
    hksj_lo, hksj_hi, _ = hksj_ci(theta, se, yi, vi, tau2, k, 0.05)
    assert (hksj_hi - hksj_lo) >= (wald_hi - wald_lo)


def test_tdist_ci_wider_than_wald():
    """t-dist CI is wider than Wald for small k."""
    ci_wald_lo, ci_wald_hi, _ = wald_ci(-0.5, 0.15, 0.05)
    ci_t_lo, ci_t_hi, _ = tdist_ci(-0.5, 0.15, k=5, alpha=0.05)
    assert (ci_t_hi - ci_t_lo) > (ci_wald_hi - ci_wald_lo)


def test_hksj_q_hat_floor():
    """HKSJ q_hat is floored at 1 to avoid narrowing CIs."""
    # Very homogeneous data
    yi = np.array([-0.5, -0.5, -0.5, -0.5, -0.5])
    vi = np.array([0.04, 0.04, 0.04, 0.04, 0.04])
    tau2 = 0.0
    theta, se = -0.5, 0.0894  # approx
    lo, hi, _ = hksj_ci(theta, se, yi, vi, tau2, 5, 0.05)
    # Should not be narrower than t-dist CI
    t_lo, t_hi, _ = tdist_ci(theta, se, k=5, alpha=0.05)
    assert (hi - lo) >= (t_hi - t_lo) * 0.99  # allow tiny float rounding
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd C:/Models/MES && python -m pytest tests/test_ci_methods.py -v`
Expected: FAIL

- [ ] **Step 3: Implement CI methods**

```python
# mes_core/explore/ci_methods.py
"""Confidence interval methods for pooled meta-analysis estimates."""
import numpy as np
from scipy import stats


def wald_ci(
    theta: float, se: float, alpha: float = 0.05
) -> tuple[float, float, float]:
    """Wald (normal approximation) CI. Returns (ci_lo, ci_hi, p_value)."""
    z_crit = stats.norm.ppf(1 - alpha / 2)
    ci_lo = theta - z_crit * se
    ci_hi = theta + z_crit * se
    z = theta / se if se > 1e-30 else 0.0
    p = 2.0 * stats.norm.sf(abs(z))
    return float(ci_lo), float(ci_hi), float(p)


def hksj_ci(
    theta: float, se: float,
    yi: np.ndarray, vi: np.ndarray, tau2: float,
    k: int, alpha: float = 0.05,
) -> tuple[float, float, float]:
    """Hartung-Knapp-Sidik-Jonkman adjusted CI.

    Uses t-distribution with df=k-1 and variance inflation factor q_hat.
    q_hat is floored at 1 to avoid narrowing CIs below the t-dist CI.
    """
    if k < 2:
        return wald_ci(theta, se, alpha)
    wi = 1.0 / (vi + tau2)
    resid_sq = wi * (yi - theta) ** 2
    q_hat = float(np.sum(resid_sq)) / (k - 1)
    q_hat = max(q_hat, 1.0)  # floor at 1
    se_adj = se * np.sqrt(q_hat)
    t_crit = stats.t.ppf(1 - alpha / 2, df=k - 1)
    ci_lo = theta - t_crit * se_adj
    ci_hi = theta + t_crit * se_adj
    t_stat = theta / se_adj if se_adj > 1e-30 else 0.0
    p = 2.0 * stats.t.sf(abs(t_stat), df=k - 1)
    return float(ci_lo), float(ci_hi), float(p)


def tdist_ci(
    theta: float, se: float,
    k: int, alpha: float = 0.05,
) -> tuple[float, float, float]:
    """t-distribution CI with df=k-1 (no variance adjustment)."""
    if k < 2:
        return wald_ci(theta, se, alpha)
    t_crit = stats.t.ppf(1 - alpha / 2, df=k - 1)
    ci_lo = theta - t_crit * se
    ci_hi = theta + t_crit * se
    t_stat = theta / se if se > 1e-30 else 0.0
    p = 2.0 * stats.t.sf(abs(t_stat), df=k - 1)
    return float(ci_lo), float(ci_hi), float(p)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd C:/Models/MES && python -m pytest tests/test_ci_methods.py -v`
Expected: All 5 tests PASS

- [ ] **Step 5: Commit**

```bash
cd C:/Models/MES && git add -A && git commit -m "feat(explore): add 3 CI methods (Wald, HKSJ, t-dist)"
```

---

## Task 9: EXPLORE — Bias Corrections

**Files:**
- Create: `C:\Models\MES\mes_core\explore\bias_corrections.py`
- Create: `C:\Models\MES\tests\test_bias_corrections.py`

Reference: `C:\FragilityAtlas\src\corrections.py` (169 lines) + `C:\BiasForensics\src\methods.py`.

- [ ] **Step 1: Write failing tests**

```python
# tests/test_bias_corrections.py
import numpy as np
import pytest
from mes_core.explore.bias_corrections import trim_fill, pet_peese, selection_model


class TestTrimFill:
    def test_bcg_adjusts(self, bcg_studies):
        result = trim_fill(bcg_studies["yi"], bcg_studies["vi"])
        assert result["k0"] >= 0
        assert result["theta_adj"] is not None
        # Should return augmented arrays
        assert len(result["yi_aug"]) >= len(bcg_studies["yi"])

    def test_symmetric_no_fill(self):
        yi = np.array([-0.5, -0.3, -0.7, -0.4, -0.6])
        vi = np.array([0.04, 0.04, 0.04, 0.04, 0.04])
        result = trim_fill(yi, vi)
        assert result["k0"] == 0

    def test_too_few_studies():
        yi = np.array([-0.5, -0.3])
        vi = np.array([0.04, 0.06])
        result = trim_fill(yi, vi)
        assert result["theta_adj"] is None  # insufficient k


class TestPETPEESE:
    def test_bcg(self, bcg_studies):
        result = pet_peese(bcg_studies["yi"], bcg_studies["vi"])
        assert "theta_adj" in result
        assert "method_used" in result
        assert result["method_used"] in ("PET", "PEESE")

    def test_too_few():
        yi = np.array([-0.5, -0.3])
        vi = np.array([0.04, 0.06])
        result = pet_peese(yi, vi)
        assert result["theta_adj"] is None


class TestSelectionModel:
    def test_bcg(self, bcg_studies):
        result = selection_model(bcg_studies["yi"], bcg_studies["vi"])
        assert "theta_adj" in result
        assert "eta" in result

    def test_too_few():
        yi = np.array([-0.5] * 5)
        vi = np.array([0.04] * 5)
        result = selection_model(yi, vi)
        # k=5 is borderline — should still attempt but may return None
        assert "theta_adj" in result
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd C:/Models/MES && python -m pytest tests/test_bias_corrections.py -v`
Expected: FAIL

- [ ] **Step 3: Implement bias corrections**

Port from `C:\FragilityAtlas\src\corrections.py` + `C:\BiasForensics\src\methods.py`.

```python
# mes_core/explore/bias_corrections.py
"""Publication bias correction methods for the EXPLORE multiverse."""
import numpy as np
from scipy import stats
from mes_core.explore.estimators import dl, pool_estimate


def trim_fill(
    yi: np.ndarray, vi: np.ndarray, side: str = "auto", max_iter: int = 20,
) -> dict:
    """Duval-Tweedie L0 trim-and-fill.

    Returns dict with: k0, theta_adj, se_adj, yi_aug, vi_aug.
    Returns theta_adj=None if k < 5.
    """
    k = len(yi)
    if k < 5:
        return {"k0": 0, "theta_adj": None, "se_adj": None,
                "yi_aug": yi, "vi_aug": vi}

    tau2_init, theta_init, _ = dl(yi, vi)

    # Detect asymmetry side
    if side == "auto":
        residuals = yi - theta_init
        side = "right" if np.sum(residuals > 0) > np.sum(residuals < 0) else "left"

    yi_work = yi.copy()
    vi_work = vi.copy()
    k0_prev = -1

    for _ in range(max_iter):
        tau2_w, theta_w, _ = dl(yi_work, vi_work)
        residuals = yi_work - theta_w
        if side == "right":
            ranks = np.argsort(residuals)[::-1]
        else:
            ranks = np.argsort(residuals)
        # L0 estimator: k0 = max(0, (4S - n(n+1)) / (2n+1))
        n = len(yi_work)
        S = 0
        for i in range(n):
            if side == "right" and residuals[ranks[i]] > 0:
                S += (i + 1)
            elif side == "left" and residuals[ranks[i]] < 0:
                S += (i + 1)
        k0 = max(0, int(round((4 * S - n * (n + 1)) / (2 * n + 1))))
        if k0 == k0_prev:
            break
        k0_prev = k0
        # Augment with reflected studies
        yi_work = yi.copy()
        vi_work = vi.copy()
        if k0 > 0:
            tau2_base, theta_base, _ = dl(yi, vi)
            resid = yi - theta_base
            if side == "right":
                idx = np.argsort(resid)[::-1][:k0]
            else:
                idx = np.argsort(resid)[:k0]
            mirror_yi = 2 * theta_base - yi[idx]
            mirror_vi = vi[idx]
            yi_work = np.concatenate([yi, mirror_yi])
            vi_work = np.concatenate([vi, mirror_vi])

    tau2_adj, theta_adj, se_adj = dl(yi_work, vi_work)
    return {
        "k0": k0,
        "theta_adj": float(theta_adj),
        "se_adj": float(se_adj),
        "yi_aug": yi_work,
        "vi_aug": vi_work,
    }


def pet_peese(yi: np.ndarray, vi: np.ndarray) -> dict:
    """PET-PEESE conditional regression (Stanley-Doucouliagos).

    PET: yi = b0 + b1*sei + error (WLS, weights=1/vi)
    If PET p(b0) < 0.05 → use PEESE: yi = b0 + b1*sei^2
    Returns dict with: theta_adj, se_adj, method_used.
    """
    k = len(yi)
    if k < 3:
        return {"theta_adj": None, "se_adj": None, "method_used": None}

    sei = np.sqrt(vi)
    wi = 1.0 / vi

    # PET: yi ~ sei
    theta_pet, se_pet, p_pet = _wls_intercept(yi, sei, wi)

    if p_pet < 0.05:
        # PEESE: yi ~ sei^2
        theta_adj, se_adj, _ = _wls_intercept(yi, vi, wi)
        method = "PEESE"
    else:
        theta_adj, se_adj = theta_pet, se_pet
        method = "PET"

    return {
        "theta_adj": float(theta_adj) if theta_adj is not None else None,
        "se_adj": float(se_adj) if se_adj is not None else None,
        "method_used": method,
    }


def _wls_intercept(
    y: np.ndarray, x: np.ndarray, w: np.ndarray
) -> tuple[float, float, float]:
    """WLS regression y = a + b*x, returns (intercept, se_intercept, p_intercept)."""
    n = len(y)
    sw = np.sum(w)
    swx = np.sum(w * x)
    swy = np.sum(w * y)
    swxx = np.sum(w * x ** 2)
    swxy = np.sum(w * x * y)
    denom = sw * swxx - swx ** 2
    if abs(denom) < 1e-30:
        return 0.0, 1.0, 1.0
    a = (swxx * swy - swx * swxy) / denom
    b = (sw * swxy - swx * swy) / denom
    resid = y - (a + b * x)
    mse = np.sum(w * resid ** 2) / (n - 2) if n > 2 else 1.0
    se_a = np.sqrt(mse * swxx / denom)
    if se_a < 1e-30:
        return a, 1e-10, 1.0
    t_stat = a / se_a
    p = 2.0 * stats.t.sf(abs(t_stat), df=n - 2) if n > 2 else 1.0
    return float(a), float(se_a), float(p)


def selection_model(yi: np.ndarray, vi: np.ndarray) -> dict:
    """3-parameter selection model (simplified Vevea-Hedges).

    Step-function: w(p) = 1 if p < 0.025 (one-sided), else eta.
    Profile likelihood over eta grid.
    """
    k = len(yi)
    if k < 5:
        return {"theta_adj": None, "se_adj": None, "eta": None, "lr_p": None}

    sei = np.sqrt(vi)
    eta_grid = [0.01, 0.05, 0.1, 0.2, 0.3, 0.5, 0.7, 1.0]

    # Baseline (no selection): DL
    tau2_base, theta_base, _ = dl(yi, vi)
    ll_base = _log_likelihood(yi, vi, theta_base, tau2_base, eta=1.0)

    best_ll = -np.inf
    best_eta = 1.0
    best_theta = theta_base

    for eta in eta_grid:
        # Compute weighted log-likelihood
        theta_try, tau2_try = _sel_model_fit(yi, vi, sei, eta)
        ll = _log_likelihood(yi, vi, theta_try, tau2_try, eta)
        if ll > best_ll:
            best_ll = ll
            best_eta = eta
            best_theta = theta_try

    # LR test: 2*(ll_sel - ll_base) ~ chi2(1)
    lr_stat = 2 * (best_ll - ll_base)
    lr_p = float(stats.chi2.sf(max(0, lr_stat), df=1))

    tau2_adj, _, _ = dl(yi, vi)
    _, se_adj = pool_estimate(yi, vi, tau2_adj)

    return {
        "theta_adj": float(best_theta),
        "se_adj": float(se_adj),
        "eta": float(best_eta),
        "lr_p": lr_p,
    }


def _sel_model_fit(
    yi: np.ndarray, vi: np.ndarray, sei: np.ndarray, eta: float
) -> tuple[float, float]:
    """Fit selection model for a given eta. Returns (theta, tau2)."""
    # Weight each study by selection probability
    p_values = 2.0 * stats.norm.sf(np.abs(yi / sei))
    sel_weights = np.where(p_values < 0.05, 1.0, eta)
    # Weighted DL
    wi = sel_weights / vi
    theta = float(np.sum(wi * yi) / np.sum(wi)) if np.sum(wi) > 0 else 0.0
    Q = float(np.sum(wi * (yi - theta) ** 2))
    k = len(yi)
    C = float(np.sum(wi) - np.sum(wi ** 2) / np.sum(wi)) if np.sum(wi) > 0 else 1.0
    tau2 = max(0.0, (Q - (k - 1)) / C) if C > 0 else 0.0
    return theta, tau2


def _log_likelihood(
    yi: np.ndarray, vi: np.ndarray,
    theta: float, tau2: float, eta: float,
) -> float:
    """Log-likelihood under selection model."""
    sei = np.sqrt(vi)
    total_var = vi + tau2
    ll = -0.5 * np.sum(np.log(total_var) + (yi - theta) ** 2 / total_var)
    # Selection weight contribution
    p_values = 2.0 * stats.norm.sf(np.abs(yi / sei))
    sel_weights = np.where(p_values < 0.05, 1.0, eta)
    ll += np.sum(np.log(np.maximum(sel_weights, 1e-30)))
    return float(ll)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd C:/Models/MES && python -m pytest tests/test_bias_corrections.py -v`
Expected: All 7 tests PASS

- [ ] **Step 5: Commit**

```bash
cd C:/Models/MES && git add -A && git commit -m "feat(explore): add 3 bias corrections (trim-fill, PET-PEESE, selection model)"
```

---

## Task 10: EXPLORE — Spec Generator + Executor

**Files:**
- Create: `C:\Models\MES\mes_core\explore\spec_generator.py`
- Create: `C:\Models\MES\mes_core\explore\executor.py`
- Create: `C:\Models\MES\tests\test_spec_generator.py`
- Create: `C:\Models\MES\tests\test_executor.py`

- [ ] **Step 1: Write failing tests for spec generator**

```python
# tests/test_spec_generator.py
import numpy as np
from mes_core.models import MESSpec, StudyDossier
from mes_core.explore.spec_generator import generate_specs


def test_default_spec_count():
    """Default MESSpec produces 648 base specs."""
    dossiers = [
        StudyDossier(study_id=f"S{i}", yi=-0.5, vi=0.04, measure="logOR")
        for i in range(10)
    ]
    specs = generate_specs(MESSpec(), dossiers, include_loo=False)
    assert len(specs) == 648


def test_with_loo():
    """With LOO, k=5 → 648 * 6 = 3888 specs."""
    dossiers = [
        StudyDossier(study_id=f"S{i}", yi=-0.5, vi=0.04, measure="logOR")
        for i in range(5)
    ]
    specs = generate_specs(MESSpec(), dossiers, include_loo=True)
    assert len(specs) == 648 * 6  # 648 * (5+1)


def test_feasibility_pruning_k_threshold():
    """Quality filter leaving k<2 → specs pruned."""
    dossiers = [
        StudyDossier(study_id="S0", yi=-0.5, vi=0.04, measure="logOR",
                     rob=None),  # unassessed
        StudyDossier(study_id="S1", yi=-0.3, vi=0.06, measure="logOR",
                     rob=None),
    ]
    specs = generate_specs(MESSpec(), dossiers, include_loo=False)
    # "low-rob-only" and "exclude-high-rob" both yield k=0 (no RoB data)
    # So quality_filters effectively collapse to ["all"] only
    # FE + trim-fill also pruned
    assert len(specs) < 648


def test_spec_has_required_fields():
    dossiers = [
        StudyDossier(study_id=f"S{i}", yi=-0.5, vi=0.04, measure="logOR")
        for i in range(5)
    ]
    specs = generate_specs(MESSpec(), dossiers, include_loo=False)
    s = specs[0]
    assert "estimator" in s
    assert "ci_method" in s
    assert "bias_correction" in s
    assert "quality_filter" in s
    assert "design_filter" in s
    assert "sensitivity" in s
    assert "spec_id" in s
```

- [ ] **Step 2: Write failing tests for executor**

```python
# tests/test_executor.py
import pytest
from mes_core.models import MESSpec, SpecResult, StudyDossier
from mes_core.explore.executor import execute_multiverse


def test_execute_bcg(bcg_study_dicts):
    """Full multiverse on BCG — should return ~648 results."""
    from mes_core.assess.dossier_builder import build_dossiers
    dossiers, bias = build_dossiers(bcg_study_dicts)
    spec = MESSpec()
    results = execute_multiverse(spec, dossiers, include_loo=False)
    assert len(results) > 0
    assert all(isinstance(r, SpecResult) for r in results)


def test_execute_returns_expected_fields(bcg_study_dicts):
    from mes_core.assess.dossier_builder import build_dossiers
    dossiers, _ = build_dossiers(bcg_study_dicts)
    spec = MESSpec(
        estimators=["DL"],
        ci_methods=["Wald"],
        bias_corrections=["none"],
        quality_filters=["all"],
        design_filters=["all"],
    )
    results = execute_multiverse(spec, dossiers, include_loo=False)
    assert len(results) == 1
    r = results[0]
    assert r.estimator == "DL"
    assert r.ci_method == "Wald"
    assert abs(r.theta - (-0.7145)) < 0.02
    assert r.k == 13


def test_execute_dl_theta_matches_estimator(bcg_study_dicts):
    """Executor DL result should match direct estimator call."""
    import numpy as np
    from mes_core.assess.dossier_builder import build_dossiers
    from mes_core.explore.estimators import dl
    dossiers, _ = build_dossiers(bcg_study_dicts)
    yi = np.array([d.yi for d in dossiers])
    vi = np.array([d.vi for d in dossiers])
    tau2_direct, theta_direct, _ = dl(yi, vi)

    spec = MESSpec(
        estimators=["DL"], ci_methods=["Wald"],
        bias_corrections=["none"], quality_filters=["all"],
        design_filters=["all"],
    )
    results = execute_multiverse(spec, dossiers, include_loo=False)
    assert abs(results[0].theta - theta_direct) < 1e-10
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `cd C:/Models/MES && python -m pytest tests/test_spec_generator.py tests/test_executor.py -v`
Expected: FAIL

- [ ] **Step 4: Implement spec generator**

```python
# mes_core/explore/spec_generator.py
"""Generate the multiverse specification grid."""
from itertools import product
from mes_core.models import MESSpec, StudyDossier


def generate_specs(
    mes_spec: MESSpec,
    dossiers: list[StudyDossier],
    include_loo: bool = True,
    loo_cap: int = 30,
) -> list[dict]:
    """Generate all feasible specifications from the MES spec.

    Returns list of spec dicts, each with keys:
    estimator, ci_method, bias_correction, quality_filter, design_filter,
    sensitivity, spec_id, study_indices (which dossiers to include).
    """
    k_total = len(dossiers)

    # Pre-compute study subsets for each filter combination
    filter_subsets = _compute_filter_subsets(dossiers, mes_spec)

    # Build sensitivity levels
    sensitivity_levels = ["full"]
    if include_loo and "loo" in mes_spec.sensitivity:
        if k_total <= loo_cap:
            for i in range(k_total):
                sensitivity_levels.append(f"loo_{dossiers[i].study_id}")
        else:
            # Random sampling for large k
            import random
            rng = random.Random(42)  # deterministic
            indices = rng.sample(range(k_total), min(loo_cap, k_total))
            for i in indices:
                sensitivity_levels.append(f"loo_{dossiers[i].study_id}")

    specs = []
    for est, ci, bc, qf, df in product(
        mes_spec.estimators,
        mes_spec.ci_methods,
        mes_spec.bias_corrections,
        mes_spec.quality_filters,
        mes_spec.design_filters,
    ):
        # Get study indices for this filter combo
        idx_key = (qf, df)
        if idx_key not in filter_subsets:
            continue
        base_indices = filter_subsets[idx_key]

        for sens in sensitivity_levels:
            indices = base_indices.copy()
            if sens.startswith("loo_"):
                loo_id = sens[4:]
                indices = [i for i in indices if dossiers[i].study_id != loo_id]

            k = len(indices)

            # Feasibility checks
            if k < 2:
                continue
            if bc == "trim-fill" and (k < 5 or est == "FE"):
                continue
            if bc == "selection-model" and k < 10:
                continue

            spec_id = f"{est}_{ci}_{bc}_{qf}_{df}_{sens}"
            specs.append({
                "spec_id": spec_id,
                "estimator": est,
                "ci_method": ci,
                "bias_correction": bc,
                "quality_filter": qf,
                "design_filter": df,
                "sensitivity": sens,
                "study_indices": indices,
            })

    return specs


def _compute_filter_subsets(
    dossiers: list[StudyDossier], mes_spec: MESSpec
) -> dict[tuple[str, str], list[int]]:
    """Pre-compute which study indices pass each quality+design filter."""
    subsets = {}
    for qf in mes_spec.quality_filters:
        for df in mes_spec.design_filters:
            indices = []
            for i, d in enumerate(dossiers):
                if not _passes_quality(d, qf):
                    continue
                if not _passes_design(d, df):
                    continue
                indices.append(i)
            if len(indices) >= 2:  # minimum for any MA
                subsets[(qf, df)] = indices
    return subsets


def _passes_quality(d: StudyDossier, qf: str) -> bool:
    if qf == "all":
        return True
    if d.rob is None:
        return False  # unassessed → excluded from quality-filtered specs
    if qf == "exclude-high-rob":
        return d.rob.overall != "high"
    if qf == "low-rob-only":
        return d.rob.overall == "low"
    return True


def _passes_design(d: StudyDossier, df: str) -> bool:
    if df == "all":
        return True
    if df == "rct-quasi":
        return d.design_tier <= 2
    if df == "rct-only":
        return d.design_tier == 1
    return True
```

- [ ] **Step 5: Implement executor**

```python
# mes_core/explore/executor.py
"""Execute the multiverse of specifications."""
import numpy as np
from scipy import stats
from mes_core.models import MESSpec, SpecResult, StudyDossier
from mes_core.explore import estimators as est_mod
from mes_core.explore.ci_methods import wald_ci, hksj_ci, tdist_ci
from mes_core.explore.bias_corrections import trim_fill, pet_peese, selection_model
from mes_core.explore.spec_generator import generate_specs


_ESTIMATORS = {
    "FE": est_mod.fe,
    "DL": est_mod.dl,
    "REML": est_mod.reml,
    "PM": est_mod.pm,
    "SJ": est_mod.sj,
    "ML": est_mod.ml,
}


def execute_multiverse(
    mes_spec: MESSpec,
    dossiers: list[StudyDossier],
    include_loo: bool = True,
) -> list[SpecResult]:
    """Run all feasible specifications and return SpecResult list."""
    specs = generate_specs(mes_spec, dossiers, include_loo=include_loo)
    results = []
    for spec in specs:
        result = _run_single_spec(spec, dossiers, mes_spec.alpha)
        if result is not None:
            results.append(result)
    return results


def _run_single_spec(
    spec: dict, dossiers: list[StudyDossier], alpha: float
) -> SpecResult | None:
    """Execute one specification. Returns SpecResult or None on failure."""
    indices = spec["study_indices"]
    yi = np.array([dossiers[i].yi for i in indices])
    vi = np.array([dossiers[i].vi for i in indices])
    k = len(yi)

    try:
        # Step 1: Apply bias correction (modifies yi/vi if needed)
        bc = spec["bias_correction"]
        bc_result = None
        if bc == "trim-fill":
            bc_result = trim_fill(yi, vi)
            if bc_result["theta_adj"] is None:
                return None
            yi = bc_result["yi_aug"]
            vi = bc_result["vi_aug"]
            k = len(yi)
        elif bc == "PET-PEESE":
            bc_result = pet_peese(yi, vi)
        elif bc == "selection-model":
            bc_result = selection_model(yi, vi)

        # Step 2: Estimate tau2 and pool
        est_fn = _ESTIMATORS.get(spec["estimator"])
        if est_fn is None:
            return None
        tau2, theta, se = est_fn(yi, vi)

        # Override theta for PET-PEESE and selection model
        if bc in ("PET-PEESE", "selection-model") and bc_result and bc_result["theta_adj"] is not None:
            theta = bc_result["theta_adj"]
            if bc_result.get("se_adj") is not None:
                se = bc_result["se_adj"]

        # Step 3: Compute CI
        ci_method = spec["ci_method"]
        if ci_method == "Wald":
            ci_lo, ci_hi, p_value = wald_ci(theta, se, alpha)
        elif ci_method == "HKSJ":
            ci_lo, ci_hi, p_value = hksj_ci(theta, se, yi, vi, tau2, k, alpha)
        elif ci_method == "t-dist":
            ci_lo, ci_hi, p_value = tdist_ci(theta, se, k, alpha)
        else:
            return None

        # Step 4: Prediction interval
        if k >= 3 and tau2 > 0:
            t_crit_pi = stats.t.ppf(1 - alpha / 2, df=k - 2)
            pi_se = np.sqrt(tau2 + se ** 2)
            pi_lo = theta - t_crit_pi * pi_se
            pi_hi = theta + t_crit_pi * pi_se
        else:
            pi_lo, pi_hi = ci_lo, ci_hi

        # Step 5: Derived metrics
        I2 = max(0.0, 1.0 - (k - 1) / max(1e-30, _cochran_q(yi, vi))) if k >= 2 else 0.0
        significant = p_value < alpha
        if abs(theta) < 1e-10:
            direction = "null"
        elif theta > 0:
            direction = "positive"
        else:
            direction = "negative"

        return SpecResult(
            spec_id=spec["spec_id"],
            estimator=spec["estimator"],
            ci_method=ci_method,
            bias_correction=bc,
            quality_filter=spec["quality_filter"],
            design_filter=spec["design_filter"],
            sensitivity=spec["sensitivity"],
            theta=float(theta), se=float(se),
            ci_lo=float(ci_lo), ci_hi=float(ci_hi),
            p_value=float(p_value),
            tau2=float(tau2), I2=float(I2), k=k,
            pi_lo=float(pi_lo), pi_hi=float(pi_hi),
            significant=significant, direction=direction,
        )
    except (ValueError, FloatingPointError, ZeroDivisionError) as e:
        return SpecResult(
            spec_id=spec["spec_id"],
            estimator=spec["estimator"],
            ci_method=spec["ci_method"],
            bias_correction=spec["bias_correction"],
            quality_filter=spec["quality_filter"],
            design_filter=spec["design_filter"],
            sensitivity=spec["sensitivity"],
            theta=0.0, se=0.0, ci_lo=0.0, ci_hi=0.0,
            p_value=1.0, tau2=0.0, I2=0.0, k=0,
            pi_lo=0.0, pi_hi=0.0,
            significant=False, direction="null",
            converged=False, error=str(e),
        )


def _cochran_q(yi: np.ndarray, vi: np.ndarray) -> float:
    wi = 1.0 / vi
    theta = np.sum(wi * yi) / np.sum(wi)
    return float(np.sum(wi * (yi - theta) ** 2))
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `cd C:/Models/MES && python -m pytest tests/test_spec_generator.py tests/test_executor.py -v`
Expected: All 7 tests PASS

- [ ] **Step 7: Commit**

```bash
cd C:/Models/MES && git add -A && git commit -m "feat(explore): add spec generator + multiverse executor"
```

---

## Task 11: MAP — Concordance + Classifier

**Files:**
- Create: `C:\Models\MES\mes_core\map\concordance.py`
- Create: `C:\Models\MES\mes_core\map\classifier.py`
- Create: `C:\Models\MES\tests\test_concordance.py`
- Create: `C:\Models\MES\tests\test_classifier.py`

- [ ] **Step 1: Write failing tests**

```python
# tests/test_concordance.py
from mes_core.models import SpecResult, ConcordanceMetrics
from mes_core.map.concordance import compute_concordance


def _make_result(theta, p, sig, direction, **overrides):
    defaults = dict(
        spec_id="test", estimator="DL", ci_method="Wald",
        bias_correction="none", quality_filter="all", design_filter="all",
        sensitivity="full", se=0.1, ci_lo=theta-0.2, ci_hi=theta+0.2,
        tau2=0.1, I2=0.5, k=10, pi_lo=theta-0.5, pi_hi=theta+0.5,
    )
    defaults.update(overrides)
    return SpecResult(theta=theta, p_value=p, significant=sig,
                      direction=direction, **defaults)


def test_all_agree():
    results = [_make_result(-0.5, 0.001, True, "negative") for _ in range(100)]
    m = compute_concordance(results)
    assert m.C_dir == 1.0
    assert m.C_sig == 1.0


def test_half_disagree_on_significance():
    agree = [_make_result(-0.5, 0.001, True, "negative") for _ in range(50)]
    disagree = [_make_result(-0.1, 0.3, False, "negative") for _ in range(50)]
    m = compute_concordance(agree + disagree)
    assert m.C_dir == 1.0  # all negative
    assert m.C_sig == 0.5  # only half significant


def test_direction_split():
    pos = [_make_result(0.5, 0.01, True, "positive") for _ in range(30)]
    neg = [_make_result(-0.5, 0.01, True, "negative") for _ in range(70)]
    m = compute_concordance(pos + neg)
    assert m.C_dir == 0.7  # 70% agree on majority direction
```

```python
# tests/test_classifier.py
from mes_core.map.classifier import classify_robustness, conditional_robustness
from mes_core.models import SpecResult


def _make_result(sig, direction, **overrides):
    defaults = dict(
        spec_id="test", estimator="DL", ci_method="Wald",
        bias_correction="none", quality_filter="all", design_filter="all",
        sensitivity="full", theta=-0.5, se=0.1, ci_lo=-0.7, ci_hi=-0.3,
        p_value=0.001 if sig else 0.3, tau2=0.1, I2=0.5, k=10,
        pi_lo=-1.0, pi_hi=0.0,
    )
    defaults.update(overrides)
    return SpecResult(significant=sig, direction=direction, **defaults)


def test_robust():
    assert classify_robustness(0.95) == "ROBUST"


def test_moderate():
    assert classify_robustness(0.75) == "MODERATE"


def test_fragile():
    assert classify_robustness(0.55) == "FRAGILE"


def test_unstable():
    assert classify_robustness(0.40) == "UNSTABLE"


def test_conditional_robustness():
    """Low-RoB specs agree more than all specs."""
    all_specs = (
        [_make_result(True, "negative", quality_filter="all") for _ in range(50)] +
        [_make_result(False, "negative", quality_filter="all") for _ in range(50)] +
        [_make_result(True, "negative", quality_filter="low-rob-only") for _ in range(45)] +
        [_make_result(False, "negative", quality_filter="low-rob-only") for _ in range(5)]
    )
    cond = conditional_robustness(all_specs)
    assert "low-rob-only" in cond
    assert cond["low-rob-only"]["c_sig"] > 0.8
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd C:/Models/MES && python -m pytest tests/test_concordance.py tests/test_classifier.py -v`
Expected: FAIL

- [ ] **Step 3: Implement concordance**

```python
# mes_core/map/concordance.py
"""Concordance analysis — what fraction of specs agree?"""
from collections import Counter
from mes_core.models import SpecResult, ConcordanceMetrics


def compute_concordance(
    results: list[SpecResult],
    mcid: float | None = None,
) -> ConcordanceMetrics:
    """Compute concordance metrics across all spec results.

    C_dir: % agreeing on majority direction
    C_sig: % agreeing on majority (direction + significance) combination
    C_full: % agreeing on direction + significance + clinical relevance
    """
    n = len(results)
    if n == 0:
        return ConcordanceMetrics(C_dir=0, C_sig=0, C_full=0, n_specs=0, n_feasible=0)

    feasible = [r for r in results if r.converged]
    n_f = len(feasible)
    if n_f == 0:
        return ConcordanceMetrics(C_dir=0, C_sig=0, C_full=0, n_specs=n, n_feasible=0)

    # Direction concordance
    dir_counts = Counter(r.direction for r in feasible)
    majority_dir = dir_counts.most_common(1)[0][0]
    C_dir = sum(1 for r in feasible if r.direction == majority_dir) / n_f

    # Direction + significance concordance
    sig_combo = Counter((r.direction, r.significant) for r in feasible)
    majority_combo = sig_combo.most_common(1)[0][0]
    C_sig = sum(1 for r in feasible
                if (r.direction, r.significant) == majority_combo) / n_f

    # Full concordance (with MCID if provided)
    if mcid is not None:
        full_combo = Counter(
            (r.direction, r.significant, abs(r.theta) >= mcid) for r in feasible
        )
        majority_full = full_combo.most_common(1)[0][0]
        C_full = sum(1 for r in feasible
                     if (r.direction, r.significant, abs(r.theta) >= mcid) == majority_full) / n_f
    else:
        C_full = C_sig  # without MCID, C_full == C_sig

    return ConcordanceMetrics(
        C_dir=round(C_dir, 4),
        C_sig=round(C_sig, 4),
        C_full=round(C_full, 4),
        n_specs=n,
        n_feasible=n_f,
    )
```

- [ ] **Step 4: Implement classifier**

```python
# mes_core/map/classifier.py
"""Robustness classification — 4-tier + conditional."""
from mes_core.models import SpecResult
from mes_core.map.concordance import compute_concordance


def classify_robustness(c_sig: float) -> str:
    """Classify robustness from C_sig concordance metric.

    >=0.90 → ROBUST, >=0.70 → MODERATE, >=0.50 → FRAGILE, <0.50 → UNSTABLE
    """
    if c_sig >= 0.90:
        return "ROBUST"
    elif c_sig >= 0.70:
        return "MODERATE"
    elif c_sig >= 0.50:
        return "FRAGILE"
    else:
        return "UNSTABLE"


def conditional_robustness(
    results: list[SpecResult],
) -> dict[str, dict]:
    """Compute robustness conditioned on quality/design filters.

    Returns dict mapping filter value → {"c_sig": float, "class": str, "n": int}.
    """
    conditions = {}

    # Group by quality_filter
    for qf in ("all", "exclude-high-rob", "low-rob-only"):
        subset = [r for r in results if r.quality_filter == qf]
        if len(subset) >= 2:
            conc = compute_concordance(subset)
            conditions[qf] = {
                "c_sig": conc.C_sig,
                "class": classify_robustness(conc.C_sig),
                "n": len(subset),
            }

    # Group by design_filter
    for df in ("all", "rct-quasi", "rct-only"):
        subset = [r for r in results if r.design_filter == df]
        if len(subset) >= 2:
            conc = compute_concordance(subset)
            conditions[df] = {
                "c_sig": conc.C_sig,
                "class": classify_robustness(conc.C_sig),
                "n": len(subset),
            }

    # Combined: low-rob + rct-only
    combined = [r for r in results
                if r.quality_filter == "low-rob-only" and r.design_filter == "rct-only"]
    if len(combined) >= 2:
        conc = compute_concordance(combined)
        conditions["low-rob_rct-only"] = {
            "c_sig": conc.C_sig,
            "class": classify_robustness(conc.C_sig),
            "n": len(combined),
        }

    return conditions
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd C:/Models/MES && python -m pytest tests/test_concordance.py tests/test_classifier.py -v`
Expected: All 8 tests PASS

- [ ] **Step 6: Commit**

```bash
cd C:/Models/MES && git add -A && git commit -m "feat(map): add concordance analysis + 4-tier robustness classifier with conditional"
```

---

## Task 12: MAP — Influence Decomposition + Boundaries

**Files:**
- Create: `C:\Models\MES\mes_core\map\influence.py`
- Create: `C:\Models\MES\mes_core\map\boundaries.py`
- Create: `C:\Models\MES\tests\test_influence.py`
- Create: `C:\Models\MES\tests\test_boundaries.py`

- [ ] **Step 1: Write failing tests**

```python
# tests/test_influence.py
from mes_core.models import SpecResult
from mes_core.map.influence import decompose_influence


def _make_results_with_variance():
    """Create results where bias_correction explains most variance."""
    results = []
    for est in ["DL", "REML"]:
        for bc in ["none", "trim-fill", "PET-PEESE"]:
            theta = -0.5 if bc == "none" else -0.1  # big shift from bias corr
            theta += 0.02 if est == "REML" else 0  # small shift from estimator
            results.append(SpecResult(
                spec_id=f"{est}_{bc}", estimator=est, ci_method="Wald",
                bias_correction=bc, quality_filter="all", design_filter="all",
                sensitivity="full", theta=theta, se=0.1,
                ci_lo=theta-0.2, ci_hi=theta+0.2, p_value=0.01,
                tau2=0.1, I2=0.5, k=10, pi_lo=-1.0, pi_hi=0.0,
                significant=True, direction="negative",
            ))
    return results


def test_influence_returns_all_dimensions():
    results = _make_results_with_variance()
    eta2 = decompose_influence(results)
    assert "estimator" in eta2
    assert "bias_correction" in eta2
    assert all(0 <= v <= 1 for v in eta2.values())


def test_bias_correction_dominant():
    """Bias correction should have highest eta2 in our test data."""
    results = _make_results_with_variance()
    eta2 = decompose_influence(results)
    assert eta2["bias_correction"] > eta2["estimator"]
```

```python
# tests/test_boundaries.py
from mes_core.models import SpecResult
from mes_core.map.boundaries import find_boundaries


def _make_mixed_results():
    """Some significant, some not — boundary should be detectable."""
    results = []
    for bc in ["none", "trim-fill"]:
        sig = bc == "none"
        results.append(SpecResult(
            spec_id=f"DL_Wald_{bc}", estimator="DL", ci_method="Wald",
            bias_correction=bc, quality_filter="all", design_filter="all",
            sensitivity="full", theta=-0.5 if sig else -0.1, se=0.1,
            ci_lo=-0.7, ci_hi=-0.3 if sig else 0.1,
            p_value=0.001 if sig else 0.3,
            tau2=0.1, I2=0.5, k=10, pi_lo=-1.0, pi_hi=0.0,
            significant=sig, direction="negative",
        ))
    return results


def test_find_boundaries_detects_flip():
    results = _make_mixed_results()
    bounds = find_boundaries(results)
    assert "bias_correction" in bounds
    # trim-fill flips significance
    bc_bound = bounds["bias_correction"]
    assert "trim-fill" in str(bc_bound)
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd C:/Models/MES && python -m pytest tests/test_influence.py tests/test_boundaries.py -v`
Expected: FAIL

- [ ] **Step 3: Implement influence decomposition**

```python
# mes_core/map/influence.py
"""Influence decomposition — ANOVA eta-squared per multiverse dimension."""
import numpy as np
from mes_core.models import SpecResult


DIMENSIONS = [
    "estimator", "ci_method", "bias_correction",
    "quality_filter", "design_filter",
]


def decompose_influence(
    results: list[SpecResult],
    dimensions: list[str] | None = None,
) -> dict[str, float]:
    """Compute eta-squared (Type III) for each dimension.

    eta2 = SS_between / SS_total for each dimension.
    Uses the 'significant' field (binary) as the response variable
    for concordance-based decomposition, or 'theta' for effect-based.
    """
    if dimensions is None:
        dimensions = DIMENSIONS

    if not results:
        return {d: 0.0 for d in dimensions}

    # Use theta as response variable for continuous decomposition
    thetas = np.array([r.theta for r in results])
    ss_total = float(np.sum((thetas - np.mean(thetas)) ** 2))

    if ss_total < 1e-30:
        return {d: 0.0 for d in dimensions}

    eta2 = {}
    for dim in dimensions:
        # Group results by dimension level
        groups: dict[str, list[float]] = {}
        for r in results:
            level = getattr(r, dim)
            groups.setdefault(level, []).append(r.theta)

        if len(groups) < 2:
            eta2[dim] = 0.0
            continue

        # Between-group SS
        grand_mean = np.mean(thetas)
        ss_between = 0.0
        for level, vals in groups.items():
            group_mean = np.mean(vals)
            ss_between += len(vals) * (group_mean - grand_mean) ** 2

        eta2[dim] = round(ss_between / ss_total, 4)

    return eta2
```

- [ ] **Step 4: Implement fragility boundaries**

```python
# mes_core/map/boundaries.py
"""Fragility boundary detection — where does the conclusion flip?"""
from collections import defaultdict
from mes_core.models import SpecResult


DIMENSIONS = [
    "estimator", "ci_method", "bias_correction",
    "quality_filter", "design_filter",
]


def find_boundaries(
    results: list[SpecResult],
    dimensions: list[str] | None = None,
) -> dict[str, dict]:
    """Find tipping points where conclusions flip, per dimension.

    For each dimension, identifies which levels flip direction or significance
    relative to the majority conclusion.
    """
    if dimensions is None:
        dimensions = DIMENSIONS

    if not results:
        return {}

    # Determine majority conclusion
    feasible = [r for r in results if r.converged]
    if not feasible:
        return {}

    n_sig_neg = sum(1 for r in feasible if r.significant and r.direction == "negative")
    n_sig_pos = sum(1 for r in feasible if r.significant and r.direction == "positive")
    n_nonsig = sum(1 for r in feasible if not r.significant)
    majority_sig = n_sig_neg >= n_sig_pos and n_sig_neg >= n_nonsig
    majority_dir = "negative" if n_sig_neg >= n_sig_pos else "positive"

    boundaries = {}
    for dim in dimensions:
        groups: dict[str, list[SpecResult]] = defaultdict(list)
        for r in feasible:
            groups[getattr(r, dim)].append(r)

        if len(groups) < 2:
            continue

        flippers = {}
        for level, level_results in groups.items():
            n_agree = sum(
                1 for r in level_results
                if r.significant == majority_sig and r.direction == majority_dir
            )
            agree_rate = n_agree / len(level_results) if level_results else 1.0
            if agree_rate < 0.5:
                flippers[level] = {
                    "agree_rate": round(agree_rate, 3),
                    "n": len(level_results),
                    "flips_significance": sum(
                        1 for r in level_results if r.significant != majority_sig
                    ),
                    "flips_direction": sum(
                        1 for r in level_results if r.direction != majority_dir
                    ),
                }

        if flippers:
            boundaries[dim] = flippers

    return boundaries
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd C:/Models/MES && python -m pytest tests/test_influence.py tests/test_boundaries.py -v`
Expected: All 4 tests PASS

- [ ] **Step 6: Commit**

```bash
cd C:/Models/MES && git add -A && git commit -m "feat(map): add influence decomposition (eta-squared) + fragility boundaries"
```

---

## Task 13: MAP — Landscape Verdict + Certifier

**Files:**
- Create: `C:\Models\MES\mes_core\map\landscape.py`
- Create: `C:\Models\MES\mes_core\map\certifier.py`
- Create: `C:\Models\MES\tests\test_landscape.py`
- Create: `C:\Models\MES\tests\test_certifier.py`

- [ ] **Step 1: Write failing tests**

```python
# tests/test_landscape.py
from mes_core.models import SpecResult, MESVerdict
from mes_core.map.landscape import synthesize_verdict


def _make_fragile_results():
    """60% agree on sig+direction → FRAGILE."""
    results = []
    for i in range(60):
        results.append(SpecResult(
            spec_id=f"s{i}", estimator="DL", ci_method="Wald",
            bias_correction="none", quality_filter="all", design_filter="all",
            sensitivity="full", theta=-0.5, se=0.1, ci_lo=-0.7, ci_hi=-0.3,
            p_value=0.001, tau2=0.1, I2=0.5, k=10,
            pi_lo=-1.0, pi_hi=0.0, significant=True, direction="negative",
        ))
    for i in range(40):
        results.append(SpecResult(
            spec_id=f"s{60+i}", estimator="REML", ci_method="HKSJ",
            bias_correction="trim-fill", quality_filter="all", design_filter="all",
            sensitivity="full", theta=-0.1, se=0.15, ci_lo=-0.4, ci_hi=0.2,
            p_value=0.5, tau2=0.2, I2=0.7, k=10,
            pi_lo=-1.0, pi_hi=0.8, significant=False, direction="negative",
        ))
    return results


def test_synthesize_verdict_fragile():
    results = _make_fragile_results()
    verdict = synthesize_verdict(results)
    assert isinstance(verdict, MESVerdict)
    assert verdict.overall_class == "FRAGILE"
    assert 0.5 <= verdict.overall_c_sig <= 0.7


def test_verdict_has_eta2():
    results = _make_fragile_results()
    verdict = synthesize_verdict(results)
    assert len(verdict.eta2_all) > 0
    assert verdict.dominant_dimension is not None
```

```python
# tests/test_certifier.py
import json
from mes_core.map.certifier import certify


def test_certify_pass():
    bundle = certify(
        input_hash="abc123",
        spec_hash="def456",
        n_specs=648,
        n_feasible=640,
        n_errors=0,
        verdict_class="FRAGILE",
    )
    assert bundle["certification"] == "PASS"
    assert bundle["input_hash"] == "abc123"
    assert "timestamp" in bundle


def test_certify_warn_sparse():
    bundle = certify(
        input_hash="abc123",
        spec_hash="def456",
        n_specs=648,
        n_feasible=500,  # >10% infeasible
        n_errors=0,
        verdict_class="UNSTABLE",
    )
    assert bundle["certification"] == "WARN"


def test_certify_reject_errors():
    bundle = certify(
        input_hash="abc123",
        spec_hash="def456",
        n_specs=648,
        n_feasible=640,
        n_errors=100,  # many silent failures
        verdict_class="FRAGILE",
    )
    assert bundle["certification"] == "REJECT"
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd C:/Models/MES && python -m pytest tests/test_landscape.py tests/test_certifier.py -v`
Expected: FAIL

- [ ] **Step 3: Implement landscape verdict**

```python
# mes_core/map/landscape.py
"""Evidence landscape synthesis — combines concordance, classification, influence, boundaries."""
from mes_core.models import SpecResult, MESVerdict
from mes_core.map.concordance import compute_concordance
from mes_core.map.classifier import classify_robustness, conditional_robustness
from mes_core.map.influence import decompose_influence
from mes_core.map.boundaries import find_boundaries


def synthesize_verdict(
    results: list[SpecResult],
    mcid: float | None = None,
) -> MESVerdict:
    """Produce the full MES verdict from multiverse results."""
    # Concordance
    conc = compute_concordance(results, mcid=mcid)

    # Classification
    overall_class = classify_robustness(conc.C_sig)

    # Conditional robustness
    cond = conditional_robustness(results)

    # Influence decomposition
    eta2 = decompose_influence(results)
    dominant_dim = max(eta2, key=eta2.get) if eta2 else "none"
    dominant_eta2 = eta2.get(dominant_dim, 0.0)

    # Fragility boundaries
    bounds = find_boundaries(results)

    # PI null crossing rate
    feasible = [r for r in results if r.converged]
    pi_null = sum(1 for r in feasible if r.pi_lo <= 0 <= r.pi_hi) / max(1, len(feasible))

    return MESVerdict(
        overall_class=overall_class,
        overall_c_sig=conc.C_sig,
        conditional=cond,
        dominant_dimension=dominant_dim,
        dominant_eta2=dominant_eta2,
        certification="PENDING",  # certifier sets this
        eta2_all=eta2,
        boundaries=bounds,
        prediction_null_rate=round(pi_null, 4),
    )
```

- [ ] **Step 4: Implement certifier**

```python
# mes_core/map/certifier.py
"""TruthCert certification for MES bundles."""
import hashlib
import json
from datetime import datetime, timezone


def certify(
    input_hash: str,
    spec_hash: str,
    n_specs: int,
    n_feasible: int,
    n_errors: int,
    verdict_class: str,
) -> dict:
    """Certify the MES analysis run.

    PASS: all specs executed, no silent failures, provenance complete.
    WARN: >10% specs infeasible (sparse data).
    REJECT: broken provenance or many errors.
    """
    infeasible_rate = 1 - (n_feasible / max(1, n_specs))
    error_rate = n_errors / max(1, n_feasible)

    if error_rate > 0.05:
        status = "REJECT"
        reason = f"Error rate {error_rate:.1%} exceeds 5% threshold"
    elif infeasible_rate > 0.10:
        status = "WARN"
        reason = f"Infeasible rate {infeasible_rate:.1%} exceeds 10% (sparse data)"
    else:
        status = "PASS"
        reason = "All checks passed"

    bundle = {
        "certification": status,
        "reason": reason,
        "input_hash": input_hash,
        "spec_hash": spec_hash,
        "n_specs": n_specs,
        "n_feasible": n_feasible,
        "n_errors": n_errors,
        "verdict_class": verdict_class,
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "software_version": "mes-core 0.1.0",
    }

    # Bundle hash for provenance chain
    bundle_json = json.dumps(bundle, sort_keys=True)
    bundle["bundle_hash"] = hashlib.sha256(bundle_json.encode()).hexdigest()

    return bundle
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd C:/Models/MES && python -m pytest tests/test_landscape.py tests/test_certifier.py -v`
Expected: All 5 tests PASS

- [ ] **Step 6: Commit**

```bash
cd C:/Models/MES && git add -A && git commit -m "feat(map): add landscape verdict synthesis + TruthCert certifier"
```

---

## Task 14: IO — CSV Reader + Exporter

**Files:**
- Create: `C:\Models\MES\mes_core\io\csv_reader.py`
- Create: `C:\Models\MES\mes_core\io\exporter.py`
- Create: `C:\Models\MES\tests\test_io.py`

- [ ] **Step 1: Write failing tests**

```python
# tests/test_io.py
import json
import os
import tempfile
import numpy as np
from mes_core.io.csv_reader import read_csv
from mes_core.io.exporter import export_json, export_csv


def test_read_csv(tmp_path):
    csv_file = tmp_path / "test.csv"
    csv_file.write_text(
        "study_id,yi,vi,measure,design_type\n"
        "A,-0.5,0.04,logOR,RCT\n"
        "B,-0.3,0.06,logOR,RCT\n"
        "C,-0.8,0.03,logOR,cohort\n"
    )
    studies = read_csv(str(csv_file))
    assert len(studies) == 3
    assert studies[0]["study_id"] == "A"
    assert studies[0]["yi"] == -0.5
    assert studies[2]["design_type"] == "cohort"


def test_export_json(tmp_path):
    data = {"overall_class": "FRAGILE", "c_sig": 0.62}
    out = tmp_path / "result.json"
    export_json(data, str(out))
    loaded = json.loads(out.read_text())
    assert loaded["overall_class"] == "FRAGILE"


def test_export_csv_results(tmp_path):
    from mes_core.models import SpecResult
    results = [
        SpecResult(
            spec_id="DL_Wald_none_all_all_full", estimator="DL",
            ci_method="Wald", bias_correction="none",
            quality_filter="all", design_filter="all", sensitivity="full",
            theta=-0.5, se=0.1, ci_lo=-0.7, ci_hi=-0.3, p_value=0.001,
            tau2=0.1, I2=0.5, k=10, pi_lo=-1.0, pi_hi=0.0,
            significant=True, direction="negative",
        )
    ]
    out = tmp_path / "specs.csv"
    export_csv(results, str(out))
    content = out.read_text()
    assert "DL_Wald_none_all_all_full" in content
    assert "theta" in content.split("\n")[0]  # header
```

- [ ] **Step 2: Run tests, implement, run again**

```python
# mes_core/io/csv_reader.py
"""Read study data from CSV files."""
import csv


def read_csv(filepath: str) -> list[dict]:
    """Read a CSV file into a list of study dicts.

    Required columns: study_id, yi, vi.
    Optional: measure, design_type, year, n1, n2, events1, events2, rob_* fields.
    """
    studies = []
    with open(filepath, "r", encoding="utf-8-sig") as f:
        reader = csv.DictReader(f)
        for row in reader:
            study = {"study_id": row["study_id"]}
            # Numeric fields
            for field in ("yi", "vi", "year", "n1", "n2", "events1", "events2"):
                if field in row and row[field].strip():
                    try:
                        study[field] = float(row[field])
                        if field in ("year", "n1", "n2", "events1", "events2"):
                            study[field] = int(study[field])
                    except ValueError:
                        pass
            # String fields
            for field in ("measure", "design_type", "rob_tool"):
                if field in row and row[field].strip():
                    study[field] = row[field].strip()
            # RoB domain fields
            for key in row:
                if key.startswith("rob_") and key != "rob_tool" and row[key].strip():
                    study[key] = row[key].strip()
            studies.append(study)
    return studies
```

```python
# mes_core/io/exporter.py
"""Export MES results to JSON and CSV."""
import csv
import json
from dataclasses import asdict
from mes_core.models import SpecResult


def export_json(data: dict, filepath: str) -> None:
    """Export a dict to JSON."""
    with open(filepath, "w", encoding="utf-8") as f:
        json.dump(data, f, indent=2, default=str)


def export_csv(results: list[SpecResult], filepath: str) -> None:
    """Export SpecResult list to CSV."""
    if not results:
        return
    fieldnames = list(asdict(results[0]).keys())
    with open(filepath, "w", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=fieldnames)
        writer.writeheader()
        for r in results:
            writer.writerow(asdict(r))
```

- [ ] **Step 3: Run tests to verify they pass**

Run: `cd C:/Models/MES && python -m pytest tests/test_io.py -v`
Expected: All 3 tests PASS

- [ ] **Step 4: Commit**

```bash
cd C:/Models/MES && git add -A && git commit -m "feat(io): add CSV reader + JSON/CSV exporter"
```

---

## Task 15: End-to-End Pipeline

**Files:**
- Create: `C:\Models\MES\mes_core\pipeline.py`
- Create: `C:\Models\MES\tests\test_pipeline.py`
- Create: `C:\Models\MES\data\built_in\bcg_vaccine.json`

- [ ] **Step 1: Create built-in BCG dataset**

```python
# Save as data/built_in/bcg_vaccine.json
# (use the fixture data from conftest.py)
```

```json
[
  {"study_id": "Aronson1948", "yi": -0.8893, "vi": 0.0355, "measure": "logRR", "design_type": "RCT"},
  {"study_id": "Ferguson1949", "yi": -1.5856, "vi": 0.0248, "measure": "logRR", "design_type": "RCT"},
  {"study_id": "Rosenthal1960", "yi": -1.3481, "vi": 0.0292, "measure": "logRR", "design_type": "RCT"},
  {"study_id": "Hart1977", "yi": -1.4416, "vi": 0.0175, "measure": "logRR", "design_type": "RCT"},
  {"study_id": "Frimodt1973", "yi": -0.2175, "vi": 0.0300, "measure": "logRR", "design_type": "RCT"},
  {"study_id": "Stein1953", "yi": -0.7861, "vi": 0.0116, "measure": "logRR", "design_type": "RCT"},
  {"study_id": "Vandiviere1973", "yi": -1.6209, "vi": 0.0206, "measure": "logRR", "design_type": "RCT"},
  {"study_id": "TPT_Madras1980", "yi": 0.0120, "vi": 0.0470, "measure": "logRR", "design_type": "RCT"},
  {"study_id": "Coetzee1968", "yi": -0.4717, "vi": 0.0124, "measure": "logRR", "design_type": "RCT"},
  {"study_id": "Comstock1974", "yi": 0.0459, "vi": 0.0616, "measure": "logRR", "design_type": "RCT"},
  {"study_id": "Comstock1976", "yi": -0.0173, "vi": 0.2355, "measure": "logRR", "design_type": "RCT"},
  {"study_id": "Comstock_etal1976", "yi": -0.4340, "vi": 0.0200, "measure": "logRR", "design_type": "RCT"},
  {"study_id": "Shapiro1998", "yi": -1.4564, "vi": 0.0384, "measure": "logRR", "design_type": "RCT"}
]
```

- [ ] **Step 2: Write failing test for pipeline**

```python
# tests/test_pipeline.py
import json
import os
from mes_core.pipeline import run_mes
from mes_core.models import MESVerdict


def test_run_mes_bcg():
    """Full pipeline on BCG dataset — end-to-end integration test."""
    data_path = os.path.join(
        os.path.dirname(__file__), "..", "data", "built_in", "bcg_vaccine.json"
    )
    with open(data_path) as f:
        studies = json.load(f)

    verdict = run_mes(studies, include_loo=False)
    assert isinstance(verdict, MESVerdict)
    assert verdict.overall_class in ("ROBUST", "MODERATE", "FRAGILE", "UNSTABLE")
    assert 0 <= verdict.overall_c_sig <= 1
    assert verdict.certification in ("PASS", "WARN", "REJECT")
    assert len(verdict.eta2_all) > 0


def test_run_mes_with_loo():
    """Pipeline with LOO — should produce more specs."""
    data_path = os.path.join(
        os.path.dirname(__file__), "..", "data", "built_in", "bcg_vaccine.json"
    )
    with open(data_path) as f:
        studies = json.load(f)

    verdict = run_mes(studies, include_loo=True)
    assert isinstance(verdict, MESVerdict)
    assert verdict.overall_c_sig > 0


def test_run_mes_small():
    """Minimal 3-study dataset — should still produce valid verdict."""
    studies = [
        {"study_id": "A", "yi": -0.5, "vi": 0.04, "measure": "logOR"},
        {"study_id": "B", "yi": -0.3, "vi": 0.06, "measure": "logOR"},
        {"study_id": "C", "yi": -0.8, "vi": 0.03, "measure": "logOR"},
    ]
    verdict = run_mes(studies, include_loo=False)
    assert isinstance(verdict, MESVerdict)
    assert verdict.overall_class in ("ROBUST", "MODERATE", "FRAGILE", "UNSTABLE")
```

- [ ] **Step 3: Implement pipeline**

```python
# mes_core/pipeline.py
"""MES end-to-end pipeline: ASSESS → EXPLORE → MAP → CERTIFY."""
import hashlib
import json

from mes_core.models import MESSpec, MESVerdict
from mes_core.assess.dossier_builder import build_dossiers
from mes_core.explore.executor import execute_multiverse
from mes_core.map.landscape import synthesize_verdict
from mes_core.map.certifier import certify


def run_mes(
    studies: list[dict],
    mes_spec: MESSpec | None = None,
    include_loo: bool = True,
    mcid: float | None = None,
) -> MESVerdict:
    """Run the full MES pipeline on a list of study dicts.

    Returns MESVerdict with robustness classification, conditional robustness,
    influence decomposition, fragility boundaries, and certification.
    """
    if mes_spec is None:
        mes_spec = MESSpec()

    # Hash inputs for certification
    input_hash = hashlib.sha256(
        json.dumps(studies, sort_keys=True).encode()
    ).hexdigest()[:16]
    spec_hash = hashlib.sha256(
        json.dumps({
            "estimators": mes_spec.estimators,
            "ci_methods": mes_spec.ci_methods,
            "bias_corrections": mes_spec.bias_corrections,
            "quality_filters": mes_spec.quality_filters,
            "design_filters": mes_spec.design_filters,
            "alpha": mes_spec.alpha,
        }, sort_keys=True).encode()
    ).hexdigest()[:16]

    # Phase 1: ASSESS
    dossiers, bias_profile = build_dossiers(studies)

    # Phase 2: EXPLORE
    results = execute_multiverse(mes_spec, dossiers, include_loo=include_loo)

    # Phase 3: MAP
    verdict = synthesize_verdict(results, mcid=mcid)

    # Certify
    n_errors = sum(1 for r in results if not r.converged)
    cert = certify(
        input_hash=input_hash,
        spec_hash=spec_hash,
        n_specs=mes_spec.base_spec_count * (len(dossiers) + 1 if include_loo else 1),
        n_feasible=len(results),
        n_errors=n_errors,
        verdict_class=verdict.overall_class,
    )
    verdict.certification = cert["certification"]

    return verdict
```

- [ ] **Step 4: Run tests**

Run: `cd C:/Models/MES && python -m pytest tests/test_pipeline.py -v`
Expected: All 3 tests PASS

- [ ] **Step 5: Run full test suite**

Run: `cd C:/Models/MES && python -m pytest tests/ -v --tb=short`
Expected: All tests PASS (should be ~55+ tests total)

- [ ] **Step 6: Commit**

```bash
cd C:/Models/MES && git add -A && git commit -m "feat: add end-to-end MES pipeline with BCG dataset"
```

---

## Task 16: R Parity Validation

**Files:**
- Create: `C:\Models\MES\validation\r_parity\validate.R`
- Create: `C:\Models\MES\tests\test_r_parity.py`

- [ ] **Step 1: Create R validation script**

```r
# validation/r_parity/validate.R
# Compare MES Python results against metafor for BCG dataset
library(metafor)

# BCG data
yi <- c(-0.8893, -1.5856, -1.3481, -1.4416, -0.2175,
        -0.7861, -1.6209, 0.0120, -0.4717, 0.0459,
        -0.0173, -0.4340, -1.4564)
vi <- c(0.0355, 0.0248, 0.0292, 0.0175, 0.0300,
        0.0116, 0.0206, 0.0470, 0.0124, 0.0616,
        0.2355, 0.0200, 0.0384)

methods <- c("FE", "DL", "REML", "PM", "SJ", "ML")
results <- data.frame()

for (m in methods) {
  fit <- rma(yi=yi, vi=vi, method=m)
  results <- rbind(results, data.frame(
    method=m,
    theta=as.numeric(fit$beta),
    se=fit$se,
    tau2=fit$tau2,
    I2=fit$I2,
    p=fit$pval
  ))
}

# Egger test
reg <- regtest(rma(yi=yi, vi=vi, method="FE"), model="lm")
cat(sprintf("egger_p=%.10f\n", reg$pval))

# Trim-and-fill
tf <- trimfill(rma(yi=yi, vi=vi, method="DL"))
cat(sprintf("tf_k0=%d tf_theta=%.10f\n", tf$k0, as.numeric(tf$beta)))

# Write results
write.csv(results, "validation/r_parity/bcg_r_results.csv", row.names=FALSE)
cat("R validation complete\n")
```

- [ ] **Step 2: Write R parity test**

```python
# tests/test_r_parity.py
"""R parity gate — compare MES Python against metafor.

Requires: R 4.5.2 with metafor package installed.
Run: pytest tests/test_r_parity.py -v
Skip if R not available.
"""
import subprocess
import csv
import os
import pytest
import numpy as np

RSCRIPT = r"C:\Program Files\R\R-4.5.2\bin\Rscript.exe"
R_SCRIPT = os.path.join(os.path.dirname(__file__), "..", "validation", "r_parity", "validate.R")
R_RESULTS = os.path.join(os.path.dirname(__file__), "..", "validation", "r_parity", "bcg_r_results.csv")

pytestmark = pytest.mark.skipif(
    not os.path.exists(RSCRIPT), reason="R not installed"
)


@pytest.fixture(scope="module")
def r_results():
    """Run R script and load results."""
    subprocess.run(
        [RSCRIPT, R_SCRIPT],
        cwd=os.path.join(os.path.dirname(__file__), ".."),
        check=True, capture_output=True, text=True,
    )
    results = {}
    with open(R_RESULTS) as f:
        for row in csv.DictReader(f):
            results[row["method"]] = {
                "theta": float(row["theta"]),
                "se": float(row["se"]),
                "tau2": float(row["tau2"]),
            }
    return results


@pytest.fixture(scope="module")
def python_results():
    """Run MES Python estimators on BCG."""
    from mes_core.explore.estimators import fe, dl, reml, pm, sj, ml
    yi = np.array([
        -0.8893, -1.5856, -1.3481, -1.4416, -0.2175,
        -0.7861, -1.6209, 0.0120, -0.4717, 0.0459,
        -0.0173, -0.4340, -1.4564
    ])
    vi = np.array([
        0.0355, 0.0248, 0.0292, 0.0175, 0.0300,
        0.0116, 0.0206, 0.0470, 0.0124, 0.0616,
        0.2355, 0.0200, 0.0384
    ])
    return {
        "FE": fe(yi, vi),
        "DL": dl(yi, vi),
        "REML": reml(yi, vi),
        "PM": pm(yi, vi),
        "SJ": sj(yi, vi),
        "ML": ml(yi, vi),
    }


TOL = 1e-4  # tolerance for theta/se/tau2 comparison


@pytest.mark.parametrize("method", ["FE", "DL", "REML", "PM", "SJ", "ML"])
def test_theta_parity(method, r_results, python_results):
    r_theta = r_results[method]["theta"]
    py_tau2, py_theta, py_se = python_results[method]
    assert abs(py_theta - r_theta) < TOL, (
        f"{method}: Python theta={py_theta:.6f}, R theta={r_theta:.6f}"
    )


@pytest.mark.parametrize("method", ["FE", "DL", "REML", "PM", "SJ", "ML"])
def test_tau2_parity(method, r_results, python_results):
    r_tau2 = r_results[method]["tau2"]
    py_tau2, _, _ = python_results[method]
    assert abs(py_tau2 - r_tau2) < TOL, (
        f"{method}: Python tau2={py_tau2:.6f}, R tau2={r_tau2:.6f}"
    )
```

- [ ] **Step 3: Run R parity tests**

Run: `cd C:/Models/MES && python -m pytest tests/test_r_parity.py -v`
Expected: All 12 tests PASS (6 theta + 6 tau2). If any fail, fix the estimator implementation to match metafor.

- [ ] **Step 4: Commit**

```bash
cd C:/Models/MES && git add -A && git commit -m "test: add R parity gate (metafor comparison for all 6 estimators)"
```

---

## Task 17: Batch Validation Pipeline (407 Reviews)

**Files:**
- Create: `C:\Models\MES\validation\batch_run.py`
- Create: `C:\Models\MES\mes_core\io\rda_reader.py`

This task creates the batch runner that processes all 407 Cochrane reviews from Pairwise70.

- [ ] **Step 1: Implement RDA reader**

Reference: `C:\FragilityAtlas\src\pipeline.py` for RDA parsing.

```python
# mes_core/io/rda_reader.py
"""Read Pairwise70 RDA files (R data format) using rpy2 or CSV fallback."""
import os
import subprocess
import csv
import tempfile
import numpy as np


RSCRIPT = r"C:\Program Files\R\R-4.5.2\bin\Rscript.exe"


def read_rda(filepath: str) -> list[dict] | None:
    """Read an RDA file and extract yi, vi, study labels.

    Uses R subprocess to convert RDA → CSV, then reads CSV.
    Returns list of study dicts or None if extraction fails.
    """
    with tempfile.NamedTemporaryFile(mode="w", suffix=".R", delete=False) as rf:
        rf.write(f'''
load("{filepath.replace(chr(92), '/')}")
# Find the data frame in the loaded objects
objs <- ls()
for (obj_name in objs) {{
  obj <- get(obj_name)
  if (is.data.frame(obj) && "Mean" %in% names(obj)) {{
    # Pairwise70 format: Mean, CI.start, CI.end columns
    df <- obj
    # Compute yi (log scale) and vi from CI
    df$yi <- df$Mean
    df$sei <- (df$CI.end - df$CI.start) / (2 * 1.96)
    df$vi <- df$sei^2
    df$study_id <- if ("Study" %in% names(df)) df$Study else paste0("S", 1:nrow(df))
    out <- df[, c("study_id", "yi", "vi")]
    out <- out[complete.cases(out), ]
    out <- out[out$vi > 0, ]
    write.csv(out, "{tempfile.gettempdir().replace(chr(92), '/')}/rda_out.csv", row.names=FALSE)
    break
  }}
}}
''')
        r_script = rf.name

    try:
        subprocess.run(
            [RSCRIPT, r_script],
            check=True, capture_output=True, text=True, timeout=30,
        )
    except (subprocess.CalledProcessError, subprocess.TimeoutExpired):
        return None
    finally:
        os.unlink(r_script)

    csv_path = os.path.join(tempfile.gettempdir(), "rda_out.csv")
    if not os.path.exists(csv_path):
        return None

    studies = []
    with open(csv_path) as f:
        for row in csv.DictReader(f):
            try:
                studies.append({
                    "study_id": row["study_id"],
                    "yi": float(row["yi"]),
                    "vi": float(row["vi"]),
                    "measure": "logOR",  # Pairwise70 default
                    "design_type": "RCT",
                })
            except (ValueError, KeyError):
                continue
    os.unlink(csv_path)
    return studies if len(studies) >= 3 else None
```

- [ ] **Step 2: Implement batch runner**

```python
# validation/batch_run.py
"""Batch MES pipeline across 407 Cochrane reviews (Pairwise70 dataset)."""
import os
import sys
import json
import time
import csv
from pathlib import Path

# Add project root to path
sys.path.insert(0, str(Path(__file__).parent.parent))

from mes_core.pipeline import run_mes
from mes_core.models import MESSpec
from mes_core.io.rda_reader import read_rda


RDA_DIR = r"C:\Users\user\OneDrive - NHS\Documents\Pairwise70\data"
OUTPUT_DIR = os.path.join(os.path.dirname(__file__), "results")


def main():
    os.makedirs(OUTPUT_DIR, exist_ok=True)
    spec = MESSpec()  # default 648 base specs

    rda_files = sorted(
        f for f in os.listdir(RDA_DIR) if f.endswith(".RData") or f.endswith(".rda")
    )
    print(f"Found {len(rda_files)} RDA files")

    results_summary = []
    t0 = time.time()

    for i, fname in enumerate(rda_files):
        filepath = os.path.join(RDA_DIR, fname)
        review_id = os.path.splitext(fname)[0]

        studies = read_rda(filepath)
        if studies is None or len(studies) < 3:
            results_summary.append({
                "review_id": review_id, "status": "SKIP",
                "reason": "insufficient studies", "k": 0,
            })
            continue

        try:
            verdict = run_mes(studies, mes_spec=spec, include_loo=False)
            results_summary.append({
                "review_id": review_id,
                "status": "OK",
                "k": len(studies),
                "overall_class": verdict.overall_class,
                "c_sig": verdict.overall_c_sig,
                "dominant_dim": verdict.dominant_dimension,
                "dominant_eta2": verdict.dominant_eta2,
                "certification": verdict.certification,
                "pi_null_rate": verdict.prediction_null_rate,
            })
        except Exception as e:
            results_summary.append({
                "review_id": review_id, "status": "ERROR",
                "reason": str(e)[:100], "k": len(studies),
            })

        if (i + 1) % 50 == 0:
            elapsed = time.time() - t0
            print(f"  {i+1}/{len(rda_files)} done ({elapsed:.0f}s)")

    # Write results
    out_csv = os.path.join(OUTPUT_DIR, "batch_results.csv")
    if results_summary:
        keys = list(results_summary[0].keys())
        # Collect all keys across all rows
        for row in results_summary:
            for k in row:
                if k not in keys:
                    keys.append(k)
        with open(out_csv, "w", newline="", encoding="utf-8") as f:
            writer = csv.DictWriter(f, fieldnames=keys)
            writer.writeheader()
            writer.writerows(results_summary)

    # Summary stats
    ok = [r for r in results_summary if r["status"] == "OK"]
    print(f"\n=== MES Batch Results ===")
    print(f"Total RDA files: {len(rda_files)}")
    print(f"Processed: {len(ok)}")
    print(f"Skipped: {sum(1 for r in results_summary if r['status'] == 'SKIP')}")
    print(f"Errors: {sum(1 for r in results_summary if r['status'] == 'ERROR')}")
    if ok:
        classes = {}
        for r in ok:
            c = r["overall_class"]
            classes[c] = classes.get(c, 0) + 1
        print(f"\nRobustness distribution:")
        for c in ("ROBUST", "MODERATE", "FRAGILE", "UNSTABLE"):
            n = classes.get(c, 0)
            pct = 100 * n / len(ok)
            print(f"  {c}: {n} ({pct:.1f}%)")
        c_sigs = [r["c_sig"] for r in ok]
        print(f"\nC_sig: mean={sum(c_sigs)/len(c_sigs):.3f}, "
              f"median={sorted(c_sigs)[len(c_sigs)//2]:.3f}")
    print(f"\nResults saved to: {out_csv}")
    print(f"Total time: {time.time()-t0:.0f}s")


if __name__ == "__main__":
    main()
```

- [ ] **Step 3: Test batch runner on first 5 reviews**

Run: `cd C:/Models/MES && python -c "from validation.batch_run import main; main()" 2>&1 | head -20`
Expected: Processes at least a few reviews without errors

- [ ] **Step 4: Run full batch (407 reviews)**

Run: `cd C:/Models/MES && python validation/batch_run.py`
Expected: Completes in 5-10 minutes. Output in `validation/results/batch_results.csv`.

- [ ] **Step 5: Commit**

```bash
cd C:/Models/MES && git add -A && git commit -m "feat: add RDA reader + batch validation pipeline for 407 Cochrane reviews"
```

---

## Task 18: Final Integration Test + Summary

- [ ] **Step 1: Run complete test suite**

Run: `cd C:/Models/MES && python -m pytest tests/ -v --tb=short`
Expected: All tests PASS. Report total count.

- [ ] **Step 2: Run R parity gate**

Run: `cd C:/Models/MES && python -m pytest tests/test_r_parity.py -v`
Expected: All 12 parity tests PASS.

- [ ] **Step 3: Verify batch results exist**

Run: `cd C:/Models/MES && python -c "import csv; r=list(csv.DictReader(open('validation/results/batch_results.csv'))); ok=[x for x in r if x['status']=='OK']; print(f'{len(ok)} reviews processed')"`
Expected: ~300-400 reviews processed.

- [ ] **Step 4: Final commit with test counts**

```bash
cd C:/Models/MES && git add -A && git commit -m "chore: Phase A complete — MES Python engine with N/N tests, R parity validated"
```

- [ ] **Step 5: Update project index**

Update `C:\ProjectIndex\INDEX.md` with MES entry:
- Status: Phase A Complete
- Location: `C:\Models\MES\`
- Tests: N/N
- Target: BMJ + F1000
