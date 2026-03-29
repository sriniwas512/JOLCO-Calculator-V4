# JOLCO Equity IRR Calculator V4 — Design Spec

**Date:** 2026-03-29
**Status:** Approved
**File:** `jolco-v3.jsx` (in-place upgrade; version badge changes to `v4`)

---

## 1. Overview

V4 upgrades the JOLCO Equity IRR Calculator from a static deal-input tool to an interactive deal-analysis platform. The financial engine (IRR solver, depreciation schedule, yearly cashflow loop, three return streams) is preserved exactly from V3. All new features are additive UX and analysis layers on top.

**Target users:** Shipbrokers, ship finance bankers, Japanese TK equity investors — desktop use on trading-floor monitors.

---

## 2. Tab Structure

```
Deal Inputs | Sensitivity | Scenarios | Depreciation | Equity Cashflows | vs Treasury
```

`Sensitivity` and `Scenarios` are new. The remaining four carry over from V3.

---

## 3. Architecture

### 3.1 File structure (single `.jsx`)

```
[top] Existing pure functions (unchanged):
      solveIRR, computeDepr, computeUsedAssetLife, VESSEL_DB, FLAG_OPTIONS

[new] computeHeatmapGrid(inputs, xVar, yVar) → 2D array of { blendedIRR, equityIRR, moic }
[new] bisectBreakeven(inputs, variable, target) → number | "< min" | "> max" | "N/A"
[new] computeTornadoData(baseInputs) → Array<{ var, label, shock, low, high }>

[new components] HeatmapCanvas({ grid, xLabels, yLabels, currentX, currentY, onCellClick })
[new components] TornadoChart({ data, baseIRR })
[new components] CumulativeCFChart({ years, withTax, withoutTax, paybackYear })
[new components] SensitivityTab({ R, inputs, onCellClick })
[new components] ScenariosTab({ scenarios, onDelete, onLoad, onSetBaseline })

[unchanged] Inp, Slider components

[main] JOLCOv3() — all useState, useMemo(R), tab routing
```

**`inputs` object shape** (passed to `computeHeatmapGrid`, `bisectBreakeven`, `computeTornadoData`):
```js
{
  vesselPrice,      // number ($M)
  debtPct,          // number (%)
  amortYrs,         // number (years)
  sofrRate,         // number (%)
  spreadBps,        // number (basis points)
  jpyBaseRate,      // number (%)
  bankSpreadBps,    // number (basis points)
  swapCostBps,      // number (basis points)
  saleCommission,   // number (%)
  bbcCommission,    // number (%)
  taxRate,          // number (%)
  capGainsTaxRate,  // number (%)
  foreignInterestTaxPct, // number (%)
  specialDeprPct,   // number (%)
  treasuryYield,    // number (%)
  vesselTypeId,     // string
  flagId,           // string
  vesselAgeYrs,     // number
  effectiveExerciseYear, // number (integer) — the year the PO is exercised (clamped to poFirstYear–poLastYear)
  poSchedule,       // array of { yr, price, obligatory, isOverridden } — PO exercise prices only, not amortization
  usefulLife,       // number (computed from vessel type + flag + age)
  poPriceMil,       // number ($) — looked up as: poSchedule.find(p => p.yr === effectiveExerciseYear).price * 1e6
}
```
These are the same values that drive the V3 `R` useMemo. The pure functions replicate the same financial computation logic as `R` but as a standalone call. `poSchedule` is used solely to look up `poPriceMil` — it is not used for amortization. All amortization is straight-line (`annualPrincipal = VP / amortYrs`).

### 3.2 New state in root component

```js
// Sensitivity tab
const [heatXVar, setHeatXVar] = useState("spreadBps")
const [heatYVar, setHeatYVar] = useState("poPremiumPct")  // derived variable
const [poMode, setPoMode]     = useState("absolute")       // "absolute" | "premium"

// Scenarios
const [scenarios, setScenarios] = useState([])  // max 5; each = { name, inputs, outputs }
```

### 3.3 PO Premium derived variable

Computed inside the existing `R` useMemo alongside the existing outputs:

```js
const remainingBalAtExercise = Math.max(0, VP - annualPrincipal * effectiveExerciseYear)
const poPremiumPct = remainingBalAtExercise > 0
  ? (poPriceMil - remainingBalAtExercise) / remainingBalAtExercise * 100
  : null
```

**Reverse formula** (premium % → absolute $M, used when user edits the `% over Balance` field):
```js
poPriceMil = remainingBalAtExercise * (1 + enteredPremiumPct / 100)
```

The `poMode` toggle switches the Deal Inputs PO field between showing absolute `$M` and `% premium over balance`. Switching converts the displayed value using the formulas above and updates `poPremium` (the absolute $M state), keeping the deal economically identical. `poPriceMil` (absolute) is always the canonical state; `poPremiumPct` is always derived.

---

## 4. Sensitivity Tab

### 4.1 Layout (top to bottom)

1. Axis picker row (two dropdowns: X-axis, Y-axis)
2. Heatmap canvas (full container width)
3. Breakeven cards row (4 cards)

### 4.2 Axis variables

| Key | Display label | Unit | Range | Step |
|-----|--------------|------|-------|------|
| `spreadBps` | Equity Spread over SOFR | bps | current ± 200 bps | 25 |
| `poPremiumPct` | PO Premium over Balance | % | −10% to +30% | 5 |
| `debtPct` | Leverage (Debt %) | % | 50–85% | 5 |
| `taxRate` | Investor Tax Rate | % | 20–40% | 2 |
| `sofrRate` | SOFR Rate | % | 2–7% | 0.5 |
| `vesselPrice` | Vessel Price | $M | current ± 30% | 10% |
| `exerciseYear` | PO Exercise Year | yr | poFirstYear–poLastYear | 1 |
| `specialDeprPct` | Special Depreciation | % | 0–flag max | 5 |

Default: X = `spreadBps`, Y = `poPremiumPct`. X and Y cannot be the same.

**Axis sweep for `poPremiumPct`:** Because `poPremiumPct` is a derived variable, sweeping it requires converting each target premium % back to an absolute `poPriceMil` for the cell calculation.

The remaining financing balance follows **straight-line amortization** throughout this model (V3 design, preserved in V4). `annualPrincipal = VP / amortYrs` is always straight-line; `poSchedule` stores PO exercise prices, not amortization amounts. Therefore:

```js
// annualPrincipal = (vesselPrice * 1e6) / amortYrs  [straight-line, same as V3]
cellRemainingBal = Math.max(0, VP - annualPrincipal * cellExerciseYear)
cellPoPriceMil   = cellRemainingBal * (1 + targetPremiumPct / 100)
```

- When `poPremiumPct` is one axis and `exerciseYear` is the **other**, use the **cell's** `exerciseYear` for `cellRemainingBal`.
- When `exerciseYear` is **not** being swept, use the current deal's `effectiveExerciseYear`.
- When neither axis involves `poPremiumPct`, this conversion does not apply.

### 4.3 Heatmap rendering

- **Grid:** 9×9 (81 cells). Center cell = current deal inputs.
- **Computation:** `computeHeatmapGrid` runs the full `R` calculation logic for each cell, swapping in the cell's (x, y) values. Everything else = current deal inputs.
- **Performance:** Debounce recalculation 200ms after any input or axis change. Use `setTimeout` chunking (process rows in batches) to avoid blocking the main thread. No web worker unless profiling shows >200ms render time.
- **Color scale:** Continuous linear interpolation across 7 anchor points (extended downward for negative IRR):
  - IRR ≤ −10% → `#2d0a0a` (near-black maroon — deep loss)
  - IRR = −5% → `#a8152e` (dark crimson)
  - IRR = 0% → `#f7768e` (bright red — breakeven)
  - IRR = 3% → `#ff9e64` (orange)
  - IRR = 6% → `#e0af68` (amber)
  - IRR = 10% → `#9ece6a` (green)
  - IRR ≥ 13% → `#1abc9c` (deep green)
  Interpolate linearly between adjacent anchors. Negative IRRs show a visible red→crimson→maroon gradient communicating depth of loss.
- **Incomputable IRR (solveIRR returns null):**
  - Cell fill: `#24283b` (neutral dark, visually distinct from any IRR color)
  - Cell label: `"—"` in `#565f89` (dim gray)
  - Hover tooltip: `"IRR not computable — cashflows do not produce a real solution at this parameter combination (e.g. all-negative cashflows, or solver non-convergent)"`
  - Click: does nothing (no deal input update for null cells)
- **Current deal marker:** White `2px` border on the cell matching current inputs.
- **IRR used for color and cell labels: `blendedIRR` (all three streams).** Cell color and displayed % both reflect `blendedIRR`. `equityIRR` and `moic` are only shown in the hover tooltip.
- **Cell labels:** Show `blendedIRR` % as text inside each cell (e.g. `7.4%`, or `−3.1%` for negative) when cell width ≥ 40px. Null cells show `"—"`.
- **Hover tooltip:** `"Spread: 300bps, PO: +15% → Blended IRR: 8.42% | Equity IRR: 5.31% | MoIC: 1.47×"`
- **Click:** Updates the two relevant deal input states to the clicked cell's values (no-op for null cells).
- **Canvas sizing:** Full container width, height = width × 0.6 (aspect ratio), min 300px tall.

### 4.4 Breakeven cards

Four cards in a single row below the heatmap:

| Card | Computation |
|------|-------------|
| Breakeven Spread | `bisectBreakeven(inputs, "spreadBps", 0)` — minimum spread (bps) at which Blended IRR = 0% |
| Breakeven PO | `bisectBreakeven(inputs, "poPremiumPct", 0)` — minimum PO premium (%) at which Blended IRR = 0% |
| Match UST at | `bisectBreakeven(inputs, "spreadBps", R.treasPostTaxYield)` — spread at which Blended IRR = post-tax Treasury yield |
| Tax Dependency | See formula below |

**`treasPostTaxYield` source:** Carried over from V3's `R` useMemo:
```js
treasPostTaxYield = treasuryYield * (1 - foreignInterestTaxPct / 100)
// result is in percentage form, e.g., 3.07 for 3.07%
```
`bisectBreakeven` targets are always in the same unit as `blendedIRR` (decimal, e.g. 0.0742). The "Match UST at" card must pass the target as `treasPostTaxYield / 100`:
```js
bisectBreakeven(inputs, "spreadBps", R.treasPostTaxYield / 100)
```
The card displays the result in bps (i.e. multiply the returned decimal spread by 10000, or return the raw bps value — see below).

**`bisectBreakeven` search bounds:** Use the same ranges defined in the axis variable table (Section 4.2). These bounds apply for all uses of `bisectBreakeven` (breakeven cards use `spreadBps` and `poPremiumPct` only). The function searches within [range min, range max] for the axis variable; these are the same bounds used to generate heatmap axis labels.

**`bisectBreakeven` return value convention:** Returns the value in the **same units as the axis variable** (e.g. bps for `spreadBps`, % for `poPremiumPct`). The function evaluates `computeBlendedIRR(inputs with variable = testValue) - target` and bisects to find the root.

**`target` is always a desired output IRR in decimal form — never an axis variable value.** Examples:
- `target = 0` means "find the axis value where Blended IRR = 0%"
- `target = 0.0307` means "find the axis value where Blended IRR = 3.07%"
- `target` is never expressed in bps or % of the axis variable itself.

**No-root fallback:** The fallback strings depend on which direction increasing the variable moves the IRR:

| Variable | IRR increases when variable... | "< min" meaning | "> max" meaning |
|----------|-------------------------------|-----------------|-----------------|
| `spreadBps` | increases | breakeven below tested range | breakeven above tested range |
| `poPremiumPct` | increases | breakeven below tested range | breakeven above tested range |
| `debtPct` | varies (leverage effect) | use "N/A" | use "N/A" |
| `taxRate` | increases (more tax shield) | breakeven below tested range | breakeven above tested range |
| `sofrRate` | decreases (cheaper equity rate) | use "N/A" | use "N/A" |
| `vesselPrice` | varies | use "N/A" | use "N/A" |
| `exerciseYear` | varies | use "N/A" | use "N/A" |
| `specialDeprPct` | increases | breakeven below tested range | breakeven above tested range |

For all breakeven cards (not tornado), the only variables used are `spreadBps` and `poPremiumPct`, both of which are monotonically positive. So "< min" and "> max" are safe to use for those two cards.

**Tax Dependency formula:**
```js
// totalProfit = sum of all three stream returns over the deal term (from R useMemo)
const totalProfit = R.totalStream1 + R.totalStream2 + R.totalStream3
// totalStream1 = cumulative hire spread; totalStream2 = cumulative tax shield (can be negative);
// totalStream3 = residual/PO net proceeds
const taxDependency = (totalProfit > 0)
  ? Math.max(0, Math.min(100, (R.totalStream2 / totalProfit) * 100))
  : null  // display "N/A" if totalProfit ≤ 0 (deal is a loss or break-even; tax dependency is undefined)
```
`totalStream2` can be negative (when SPC is net taxable over the deal life). If `totalProfit ≤ 0`, the tax dependency concept is not meaningful — display "N/A". The `Math.min/max` clamp to [0, 100] handles the case where `totalStream2 > totalProfit` (tax shield > total profit, unusual edge case) or `totalStream2 < 0` (net tax cost).

Tax dependency > 60% → amber card border + note: *"Tax capacity risk: >60% of return depends on depreciation losses."*

---

## 5. Scenarios Tab

### 5.1 Save

- "Save Current as Scenario" button (disabled when 5 scenarios exist, tooltip: "Remove a scenario to add another")
- Inline name input (text field + confirm button, no modal)
- Snapshot stores the following input state variables:
  ```
  vesselTypeId, flagId, vesselPrice, vesselAgeYrs, debtPct, amortYrs, leaseTerm,
  sofrRate, spreadBps, jpyBaseRate, bankSpreadBps, swapCostBps,
  saleCommission, bbcCommission, poFirstYear, poLastYear, poPremium,
  poOverrides, exerciseYear, taxRate, capGainsTaxRate, foreignInterestTaxPct,
  specialDeprPct, treasuryYield
  ```
  V4 UI-only state (`poMode`, `heatXVar`, `heatYVar`) is **not** included in the snapshot (these are display preferences, not deal inputs).
- Snapshot also stores pre-computed outputs: `{ totalStream1, totalStream2, totalStream3, totalEquityDeployed, blendedIRR, equityIRR, jolcoProfit, spread, poPremiumPct }` — computed from `R` at save time, not recalculated on load.
  - `spread` = V3 variable: `blendedIRR - treasPostTaxYield / 100` (decimal). Displayed in bps in the "vs UST" column: `(spread * 10000).toFixed(0) + "bps"`. Positive = green, negative = red.
- First saved scenario auto-tagged as ★ Base Case

### 5.2 Comparison table

Bloomberg-style, columns = scenarios (up to 5), rows = metrics.

**Column header:** scenario name | Load → button | ✕ delete button

**Rows:**

| Metric | Color |
|--------|-------|
| Equity Deployed | `#7aa2f7` |
| Blended IRR | `#9ece6a` / `#f7768e` |
| Charter Economics IRR (pre-tax, ①+③) | `#e0af68` |
| Stream ① Hire Spread | `#9ece6a` |
| Stream ② Tax Shield | `#bb9af7` |
| Stream ③ Residual | `#e0af68` |
| MoIC | `#c0caf5` |
| vs UST | `#9ece6a` / `#f7768e` |

**Delta display:** Columns 2–5 show value + delta vs Base Case inline. Positive delta = green, negative = red.

**Empty state:** Single centered card: *"Save your current deal as a scenario to start comparing. Up to 5 scenarios."*

**Load:** Clicking "Load →" on a scenario column:
1. Restores all of that scenario's saved input state values into the root component's state setters
2. Does **not** change the active tab (user stays on Scenarios tab)
3. Heatmap recalculates via the normal 200ms debounce triggered by the state update
4. If the inline name input is currently open (user is mid-save), it is closed and discarded before the load executes

---

## 6. UX Improvements (Deal Inputs + Equity Cashflows tabs)

### 6.1 Relabeling

| Old | New |
|-----|-----|
| Fixed Hire | Scheduled Amortization (rate-insensitive) |
| Variable Hire | Financing Return (rate-sensitive, tied to outstanding balance) |
| "Interest earned on the equity portion of outstanding balance" | "Equity return embedded in charter hire, net of brokerage, allocated to TK investors" |

### 6.2 Stream subtitle badges

In the 3 summary cards at top of Deal Inputs:
- ① subtitle: `"Cash yield from charter operations (pre-tax)"`
- ② subtitle: `"Tax arbitrage — depends on investor's taxable income capacity"`
- ③ subtitle: `"Terminal event — PO exercise at lease end"`

### 6.3 Dual IRR display

Replace single IRR metric in summary row with two side-by-side blocks:

```
Charter Economics  |  Including Tax Shield
   5.31%  (①+③)        7.42%  (all streams)
```

Both always visible. Cannot be confused for each other.

**Variable mapping:**
- "Charter Economics" = `R.equityIRR` — this is the V3 variable computed from `equityCF_noTax` (streams ①+③ only, no tax shield). `equityIRR` and "Charter Economics IRR" are identical; these are two names for the same computed value.
- "Including Tax Shield" = `R.blendedIRR` — computed from `equityCF` (all three streams)

The scenarios snapshot saves `equityIRR` — this is the Charter Economics IRR displayed in the comparison table. No separate field is needed.

### 6.4 BBC commission tax offset line

In expandable year-by-year rows (Deal Inputs + Equity Cashflows tabs):
```
BBC commission tax offset: +$XX,XXX  (BBC comm × tax rate — partially recovered via SPC deduction)
```
Computed as: `bbcCommCost × (taxRate / 100)`

### 6.5 PO mode toggle

In Deal Inputs, next to PO Premium input:
- Toggle button: `$ Absolute` | `% over Balance`
- Switching converts displayed value; underlying `poPriceMil` stays economically identical

### 6.6 Tax capacity caveat

Small italic note below Blended IRR in Deal Inputs summary and Equity Cashflows tab:
> *"Assumes investor has sufficient other taxable income to absorb full depreciation losses each year. If tax capacity is limited, actual returns will be lower."*

---

## 7. Charts (Equity Cashflows Tab)

Two new canvas charts added below the existing year-by-year cashflow table.

### 7.1 Tornado Chart

- Horizontal bar chart on `<canvas>`, full width, ~220px tall
- One row per the following **7 variables** (exactly), sorted by absolute `blendedIRR` impact (largest at top):

| Variable | Input key | Shock | Unit |
|----------|-----------|-------|------|
| Equity Spread | `spreadBps` | ±100 | bps |
| PO Premium | `poPremiumPct` | ±5 | % |
| Leverage (Debt %) | `debtPct` | ±5 | % |
| SOFR Rate | `sofrRate` | ±0.5 | % |
| Vessel Price | `vesselPrice` | ±10% of current | $M |
| Investor Tax Rate | `taxRate` | ±2 | % |
| Special Depreciation | `specialDeprPct` | ±5 | % |

For each variable, compute `blendedIRR` at (current − shock) and (current + shock). "Downside bar" = whichever extreme gives the lower IRR; "Upside bar" = whichever gives the higher IRR. Both halves are computed, not assumed directional.

- **IRR used throughout tornado: `blendedIRR` (all three streams).** `equityIRR` is not used in this chart.
- Each bar: left half = downside (`#f7768e`), right half = upside (`#9ece6a`), anchored at current `blendedIRR`
- Current `blendedIRR` = vertical reference line in `#7aa2f7`
- Left labels = variable name, bar-end labels = `blendedIRR` at that extreme (formatted as `X.XX%`)
- "Downside" = lower blended IRR; "Upside" = higher blended IRR for each variable's shock direction

### 7.2 Cumulative Equity Cashflow Chart

- Line chart on `<canvas>`, full width, ~220px tall
- **Two lines, both derived from the V3 cashflow arrays:**
  - "With Tax Shield" (`#bb9af7`): running sum of `R.equityCF` array (year 0 = initial equity outflow, years 1–N = full cashflows including tax shield Stream ②). This is `cumulativeEquityCF` already computed per-year in V3's loop.
  - "Without Tax Shield" (`#e0af68`): running sum of `R.equityCF_noTax` array (same structure but tax shield excluded — streams ①+③ only). Compute by cumulating `equityCF_noTax` the same way.
- X-axis: years 0 → effectiveExerciseYear; Y-axis: cumulative equity CF in $M (divide by 1e6)
- Payback year = first year where cumulative value crosses from negative to positive → dotted vertical line + label "Year N payback"
- If payback never occurs (cumulative remains negative throughout), no payback line is drawn
- Zero line = `#3b4261`

**Cash flow series construction (from V3):**
```
equityCF[0]      = -(equity + saleCommCost)          [year 0 outflow, same for both series]
equityCF[yr]     = equityPrincipalReturn + hireSpread + taxShieldThisYear + residualToEquity
equityCF_noTax[yr] = equityPrincipalReturn + hireSpread + residualToEquity  [no taxShield]
```
Both arrays are already computed in V3's `R` useMemo and available as `R.equityCF` and `R.equityCF_noTax`.

---

## 8. Technical Constraints

- Single `.jsx` file (esbuild bundle, no multi-file split)
- No external component libraries (no MUI, no Ant Design)
- Heatmap and tornado: `<canvas>` elements only — no charting library
- Cumulative CF chart: `<canvas>` (Recharts fallback acceptable if canvas proves awkward)
- Tokyo Night theme throughout (colors as specified in build prompt)
- `JetBrains Mono` for numbers, `Inter` for labels
- All state: `useState` + `useMemo` in root component; no Redux, no context
- Mobile: "do not break" means the page must not overflow or clip on a narrow viewport. Canvas charts should be scrollable horizontally if they exceed viewport width. Touch-tap on heatmap cells to update inputs is **not required** — the heatmap is a read-only visualization on touch devices. No touch event handling needed.

---

## 9. Financial Model

Preserve V3 engine exactly. Reference: `JOLCO-V4-Build-Prompt.md` Appendix. No changes to:
- `solveIRR` (Newton-Raphson + bisection)
- `computeDepr` (200% DB → SL switch, MOF post-FY2012)
- `computeUsedAssetLife` (NTA Art. 3 remaining life formula)
- Yearly cashflow loop (equityCF, equityCF_noTax, three streams)
- `R` useMemo dependency array

**Key V3 variable clarifications for V4:**
- `R.equityCF` — full cashflow array including all three streams (blendedIRR is IRR of this)
- `R.equityCF_noTax` — cashflow array with tax shield (Stream ②) removed; years 1–N = `equityPrincipalReturn + hireSpread + residualToEquity` only. `R.equityIRR` = IRR of this array = Charter Economics IRR.
- `R.totalStream1` — sum of `hireSpread` across all years
- `R.totalStream2` — sum of `taxShieldThisYear` across all years (can be negative)
- `R.totalStream3` — `residualToEquity` at exit year only
- `R.spread` — `blendedIRR - treasPostTaxYield / 100` (decimal); stored in scenario snapshots as "vs UST" source

New additions to `R` useMemo output: `poPremiumPct`, `remainingBalAtExercise`.

---

## 10. Success Criteria

1. Plug in a real deal in 60 seconds on Deal Inputs tab
2. Sensitivity tab heatmap updates within 200ms of any input change
3. Clicking a heatmap cell updates Deal Inputs and loads that scenario into cashflow detail
4. Saving two scenarios and comparing them shows correct deltas vs base case
5. Breakeven cards show correct bisection results for current inputs
6. Tornado chart correctly ranks variables by IRR impact
7. Charter Economics IRR and Blended IRR always shown side by side — never confused
8. Tax dependency > 60% triggers amber warning
