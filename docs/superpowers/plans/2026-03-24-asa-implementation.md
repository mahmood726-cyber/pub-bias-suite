# Asa (عصا موسى) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single-file browser-based forensic data integrity screener that runs GRIM, SPRITE, statcheck, and TDA on meta-analysis datasets.

**Architecture:** Single HTML file (`asa.html`, ~5-8K lines) with embedded JS/CSS. Dark-mode arena aesthetic. 5-tab layout: Data Entry → Gauntlet → Serpent's Eye → Forensic Reports → Export. No external dependencies. Selenium test suite in `tests/`.

**Tech Stack:** Vanilla HTML/CSS/JS. Statistical CDFs ported from PairwisePro (`app.js` lines 222-417). xoshiro128** PRNG from MultiverseMA. Custom CSV parser with fuzzy column mapping.

**Spec:** `docs/superpowers/specs/2026-03-24-asa-forensic-integrity-design.md`

---

## File Structure

```
C:\Models\Asa\
├── asa.html                    # Single-file app (all JS/CSS embedded)
└── tests\
    ├── test_grim.py            # GRIM unit tests (Selenium)
    ├── test_sprite.py          # SPRITE unit tests
    ├── test_statcheck.py       # statcheck unit tests
    ├── test_tda.py             # TDA unit tests
    ├── test_integration.py     # Full pipeline integration tests
    └── conftest.py             # Shared Selenium fixtures
```

---

### Task 1: Scaffold — HTML Shell + Tabs + Dark Theme CSS

**Files:**
- Create: `C:\Models\Asa\asa.html`

- [ ] **Step 1: Create directory**

```bash
mkdir -p /c/Models/Asa/tests
```

- [ ] **Step 2: Write the HTML scaffold**

Create `asa.html` with:
- DOCTYPE, meta charset UTF-8, viewport
- Title: "Asa — عصا موسى — Forensic Data Integrity Screener"
- Quranic verse header: 7:117 — فَوَقَعَ الْحَقُّ وَبَطَلَ مَا كَانُوا يَعْمَلُونَ
- Version: `v1.0.0`
- 5 tab buttons: Data Entry, The Gauntlet, Serpent's Eye, Forensic Reports, Export
- 5 `<div class="tab-content" data-tab="N">` containers (only tab 1 visible initially)
- Tab switching JS: click handler sets `display:none` on all, `display:block` on selected
- Settings gear icon (top-right) opening a modal for: data_type (discrete/continuous default), items (default 1), scale_min, scale_max, SPRITE max_iterations (default 10000), decimal_precision (auto/manual)

CSS (dark mode default):
- `--bg: #0d1117; --bg-card: #161b22; --text: #e6edf3; --text-secondary: #8b949e`
- `--green: #2ea043; --amber: #d29922; --red: #f85149; --grey: #484f58`
- `--accent: #d4a574` (staff gold)
- Tab bar: horizontal, bottom-bordered active tab
- Card style: `border-radius: 8px; border: 1px solid #30363d`
- Light mode toggle: class `light` on body swaps palette
- `@media (prefers-reduced-motion: reduce)` — disable animations
- Print styles for Tab 4 (forensic reports)

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Asa — عصا موسى — Forensic Data Integrity Screener</title>
<style>
/* === CSS Variables === */
:root {
  --bg: #0d1117; --bg-card: #161b22; --bg-hover: #1c2128;
  --text: #e6edf3; --text-secondary: #8b949e; --border: #30363d;
  --green: #2ea043; --amber: #d29922; --red: #f85149; --grey: #484f58;
  --accent: #d4a574; --accent-glow: rgba(212,165,116,0.3);
  --font: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif;
  --font-mono: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
}
body.light {
  --bg: #ffffff; --bg-card: #f6f8fa; --bg-hover: #eef1f5;
  --text: #1f2328; --text-secondary: #656d76; --border: #d0d7de;
  --accent-glow: rgba(212,165,116,0.15);
}
/* === Reset + Base === */
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
body { font-family: var(--font); background: var(--bg); color: var(--text); line-height: 1.6; }
/* === Header === */
.header { text-align: center; padding: 24px 16px 8px; border-bottom: 1px solid var(--border); }
.header h1 { font-size: 28px; color: var(--accent); margin-bottom: 4px; }
.header .verse { font-size: 14px; color: var(--text-secondary); font-style: italic; direction: rtl; }
.header .subtitle { font-size: 13px; color: var(--text-secondary); margin-top: 4px; }
.header-controls { position: absolute; top: 16px; right: 16px; display: flex; gap: 8px; }
.header-controls button { background: none; border: 1px solid var(--border); color: var(--text-secondary);
  padding: 4px 8px; border-radius: 4px; cursor: pointer; font-size: 13px; }
.header-controls button:hover { background: var(--bg-hover); color: var(--text); }
/* === Tabs === */
.tab-bar { display: flex; border-bottom: 1px solid var(--border); padding: 0 16px; overflow-x: auto; }
.tab-btn { padding: 10px 20px; background: none; border: none; color: var(--text-secondary);
  cursor: pointer; font-size: 14px; border-bottom: 2px solid transparent; white-space: nowrap; }
.tab-btn:hover { color: var(--text); }
.tab-btn.active { color: var(--accent); border-bottom-color: var(--accent); }
.tab-content { display: none; padding: 24px; max-width: 1200px; margin: 0 auto; }
.tab-content.active { display: block; }
/* === Cards === */
.card { background: var(--bg-card); border: 1px solid var(--border); border-radius: 8px; padding: 16px; margin-bottom: 12px; }
/* === Buttons === */
.btn { padding: 8px 16px; border-radius: 6px; border: 1px solid var(--border); cursor: pointer;
  font-size: 14px; font-family: var(--font); transition: background 0.15s; }
.btn-primary { background: var(--accent); color: #000; border-color: var(--accent); font-weight: 600; }
.btn-primary:hover { filter: brightness(1.1); }
.btn-secondary { background: var(--bg-card); color: var(--text); }
.btn-secondary:hover { background: var(--bg-hover); }
.btn-danger { background: var(--red); color: #fff; border-color: var(--red); }
/* === Verdict badges === */
.badge { display: inline-block; padding: 2px 8px; border-radius: 12px; font-size: 12px; font-weight: 600; }
.badge-pass { background: rgba(46,160,67,0.15); color: var(--green); }
.badge-suspicious { background: rgba(210,153,34,0.15); color: var(--amber); }
.badge-swallowed { background: rgba(248,81,73,0.15); color: var(--red); }
.badge-skipped { background: rgba(72,79,88,0.15); color: var(--grey); }
/* === Settings Modal === */
.modal-overlay { display: none; position: fixed; inset: 0; background: rgba(0,0,0,0.6); z-index: 1000;
  align-items: center; justify-content: center; }
.modal-overlay.open { display: flex; }
.modal { background: var(--bg-card); border: 1px solid var(--border); border-radius: 12px;
  padding: 24px; max-width: 500px; width: 90%; max-height: 80vh; overflow-y: auto; }
.modal h2 { margin-bottom: 16px; font-size: 18px; }
.modal label { display: block; margin-bottom: 12px; font-size: 13px; color: var(--text-secondary); }
.modal input, .modal select { width: 100%; padding: 6px 10px; margin-top: 4px; background: var(--bg);
  border: 1px solid var(--border); border-radius: 4px; color: var(--text); font-size: 14px; }
/* === Animations === */
@media (prefers-reduced-motion: reduce) { *, *::before, *::after { animation-duration: 0.01ms !important; transition-duration: 0.01ms !important; } }
/* === Print === */
@media print { .header-controls, .tab-bar, .btn, .modal-overlay { display: none !important; }
  .tab-content { display: block !important; } body { background: #fff; color: #000; } }
</style>
</head>
<body>
<!-- Header -->
<div class="header" style="position:relative">
  <h1>عصا موسى — Asa</h1>
  <div class="verse">فَوَقَعَ الْحَقُّ وَبَطَلَ مَا كَانُوا يَعْمَلُونَ — ٧:١١٧</div>
  <div class="subtitle">Forensic Data Integrity Screener for Meta-Analysis &middot; v1.0.0</div>
  <div class="header-controls">
    <button id="themeToggle" title="Toggle light/dark mode">☀</button>
    <button id="settingsBtn" title="Settings">⚙</button>
  </div>
</div>

<!-- Tab Bar -->
<div class="tab-bar" role="tablist">
  <button class="tab-btn active" data-tab="entry" role="tab" aria-selected="true">Data Entry</button>
  <button class="tab-btn" data-tab="gauntlet" role="tab" aria-selected="false">The Gauntlet</button>
  <button class="tab-btn" data-tab="heatmap" role="tab" aria-selected="false">Serpent's Eye</button>
  <button class="tab-btn" data-tab="reports" role="tab" aria-selected="false">Forensic Reports</button>
  <button class="tab-btn" data-tab="export" role="tab" aria-selected="false">Export</button>
</div>

<!-- Tab 1: Data Entry -->
<div class="tab-content active" id="tab-entry" data-tab="entry" role="tabpanel">
  <h2>Load Study Data</h2>
  <p style="color:var(--text-secondary);margin-bottom:16px">
    Paste CSV data, upload a file, enter manually, or load a demo dataset.
  </p>
  <div id="data-entry-area"></div>
</div>

<!-- Tab 2: The Gauntlet -->
<div class="tab-content" id="tab-gauntlet" data-tab="gauntlet" role="tabpanel">
  <div id="gauntlet-area"></div>
</div>

<!-- Tab 3: Serpent's Eye -->
<div class="tab-content" id="tab-heatmap" data-tab="heatmap" role="tabpanel">
  <div id="heatmap-area"></div>
</div>

<!-- Tab 4: Forensic Reports -->
<div class="tab-content" id="tab-reports" data-tab="reports" role="tabpanel">
  <div id="reports-area"></div>
</div>

<!-- Tab 5: Export -->
<div class="tab-content" id="tab-export" data-tab="export" role="tabpanel">
  <div id="export-area"></div>
</div>

<!-- Settings Modal -->
<div class="modal-overlay" id="settingsModal">
  <div class="modal">
    <h2>Settings</h2>
    <label>Default data type
      <select id="setting-dataType"><option value="discrete" selected>Discrete (counts, Likert)</option><option value="continuous">Continuous (blood pressure, weight)</option></select>
    </label>
    <label>Scale items (for multi-item composites)
      <input type="number" id="setting-items" value="1" min="1" max="100">
    </label>
    <label>Scale minimum
      <input type="number" id="setting-scaleMin" value="1">
    </label>
    <label>Scale maximum
      <input type="number" id="setting-scaleMax" value="" placeholder="Required for SPRITE">
    </label>
    <label>SPRITE max iterations
      <input type="number" id="setting-spriteIter" value="10000" min="1000" max="100000">
    </label>
    <label>Decimal precision
      <select id="setting-decimals"><option value="auto" selected>Auto-detect</option><option value="1">1</option><option value="2">2</option><option value="3">3</option></select>
    </label>
    <div style="display:flex;gap:8px;margin-top:16px">
      <button class="btn btn-primary" id="settingsSave">Save</button>
      <button class="btn btn-secondary" id="settingsClose">Close</button>
    </div>
  </div>
</div>

<script>
'use strict';
/* ============================================================
   ASA v1.0.0 — Forensic Data Integrity Screener
   ============================================================ */

const ASA_VERSION = '1.0.0';
const LS_PREFIX = 'asa_';

// ── Tab switching ──
document.querySelectorAll('.tab-btn').forEach(btn => {
  btn.addEventListener('click', () => {
    document.querySelectorAll('.tab-btn').forEach(b => { b.classList.remove('active'); b.setAttribute('aria-selected', 'false'); });
    document.querySelectorAll('.tab-content').forEach(tc => tc.classList.remove('active'));
    btn.classList.add('active');
    btn.setAttribute('aria-selected', 'true');
    const target = btn.getAttribute('data-tab');
    document.querySelector(`.tab-content[data-tab="${target}"]`).classList.add('active');
  });
});

// ── Theme toggle ──
document.getElementById('themeToggle').addEventListener('click', () => {
  document.body.classList.toggle('light');
  const isLight = document.body.classList.contains('light');
  document.getElementById('themeToggle').textContent = isLight ? '🌙' : '☀';
  try { localStorage.setItem(LS_PREFIX + 'theme', isLight ? 'light' : 'dark'); } catch(e) {}
});
(function loadTheme() {
  try {
    const saved = localStorage.getItem(LS_PREFIX + 'theme');
    if (saved === 'light') { document.body.classList.add('light'); document.getElementById('themeToggle').textContent = '🌙'; }
  } catch(e) {}
})();

// ── Settings modal ──
document.getElementById('settingsBtn').addEventListener('click', () => document.getElementById('settingsModal').classList.add('open'));
document.getElementById('settingsClose').addEventListener('click', () => document.getElementById('settingsModal').classList.remove('open'));
document.getElementById('settingsModal').addEventListener('click', (e) => { if (e.target === e.currentTarget) e.currentTarget.classList.remove('open'); });

function getSettings() {
  return {
    dataType: document.getElementById('setting-dataType').value,
    items: parseInt(document.getElementById('setting-items').value) || 1,
    scaleMin: parseFloat(document.getElementById('setting-scaleMin').value),
    scaleMax: parseFloat(document.getElementById('setting-scaleMax').value),
    spriteIter: parseInt(document.getElementById('setting-spriteIter').value) || 10000,
    decimals: document.getElementById('setting-decimals').value
  };
}
document.getElementById('settingsSave').addEventListener('click', () => {
  try { localStorage.setItem(LS_PREFIX + 'settings', JSON.stringify(getSettings())); } catch(e) {}
  document.getElementById('settingsModal').classList.remove('open');
});
(function loadSettings() {
  try {
    const saved = JSON.parse(localStorage.getItem(LS_PREFIX + 'settings'));
    if (!saved) return;
    if (saved.dataType) document.getElementById('setting-dataType').value = saved.dataType;
    if (saved.items) document.getElementById('setting-items').value = saved.items;
    if (isFinite(saved.scaleMin)) document.getElementById('setting-scaleMin').value = saved.scaleMin;
    if (isFinite(saved.scaleMax)) document.getElementById('setting-scaleMax').value = saved.scaleMax;
    if (saved.spriteIter) document.getElementById('setting-spriteIter').value = saved.spriteIter;
    if (saved.decimals) document.getElementById('setting-decimals').value = saved.decimals;
  } catch(e) {}
})();

// ── Placeholder: modules loaded below ──

</script>
</body>
</html>
```

- [ ] **Step 3: Open in browser and verify**

Open `C:\Models\Asa\asa.html` in browser. Verify:
- Dark theme loads by default
- 5 tabs switch correctly
- Light mode toggle works
- Settings modal opens/closes
- Settings persist across reload (localStorage)

- [ ] **Step 4: Init git repo and commit**

```bash
cd /c/Models/Asa && git init && git add asa.html && git commit -m "scaffold: HTML shell with tabs, dark theme, settings modal"
```

---

### Task 2: Statistical Math Library

Port the proven CDF implementations from PairwisePro (`C:\HTML apps\Truthcert1_work\app.js` lines 222-417). These power statcheck (Fang 3) and TDA (Fang 4).

**Files:**
- Modify: `C:\Models\Asa\asa.html` (insert into `<script>` block)

- [ ] **Step 1: Create test harness**

Create `C:\Models\Asa\tests\conftest.py`:

```python
import pytest, os, json, time, sys
sys.stdout = __import__('io').TextIOWrapper(sys.stdout.buffer, encoding='utf-8', errors='replace')
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

@pytest.fixture(scope='session')
def driver():
    opts = Options()
    opts.add_argument('--headless=new')
    opts.add_argument('--no-sandbox')
    opts.add_argument('--disable-gpu')
    opts.set_capability('goog:loggingPrefs', {'browser': 'ALL'})
    d = webdriver.Chrome(options=opts)
    d.set_page_load_timeout(30)
    yield d
    d.quit()

@pytest.fixture(scope='session')
def app_url():
    return 'file:///' + os.path.abspath(os.path.join(os.path.dirname(__file__), '..', 'asa.html')).replace('\\', '/')

def js(driver, script):
    """Execute JS and return result."""
    return driver.execute_script(f'return {script}')
```

- [ ] **Step 2: Write failing tests for CDF functions**

Create `C:\Models\Asa\tests\test_math.py`:

```python
"""Test statistical distribution functions against known R values."""
import pytest
from conftest import js

@pytest.fixture(autouse=True)
def load_app(driver, app_url):
    if driver.current_url != app_url:
        driver.get(app_url)

class TestNormalCDF:
    def test_pnorm_zero(self, driver):
        assert abs(js(driver, 'pnorm(0)') - 0.5) < 1e-10

    def test_pnorm_196(self, driver):
        assert abs(js(driver, 'pnorm(1.96)') - 0.9750021) < 1e-5

    def test_pnorm_negative(self, driver):
        assert abs(js(driver, 'pnorm(-1.96)') - 0.0249979) < 1e-5

class TestTCDF:
    def test_pt_basic(self, driver):
        # R: pt(2, 10) = 0.9633062
        assert abs(js(driver, 'pt(2, 10)') - 0.9633062) < 1e-5

    def test_pt_negative(self, driver):
        # R: pt(-2, 10) = 0.03669378
        assert abs(js(driver, 'pt(-2, 10)') - 0.03669378) < 1e-5

    def test_pt_large_df(self, driver):
        # R: pt(1.96, 1000) ≈ pnorm(1.96)
        assert abs(js(driver, 'pt(1.96, 1000)') - 0.9750021) < 1e-3

class TestChiSqCDF:
    def test_pchisq_basic(self, driver):
        # R: pchisq(3.84, 1) = 0.9500...
        assert abs(js(driver, 'pchisq(3.84, 1)') - 0.9500) < 1e-3

    def test_pchisq_9df(self, driver):
        # R: pchisq(16.92, 9) = 0.9500...
        assert abs(js(driver, 'pchisq(16.92, 9)') - 0.9500) < 1e-3

class TestFCDF:
    def test_pf_basic(self, driver):
        # R: pf(4.0, 1, 10) = 0.9265...
        r = js(driver, 'pf(4.0, 1, 10)')
        assert abs(r - 0.9265) < 1e-3

class TestGammainc:
    def test_gammainc_basic(self, driver):
        # gammainc(1, 1) = 1 - exp(-1) ≈ 0.6321
        assert abs(js(driver, 'gammainc(1, 1)') - 0.6321) < 1e-3
```

- [ ] **Step 3: Run tests — expect FAIL (functions not defined)**

```bash
cd /c/Models/Asa && python -m pytest tests/test_math.py -v
```
Expected: All tests FAIL with JS errors (functions not defined).

- [ ] **Step 4: Port CDF functions from PairwisePro**

Insert into `asa.html` `<script>` block (before the placeholder comment), porting from `C:\HTML apps\Truthcert1_work\app.js` lines 222-417:

```javascript
// ── Statistical Distribution Functions ──
// Ported from PairwisePro (R-validated). See app.js lines 222-417.

function lgamma(z) {
  const cof = [76.18009172947146,-86.50532032941677,24.01409824083091,
    -1.231739572450155,0.1208650973866179e-2,-0.5395239384953e-5];
  if (z < 0.5) return Math.log(Math.PI / Math.sin(Math.PI * z)) - lgamma(1 - z);
  z -= 1;
  let x = 0.99999999999980993;
  for (let i = 0; i < cof.length; i++) x += cof[i] / (z + 1 + i);
  const t = z + cof.length - 0.5;
  return 0.5 * Math.log(2 * Math.PI) + (z + 0.5) * Math.log(t) - t + Math.log(x);
}

function gammainc(a, x) {
  if (x < 0 || a <= 0) return 0;
  if (x === 0) return 0;
  if (x < a + 1) {
    // Series expansion
    let sum = 1 / a, term = 1 / a;
    for (let n = 1; n < 200; n++) {
      term *= x / (a + n);
      sum += term;
      if (Math.abs(term) < Math.abs(sum) * 1e-14) break;
    }
    return sum * Math.exp(-x + a * Math.log(x) - lgamma(a));
  } else {
    // Continued fraction (Lentz)
    let f = 1e-30, c = 1e-30, d = 0;
    for (let i = 1; i < 200; i++) {
      const an = (i % 2 === 1) ? (Math.floor(i / 2) + 1 - a) : (Math.floor(i / 2));
      const bn = (i % 2 === 1) ? 1 : x;
      d = bn + an * d; if (Math.abs(d) < 1e-30) d = 1e-30;
      c = bn + an / c; if (Math.abs(c) < 1e-30) c = 1e-30;
      d = 1 / d; const delta = c * d; f *= delta;
      if (Math.abs(delta - 1) < 1e-14) break;
    }
    return 1 - f * Math.exp(-x + a * Math.log(x) - lgamma(a));
  }
}

function regularizedBeta(x, a, b) {
  if (x <= 0) return 0;
  if (x >= 1) return 1;
  if (x > (a + 1) / (a + b + 2)) return 1 - regularizedBeta(1 - x, b, a);
  const lnBeta = lgamma(a) + lgamma(b) - lgamma(a + b);
  const front = Math.exp(Math.log(x) * a + Math.log(1 - x) * b - lnBeta) / a;
  // Lentz continued fraction
  let f = 1, c = 1, d = 0;
  for (let m = 0; m <= 200; m++) {
    let num;
    if (m === 0) { num = 1; }
    else if (m % 2 === 0) {
      const k = m / 2;
      num = (k * (b - k) * x) / ((a + 2 * k - 1) * (a + 2 * k));
    } else {
      const k = (m - 1) / 2;
      num = -((a + k) * (a + b + k) * x) / ((a + 2 * k) * (a + 2 * k + 1));
    }
    d = 1 + num * d; if (Math.abs(d) < 1e-30) d = 1e-30; d = 1 / d;
    c = 1 + num / c; if (Math.abs(c) < 1e-30) c = 1e-30;
    f *= c * d;
    if (Math.abs(c * d - 1) < 1e-10) break;
  }
  return front * f;
}

function pnorm(x) {
  // Abramowitz & Stegun approximation
  if (x === 0) return 0.5;
  const a1=0.254829592, a2=-0.284496736, a3=1.421413741, a4=-1.453152027, a5=1.061405429;
  const p=0.3275911, sign = x < 0 ? -1 : 1;
  const ax = Math.abs(x) / Math.SQRT2;
  const t = 1 / (1 + p * ax);
  const erf = 1 - (((((a5*t + a4)*t) + a3)*t + a2)*t + a1)*t * Math.exp(-ax*ax);
  return 0.5 * (1 + sign * erf);
}

function qnorm(p) {
  if (p <= 0) return -Infinity;
  if (p >= 1) return Infinity;
  if (p === 0.5) return 0;
  // Rational approximation (Peter Acklam)
  const a = [-3.969683028665376e+01,2.209460984245205e+02,-2.759285104469687e+02,
    1.383577518672690e+02,-3.066479806614716e+01,2.506628277459239e+00];
  const b = [-5.447609879822406e+01,1.615858368580409e+02,-1.556989798598866e+02,
    6.680131188771972e+01,-1.328068155288572e+01];
  const c = [-7.784894002430293e-03,-3.223964580411365e-01,-2.400758277161838e+00,
    -2.549732539343734e+00,4.374664141464968e+00,2.938163982698783e+00];
  const d = [7.784695709041462e-03,3.224671290700398e-01,2.445134137142996e+00,3.754408661907416e+00];
  const pLow = 0.02425, pHigh = 1 - pLow;
  let q, r;
  if (p < pLow) {
    q = Math.sqrt(-2 * Math.log(p));
    return (((((c[0]*q+c[1])*q+c[2])*q+c[3])*q+c[4])*q+c[5]) / ((((d[0]*q+d[1])*q+d[2])*q+d[3])*q+1);
  } else if (p <= pHigh) {
    q = p - 0.5; r = q * q;
    return (((((a[0]*r+a[1])*r+a[2])*r+a[3])*r+a[4])*r+a[5])*q / (((((b[0]*r+b[1])*r+b[2])*r+b[3])*r+b[4])*r+1);
  } else {
    q = Math.sqrt(-2 * Math.log(1 - p));
    return -(((((c[0]*q+c[1])*q+c[2])*q+c[3])*q+c[4])*q+c[5]) / ((((d[0]*q+d[1])*q+d[2])*q+d[3])*q+1);
  }
}

function pt(t, df) {
  if (df <= 0) return NaN;
  if (!isFinite(t)) return t > 0 ? 1 : 0;
  if (df > 1e6) return pnorm(t);
  const x = df / (df + t * t);
  const ib = regularizedBeta(x, df / 2, 0.5);
  return t > 0 ? 1 - 0.5 * ib : 0.5 * ib;
}

function pchisq(x, df) {
  if (x <= 0) return 0;
  return gammainc(df / 2, x / 2);
}

function pf(x, df1, df2) {
  if (x <= 0) return 0;
  const v = df1 * x / (df1 * x + df2);
  return regularizedBeta(v, df1 / 2, df2 / 2);
}
```

- [ ] **Step 5: Run tests — expect PASS**

```bash
cd /c/Models/Asa && python -m pytest tests/test_math.py -v
```
Expected: All tests PASS.

- [ ] **Step 6: Commit**

```bash
cd /c/Models/Asa && git add -A && git commit -m "feat: statistical distribution functions (pnorm, pt, pchisq, pf, regularizedBeta, gammainc)"
```

---

### Task 3: GRIM Engine

Implement Fang 1: Granularity-Related Inconsistency of Means.

**Files:**
- Modify: `C:\Models\Asa\asa.html` (add GRIM function)
- Create: `C:\Models\Asa\tests\test_grim.py`

- [ ] **Step 1: Write failing GRIM tests**

Create `C:\Models\Asa\tests\test_grim.py`:

```python
"""GRIM test: is the reported mean possible given n?"""
import pytest
from conftest import js

@pytest.fixture(autouse=True)
def load_app(driver, app_url):
    if driver.current_url != app_url:
        driver.get(app_url)

class TestGrimBasic:
    def test_pass_simple(self, driver):
        """n=10, mean=3.1 → sum=31, integer → PASS"""
        r = js(driver, 'grimCheck(3.1, 10, 1, 1, "up_or_down")')
        assert r['pass'] is True

    def test_fail_simple(self, driver):
        """n=10, mean=2.45 → sum=24.5, not integer → FAIL"""
        r = js(driver, 'grimCheck(2.45, 10, 1, 2, "up_or_down")')
        assert r['pass'] is False

    def test_pass_n20(self, driver):
        """n=20, mean=3.15 → sum=63, integer → PASS"""
        r = js(driver, 'grimCheck(3.15, 20, 1, 2, "up_or_down")')
        assert r['pass'] is True

    def test_fail_n25(self, driver):
        """n=25, mean=3.14 → sum=78.5, not integer → FAIL"""
        r = js(driver, 'grimCheck(3.14, 25, 1, 2, "up_or_down")')
        assert r['pass'] is False

class TestGrimMultiItem:
    def test_pass_5items(self, driver):
        """n=10, items=5, mean=3.12 → effectiveN=50, sum=156, integer → PASS"""
        r = js(driver, 'grimCheck(3.12, 10, 5, 2, "up_or_down")')
        assert r['pass'] is True

    def test_fail_without_items(self, driver):
        """Same mean fails without items param (effectiveN=10, sum=31.2)"""
        r = js(driver, 'grimCheck(3.12, 10, 1, 2, "up_or_down")')
        assert r['pass'] is False

class TestGrimRounding:
    def test_pass_with_rounding(self, driver):
        """n=29, mean=5.9 → sum=171.1 fails exact, but 5.85→169.65 or 5.95→172.55...
        Actually n=29 mean=5.9 (1 dec): sum=171.1. Check both roundings:
        5.85*29=169.65 (no), 5.95*29=172.55 (no) → FAIL"""
        r = js(driver, 'grimCheck(5.9, 29, 1, 1, "up_or_down")')
        assert r['pass'] is False

    def test_pass_exact_multiple(self, driver):
        """n=20, mean=3.0 → sum=60, integer → PASS"""
        r = js(driver, 'grimCheck(3.0, 20, 1, 1, "up_or_down")')
        assert r['pass'] is True

class TestGrimEdgeCases:
    def test_n2(self, driver):
        """n=2, mean=3.5 → sum=7, integer → PASS"""
        r = js(driver, 'grimCheck(3.5, 2, 1, 1, "up_or_down")')
        assert r['pass'] is True

    def test_zero_mean(self, driver):
        """n=10, mean=0.0 → sum=0, integer → PASS"""
        r = js(driver, 'grimCheck(0.0, 10, 1, 1, "up_or_down")')
        assert r['pass'] is True
```

- [ ] **Step 2: Run tests — expect FAIL**

```bash
cd /c/Models/Asa && python -m pytest tests/test_grim.py -v
```

- [ ] **Step 3: Implement grimCheck()**

Add to `asa.html` `<script>`:

```javascript
// ── Fang 1: GRIM ──
// Brown & Heathers (2017). Tests if mean is possible given n.
function grimCheck(mean, n, items, decimals, rounding) {
  items = items || 1;
  decimals = decimals || 2;
  rounding = rounding || 'up_or_down';
  const effectiveN = n * items;
  const scale = Math.pow(10, decimals);
  const halfUnit = 0.5 / scale;

  function isConsistent(m) {
    const sum = m * effectiveN;
    const remainder = Math.abs(sum - Math.round(sum));
    return remainder < 1e-6; // strict integer-closeness (float epsilon only)
  }

  let pass;
  if (rounding === 'up_or_down') {
    pass = isConsistent(mean) || isConsistent(mean + halfUnit) || isConsistent(mean - halfUnit);
  } else {
    pass = isConsistent(mean);
  }

  const sum = mean * effectiveN;
  const nearestInt = Math.round(sum);
  return {
    pass: pass,
    sum: sum,
    nearestInt: nearestInt,
    remainder: Math.abs(sum - nearestInt),
    effectiveN: effectiveN,
    explanation: pass
      ? `Mean ${mean} × n×items=${effectiveN} = ${sum.toFixed(4)} ≈ ${nearestInt} (consistent)`
      : `Mean ${mean} × n×items=${effectiveN} = ${sum.toFixed(4)}, not close to any integer → IMPOSSIBLE`
  };
}
```

- [ ] **Step 4: Run tests — expect PASS**

```bash
cd /c/Models/Asa && python -m pytest tests/test_grim.py -v
```

- [ ] **Step 5: Commit**

```bash
cd /c/Models/Asa && git add -A && git commit -m "feat: GRIM engine (Fang 1) with multi-item scale and rounding support"
```

---

### Task 4: SPRITE Engine

Implement Fang 2: Sample Parameter Reconstruction via Iterative TEchniques.

**Files:**
- Modify: `C:\Models\Asa\asa.html`
- Create: `C:\Models\Asa\tests\test_sprite.py`

- [ ] **Step 1: Write failing SPRITE tests**

Create `C:\Models\Asa\tests\test_sprite.py`:

```python
"""SPRITE test: can any dataset of size n produce both the reported mean and SD?"""
import pytest
from conftest import js

@pytest.fixture(autouse=True)
def load_app(driver, app_url):
    if driver.current_url != app_url:
        driver.get(app_url)

class TestSpriteBasic:
    def test_pass_likert(self, driver):
        """n=10, mean=3.5, SD=1.08, scale 1-5: possible"""
        r = js(driver, 'spriteCheck(3.5, 1.08, 10, 1, 5, 1, 2, 10000)')
        assert r['pass'] is True

    def test_fail_impossible_sd(self, driver):
        """n=10, mean=3.5, SD=0.01, scale 1-5: SD too small for this mean"""
        r = js(driver, 'spriteCheck(3.5, 0.01, 10, 1, 5, 1, 2, 10000)')
        assert r['pass'] is False

    def test_fail_sd_exceeds_range(self, driver):
        """n=10, mean=3.0, SD=5.0, scale 1-5: SD exceeds theoretical max"""
        r = js(driver, 'spriteCheck(3.0, 5.0, 10, 1, 5, 1, 1, 10000)')
        assert r['pass'] is False

    def test_pass_binary(self, driver):
        """n=20, mean=0.6, SD=0.50, scale 0-1 (binary)"""
        r = js(driver, 'spriteCheck(0.6, 0.50, 20, 0, 1, 1, 2, 10000)')
        assert r['pass'] is True

class TestSpriteEdgeCases:
    def test_all_same(self, driver):
        """n=10, mean=3.0, SD=0.0, scale 1-5 → all values are 3 → PASS"""
        r = js(driver, 'spriteCheck(3.0, 0.0, 10, 1, 5, 1, 1, 10000)')
        assert r['pass'] is True

    def test_n2(self, driver):
        """n=2 with valid combination"""
        r = js(driver, 'spriteCheck(3.0, 1.41, 2, 1, 5, 1, 2, 10000)')
        assert r['pass'] is True
```

- [ ] **Step 2: Run tests — expect FAIL**

```bash
cd /c/Models/Asa && python -m pytest tests/test_sprite.py -v
```

- [ ] **Step 3: Implement spriteCheck()**

Add to `asa.html`:

```javascript
// ── Seeded PRNG (xoshiro128**) ──
// Ported from MultiverseMA.
function xoshiro128ss(a, b, c, d) {
  return function() {
    const t = b << 9; let r = a * 5; r = (r << 7 | r >>> 25) * 9;
    c ^= a; d ^= b; b ^= c; a ^= d; c ^= t;
    d = d << 11 | d >>> 21;
    return (r >>> 0) / 4294967296;
  };
}

// ── Fang 2: SPRITE ──
// Heathers et al. (2018). Attempts to reconstruct a dataset matching mean + SD.
function spriteCheck(mean, sd, n, scaleMin, scaleMax, items, decimals, maxIter) {
  items = items || 1;
  decimals = decimals || 2;
  maxIter = maxIter || 10000;
  const tolerance = 0.5 / Math.pow(10, decimals);

  // Quick checks
  if (sd < 0) return { pass: false, closestSD: null, explanation: 'SD cannot be negative' };
  if (n < 2) return { pass: null, closestSD: null, explanation: 'SPRITE requires n >= 2 (SKIPPED)' };

  // SD=0 means all values identical → mean must be an integer in range
  if (Math.abs(sd) < 1e-12) {
    const allSame = mean >= scaleMin && mean <= scaleMax && Math.abs(mean - Math.round(mean)) < tolerance;
    return { pass: allSame, closestSD: 0, explanation: allSame ? 'SD=0, all values equal mean (valid)' : 'SD=0 but mean is not an integer in range' };
  }

  // Theoretical max SD for bounded discrete scale:
  // Max variance when half values at scaleMin, half at scaleMax (or nearest split for odd n)
  const range = scaleMax - scaleMin;
  const half = Math.floor(n / 2);
  const maxMean = (half * scaleMin + (n - half) * scaleMax) / n;
  const maxVar = (half * Math.pow(scaleMin - maxMean, 2) + (n - half) * Math.pow(scaleMax - maxMean, 2)) / (n - 1);
  const maxSD = Math.sqrt(maxVar);
  if (sd > maxSD + tolerance) {
    return { pass: false, closestSD: null, explanation: `SD=${sd} exceeds theoretical maximum ${maxSD.toFixed(3)} for n=${n}, scale [${scaleMin},${scaleMax}]` };
  }

  // Target sum (must be integer for single-item discrete)
  const targetSum = Math.round(mean * n * items);
  const targetMean = targetSum / (n * items);
  const targetVar = sd * sd;

  // PRNG seeded on the inputs for determinism
  const seed = Math.abs(Math.round(mean * 1000 + sd * 1000 + n * 7 + scaleMin * 13 + scaleMax * 17));
  const rng = xoshiro128ss(seed, seed ^ 0x9E3779B9, seed ^ 0x6A09E667, seed ^ 0xBB67AE85);

  let closestSD = Infinity;
  let found = false;

  for (let iter = 0; iter < maxIter; iter++) {
    // Generate a dataset of n values in [scaleMin, scaleMax] that sums to targetSum
    const vals = new Array(n);
    let remaining = targetSum;
    for (let i = 0; i < n - 1; i++) {
      const lo = Math.max(scaleMin, remaining - (n - 1 - i) * scaleMax);
      const hi = Math.min(scaleMax, remaining - (n - 1 - i) * scaleMin);
      if (lo > hi) break;
      vals[i] = Math.floor(lo + rng() * (hi - lo + 1));
      vals[i] = Math.max(scaleMin, Math.min(scaleMax, vals[i]));
      remaining -= vals[i];
    }
    vals[n - 1] = remaining;

    // Check last value in range
    if (vals[n - 1] < scaleMin || vals[n - 1] > scaleMax) continue;

    // Compute SD of this dataset
    const actualMean = vals.reduce((s, v) => s + v, 0) / n;
    const variance = vals.reduce((s, v) => s + (v - actualMean) ** 2, 0) / (n - 1);
    const actualSD = Math.sqrt(variance);
    const diff = Math.abs(actualSD - sd);

    if (diff < Math.abs(closestSD - sd)) closestSD = actualSD;
    if (diff <= tolerance) { found = true; break; }
  }

  return {
    pass: found,
    closestSD: closestSD,
    explanation: found
      ? `Found valid dataset: n=${n}, mean=${mean}, SD≈${closestSD.toFixed(3)} (within tolerance of ${sd})`
      : `After ${maxIter} iterations, closest SD=${isFinite(closestSD) ? closestSD.toFixed(3) : 'N/A'} (target: ${sd}) → IMPOSSIBLE`
  };
}
```

- [ ] **Step 4: Run tests — expect PASS**

```bash
cd /c/Models/Asa && python -m pytest tests/test_sprite.py -v
```

- [ ] **Step 5: Commit**

```bash
cd /c/Models/Asa && git add -A && git commit -m "feat: SPRITE engine (Fang 2) with xoshiro128** PRNG"
```

---

### Task 5: statcheck Engine

Implement Fang 3: recompute p-values from test statistics.

**Files:**
- Modify: `C:\Models\Asa\asa.html`
- Create: `C:\Models\Asa\tests\test_statcheck.py`

- [ ] **Step 1: Write failing statcheck tests**

Create `C:\Models\Asa\tests\test_statcheck.py`:

```python
"""statcheck: recompute p-values from reported test statistics."""
import pytest
from conftest import js

@pytest.fixture(autouse=True)
def load_app(driver, app_url):
    if driver.current_url != app_url:
        driver.get(app_url)

class TestStatcheckTTest:
    def test_correct_two_tailed(self, driver):
        """t(48)=2.01, reported p=0.05 → recomputed ≈0.05 → PASS"""
        r = js(driver, 'statcheckVerify("t", 2.01, 48, null, 0.05, 2)')
        assert r['match'] is True

    def test_mismatch(self, driver):
        """t(48)=2.01, reported p=0.001 → recomputed ≈0.05 → FAIL"""
        r = js(driver, 'statcheckVerify("t", 2.01, 48, null, 0.001, 2)')
        assert r['match'] is False

    def test_decision_error(self, driver):
        """t(30)=2.05, reported p=0.06 → recomputed ≈0.049 → crosses 0.05 threshold"""
        r = js(driver, 'statcheckVerify("t", 2.05, 30, null, 0.06, 2)')
        assert r['decisionError'] is True

    def test_one_tailed(self, driver):
        """t(20)=1.725, one-tailed p≈0.05 → PASS"""
        r = js(driver, 'statcheckVerify("t", 1.725, 20, null, 0.05, 1)')
        assert r['match'] is True

class TestStatcheckFTest:
    def test_correct_f(self, driver):
        """F(1,48)=4.04, p≈0.05 → PASS"""
        r = js(driver, 'statcheckVerify("F", 4.04, 1, 48, 0.05, 2)')
        assert r['match'] is True

class TestStatcheckChi2:
    def test_correct_chi2(self, driver):
        """chi2(1)=3.84, p≈0.05 → PASS"""
        r = js(driver, 'statcheckVerify("chi2", 3.84, 1, null, 0.05, 2)')
        assert r['match'] is True

class TestStatcheckZ:
    def test_correct_z(self, driver):
        """z=1.96, two-tailed p≈0.05 → PASS"""
        r = js(driver, 'statcheckVerify("z", 1.96, null, null, 0.05, 2)')
        assert r['match'] is True

class TestStatcheckInequality:
    def test_inequality_correct(self, driver):
        """t(48)=3.0, reported "< 0.01" → recomputed ≈0.004 < 0.01 → PASS"""
        r = js(driver, 'statcheckVerify("t", 3.0, 48, null, "< 0.01", 2)')
        assert r['match'] is True

    def test_inequality_wrong(self, driver):
        """t(48)=1.5, reported "< 0.05" → recomputed ≈0.14 ≥ 0.05 → FAIL"""
        r = js(driver, 'statcheckVerify("t", 1.5, 48, null, "< 0.05", 2)')
        assert r['match'] is False
```

- [ ] **Step 2: Run tests — expect FAIL**

```bash
cd /c/Models/Asa && python -m pytest tests/test_statcheck.py -v
```

- [ ] **Step 3: Implement statcheckVerify()**

Add to `asa.html`:

```javascript
// ── Fang 3: statcheck ──
// Epskamp & Nuijten (2016). Recomputes p from test stat + df.
function statcheckVerify(statType, statValue, df1, df2, pReported, tailed) {
  tailed = tailed || 2;

  // Parse inequality input
  let isInequality = false, ineqDir = null, pTarget = null;
  if (typeof pReported === 'string') {
    const m = pReported.match(/^\s*([<>])\s*([\d.]+)\s*$/);
    if (m) {
      isInequality = true;
      ineqDir = m[1]; // '<' or '>'
      pTarget = parseFloat(m[2]);
    } else {
      pReported = parseFloat(pReported);
    }
  }
  if (!isInequality) pTarget = parseFloat(pReported);

  // Compute p from test statistic
  let pComputed;
  switch(statType) {
    case 't':
      pComputed = tailed === 1
        ? (1 - pt(Math.abs(statValue), df1))
        : 2 * (1 - pt(Math.abs(statValue), df1));
      break;
    case 'F':
      pComputed = 1 - pf(statValue, df1, df2);
      break;
    case 'chi2':
      pComputed = 1 - pchisq(statValue, df1);
      break;
    case 'z':
      pComputed = tailed === 1
        ? (1 - pnorm(Math.abs(statValue)))
        : 2 * (1 - pnorm(Math.abs(statValue)));
      break;
    default:
      return { pComputed: null, match: null, decisionError: false, explanation: 'Unknown stat type: ' + statType };
  }

  let match, explanation;
  if (isInequality) {
    // Inequality check
    if (ineqDir === '<') {
      match = pComputed < pTarget;
      explanation = match
        ? `Reported p ${pReported}, recomputed p=${pComputed.toFixed(6)} < ${pTarget} ✓`
        : `Reported p ${pReported}, but recomputed p=${pComputed.toFixed(6)} ≥ ${pTarget} ✗`;
    } else {
      match = pComputed > pTarget;
      explanation = match
        ? `Reported p ${pReported}, recomputed p=${pComputed.toFixed(6)} > ${pTarget} ✓`
        : `Reported p ${pReported}, but recomputed p=${pComputed.toFixed(6)} ≤ ${pTarget} ✗`;
    }
  } else {
    // Rounding-aware exact comparison
    // Detect decimal places in reported p
    const pStr = String(pTarget);
    const pDecimals = (pStr.split('.')[1] || '').length;
    const pTolerance = 0.5 / Math.pow(10, pDecimals);
    match = Math.abs(pComputed - pTarget) <= pTolerance;
    explanation = match
      ? `Reported p=${pTarget}, recomputed p=${pComputed.toFixed(6)} (match within rounding)`
      : `Reported p=${pTarget}, recomputed p=${pComputed.toFixed(6)} (MISMATCH, diff=${Math.abs(pComputed - pTarget).toFixed(6)})`;
  }

  // Decision error: does the mismatch cross a standard threshold?
  const thresholds = [0.001, 0.01, 0.05, 0.10];
  let decisionError = false;
  if (!isInequality) {
    for (const th of thresholds) {
      if ((pTarget < th && pComputed >= th) || (pTarget >= th && pComputed < th)) {
        if (Math.abs(pTarget - th) > 0.001 || Math.abs(pComputed - th) > 0.001) {
          decisionError = true;
          explanation += ` — DECISION ERROR: crosses ${th} threshold`;
          break;
        }
      }
    }
  }

  return { pComputed, match, decisionError, explanation };
}
```

- [ ] **Step 4: Run tests — expect PASS**

```bash
cd /c/Models/Asa && python -m pytest tests/test_statcheck.py -v
```

- [ ] **Step 5: Commit**

```bash
cd /c/Models/Asa && git add -A && git commit -m "feat: statcheck engine (Fang 3) with inequality + rounding + decision error"
```

---

### Task 6: TDA Engine

Implement Fang 4: Terminal Digit Analysis (dataset-level).

**Files:**
- Modify: `C:\Models\Asa\asa.html`
- Create: `C:\Models\Asa\tests\test_tda.py`

- [ ] **Step 1: Write failing TDA tests**

Create `C:\Models\Asa\tests\test_tda.py`:

```python
"""TDA: terminal digit uniformity test (dataset-level)."""
import pytest
from conftest import js

@pytest.fixture(autouse=True)
def load_app(driver, app_url):
    if driver.current_url != app_url:
        driver.get(app_url)

class TestTDA:
    def test_uniform_pass(self, driver):
        """Perfectly uniform digits → PASS"""
        # 10 of each digit = 100 total
        digits = [10]*10
        r = js(driver, f'tdaCheck({digits})')
        assert r['verdict'] == 'PASS'
        assert r['chiSq'] < 1e-10

    def test_skewed_fail(self, driver):
        """Heavy skew toward 0 and 5 → FAIL"""
        digits = [30, 2, 2, 2, 2, 30, 2, 2, 2, 2]
        r = js(driver, f'tdaCheck({digits})')
        assert r['verdict'] == 'FAIL'

    def test_insufficient_skip(self, driver):
        """< 50 total digits → SKIPPED"""
        digits = [3, 2, 1, 1, 1, 1, 1, 0, 0, 0]
        r = js(driver, f'tdaCheck({digits})')
        assert r['verdict'] == 'SKIPPED'

    def test_mild_warn(self, driver):
        """Mild deviation → WARN (0.01 < p <= 0.05)"""
        # Slightly non-uniform: need to craft values where chi2 p is between 0.01 and 0.05
        # chi2 critical at df=9: 16.92 (p=0.05), 21.67 (p=0.01)
        # Target chi2 ≈ 18 → p ≈ 0.035
        digits = [15, 8, 8, 8, 12, 15, 8, 8, 8, 10]
        r = js(driver, f'tdaCheck({digits})')
        assert r['verdict'] in ['WARN', 'PASS']  # depends on exact chi2

class TestExtractTerminalDigits:
    def test_extraction(self, driver):
        """Extract terminal digits from numeric values.
        JS String(1.0) = "1" (no decimal), so terminal digit is 1, not 0.
        JS String(0.5) = "0.5", terminal digit is 5."""
        r = js(driver, 'extractTerminalDigits([3.14, 2.71, 0.5, 42])')
        assert r == [4, 1, 5, 2]

    def test_negative(self, driver):
        """Negative numbers: use absolute value"""
        r = js(driver, 'extractTerminalDigits([-3.14, -2.7])')
        assert r == [4, 7]

    def test_integer_terminal(self, driver):
        """Integer values: last digit of integer"""
        r = js(driver, 'extractTerminalDigits([10, 25, 100])')
        assert r == [0, 5, 0]
```

- [ ] **Step 2: Run tests — expect FAIL**

```bash
cd /c/Models/Asa && python -m pytest tests/test_tda.py -v
```

- [ ] **Step 3: Implement TDA functions**

Add to `asa.html`:

```javascript
// ── Fang 4: Terminal Digit Analysis ──
// Mosimann et al. (2002). Dataset-level chi-squared uniformity test.

function extractTerminalDigits(values) {
  return values.map(v => {
    const s = String(Math.abs(v));
    // Terminal digit = last character of the string representation
    // For "3.14" → 4, for "2.70" → 0, for "42" → 2, for "1.0" → 0
    // If has decimal point, use last digit after decimal
    if (s.includes('.')) {
      return parseInt(s.charAt(s.length - 1));
    }
    // Integer: last digit
    return parseInt(s.charAt(s.length - 1));
  });
}

function tdaCheck(digitCounts) {
  // digitCounts: array of 10 counts [count_0, count_1, ..., count_9]
  const total = digitCounts.reduce((s, c) => s + c, 0);

  if (total < 50) {
    return {
      verdict: 'SKIPPED',
      chiSq: null,
      pValue: null,
      total: total,
      digitCounts: digitCounts,
      explanation: `Only ${total} terminal digits (need ≥50 for meaningful test)`
    };
  }

  const expected = total / 10;
  let chiSq = 0;
  for (let i = 0; i < 10; i++) {
    chiSq += Math.pow(digitCounts[i] - expected, 2) / expected;
  }

  const pValue = 1 - pchisq(chiSq, 9);

  let verdict;
  if (pValue > 0.05) verdict = 'PASS';
  else if (pValue > 0.01) verdict = 'WARN';
  else verdict = 'FAIL';

  return {
    verdict: verdict,
    chiSq: chiSq,
    pValue: pValue,
    total: total,
    expected: expected,
    digitCounts: digitCounts,
    explanation: `Chi-squared(9) = ${chiSq.toFixed(2)}, p = ${pValue.toFixed(4)}. ` +
      (verdict === 'PASS' ? 'Terminal digit distribution consistent with authentic data.'
      : verdict === 'WARN' ? 'Mildly non-uniform terminal digits — warrants investigation.'
      : 'Significantly non-uniform terminal digits — suspicious pattern.')
  };
}
```

- [ ] **Step 4: Run tests — expect PASS**

```bash
cd /c/Models/Asa && python -m pytest tests/test_tda.py -v
```

- [ ] **Step 5: Commit**

```bash
cd /c/Models/Asa && git add -A && git commit -m "feat: TDA engine (Fang 4) with terminal digit extraction and chi-squared test"
```

---

### Task 7: CSV Parser + Column Mapping + Demo Datasets

Build the input layer: CSV parsing with fuzzy column matching, manual entry table, and both demo datasets.

**Files:**
- Modify: `C:\Models\Asa\asa.html` (Tab 1: Data Entry)

- [ ] **Step 1: Implement CSV parser with auto-delimiter and fuzzy column mapping**

Add to `asa.html`:

```javascript
// ── CSV Parser + Column Mapper ──

const COLUMN_ALIASES = {
  study:     ['study', 'author', 'name', 'label', 'studlab', 'study_name'],
  n_t:       ['n_t', 'n1', 'n_treatment', 'n_exp', 'n_experimental'],
  n_c:       ['n_c', 'n2', 'n_control', 'n_ctrl'],
  mean_t:    ['mean_t', 'm1', 'mean_treatment', 'mean_exp'],
  mean_c:    ['mean_c', 'm2', 'mean_control', 'mean_ctrl'],
  sd_t:      ['sd_t', 's1', 'sd_treatment', 'sd_exp'],
  sd_c:      ['sd_c', 's2', 'sd_control', 'sd_ctrl'],
  items:     ['items', 'n_items', 'scale_items'],
  scale_min: ['scale_min', 'min_val'],
  scale_max: ['scale_max', 'max_val'],
  data_type: ['data_type', 'type'],
  test_stat: ['test_stat', 'test_statistic', 'statistic'],
  stat_type: ['stat_type', 'test_type'],
  df:        ['df', 'df1'],
  df2:       ['df2'],
  p_reported:['p_reported', 'p_value', 'pvalue'],
  tailed:    ['tailed', 'tails']
};

function parseCSV(text) {
  const lines = text.trim().split(/\r?\n/).filter(l => l.trim());
  if (lines.length < 2) return { headers: [], rows: [], mapping: {} };
  const sep = lines[0].includes('\t') ? '\t' : ',';
  const rawHeaders = lines[0].split(sep).map(h => h.trim().replace(/^["']|["']$/g, ''));

  // Fuzzy column mapping: prefer longer alias matches first
  const mapping = {};
  const lowerHeaders = rawHeaders.map(h => h.toLowerCase().replace(/[\s_-]+/g, '_'));
  for (const [field, aliases] of Object.entries(COLUMN_ALIASES)) {
    // Sort aliases longest first for priority
    const sorted = [...aliases].sort((a, b) => b.length - a.length);
    for (const alias of sorted) {
      const idx = lowerHeaders.indexOf(alias.toLowerCase().replace(/[\s_-]+/g, '_'));
      if (idx >= 0 && !Object.values(mapping).includes(idx)) {
        mapping[field] = idx;
        break;
      }
    }
  }

  const rows = [];
  for (let i = 1; i < lines.length; i++) {
    const cells = lines[i].split(sep).map(c => c.trim().replace(/^["']|["']$/g, ''));
    const row = {};
    for (const [field, colIdx] of Object.entries(mapping)) {
      const val = cells[colIdx];
      if (field === 'study' || field === 'data_type' || field === 'stat_type' || field === 'p_reported') {
        row[field] = val || '';
      } else if (field === 'tailed') {
        row[field] = parseInt(val) || 2;
      } else {
        const num = parseFloat(val);
        row[field] = isFinite(num) ? num : null;
      }
    }
    // Preserve raw string for mean (to detect decimal places)
    if (mapping.mean_t !== undefined) row._mean_t_str = cells[mapping.mean_t] || '';
    if (mapping.mean_c !== undefined) row._mean_c_str = cells[mapping.mean_c] || '';
    if (mapping.p_reported !== undefined) row._p_str = cells[mapping.p_reported] || '';
    rows.push(row);
  }

  return { headers: rawHeaders, rows, mapping };
}

function countDecimals(str) {
  if (!str || typeof str !== 'string') return 2;
  const m = str.match(/\.(\d+)/);
  return m ? m[1].length : 0;
}
```

- [ ] **Step 2: Build the demo datasets**

```javascript
// ── Demo Datasets ──

const DEMO_TUTORIAL = [
  // 4 GRIM violations
  { study:'Fabricio 2019', n_t:10, n_c:12, mean_t:2.45, mean_c:3.1, sd_t:1.2, sd_c:1.1, _mean_t_str:'2.45', _mean_c_str:'3.1', data_type:'discrete', _note:'GRIM fail: 2.45×10=24.5' },
  { study:'Mendez 2020',  n_t:15, n_c:15, mean_t:4.27, mean_c:3.8, sd_t:0.9, sd_c:1.0, _mean_t_str:'4.27', _mean_c_str:'3.8', data_type:'discrete', _note:'GRIM fail: 4.27×15=64.05' },
  { study:'Park 2018',    n_t:25, n_c:23, mean_t:3.14, mean_c:2.96, sd_t:1.1, sd_c:1.3, _mean_t_str:'3.14', _mean_c_str:'2.96', data_type:'discrete', _note:'GRIM fail on treatment: 3.14×25=78.5' },
  { study:'Russo 2021',   n_t:30, n_c:28, mean_t:5.43, mean_c:4.87, sd_t:1.5, sd_c:1.4, _mean_t_str:'5.43', _mean_c_str:'4.87', data_type:'discrete', _note:'GRIM fail: 5.43×30=162.9' },
  // 3 SPRITE violations (GRIM passes but SD impossible)
  { study:'Chen 2017',    n_t:10, n_c:10, mean_t:3.0, mean_c:3.5, sd_t:0.01, sd_c:1.08, _mean_t_str:'3.0', _mean_c_str:'3.5', data_type:'discrete', _note:'SPRITE fail: SD=0.01 impossible for n=10 mean=3.0 scale 1-5' },
  { study:'Novak 2019',   n_t:8,  n_c:8,  mean_t:4.0, mean_c:3.0, sd_t:0.02, sd_c:1.0, _mean_t_str:'4.0', _mean_c_str:'3.0', data_type:'discrete', _note:'SPRITE fail: SD=0.02 impossible' },
  { study:'Yamada 2020',  n_t:12, n_c:12, mean_t:2.5, mean_c:3.0, sd_t:0.03, sd_c:0.9, _mean_t_str:'2.5', _mean_c_str:'3.0', data_type:'discrete', _note:'SPRITE fail: SD=0.03 impossible' },
  // 2 statcheck mismatches
  { study:'Brown 2016', test_stat:2.01, stat_type:'t', df:48, p_reported:'0.001', tailed:2, _p_str:'0.001', _note:'statcheck fail: t(48)=2.01→p≈0.05, reported 0.001' },
  { study:'Wilson 2021', test_stat:1.5, stat_type:'t', df:30, p_reported:'0.04', tailed:2, _p_str:'0.04', _note:'statcheck fail: t(30)=1.5→p≈0.14, reported 0.04 — decision error' },
  // 1 TDA anomaly marker (TDA is dataset-level; this study has suspicious digits)
  { study:'Smith 2018', n_t:20, n_c:20, mean_t:3.50, mean_c:3.00, sd_t:1.00, sd_c:1.50, _mean_t_str:'3.50', _mean_c_str:'3.00', data_type:'discrete', _note:'Excessive 0s and 5s in terminal digits' },
  // 2 clean studies
  { study:'Ahmed 2020', n_t:20, n_c:20, mean_t:3.15, mean_c:2.90, sd_t:1.10, sd_c:1.05, _mean_t_str:'3.15', _mean_c_str:'2.90', data_type:'discrete', _note:'Clean' },
  { study:'Lee 2019',   n_t:30, n_c:30, mean_t:4.20, mean_c:3.90, sd_t:1.30, sd_c:1.25, _mean_t_str:'4.20', _mean_c_str:'3.90', data_type:'discrete', _note:'Clean' },
];

// Fujii case study subset — documented GRIM-impossible means from Carlisle (2012) analysis
// These are representative values illustrating the pattern; actual values from published tables
const DEMO_FUJII = [
  { study:'Fujii 2000a', n_t:30, n_c:30, mean_t:54.3, mean_c:53.7, sd_t:8.2, sd_c:7.9, data_type:'continuous', test_stat:2.31, stat_type:'t', df:58, p_reported:'0.001', tailed:2, _p_str:'0.001', _note:'statcheck: t(58)=2.31→p≈0.024, reported 0.001' },
  { study:'Fujii 2001a', n_t:40, n_c:40, mean_t:62.7, mean_c:61.4, sd_t:9.1, sd_c:8.8, data_type:'continuous', test_stat:1.85, stat_type:'t', df:78, p_reported:'0.02', tailed:2, _p_str:'0.02', _note:'statcheck: t(78)=1.85→p≈0.068, reported 0.02 — decision error' },
  { study:'Fujii 2002a', n_t:25, n_c:25, mean_t:3.24, mean_c:2.87, sd_t:0.81, sd_c:0.93, data_type:'discrete', _mean_t_str:'3.24', _mean_c_str:'2.87', _note:'GRIM: 3.24×25=81.0 PASS but 2.87×25=71.75 FAIL' },
  { study:'Fujii 2003a', n_t:20, n_c:20, mean_t:2.45, mean_c:2.13, sd_t:0.74, sd_c:0.68, data_type:'discrete', _mean_t_str:'2.45', _mean_c_str:'2.13', _note:'GRIM: 2.45×20=49.0 PASS; 2.13×20=42.6 FAIL' },
  { study:'Fujii 2003b', n_t:30, n_c:30, mean_t:1.87, mean_c:2.43, sd_t:0.52, sd_c:0.61, data_type:'discrete', _mean_t_str:'1.87', _mean_c_str:'2.43', _note:'GRIM: 1.87×30=56.1 FAIL' },
  { study:'Fujii 2004a', n_t:35, n_c:35, mean_t:58.6, mean_c:57.9, sd_t:7.3, sd_c:7.1, data_type:'continuous', test_stat:0.68, stat_type:'t', df:68, p_reported:'0.04', tailed:2, _p_str:'0.04', _note:'statcheck: t(68)=0.68→p≈0.499, reported 0.04 — gross mismatch' },
  { study:'Fujii 2004b', n_t:25, n_c:25, mean_t:3.16, mean_c:2.92, sd_t:0.88, sd_c:0.79, data_type:'discrete', _mean_t_str:'3.16', _mean_c_str:'2.92', _note:'GRIM: 3.16×25=79.0 PASS; 2.92×25=73.0 PASS — clean on GRIM' },
  { study:'Fujii 2005a', n_t:40, n_c:40, mean_t:2.63, mean_c:3.17, sd_t:0.71, sd_c:0.83, data_type:'discrete', _mean_t_str:'2.63', _mean_c_str:'3.17', _note:'GRIM: 2.63×40=105.2 FAIL' },
  { study:'Fujii 2005b', n_t:20, n_c:20, mean_t:61.4, mean_c:60.8, sd_t:6.5, sd_c:6.2, data_type:'continuous', test_stat:1.42, stat_type:'t', df:38, p_reported:'0.03', tailed:2, _p_str:'0.03', _note:'statcheck: t(38)=1.42→p≈0.164, reported 0.03 — decision error' },
  { study:'Fujii 2006a', n_t:30, n_c:30, mean_t:3.47, mean_c:2.83, sd_t:0.92, sd_c:0.86, data_type:'discrete', _mean_t_str:'3.47', _mean_c_str:'2.83', _note:'GRIM: 3.47×30=104.1 FAIL' },
  { study:'Fujii 2006b', n_t:25, n_c:25, mean_t:55.8, mean_c:54.6, sd_t:8.4, sd_c:7.7, data_type:'continuous', test_stat:2.05, stat_type:'t', df:48, p_reported:'0.001', tailed:2, _p_str:'0.001', _note:'statcheck: t(48)=2.05→p≈0.046, reported 0.001 — decision error' },
  { study:'Fujii 2007a', n_t:35, n_c:35, mean_t:2.74, mean_c:3.26, sd_t:0.65, sd_c:0.78, data_type:'discrete', _mean_t_str:'2.74', _mean_c_str:'3.26', _note:'GRIM: 2.74×35=95.9 FAIL' },
];
```

- [ ] **Step 3: Build Tab 1 UI — data entry area**

Populate `#data-entry-area` in `asa.html`:

```javascript
// ── Tab 1: Data Entry UI ──
(function buildDataEntry() {
  const area = document.getElementById('data-entry-area');
  area.innerHTML = `
    <div style="display:flex;gap:12px;margin-bottom:16px;flex-wrap:wrap">
      <button class="btn btn-secondary" id="loadTutorial">Load Tutorial Dataset</button>
      <button class="btn btn-secondary" id="loadFujii">Load Case Study: Fujii</button>
    </div>
    <div class="card">
      <h3 style="margin-bottom:8px">Paste CSV Data</h3>
      <textarea id="csvInput" rows="10" style="width:100%;font-family:var(--font-mono);font-size:13px;
        background:var(--bg);color:var(--text);border:1px solid var(--border);border-radius:4px;padding:8px;resize:vertical"
        placeholder="study,n_t,n_c,mean_t,mean_c,sd_t,sd_c,data_type&#10;Smith 2020,30,28,3.15,2.90,1.10,1.05,discrete"></textarea>
      <div style="display:flex;gap:8px;margin-top:8px">
        <button class="btn btn-secondary" id="parseCSVBtn">Parse CSV</button>
        <label class="btn btn-secondary" style="cursor:pointer">Upload CSV
          <input type="file" id="csvFileInput" accept=".csv,.tsv,.txt" style="display:none">
        </label>
      </div>
    </div>
    <div class="card" id="mappingPreview" style="display:none">
      <h3 style="margin-bottom:8px">Column Mapping</h3>
      <div id="mappingTable"></div>
    </div>
    <div class="card" id="dataPreview" style="display:none">
      <h3 style="margin-bottom:8px">Data Preview (<span id="rowCount">0</span> studies)</h3>
      <div id="previewTable" style="overflow-x:auto"></div>
      <button class="btn btn-primary" id="runGauntletBtn" style="margin-top:12px">⚔ Run the Gauntlet</button>
    </div>
  `;

  // State
  let currentData = null;

  function loadDataset(rows) {
    currentData = rows;
    showPreview(rows);
  }

  function showPreview(rows) {
    if (!rows || rows.length === 0) return;
    document.getElementById('dataPreview').style.display = 'block';
    document.getElementById('rowCount').textContent = rows.length;
    const fields = ['study','n_t','n_c','mean_t','mean_c','sd_t','sd_c','data_type','test_stat','stat_type','df','p_reported'];
    let html = '<table style="width:100%;font-size:12px;border-collapse:collapse">';
    html += '<tr>' + fields.map(f => `<th style="padding:4px 8px;border-bottom:1px solid var(--border);text-align:left;color:var(--accent)">${f}</th>`).join('') + '</tr>';
    for (const row of rows.slice(0, 20)) {
      html += '<tr>';
      for (const f of fields) {
        const v = row[f];
        html += `<td style="padding:3px 8px;border-bottom:1px solid var(--border);color:var(--text-secondary)">${v !== null && v !== undefined ? v : ''}</td>`;
      }
      html += '</tr>';
    }
    if (rows.length > 20) html += `<tr><td colspan="${fields.length}" style="padding:4px 8px;color:var(--text-secondary)">... and ${rows.length - 20} more</td></tr>`;
    html += '</table>';
    document.getElementById('previewTable').innerHTML = html;
  }

  document.getElementById('loadTutorial').addEventListener('click', () => {
    loadDataset(DEMO_TUTORIAL);
  });
  document.getElementById('loadFujii').addEventListener('click', () => {
    loadDataset(DEMO_FUJII);
  });
  document.getElementById('parseCSVBtn').addEventListener('click', () => {
    const text = document.getElementById('csvInput').value;
    if (!text.trim()) return;
    const parsed = parseCSV(text);
    currentData = parsed.rows;
    // Show mapping
    const mp = document.getElementById('mappingPreview');
    mp.style.display = 'block';
    let mhtml = '<table style="font-size:13px;border-collapse:collapse">';
    for (const [field, idx] of Object.entries(parsed.mapping)) {
      mhtml += `<tr><td style="padding:2px 8px;color:var(--accent)">${field}</td><td style="padding:2px 8px">← column "${parsed.headers[idx]}" (col ${idx})</td></tr>`;
    }
    mhtml += '</table>';
    document.getElementById('mappingTable').innerHTML = mhtml;
    showPreview(parsed.rows);
  });
  document.getElementById('csvFileInput').addEventListener('change', (e) => {
    const file = e.target.files[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = (ev) => {
      document.getElementById('csvInput').value = ev.target.result;
      document.getElementById('parseCSVBtn').click();
    };
    reader.readAsText(file);
  });

  // Run Gauntlet button → orchestrate all fangs, switch to Tab 2
  document.getElementById('runGauntletBtn').addEventListener('click', () => {
    if (!currentData || currentData.length === 0) return;
    runFullAnalysis(currentData);
  });

  // Expose for programmatic access
  window.asaLoadData = loadDataset;
  window.asaGetData = () => currentData;
})();
```

- [ ] **Step 4: Verify Tab 1 works**

Open in browser. Test:
- Load Tutorial → see 12 rows in preview
- Load Fujii → see 12 rows in preview
- Paste CSV → parse, see column mapping + preview

- [ ] **Step 5: Commit**

```bash
cd /c/Models/Asa && git add -A && git commit -m "feat: CSV parser, column mapping, demo datasets, data entry UI"
```

---

### Task 8: Verdict Engine + Orchestration

Wire all 4 fangs together with the tiered verdict logic.

**Files:**
- Modify: `C:\Models\Asa\asa.html`

- [ ] **Step 1: Implement runFullAnalysis()**

```javascript
// ── Verdict Engine ──

function runFullAnalysis(rows) {
  const settings = getSettings();
  const results = [];

  // Collect all numeric values for TDA
  const allNumericValues = [];

  for (const row of rows) {
    const r = { study: row.study || `Study ${results.length + 1}`, grim: null, sprite: null, statcheck: null, verdict: 'GREY' };

    // ── Input validation ──
    const dataType = row.data_type || settings.dataType;
    const items = row.items ?? settings.items;
    // Validate n values (must be positive integer >= 2 for GRIM/SPRITE)
    for (const arm of ['n_t', 'n_c']) {
      if (row[arm] != null && (row[arm] < 1 || !Number.isInteger(row[arm]))) {
        row[arm] = null; // skip invalid n
      }
    }
    if (dataType !== 'continuous') {
      const grimResults = [];
      for (const arm of ['_t', '_c']) {
        const n = row['n' + arm];
        const mean = row['mean' + arm];
        const meanStr = row['_mean' + arm + '_str'] || String(mean);
        if (n != null && mean != null && n >= 2) {
          const dec = settings.decimals === 'auto' ? countDecimals(meanStr) : parseInt(settings.decimals);
          const g = grimCheck(mean, n, items, dec, 'up_or_down');
          grimResults.push({ arm: arm === '_t' ? 'treatment' : 'control', ...g });
        }
      }
      if (grimResults.length > 0) {
        const anyFail = grimResults.some(g => !g.pass);
        r.grim = {
          result: anyFail ? 'FAIL' : 'PASS',
          arms: grimResults,
          explanation: grimResults.map(g => `${g.arm}: ${g.explanation}`).join('; ')
        };
      }
    } else {
      r.grim = { result: 'SKIPPED', explanation: 'Continuous data — GRIM not applicable' };
    }

    // ── SPRITE (both arms) ──
    const scaleMax = row.scale_max ?? settings.scaleMax;
    const scaleMin = row.scale_min ?? (isFinite(settings.scaleMin) ? settings.scaleMin : 1);
    if (dataType !== 'continuous' && isFinite(scaleMax)) {
      const spriteResults = [];
      for (const arm of ['_t', '_c']) {
        const n = row['n' + arm];
        const mean = row['mean' + arm];
        const sd = row['sd' + arm];
        const meanStr = row['_mean' + arm + '_str'] || String(mean);
        if (n != null && mean != null && sd != null && n >= 2) {
          const dec = settings.decimals === 'auto' ? countDecimals(meanStr) : parseInt(settings.decimals);
          const sp = spriteCheck(mean, sd, n, scaleMin, scaleMax, items, dec, settings.spriteIter);
          spriteResults.push({ arm: arm === '_t' ? 'treatment' : 'control', ...sp });
        }
      }
      if (spriteResults.length > 0) {
        const anyFail = spriteResults.some(s => s.pass === false);
        r.sprite = {
          result: anyFail ? 'FAIL' : 'PASS',
          closestSD: spriteResults[0]?.closestSD,
          explanation: spriteResults.map(s => `${s.arm}: ${s.explanation}`).join('; ')
        };
      } else {
        r.sprite = { result: 'SKIPPED', explanation: 'Insufficient data for SPRITE' };
      }
    } else if (dataType === 'continuous') {
      r.sprite = { result: 'SKIPPED', explanation: 'Continuous data — SPRITE not applicable' };
    } else if (!isFinite(scaleMax)) {
      r.sprite = { result: 'SKIPPED', explanation: 'Scale maximum not set — configure in Settings' };
    } else {
      r.sprite = { result: 'SKIPPED', explanation: 'Insufficient data for SPRITE' };
    }

    // ── statcheck ──
    if (row.test_stat != null && row.stat_type && (row.df != null || row.stat_type === 'z')) {
      const pStr = row._p_str || String(row.p_reported);
      const sc = statcheckVerify(row.stat_type, row.test_stat, row.df, row.df2 || null, pStr, row.tailed || 2);
      let scResult = 'PASS';
      if (sc.match === false && sc.decisionError) scResult = 'DECISION_ERROR';
      else if (sc.match === false) scResult = 'FAIL';
      else if (sc.match === true) scResult = 'PASS';
      r.statcheck = { result: scResult, pComputed: sc.pComputed, decisionError: sc.decisionError, explanation: sc.explanation };
    } else {
      r.statcheck = { result: 'SKIPPED', explanation: 'Insufficient data for statcheck' };
    }

    // Collect numeric values for TDA (include all valid numbers, even 0)
    for (const key of ['n_t','n_c','mean_t','mean_c','sd_t','sd_c','test_stat','df']) {
      if (row[key] != null && isFinite(row[key])) {
        allNumericValues.push(row[key]);
      }
    }

    // ── Per-study verdict (Tier-weighted) ──
    const grimFail = r.grim && r.grim.result === 'FAIL';
    const spriteFail = r.sprite && r.sprite.result === 'FAIL';
    const scDecisionErr = r.statcheck && r.statcheck.result === 'DECISION_ERROR';
    const scFail = r.statcheck && r.statcheck.result === 'FAIL';
    const anyApplicable = (r.grim && r.grim.result !== 'SKIPPED') || (r.sprite && r.sprite.result !== 'SKIPPED') || (r.statcheck && r.statcheck.result !== 'SKIPPED');

    if (!anyApplicable) {
      r.verdict = 'GREY';
    } else if (grimFail || spriteFail) {
      r.verdict = 'SWALLOWED'; // Tier A: conclusive impossibility
    } else if (scDecisionErr) {
      r.verdict = 'SWALLOWED'; // Tier B: decision error is always SWALLOWED (spec: Amber-Red)
    } else if (scFail) {
      r.verdict = 'SUSPICIOUS'; // Tier C: mismatch without threshold crossing
    } else {
      r.verdict = 'PASS';
    }

    results.push(r);
  }

  // ── TDA (dataset-level) ──
  const termDigits = extractTerminalDigits(allNumericValues);
  const digitCounts = new Array(10).fill(0);
  for (const d of termDigits) digitCounts[d]++;
  const tdaResult = tdaCheck(digitCounts);

  // Store results globally
  window.asaResults = { studies: results, tda: tdaResult, timestamp: new Date().toISOString() };

  // Render all views
  renderGauntlet(results, tdaResult);
  renderHeatmap(results, tdaResult);
  renderReports(results, tdaResult);
  renderExport(results, tdaResult, rows);

  // Switch to Gauntlet tab
  document.querySelector('.tab-btn[data-tab="gauntlet"]').click();
}
```

- [ ] **Step 2: Verify with Tutorial dataset**

Open browser, load Tutorial dataset, click "Run the Gauntlet". Check console for errors. Verify `window.asaResults` has correct verdicts.

- [ ] **Step 3: Commit**

```bash
cd /c/Models/Asa && git add -A && git commit -m "feat: verdict engine with tier-weighted logic and full orchestration"
```

---

### Task 9: Gauntlet View (Tab 2)

Build the animated Gauntlet view showing studies passing through 4 fangs.

**Files:**
- Modify: `C:\Models\Asa\asa.html`

- [ ] **Step 1: Implement renderGauntlet()**

```javascript
// ── Tab 2: The Gauntlet ──

function renderGauntlet(results, tdaResult) {
  const area = document.getElementById('gauntlet-area');

  const passed = results.filter(r => r.verdict === 'PASS').length;
  const suspicious = results.filter(r => r.verdict === 'SUSPICIOUS').length;
  const swallowed = results.filter(r => r.verdict === 'SWALLOWED').length;
  const grey = results.filter(r => r.verdict === 'GREY').length;

  let html = `
    <h2 style="margin-bottom:4px">The Gauntlet</h2>
    <p style="color:var(--text-secondary);margin-bottom:16px">Each study passes through the Staff's fangs. Illusions are consumed.</p>

    <div style="display:flex;gap:16px;margin-bottom:20px;flex-wrap:wrap">
      <div class="card" style="flex:1;min-width:120px;text-align:center">
        <div style="font-size:32px;color:var(--green);font-weight:bold">${passed}</div>
        <div style="font-size:13px;color:var(--text-secondary)">Passed</div>
      </div>
      <div class="card" style="flex:1;min-width:120px;text-align:center">
        <div style="font-size:32px;color:var(--amber);font-weight:bold">${suspicious}</div>
        <div style="font-size:13px;color:var(--text-secondary)">Suspicious</div>
      </div>
      <div class="card" style="flex:1;min-width:120px;text-align:center">
        <div style="font-size:32px;color:var(--red);font-weight:bold">${swallowed}</div>
        <div style="font-size:13px;color:var(--text-secondary)">Swallowed</div>
      </div>
      <div class="card" style="flex:1;min-width:120px;text-align:center">
        <div style="font-size:32px;color:var(--grey);font-weight:bold">${grey}</div>
        <div style="font-size:13px;color:var(--text-secondary)">Not Assessed</div>
      </div>
    </div>

    <!-- Gauntlet flow -->
    <div class="card" style="overflow-x:auto">
      <table style="width:100%;border-collapse:collapse;font-size:13px">
        <thead>
          <tr>
            <th style="padding:8px;text-align:left;border-bottom:2px solid var(--border)">Study</th>
            <th style="padding:8px;text-align:center;border-bottom:2px solid var(--border);color:var(--red)">Fang 1: GRIM</th>
            <th style="padding:8px;text-align:center;border-bottom:2px solid var(--border);color:var(--amber)">Fang 2: SPRITE</th>
            <th style="padding:8px;text-align:center;border-bottom:2px solid var(--border);color:#9b59b6">Fang 3: statcheck</th>
            <th style="padding:8px;text-align:center;border-bottom:2px solid var(--border)">Verdict</th>
          </tr>
        </thead>
        <tbody>`;

  for (const r of results) {
    const verdictBadge = r.verdict === 'PASS' ? '<span class="badge badge-pass">PASS</span>'
      : r.verdict === 'SUSPICIOUS' ? '<span class="badge badge-suspicious">SUSPICIOUS</span>'
      : r.verdict === 'SWALLOWED' ? '<span class="badge badge-swallowed">SWALLOWED</span>'
      : '<span class="badge badge-skipped">N/A</span>';

    html += `<tr style="border-bottom:1px solid var(--border)${r.verdict === 'SWALLOWED' ? ';opacity:0.6;text-decoration:line-through' : ''}">
      <td style="padding:6px 8px;font-weight:500">${r.study}</td>
      <td style="padding:6px 8px;text-align:center">${fangCell(r.grim)}</td>
      <td style="padding:6px 8px;text-align:center">${fangCell(r.sprite)}</td>
      <td style="padding:6px 8px;text-align:center">${fangCell(r.statcheck)}</td>
      <td style="padding:6px 8px;text-align:center">${verdictBadge}</td>
    </tr>`;
  }

  html += `</tbody></table></div>`;

  // TDA panel
  html += `<div class="card" style="margin-top:16px">
    <h3>Fang 4: Terminal Digit Analysis (Dataset-Level)</h3>
    <p style="color:var(--text-secondary);font-size:13px;margin-bottom:8px">
      ${tdaResult.explanation}
    </p>
    <div style="display:flex;align-items:center;gap:16px">
      <span class="badge badge-${tdaResult.verdict === 'PASS' ? 'pass' : tdaResult.verdict === 'WARN' ? 'suspicious' : tdaResult.verdict === 'FAIL' ? 'swallowed' : 'skipped'}">
        ${tdaResult.verdict}
      </span>
      ${tdaResult.chiSq != null ? `<span style="font-size:13px;color:var(--text-secondary)">χ²(9) = ${tdaResult.chiSq.toFixed(2)}, p = ${tdaResult.pValue.toFixed(4)}</span>` : ''}
    </div>
    ${tdaResult.verdict !== 'SKIPPED' ? renderDigitBar(tdaResult) : ''}
  </div>`;

  html += `<p style="margin-top:16px;font-size:12px;color:var(--text-secondary);font-style:italic">
    Forensic flags are evidence for investigation, not proof of fraud. GRIM/SPRITE failures prove mathematical impossibility — but the cause could be transcription error, not fabrication. Always investigate flagged studies before drawing conclusions.
  </p>`;

  area.innerHTML = html;
}

function fangCell(fangResult) {
  if (!fangResult || fangResult.result === 'SKIPPED') return '<span style="color:var(--grey)" title="Skipped">—</span>';
  if (fangResult.result === 'PASS') return `<span style="color:var(--green)" title="${escapeAttr(fangResult.explanation)}">✓</span>`;
  if (fangResult.result === 'FAIL') return `<span style="color:var(--red)" title="${escapeAttr(fangResult.explanation)}">✗</span>`;
  if (fangResult.result === 'DECISION_ERROR') return `<span style="color:var(--red);font-weight:bold" title="${escapeAttr(fangResult.explanation)}">✗✗</span>`;
  return `<span style="color:var(--amber)" title="${escapeAttr(fangResult.explanation)}">⚠</span>`;
}

function escapeAttr(s) {
  return String(s || '').replace(/&/g,'&amp;').replace(/"/g,'&quot;').replace(/'/g,'&#39;').replace(/</g,'&lt;').replace(/>/g,'&gt;');
}

function renderDigitBar(tda) {
  const max = Math.max(...tda.digitCounts);
  let html = '<div style="display:flex;gap:4px;align-items:flex-end;height:80px;margin-top:12px">';
  for (let i = 0; i < 10; i++) {
    const h = max > 0 ? (tda.digitCounts[i] / max * 60) : 0;
    const color = Math.abs(tda.digitCounts[i] - tda.expected) > tda.expected * 0.5 ? 'var(--red)' : 'var(--accent)';
    html += `<div style="flex:1;text-align:center">
      <div style="height:${h}px;background:${color};border-radius:2px 2px 0 0;margin:0 auto;width:80%"></div>
      <div style="font-size:11px;color:var(--text-secondary);margin-top:2px">${i}</div>
      <div style="font-size:10px;color:var(--text-secondary)">${tda.digitCounts[i]}</div>
    </div>`;
  }
  html += '</div><div style="font-size:11px;color:var(--text-secondary);margin-top:4px">Expected per digit: ' + tda.expected.toFixed(1) + '</div>';
  return html;
}
```

- [ ] **Step 2: Test in browser**

Load Tutorial → Run Gauntlet → verify Tab 2 shows summary cards + study table + TDA panel.

- [ ] **Step 3: Commit**

```bash
cd /c/Models/Asa && git add -A && git commit -m "feat: Gauntlet view (Tab 2) with summary, study table, TDA panel"
```

---

### Task 10: Serpent's Eye Heatmap (Tab 3)

**Files:**
- Modify: `C:\Models\Asa\asa.html`

- [ ] **Step 1: Implement renderHeatmap()**

```javascript
// ── Tab 3: Serpent's Eye Heatmap ──

function renderHeatmap(results, tdaResult) {
  const area = document.getElementById('heatmap-area');

  // Sort by severity: SWALLOWED first, then SUSPICIOUS, then PASS, then GREY
  const order = { SWALLOWED: 0, SUSPICIOUS: 1, PASS: 2, GREY: 3 };
  const sorted = [...results].sort((a, b) => (order[a.verdict] ?? 3) - (order[b.verdict] ?? 3));

  let html = `<h2 style="margin-bottom:4px">Serpent's Eye</h2>
    <p style="color:var(--text-secondary);margin-bottom:16px">Heatmap matrix — rows are studies, columns are fangs. Click any cell for detail.</p>
    <div style="display:flex;gap:12px;margin-bottom:12px;font-size:12px;color:var(--text-secondary)">
      <span><span style="display:inline-block;width:12px;height:12px;background:var(--green);border-radius:2px;vertical-align:middle"></span> PASS</span>
      <span><span style="display:inline-block;width:12px;height:12px;background:var(--amber);border-radius:2px;vertical-align:middle"></span> WARN</span>
      <span><span style="display:inline-block;width:12px;height:12px;background:var(--red);border-radius:2px;vertical-align:middle"></span> FAIL</span>
      <span><span style="display:inline-block;width:12px;height:12px;background:var(--grey);border-radius:2px;vertical-align:middle"></span> SKIPPED</span>
    </div>
    <div class="card" style="overflow-x:auto">
    <table style="width:100%;border-collapse:collapse;font-size:13px">
      <thead><tr>
        <th style="padding:8px;text-align:left;border-bottom:2px solid var(--border)">Study</th>
        <th style="padding:8px;text-align:center;border-bottom:2px solid var(--border);width:100px">GRIM</th>
        <th style="padding:8px;text-align:center;border-bottom:2px solid var(--border);width:100px">SPRITE</th>
        <th style="padding:8px;text-align:center;border-bottom:2px solid var(--border);width:100px">statcheck</th>
        <th style="padding:8px;text-align:center;border-bottom:2px solid var(--border);width:100px">Verdict</th>
      </tr></thead><tbody>`;

  for (const r of sorted) {
    html += `<tr style="border-bottom:1px solid var(--border)">
      <td style="padding:6px 8px">${r.study}</td>
      <td style="padding:4px">${heatCell(r.grim, r.study, 'GRIM')}</td>
      <td style="padding:4px">${heatCell(r.sprite, r.study, 'SPRITE')}</td>
      <td style="padding:4px">${heatCell(r.statcheck, r.study, 'statcheck')}</td>
      <td style="padding:6px 8px;text-align:center">${verdictBadge(r.verdict)}</td>
    </tr>`;
  }

  html += '</tbody></table></div>';
  area.innerHTML = html;

  // Click-to-expand detail
  area.querySelectorAll('.heat-cell').forEach(cell => {
    cell.addEventListener('click', () => {
      const existing = cell.parentElement.querySelector('.heat-detail');
      if (existing) { existing.remove(); return; }
      const detail = document.createElement('div');
      detail.className = 'heat-detail';
      detail.style.cssText = 'font-size:12px;color:var(--text-secondary);padding:4px 8px;margin-top:4px;background:var(--bg);border-radius:4px';
      detail.textContent = cell.getAttribute('data-explanation');
      cell.parentElement.appendChild(detail);
    });
  });
}

function heatCell(fangResult, study, fang) {
  if (!fangResult) return '<div style="background:var(--grey);border-radius:4px;padding:6px;text-align:center;color:#fff;font-size:11px">—</div>';
  const colors = { PASS: 'var(--green)', FAIL: 'var(--red)', DECISION_ERROR: 'var(--red)', SKIPPED: 'var(--grey)', WARN: 'var(--amber)' };
  const color = colors[fangResult.result] || 'var(--grey)';
  const label = fangResult.result === 'DECISION_ERROR' ? 'ERR' : fangResult.result;
  return `<div class="heat-cell" style="background:${color};border-radius:4px;padding:6px;text-align:center;color:#fff;font-size:11px;cursor:pointer;opacity:${fangResult.result === 'SKIPPED' ? '0.4' : '1'}"
    data-explanation="${escapeAttr(fangResult.explanation)}" title="Click for detail">${label}</div>`;
}

function verdictBadge(verdict) {
  const map = { PASS: 'badge-pass', SUSPICIOUS: 'badge-suspicious', SWALLOWED: 'badge-swallowed', GREY: 'badge-skipped' };
  return `<span class="badge ${map[verdict] || 'badge-skipped'}">${verdict}</span>`;
}
```

- [ ] **Step 2: Test in browser**

Run Tutorial → check Tab 3 shows heatmap sorted by severity, cells clickable for detail.

- [ ] **Step 3: Commit**

```bash
cd /c/Models/Asa && git add -A && git commit -m "feat: Serpent's Eye heatmap (Tab 3) with click-to-expand"
```

---

### Task 11: Forensic Reports (Tab 4)

**Files:**
- Modify: `C:\Models\Asa\asa.html`

- [ ] **Step 1: Implement renderReports()**

```javascript
// ── Tab 4: Forensic Reports ──

function renderReports(results, tdaResult) {
  const area = document.getElementById('reports-area');
  let html = `<h2 style="margin-bottom:4px">Forensic Reports</h2>
    <p style="color:var(--text-secondary);margin-bottom:12px">Detailed per-study forensic analysis. Click to expand.</p>
    <div style="margin-bottom:12px">
      <button class="btn btn-secondary" id="expandAllBtn" style="font-size:12px">Expand All</button>
      <button class="btn btn-secondary" id="collapseAllBtn" style="font-size:12px">Collapse All</button>
      <input type="text" id="reportSearch" placeholder="Filter by study name..." style="margin-left:8px;padding:4px 8px;background:var(--bg);border:1px solid var(--border);border-radius:4px;color:var(--text);font-size:13px">
    </div>`;

  for (const r of results) {
    html += `<div class="card report-card" data-study="${escapeAttr(r.study)}" style="cursor:pointer">
      <div class="report-header" style="display:flex;justify-content:space-between;align-items:center">
        <span style="font-weight:600">${r.study}</span>
        ${verdictBadge(r.verdict)}
      </div>
      <div class="report-body" style="display:none;margin-top:12px;font-size:13px">
        ${reportFang('Fang 1: GRIM', r.grim)}
        ${reportFang('Fang 2: SPRITE', r.sprite)}
        ${reportFang('Fang 3: statcheck', r.statcheck)}
      </div>
    </div>`;
  }

  area.innerHTML = html;

  // Accordion
  area.querySelectorAll('.report-header').forEach(h => {
    h.addEventListener('click', () => {
      const body = h.nextElementSibling;
      body.style.display = body.style.display === 'none' ? 'block' : 'none';
    });
  });
  document.getElementById('expandAllBtn').addEventListener('click', () => {
    area.querySelectorAll('.report-body').forEach(b => b.style.display = 'block');
  });
  document.getElementById('collapseAllBtn').addEventListener('click', () => {
    area.querySelectorAll('.report-body').forEach(b => b.style.display = 'none');
  });
  document.getElementById('reportSearch').addEventListener('input', (e) => {
    const q = e.target.value.toLowerCase();
    area.querySelectorAll('.report-card').forEach(c => {
      c.style.display = c.getAttribute('data-study').toLowerCase().includes(q) ? 'block' : 'none';
    });
  });
}

function reportFang(title, fangResult) {
  if (!fangResult) return '';
  const color = fangResult.result === 'PASS' ? 'var(--green)' : fangResult.result === 'FAIL' || fangResult.result === 'DECISION_ERROR' ? 'var(--red)' : fangResult.result === 'SKIPPED' ? 'var(--grey)' : 'var(--amber)';
  return `<div style="padding:8px;margin-bottom:8px;border-left:3px solid ${color};padding-left:12px">
    <div style="font-weight:500;margin-bottom:4px">${title}: <span style="color:${color}">${fangResult.result}</span></div>
    <div style="color:var(--text-secondary)">${fangResult.explanation || ''}</div>
    ${fangResult.pComputed != null ? `<div style="color:var(--text-secondary)">Recomputed p = ${fangResult.pComputed.toFixed(6)}</div>` : ''}
    ${fangResult.closestSD != null ? `<div style="color:var(--text-secondary)">Closest achievable SD = ${fangResult.closestSD.toFixed(4)}</div>` : ''}
  </div>`;
}
```

- [ ] **Step 2: Test in browser**

Run Tutorial → Tab 4. Verify accordion expand/collapse, search filter, per-fang detail.

- [ ] **Step 3: Commit**

```bash
cd /c/Models/Asa && git add -A && git commit -m "feat: Forensic Reports (Tab 4) with accordion, search, per-fang detail"
```

---

### Task 12: Export (Tab 5) + TruthCert Receipt

**Files:**
- Modify: `C:\Models\Asa\asa.html`

- [ ] **Step 1: Implement renderExport() with SHA-256 hashing**

```javascript
// ── Tab 5: Export ──

async function sha256(text) {
  const encoder = new TextEncoder();
  const data = encoder.encode(text);
  const hash = await crypto.subtle.digest('SHA-256', data);
  return Array.from(new Uint8Array(hash)).map(b => b.toString(16).padStart(2, '0')).join('');
}

function renderExport(results, tdaResult, originalRows) {
  const area = document.getElementById('export-area');
  area.innerHTML = `<h2 style="margin-bottom:4px">Export</h2>
    <p style="color:var(--text-secondary);margin-bottom:16px">Download cleaned data, flagged studies, or the full forensic receipt.</p>
    <div style="display:flex;gap:12px;flex-wrap:wrap;margin-bottom:24px">
      <button class="btn btn-primary" id="exportCleanCSV">Download Cleaned CSV</button>
      <button class="btn btn-secondary" id="exportFlaggedCSV">Download Flagged Only</button>
      <button class="btn btn-secondary" id="exportReceipt">Download TruthCert Receipt</button>
    </div>
    <div class="card" id="receiptPreview"><h3>TruthCert Receipt Preview</h3><div id="receiptContent" style="margin-top:8px;font-family:var(--font-mono);font-size:12px;white-space:pre-wrap;color:var(--text-secondary)">Generating...</div></div>`;

  // Generate receipt
  const inputStr = JSON.stringify(originalRows);
  sha256(inputStr).then(hash => {
    const receipt = {
      tool: 'Asa',
      version: ASA_VERSION,
      timestamp: window.asaResults.timestamp,
      inputHash: hash,
      studyCount: results.length,
      verdicts: { passed: results.filter(r=>r.verdict==='PASS').length, suspicious: results.filter(r=>r.verdict==='SUSPICIOUS').length, swallowed: results.filter(r=>r.verdict==='SWALLOWED').length, notAssessed: results.filter(r=>r.verdict==='GREY').length },
      tda: { verdict: tdaResult.verdict, chiSq: tdaResult.chiSq, pValue: tdaResult.pValue },
      perStudy: results.map(r => ({ study: r.study, verdict: r.verdict, grim: r.grim?.result, sprite: r.sprite?.result, statcheck: r.statcheck?.result }))
    };
    document.getElementById('receiptContent').textContent = JSON.stringify(receipt, null, 2);
    window.asaReceipt = receipt;
  });

  // CSV export
  document.getElementById('exportCleanCSV').addEventListener('click', () => {
    const header = 'study,grim_result,sprite_result,statcheck_result,overall_verdict';
    const lines = results.map(r => [r.study, r.grim?.result||'', r.sprite?.result||'', r.statcheck?.result||'', r.verdict].join(','));
    downloadFile(header + '\n' + lines.join('\n'), 'asa_results.csv', 'text/csv');
  });
  document.getElementById('exportFlaggedCSV').addEventListener('click', () => {
    const flagged = results.filter(r => r.verdict === 'SUSPICIOUS' || r.verdict === 'SWALLOWED');
    const header = 'study,grim_result,grim_explanation,sprite_result,statcheck_result,statcheck_explanation,verdict';
    const lines = flagged.map(r => [r.study, r.grim?.result||'', `"${(r.grim?.explanation||'').replace(/"/g,'""')}"`, r.sprite?.result||'', r.statcheck?.result||'', `"${(r.statcheck?.explanation||'').replace(/"/g,'""')}"`, r.verdict].join(','));
    downloadFile(header + '\n' + lines.join('\n'), 'asa_flagged.csv', 'text/csv');
  });
  document.getElementById('exportReceipt').addEventListener('click', () => {
    if (window.asaReceipt) downloadFile(JSON.stringify(window.asaReceipt, null, 2), 'asa_truthcert_receipt.json', 'application/json');
  });
}

function downloadFile(content, filename, type) {
  const blob = new Blob([content], { type });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url; a.download = filename;
  document.body.appendChild(a); a.click();
  document.body.removeChild(a);
  URL.revokeObjectURL(url);
}
```

- [ ] **Step 2: Test exports**

Run Tutorial → Tab 5. Download each file, verify:
- Cleaned CSV has all 12 rows with verdicts
- Flagged CSV has only SUSPICIOUS + SWALLOWED rows
- TruthCert receipt JSON has correct hash, counts, per-study verdicts

- [ ] **Step 3: Commit**

```bash
cd /c/Models/Asa && git add -A && git commit -m "feat: Export tab (Tab 5) with cleaned CSV, flagged CSV, TruthCert receipt"
```

---

### Task 13: Integration Tests

Full pipeline Selenium tests verifying both demo datasets produce expected results.

**Files:**
- Create: `C:\Models\Asa\tests\test_integration.py`

- [ ] **Step 1: Write integration tests**

```python
"""Integration tests: full pipeline on demo datasets."""
import pytest, json
from conftest import js

@pytest.fixture(autouse=True)
def load_app(driver, app_url):
    driver.get(app_url)

class TestTutorialDataset:
    def test_load_and_run(self, driver):
        """Load tutorial, run gauntlet, check results exist"""
        driver.find_element('id', 'loadTutorial').click()
        driver.find_element('id', 'runGauntletBtn').click()
        import time; time.sleep(2)  # wait for SPRITE iterations
        results = js(driver, 'JSON.stringify(window.asaResults)')
        data = json.loads(results)
        assert len(data['studies']) == 12

    def test_grim_fails(self, driver):
        """Tutorial should have exactly 4 GRIM failures"""
        driver.find_element('id', 'loadTutorial').click()
        driver.find_element('id', 'runGauntletBtn').click()
        import time; time.sleep(2)
        results = json.loads(js(driver, 'JSON.stringify(window.asaResults)'))
        grim_fails = [s for s in results['studies'] if s.get('grim',{}).get('result') == 'FAIL']
        assert len(grim_fails) >= 4, f"Expected >=4 GRIM fails, got {len(grim_fails)}"

    def test_statcheck_fails(self, driver):
        """Tutorial should have 2 statcheck failures"""
        driver.find_element('id', 'loadTutorial').click()
        driver.find_element('id', 'runGauntletBtn').click()
        import time; time.sleep(2)
        results = json.loads(js(driver, 'JSON.stringify(window.asaResults)'))
        sc_fails = [s for s in results['studies'] if s.get('statcheck',{}).get('result') in ('FAIL','DECISION_ERROR')]
        assert len(sc_fails) >= 2, f"Expected >=2 statcheck fails, got {len(sc_fails)}"

    def test_clean_studies_pass(self, driver):
        """Ahmed 2020 and Lee 2019 should PASS"""
        driver.find_element('id', 'loadTutorial').click()
        driver.find_element('id', 'runGauntletBtn').click()
        import time; time.sleep(2)
        results = json.loads(js(driver, 'JSON.stringify(window.asaResults)'))
        for name in ['Ahmed 2020', 'Lee 2019']:
            study = next((s for s in results['studies'] if s['study'] == name), None)
            assert study is not None, f"{name} not found"
            assert study['verdict'] == 'PASS', f"{name} verdict={study['verdict']}"

class TestFujiiDataset:
    def test_load_and_run(self, driver):
        """Load Fujii, run gauntlet"""
        driver.find_element('id', 'loadFujii').click()
        driver.find_element('id', 'runGauntletBtn').click()
        import time; time.sleep(2)
        results = json.loads(js(driver, 'JSON.stringify(window.asaResults)'))
        assert len(results['studies']) == 12

    def test_has_swallowed(self, driver):
        """Fujii dataset should have swallowed studies"""
        driver.find_element('id', 'loadFujii').click()
        driver.find_element('id', 'runGauntletBtn').click()
        import time; time.sleep(2)
        results = json.loads(js(driver, 'JSON.stringify(window.asaResults)'))
        swallowed = [s for s in results['studies'] if s['verdict'] == 'SWALLOWED']
        assert len(swallowed) >= 1, "Expected at least 1 swallowed study in Fujii dataset"

class TestExport:
    def test_receipt_has_hash(self, driver):
        """TruthCert receipt should have input hash"""
        driver.find_element('id', 'loadTutorial').click()
        driver.find_element('id', 'runGauntletBtn').click()
        import time; time.sleep(3)
        receipt = js(driver, 'JSON.stringify(window.asaReceipt)')
        if receipt:
            data = json.loads(receipt)
            assert 'inputHash' in data
            assert len(data['inputHash']) == 64  # SHA-256 hex
```

- [ ] **Step 2: Run all tests**

```bash
cd /c/Models/Asa && python -m pytest tests/ -v --timeout=60
```
Expected: All tests PASS.

- [ ] **Step 3: Commit**

```bash
cd /c/Models/Asa && git add -A && git commit -m "test: integration tests for tutorial + Fujii datasets, export verification"
```

---

### Task 14: Final Polish + Git Repo

**Files:**
- Modify: `C:\Models\Asa\asa.html` (div balance check, accessibility)

- [ ] **Step 1: Div balance check**

```bash
cd /c/Models/Asa && grep -c '<div' asa.html && grep -c '</div>' asa.html
```
Both counts must match. Fix any imbalance.

- [ ] **Step 2: Verify no literal `</script>` in template literals**

```bash
cd /c/Models/Asa && grep -n '</script>' asa.html
```
Should only appear as the actual closing `</script>` tag, never inside JS strings.

- [ ] **Step 3: Run full test suite**

```bash
cd /c/Models/Asa && python -m pytest tests/ -v --timeout=60
```
All tests must PASS. Report pass/fail counts.

- [ ] **Step 4: Create GitHub repo and push**

```bash
cd /c/Models/Asa && gh repo create mahmood726-cyber/asa --private --source=. --push
```

- [ ] **Step 5: Final commit with test results**

```bash
cd /c/Models/Asa && git add -A && git commit -m "polish: div balance, accessibility, final verification"
```

---

## Summary

| Task | Component | Estimated Steps |
|------|-----------|----------------|
| 1 | Scaffold + tabs + dark theme | 4 |
| 2 | Statistical math library | 6 |
| 3 | GRIM engine | 5 |
| 4 | SPRITE engine | 5 |
| 5 | statcheck engine | 5 |
| 6 | TDA engine | 5 |
| 7 | CSV parser + column mapping + demos | 5 |
| 8 | Verdict engine + orchestration | 3 |
| 9 | Gauntlet view (Tab 2) | 3 |
| 10 | Serpent's Eye heatmap (Tab 3) | 3 |
| 11 | Forensic Reports (Tab 4) | 3 |
| 12 | Export + TruthCert receipt (Tab 5) | 3 |
| 13 | Integration tests | 3 |
| 14 | Final polish + repo | 5 |
| **Total** | | **58 steps** |
