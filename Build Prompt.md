# JOLCO Equity IRR Calculator V4 — Build Prompt

> Hand this entire document to your coding agent. It contains everything needed to build V4 from the V3 codebase.

---

## CONTEXT FOR THE DEVELOPER

You are upgrading a JOLCO (Japanese Operating Lease with Call Option) equity IRR calculator from V3 to V4. The V3 codebase is a single-file React component (`jolco-v3.jsx`, ~900 lines) with a Tokyo Night dark theme. The financial model is sound but the tool lacks the interactive analysis features that make it genuinely useful in a deal negotiation or investor pitch.

**Who uses this:** Shipbrokers, ship finance bankers, and Japanese TK equity investors evaluating JOLCO deals. These people need to quickly see "if I push the charter margin up 50bps, how much does my IRR move?" or "at what PO premium does this deal stop making sense?" — the kind of real-time sensitivity analysis they currently do in Excel.

**V3 source files are provided alongside this prompt.** Read `jolco-v3.jsx` thoroughly before starting — the financial engine (IRR solver, depreciation schedule, yearly cashflow loop, three return streams) is correct and should be preserved. You are adding new features and UX improvements on top of it.

---

## PART 1: SENSITIVITY HEATMAP TAB

This is the headline V4 feature. Add a new tab called **"Sensitivity"** (or "What-If").

### 1.1 Concept
A 2D heatmap/matrix where the user picks any two input variables for the X and Y axes, and every cell shows the resulting Blended IRR (color-coded). The user's current deal sits at the center of the grid, highlighted.

### 1.2 Axis variable picker
Let the user choose from these pairs (default: Equity Spread vs PO Premium):

| Variable | Display name | Unit | Suggested range/step |
|----------|-------------|------|---------------------|
| `spreadBps` | Equity Spread over SOFR | bps | ±200bps around current, step 25 |
| PO premium % | PO Premium over remaining balance | % | -10% to +30%, step 5% |
| `debtPct` | Leverage (Debt %) | % | 50–85%, step 5% |
| `taxRate` | Investor Tax Rate | % | 20–40%, step 2% |
| `sofrRate` | SOFR Rate | % | 2–7%, step 0.5% |
| `vesselPrice` | Vessel Price | $M | ±30% around current, step 10% |
| `exerciseYear` | PO Exercise Year | yr | range within PO window, step 1 |
| `specialDeprPct` | Special Depreciation | % | 0 to flag max, step 5% |

### 1.3 PO Premium — new derived variable
V3 models PO as an absolute dollar price. V4 should ALSO express PO as a **premium (or discount) over remaining financing balance** at the exercise year:

```
PO Premium % = (PO Price - Remaining Balance at Exercise Year) / Remaining Balance × 100
```

This is how the market actually quotes it. The user should be able to toggle between "PO as absolute $M" and "PO as % premium over balance." The sensitivity heatmap should use the % premium as one of the axis options.

### 1.4 Heatmap rendering
- Grid of ~8–12 steps on each axis (so roughly 64–144 cells)
- Each cell = run the full IRR calculation with those two variables changed, everything else held at current deal inputs
- Color scale: deep red (IRR < 0%) → orange (0–3%) → yellow (3–6%) → green (6–10%) → deep green (>10%). Use a continuous gradient, not discrete buckets
- The cell matching the user's CURRENT inputs should have a white border or crosshair marker
- Hovering a cell shows a tooltip: "Spread: 300bps, PO Premium: +15% → Blended IRR: 8.42%, Equity IRR: 5.31%, MoIC: 1.47x"
- Clicking a cell should UPDATE the main deal inputs to those values (so the user can then flip to other tabs and see the full cashflow detail for that scenario)

### 1.5 Performance
Running the IRR solver 100+ times on every slider change will lag. Solutions:
- Debounce axis range changes (200ms)
- Use `requestAnimationFrame` or `setTimeout` chunking to avoid blocking the UI
- The IRR solver is already fast (Newton-Raphson + bisection) — profile before over-optimizing
- Consider a web worker if it's still slow, but try without first

---

## PART 2: NEW FEATURE — SCENARIO COMPARISON

### 2.1 Save & compare scenarios
Let the user **snapshot** their current deal inputs as a named scenario (e.g. "Base case", "Aggressive PO", "Low leverage"). Store up to 5 scenarios in React state.

### 2.2 Comparison table
A side-by-side table showing key outputs for each saved scenario:

| Metric | Base Case | Aggressive PO | Low Leverage |
|--------|----------|---------------|-------------|
| Equity Deployed | $8.82M | $8.82M | $12.74M |
| Blended IRR | 7.42% | 9.18% | 6.03% |
| Equity IRR (no tax) | 5.31% | 6.92% | 4.88% |
| Stream ① (Hire) | $2.1M | $2.1M | $3.4M |
| Stream ② (Tax) | $1.8M | $1.8M | $1.8M |
| Stream ③ (Residual) | $0.4M | $1.2M | $0.4M |
| MoIC | 1.47x | 1.58x | 1.44x |
| vs UST (bps) | +317 | +493 | +178 |

### 2.3 Highlight deltas
Show the difference from the "Base case" scenario in each cell (e.g. "+176bps" in green, "−139bps" in red).

---

## PART 3: NEW FEATURE — BREAKEVEN ANALYSIS

### 3.1 Auto-calculate breakeven points
Below the sensitivity heatmap (or as a separate card), show:

- **Breakeven charter spread:** The minimum equity spread (bps over SOFR) at which Blended IRR = 0%. Use bisection on the spread variable.
- **Breakeven PO premium:** The minimum PO premium at which IRR = 0%.
- **Breakeven vs Treasury:** The spread/PO at which JOLCO IRR = Treasury yield (i.e., the deal just matches risk-free). This is the "why bother?" line.
- **Tax shield dependency:** What % of total IRR comes from Stream ②? Show: "X% of your return is tax-dependent." If >60%, flag it in amber with a note about tax capacity risk.

### 3.2 Display
Show these as a row of cards with clean numbers. Example:
```
Breakeven Spread: 142 bps    |    Breakeven PO: -8.3%    |    Match UST at: 195 bps    |    Tax dependency: 54%
```

---

## PART 4: LABELING & UX IMPROVEMENTS

### 4.1 Rename hire components
Everywhere "Fixed Hire" appears → **"Scheduled Amortization (rate-insensitive)"**
Everywhere "Variable Hire" appears → **"Financing Return (rate-sensitive, tied to outstanding balance)"**

### 4.2 Rename Stream ① label
Change "Interest earned on the equity portion of outstanding balance" →
**"Equity return embedded in charter hire, net of brokerage, allocated to TK investors"**

### 4.3 Add stream type subtitles
In the three-stream summary cards at the top of Deal Inputs:
- Stream ① subtitle: "Cash yield from charter operations (pre-tax)"
- Stream ② subtitle: "Tax arbitrage — depends on investor's taxable income capacity"
- Stream ③ subtitle: "Terminal event — PO exercise at lease end"

### 4.4 Show BBC commission tax offset
In the expandable year-by-year detail rows, add a line:
```
BBC commission tax offset: +$XX,XXX (= BBC comm × tax rate — partially recovered via SPC deduction)
```

### 4.5 Tax capacity caveat
Below the Blended IRR display (both in Deal Inputs summary and Equity Cashflows tab), add small text:
> "Assumes investor has sufficient other taxable income to absorb full depreciation losses each year. If tax capacity is limited, actual returns will be lower."

### 4.6 Separate economics from tax visually
In the IRR summary area, always show TWO IRR numbers side by side:
- **Pre-tax Equity IRR** (Streams ① + ③ only) — labeled "Charter Economics"
- **Blended IRR** (all three streams) — labeled "Including Tax Shield"
Make it impossible to miss the distinction.

---

## PART 5: CHARTING IMPROVEMENTS

### 5.1 IRR waterfall / tornado chart
Add a horizontal tornado chart showing IRR sensitivity to each input variable independently. For each variable, show:
- What happens to Blended IRR if this variable moves ±1 standard unit (e.g., ±100bps for spread, ±$2M for vessel price, ±5% for debt ratio)
- Sort by absolute impact (biggest mover at top)
- This tells the user instantly: "charter spread matters most, vessel price matters least" (or whatever the actual ranking is for their deal)

### 5.2 Cumulative equity cashflow chart
A simple line chart showing cumulative equity cashflow over the lease term (already available in the data as `cumulativeEquityCF`). Mark the payback year (where cumulative crosses zero). Show two lines: with tax shield and without.

---

## PART 6: TECHNICAL REQUIREMENTS

### 6.1 Stack
- Single-file React component (keep the V3 architecture)
- No external component libraries (no MUI, no Ant Design) — continue with inline styles, Tokyo Night theme
- For the heatmap: render with a `<canvas>` element or SVG grid. Do NOT use a charting library — keep it self-contained
- For the tornado chart and cumulative CF chart: use `<canvas>` or simple SVG. If you must use a library, only Recharts (it's already available)

### 6.2 Theme
Continue the Tokyo Night dark theme from V3:
- Background: `#16161e`, `#1a1b26`, `#24283b`
- Text: `#c0caf5`, `#a9b1d6`, `#565f89`
- Accents: `#9ece6a` (green/positive), `#f7768e` (red/negative), `#7aa2f7` (blue/info), `#bb9af7` (purple/tax), `#e0af68` (amber/residual)
- Font: JetBrains Mono for numbers, Inter for labels

### 6.3 Tab structure (V4)
```
Deal Inputs | Sensitivity | Scenarios | Depreciation | Equity Cashflows | vs Treasury
```
("Sensitivity" and "Scenarios" are new. The rest carry over from V3.)

### 6.4 Mobile
Not a priority. This is a desktop tool used on trading floor monitors. Don't break the layout on mobile, but don't optimize for it either.

### 6.5 State management
All state stays in React `useState` + `useMemo`. No Redux, no context providers, no external state. Scenarios can be stored as an array of frozen input snapshots in state.

---

## PART 7: WHAT SUCCESS LOOKS LIKE

A shipbroker sitting with a Japanese investor should be able to:

1. Plug in a real deal's numbers in 60 seconds
2. Drag the charter spread slider and watch the heatmap update in real-time
3. See instantly: "if we push the PO premium from 5% to 15%, IRR goes from 6.8% to 9.2%"
4. Save "conservative" and "aggressive" scenarios, show them side by side
5. See that 54% of the return is tax-dependent and have a conversation about that
6. See the breakeven spread — "below 142bps this deal doesn't work"
7. Show the tornado chart to identify which negotiation lever has the most impact
8. Click any cell in the heatmap to load that scenario into the full cashflow model
9. Clearly distinguish "this is what the charter earns you" from "this is what the tax structure earns you"

The tool should feel like a Bloomberg terminal panel, not a textbook exercise.

---

## APPENDIX: V3 FINANCIAL MODEL REFERENCE

Preserve this logic exactly (with the bug fixes above). Do not rewrite the financial engine.

```
YEAR 0:
  equityCF[0] = -(equity + saleCommCost)

EACH YEAR 1..exerciseYear:
  CASH IN:
    fixedHire = VP / amortYrs                              (scheduled amortization)
    variableHire = outDebt × bankRate + outEquity × eqRate (financing return)
    bbcComm = totalHire × bbcCommission%
    netHire = totalHire - bbcComm

  CASH OUT TO BANK:
    bankPrincipal = annualPrincipal × debtPct%
    bankInterest  = outDebt × bankRate

  NET TO EQUITY:
    equityPrincipal = annualPrincipal × equityPct%         (return OF capital)
    hireSpread = outEquity × eqRate                        (return ON capital = Stream ①)

  TAX:
    spcTaxablePL = netHire - depreciation - bankInterest
    taxShield = -spcTaxablePL × taxRate%                   (Stream ②)

  RESIDUAL (exit year only):
    grossResidual = PO price - remaining debt
    capGainTax = max(0, PO - bookValue) × taxRate%
    residualToEquity = grossResidual - capGainTax          (Stream ③)

  EQUITY CF = equityPrincipal + hireSpread - bbcComm + taxShield + residual

IRR:
  Blended IRR = IRR(equityCF)        ← all 3 streams
  Equity IRR  = IRR(equityCF_noTax)  ← streams ① + ③ only
```

Bank rate = (jpyBaseRate + bankSpread) / 100 + swapCost / 10000
Equity rate = (sofrRate + equitySpread) / 100

Depreciation: 200% Declining Balance → Straight-Line switch (post-FY2012 MOF).
Vessel useful lives per MOF Beppyō 1 (15 types × 2 flag categories).
Special depreciation: Year 1 only, for MLIT-certified advanced vessels.
