# JOLCO V4 Extended Features Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add 8 high-value features to `jolco-v3.jsx`: URL share link, investment memo export, Monte Carlo simulation, deal optimizer, tax law shock panel, CSV cashflow export, IRR attribution waterfall chart, and JPY/USD FX sensitivity axis.

**Architecture:** All features are additions to the single-file `jolco-v3.jsx` (currently 2103 lines). New pure-function helpers are inserted before the `export default function JOLCOv3()` block (~line 1067). New state variables are added inside `JOLCOv3()`. New tabs are added to the tab bar string and rendered in the existing `tab === "..."` conditional chain. Build via `npx esbuild entry.jsx --bundle --outfile=jolco-bundle.js --format=iife`.

**Tech Stack:** React 18 (CDN, no JSX transpile beyond esbuild), HTML5 Canvas, browser APIs (URL, Blob, window.print). No new npm dependencies.

**Context — key existing APIs to use:**
- `computeDealOutputs(inp)` — pure fn, inp is a flat object `{ vesselPrice, debtPct, amortYrs, sofrRate, spreadBps, jpyBaseRate, bankSpreadBps, swapCostBps, saleCommission, bbcCommission, taxRate, capGainsTaxRate, foreignInterestTaxPct, usefulLife, specialDeprPct, treasuryYield, effectiveExerciseYear, poPriceMil, fxFactor? }`. Returns `{ blendedIRR, equityIRR, years[], equityCF[], equityCF_noTax[], totalStream1, totalStream2, totalStream3, totalEquityDeployed, jolcoProfit, spread, poPremiumPct, remainingBalAtExercise, VP, ... }`.
- `solveIRR(cf[])` — Newton-Raphson + bisection; returns IRR as decimal or null.
- `bisectBreakeven(baseInputs, varKey, target)` — already handles "spreadBps" and "poPremiumPct".
- `irrToColor(irr)` — returns CSS color string.
- `$d(n, d)`, `pct(n)`, `$(n)` — formatting helpers.
- `F` — font string `"'JetBrains Mono', monospace"`.
- `C` — card style object (defined inside JOLCOv3, must be re-declared in standalone components or passed as prop).

**Build command:** `npx esbuild entry.jsx --bundle --outfile=jolco-bundle.js --format=iife`

**Deploy:** `git add jolco-v3.jsx jolco-bundle.js && git commit -m "..." && git push`

---

## Chunk 1: Infrastructure + Quick Wins (Tasks 1, 5, 6, 7)

These four tasks require no new tabs, minimal state, and are the fastest to validate.

---

### Task 1: FX Factor in computeDealOutputs + JPY/USD State

**Purpose:** Extend `computeDealOutputs` with an optional `fxFactor` parameter (defaults to 1) that scales the tax shield stream. This enables the FX sensitivity axis in Task 8 and also exposes the sensitivity in the tornado chart automatically.

**Files:**
- Modify: `jolco-v3.jsx` (computeDealOutputs destructuring + tax shield line + R useMemo call)

- [ ] **Step 1: Add `fxFactor = 1` to the computeDealOutputs internal destructure**

The function signature is `function computeDealOutputs(inp)`. The destructuring is INSIDE the function body (lines 119–124). Find this exact block:
```jsx
  const {
    vesselPrice, debtPct, amortYrs, sofrRate, spreadBps,
    jpyBaseRate, bankSpreadBps, swapCostBps, saleCommission, bbcCommission,
    taxRate, capGainsTaxRate, foreignInterestTaxPct, usefulLife, specialDeprPct,
    treasuryYield, effectiveExerciseYear, poPriceMil,
  } = inp;
```

Replace with:
```jsx
  const {
    vesselPrice, debtPct, amortYrs, sofrRate, spreadBps,
    jpyBaseRate, bankSpreadBps, swapCostBps, saleCommission, bbcCommission,
    taxRate, capGainsTaxRate, foreignInterestTaxPct, usefulLife, specialDeprPct,
    treasuryYield, effectiveExerciseYear, poPriceMil,
    fxFactor = 1,
  } = inp;
```

- [ ] **Step 2: Apply fxFactor to tax shield line**

Find:
```jsx
    const taxShieldThisYear = -spcTaxablePL * (taxRate / 100);
```

Replace with:
```jsx
    const taxShieldThisYear = -spcTaxablePL * (taxRate / 100) * fxFactor;
```

- [ ] **Step 3: Add `JPY_USD_BASE` constant at module scope and `jpyUsdRate` state inside component**

At module scope (after the `const F = ...` line, before `export default function JOLCOv3()`), insert:
```jsx
const JPY_USD_BASE = 150; // Reference JPY/USD rate for FX factor normalization
```

Then, inside `JOLCOv3()`, after `const [savingScenario, setSavingScenario] = useState(false);`, insert:
```jsx
  const [jpyUsdRate, setJpyUsdRate] = useState(150);
```

- [ ] **Step 4: Pass fxFactor into R = useMemo**

Find the `computeDealOutputs({` call inside `R = useMemo(...)`:
```jsx
    return computeDealOutputs({
      vesselPrice, debtPct, amortYrs, sofrRate, spreadBps,
      jpyBaseRate, bankSpreadBps, swapCostBps, saleCommission, bbcCommission,
      taxRate, capGainsTaxRate, foreignInterestTaxPct, usefulLife, specialDeprPct,
      treasuryYield, effectiveExerciseYear, poPriceMil,
    });
```

Replace with:
```jsx
    return computeDealOutputs({
      vesselPrice, debtPct, amortYrs, sofrRate, spreadBps,
      jpyBaseRate, bankSpreadBps, swapCostBps, saleCommission, bbcCommission,
      taxRate, capGainsTaxRate, foreignInterestTaxPct, usefulLife, specialDeprPct,
      treasuryYield, effectiveExerciseYear, poPriceMil,
      fxFactor: JPY_USD_BASE / jpyUsdRate,
    });
```

- [ ] **Step 5: Add `jpyUsdRate` to R useMemo deps array**

Find the deps array:
```jsx
  }, [vesselPrice, debtPct, amortYrs, sofrRate, spreadBps, jpyBaseRate, bankSpreadBps,
      swapCostBps, saleCommission, bbcCommission, taxRate, capGainsTaxRate,
      foreignInterestTaxPct, usefulLife, specialDeprPct, treasuryYield, flagId,
      effectiveExerciseYear, poSchedule, vesselAgeYrs]);
```

Add `jpyUsdRate` to it:
```jsx
  }, [vesselPrice, debtPct, amortYrs, sofrRate, spreadBps, jpyBaseRate, bankSpreadBps,
      swapCostBps, saleCommission, bbcCommission, taxRate, capGainsTaxRate,
      foreignInterestTaxPct, usefulLife, specialDeprPct, treasuryYield, flagId,
      effectiveExerciseYear, poSchedule, vesselAgeYrs, jpyUsdRate]);
```

- [ ] **Step 6: Build and verify**

```bash
cd "/Users/sriniwasghate/Programs/JOLCO Calculator V4"
npx esbuild entry.jsx --bundle --outfile=jolco-bundle.js --format=iife
```

Expected: no errors, `jolco-bundle.js 1.2mb`. Open in browser — deal should still work exactly as before (jpyUsdRate=150 = JPY_USD_BASE so fxFactor=1).

- [ ] **Step 7: Commit**

```bash
git add jolco-v3.jsx jolco-bundle.js
git commit -m "feat: add fxFactor param to computeDealOutputs for JPY/USD sensitivity"
```

---

### Task 2: CSV Cashflow Export

**Purpose:** "Download CSV" button on the Equity Cashflows tab. Downloads the full year-by-year cashflow table as a spreadsheet.

**Files:**
- Modify: `jolco-v3.jsx` (add `exportCashflowCSV` helper before JOLCOv3, add button in `tab === "cf"` section)

- [ ] **Step 1: Add `exportCashflowCSV` function**

Insert before `export default function JOLCOv3()`:

```jsx
function exportCashflowCSV(years, R, vesselPrice, spreadBps) {
  const headers = [
    "Year", "Fixed Hire ($)", "Variable Hire ($)", "Total Hire ($)", "BBC Comm ($)",
    "Net Hire ($)", "Bank Principal ($)", "Bank Interest ($)", "Equity Principal ($)",
    "Hire Spread ($)", "Depreciation ($)", "SPC Taxable P&L ($)", "Tax Shield ($)",
    "Residual Gain ($)", "Net CF ($)", "Net CF no Tax ($)", "Cumulative CF ($)",
    "Outstanding Debt ($)", "Outstanding Equity ($)", "Book Value ($)"
  ];
  const rows = years.map(y => [
    y.yr,
    y.fixedHire.toFixed(0), y.variableHire.toFixed(0), y.totalHire.toFixed(0),
    y.bbcCommCost.toFixed(0), y.netHire.toFixed(0),
    y.bankPrincipal.toFixed(0), y.bankInterest.toFixed(0),
    y.equityPrincipalReturn.toFixed(0), y.hireSpread.toFixed(0),
    y.dep.toFixed(0), y.spcTaxablePL.toFixed(0), y.taxShieldThisYear.toFixed(0),
    y.residualGain.toFixed(0), y.netCF.toFixed(0), y.netCF_noTax.toFixed(0),
    y.cumulativeEquityCF.toFixed(0), y.outstandingDebt.toFixed(0),
    y.outstandingEquity.toFixed(0), y.bookVal.toFixed(0),
  ]);
  const meta = [
    ["JOLCO IRR Calculator - Equity Cashflow Export"],
    ["Vessel Price ($M)", (R.VP / 1e6).toFixed(2)],
    ["Equity Spread (bps)", spreadBps],
    ["Blended IRR", R.blendedIRR != null ? (R.blendedIRR * 100).toFixed(2) + "%" : "N/A"],
    ["Charter Economics IRR", R.equityIRR != null ? (R.equityIRR * 100).toFixed(2) + "%" : "N/A"],
    [],
  ];
  const csv = [...meta, headers, ...rows]
    .map(row => row.map(v => `"${String(v).replace(/"/g, '""')}"`).join(","))
    .join("\n");
  const blob = new Blob([csv], { type: "text/csv" });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = `jolco-cashflows-${new Date().toISOString().slice(0, 10)}.csv`;
  a.click();
  URL.revokeObjectURL(url);
}
```

- [ ] **Step 2: Add download button to the Equity Cashflows tab**

In the `tab === "cf"` section, find this specific anchor (the tax capacity caveat div that comes right before the tornado/cumulative CF charts section):
```jsx
        {/* ═══ EQUITY CASHFLOWS ═══ */}
        {tab === "cf" && (
```

Immediately after that opening line (before the first `<div style={C}>` of the cf tab), add:

```jsx
<div style={{ display: "flex", justifyContent: "flex-end", marginBottom: 10 }}>
  <button
    onClick={() => exportCashflowCSV(years, R, vesselPrice, spreadBps)}
    style={{ padding: "7px 16px", borderRadius: 6, border: "1px solid #3b4261", background: "#24283b", color: "#9ece6a", fontSize: 12, fontFamily: F, fontWeight: 600, cursor: "pointer", letterSpacing: "0.04em" }}
  >
    ↓ Download CSV
  </button>
</div>
```

- [ ] **Step 3: Build and verify**

```bash
npx esbuild entry.jsx --bundle --outfile=jolco-bundle.js --format=iife
```

Open in browser → Equity Cashflows tab → click "↓ Download CSV" → verify a `.csv` file downloads with correct columns and data.

- [ ] **Step 4: Commit**

```bash
git add jolco-v3.jsx jolco-bundle.js
git commit -m "feat: CSV cashflow export on Equity Cashflows tab"
```

---

### Task 3: IRR Attribution Waterfall Chart

**Purpose:** A horizontal bar chart on the Deal Inputs tab showing how much each of the three streams contributes to total blended IRR (in bps). Gives non-JOLCO experts an instant "what's driving returns" picture.

**Files:**
- Modify: `jolco-v3.jsx` (add `IrrWaterfallChart` component before JOLCOv3, insert in Deal Inputs tab)

- [ ] **Step 1: Add `IrrWaterfallChart` component**

Insert before `export default function JOLCOv3()`:

```jsx
function IrrWaterfallChart({ R }) {
  const canvasRef = React.useRef(null);
  const containerRef = React.useRef(null);
  const [width, setWidth] = React.useState(600);

  React.useEffect(() => {
    if (!containerRef.current) return;
    setWidth(containerRef.current.offsetWidth);
    const ro = new ResizeObserver(([e]) => setWidth(e.contentRect.width));
    ro.observe(containerRef.current);
    return () => ro.disconnect();
  }, []);

  React.useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;
    const ctx = canvas.getContext("2d");
    const W = canvas.width, H = canvas.height;
    const padL = 130, padR = 80, padT = 18, padB = 24;
    ctx.clearRect(0, 0, W, H);
    ctx.fillStyle = "#16161e";
    ctx.fillRect(0, 0, W, H);

    const baseIRR = R.blendedIRR ?? 0;
    const total = R.totalStream1 + R.totalStream2 + R.totalStream3;
    if (total <= 0 || baseIRR <= 0) {
      ctx.fillStyle = "#565f89";
      ctx.font = "11px 'Inter', sans-serif";
      ctx.textAlign = "center";
      ctx.fillText("No positive IRR to attribute", W / 2, H / 2);
      return;
    }

    // Approximate IRR contribution by stream weight
    const stream1Share = R.totalStream1 / total;
    const stream2Share = R.totalStream2 / total;
    const stream3Share = R.totalStream3 / total;
    const irrBps = baseIRR * 10000;

    const bars = [
      { label: "① Hire Spread", bps: irrBps * stream1Share, color: "#9ece6a" },
      { label: "② Tax Shield",  bps: irrBps * stream2Share, color: "#bb9af7" },
      { label: "③ Residual/PO", bps: irrBps * stream3Share, color: "#e0af68" },
      { label: "Blended IRR",   bps: irrBps,                color: "#7aa2f7", isTotal: true },
    ];

    const maxBps = Math.max(...bars.map(b => b.bps)) * 1.15;
    const xScale = (v) => padL + (v / maxBps) * (W - padL - padR);
    const barH = 22, gap = 12;
    const totalH = bars.length * (barH + gap);
    const startY = padT + (H - padT - padB - totalH) / 2;

    bars.forEach((bar, i) => {
      const y = startY + i * (barH + gap);
      const barW = Math.max(2, xScale(bar.bps) - padL);

      if (bar.isTotal) {
        ctx.fillStyle = "#292e42";
        ctx.fillRect(padL, y, W - padL - padR, barH);
      }
      ctx.fillStyle = bar.color + (bar.isTotal ? "" : "cc");
      ctx.fillRect(padL, y, barW, barH);

      ctx.fillStyle = "#a9b1d6";
      ctx.font = `${bar.isTotal ? "bold " : ""}11px 'Inter', sans-serif`;
      ctx.textAlign = "right";
      ctx.textBaseline = "middle";
      ctx.fillText(bar.label, padL - 6, y + barH / 2);

      ctx.fillStyle = bar.isTotal ? "#ffffff" : "#c0caf5";
      ctx.font = `bold 11px 'JetBrains Mono', monospace`;
      ctx.textAlign = "left";
      ctx.fillText(bar.bps.toFixed(0) + " bps", xScale(bar.bps) + 5, y + barH / 2);
    });

    // Zero line
    ctx.strokeStyle = "#3b4261";
    ctx.lineWidth = 1;
    ctx.beginPath();
    ctx.moveTo(padL, padT);
    ctx.lineTo(padL, H - padB);
    ctx.stroke();
  }, [R, width]);

  return (
    <div ref={containerRef} style={{ width: "100%" }}>
      <canvas ref={canvasRef} width={width} height={160}
        style={{ width: "100%", height: "auto", display: "block", borderRadius: 6 }} />
    </div>
  );
}
```

- [ ] **Step 2: Insert IrrWaterfallChart into Deal Inputs tab**

In the Deal Inputs tab, find the closing `</div>` of the dual-IRR card. The unique anchor is the card's last line:
```jsx
                  <div style={{ fontSize: 10, color: "#565f89", marginTop: 2 }}>Blended IRR — full deal return</div>
```
After the two closing `</div></div>` that close that dual-IRR card and its wrapper, add:

```jsx
<div style={{ gridColumn: "1 / -1", background: "#1a1b26", borderRadius: 10, padding: 16, border: "1px solid #292e42", marginBottom: 14 }}>
  <div style={{ fontSize: 11, fontWeight: 700, color: "#565f89", textTransform: "uppercase", letterSpacing: "0.06em", marginBottom: 10, fontFamily: F }}>IRR Attribution — Contribution by Stream (bps)</div>
  <IrrWaterfallChart R={R} />
</div>
```

- [ ] **Step 3: Build and verify**

```bash
npx esbuild entry.jsx --bundle --outfile=jolco-bundle.js --format=iife
```

Open in browser → Deal Inputs tab → verify horizontal bar chart shows three colored bars (green, purple, orange) and a blue total bar. Change deal inputs → bars update in real time.

- [ ] **Step 4: Commit**

```bash
git add jolco-v3.jsx jolco-bundle.js
git commit -m "feat: IRR attribution waterfall chart on Deal Inputs tab"
```

---

### Task 4: Tax Law Shock Panel

**Purpose:** An inline panel on the Deal Inputs tab where the user can simulate a change to the special depreciation rate (e.g., "what if special depr drops from 30% to 20% due to a tax reform?"). Shows ΔBlendedIRR and ΔEquityIRR in real time.

**Files:**
- Modify: `jolco-v3.jsx` (add state + panel in Deal Inputs tab)

- [ ] **Step 1: Add tax shock state variables**

After the `jpyUsdRate` state, add:

```jsx
  const [showTaxShock, setShowTaxShock] = useState(false);
  const [shockDeprPct, setShockDeprPct] = useState(20);
```

- [ ] **Step 2: Compute shock outputs**

After the `years` useMemo, add:

```jsx
  const taxShockR = useMemo(() => {
    if (!showTaxShock) return null;
    const poEntry = poSchedule.find(p => p.yr === effectiveExerciseYear);
    const poPriceMil = poEntry ? poEntry.price * 1e6 : 0;
    return computeDealOutputs({
      vesselPrice, debtPct, amortYrs, sofrRate, spreadBps,
      jpyBaseRate, bankSpreadBps, swapCostBps, saleCommission, bbcCommission,
      taxRate, capGainsTaxRate, foreignInterestTaxPct, usefulLife,
      specialDeprPct: shockDeprPct,
      treasuryYield, effectiveExerciseYear, poPriceMil,
      fxFactor: JPY_USD_BASE / jpyUsdRate,
    });
  }, [showTaxShock, shockDeprPct, vesselPrice, debtPct, amortYrs, sofrRate, spreadBps,
      jpyBaseRate, bankSpreadBps, swapCostBps, saleCommission, bbcCommission,
      taxRate, capGainsTaxRate, foreignInterestTaxPct, usefulLife, treasuryYield,
      effectiveExerciseYear, poSchedule, jpyUsdRate]);
  // Note: taxShockR is null when showTaxShock is false. Always null-guard before using.
```

- [ ] **Step 3: Add Tax Law Shock panel to Deal Inputs tab**

In the Deal Inputs tab, at the end of the `gridColumn: "1 / -1"` summary section (after the IRR attribution chart card), add:

```jsx
<div style={{ gridColumn: "1 / -1", background: "#1a1b26", borderRadius: 10, padding: 16, border: "1px solid #292e42", marginBottom: 14 }}>
  <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginBottom: showTaxShock ? 14 : 0 }}>
    <div style={{ fontSize: 11, fontWeight: 700, color: "#565f89", textTransform: "uppercase", letterSpacing: "0.06em", fontFamily: F }}>
      ⚡ Tax Law Shock Simulator
    </div>
    <button
      onClick={() => setShowTaxShock(s => !s)}
      style={{ padding: "5px 12px", borderRadius: 5, border: `1px solid ${showTaxShock ? "#f7768e" : "#3b4261"}`, background: showTaxShock ? "rgba(247,118,142,0.12)" : "#24283b", color: showTaxShock ? "#f7768e" : "#a9b1d6", fontSize: 11, fontFamily: F, cursor: "pointer", fontWeight: 600 }}
    >
      {showTaxShock ? "Hide" : "Show"}
    </button>
  </div>
  {showTaxShock && taxShockR && (
    <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr 1fr 1fr", gap: 12, alignItems: "end" }}>
      <div>
        <div style={{ fontSize: 10, color: "#565f89", textTransform: "uppercase", marginBottom: 4, fontFamily: F }}>Current Special Depr</div>
        <div style={{ fontSize: 18, fontWeight: 700, color: "#bb9af7", fontFamily: F }}>{specialDeprPct}%</div>
      </div>
      <div>
        <label style={{ display: "block", fontSize: 10, color: "#7aa2f7", textTransform: "uppercase", marginBottom: 4, fontFamily: F, fontWeight: 600 }}>Shocked Depr %</label>
        <input type="number" value={shockDeprPct} onChange={e => setShockDeprPct(parseFloat(e.target.value) || 0)}
          min={0} max={50} step={1}
          style={{ width: "100%", padding: "6px 8px", borderRadius: 5, border: "1px solid #3b4261", background: "#16161e", color: "#c0caf5", fontSize: 14, fontFamily: F }} />
      </div>
      <div>
        <div style={{ fontSize: 10, color: "#565f89", textTransform: "uppercase", marginBottom: 4, fontFamily: F }}>Shocked Blended IRR</div>
        <div style={{ fontSize: 18, fontWeight: 700, fontFamily: F, color: taxShockR.blendedIRR > 0 ? "#9ece6a" : "#f7768e" }}>
          {pct(taxShockR.blendedIRR)}
          {taxShockR.blendedIRR != null && R.blendedIRR != null && (
            <span style={{ fontSize: 11, marginLeft: 6, color: (taxShockR.blendedIRR - R.blendedIRR) >= 0 ? "#9ece6a" : "#f7768e" }}>
              ({(((taxShockR.blendedIRR - R.blendedIRR)) * 10000).toFixed(0)} bps)
            </span>
          )}
        </div>
      </div>
      <div>
        <div style={{ fontSize: 10, color: "#565f89", textTransform: "uppercase", marginBottom: 4, fontFamily: F }}>Shocked Charter IRR</div>
        <div style={{ fontSize: 18, fontWeight: 700, fontFamily: F, color: "#e0af68" }}>
          {pct(taxShockR.equityIRR)}
          {taxShockR.equityIRR != null && R.equityIRR != null && (
            <span style={{ fontSize: 11, marginLeft: 6, color: (taxShockR.equityIRR - R.equityIRR) >= 0 ? "#9ece6a" : "#f7768e" }}>
              ({(((taxShockR.equityIRR - R.equityIRR)) * 10000).toFixed(0)} bps)
            </span>
          )}
        </div>
      </div>
    </div>
  )}
</div>
```

- [ ] **Step 4: Build and verify**

```bash
npx esbuild entry.jsx --bundle --outfile=jolco-bundle.js --format=iife
```

Open browser → Deal Inputs tab → click "Show" on Tax Law Shock → change shocked depr % → verify blended IRR delta updates in real time.

- [ ] **Step 5: Commit**

```bash
git add jolco-v3.jsx jolco-bundle.js
git commit -m "feat: tax law shock simulator panel on Deal Inputs tab"
```

---

## Chunk 2: New Tabs + Complex Features (Tasks 5, 6, 7, 8)

---

### Task 5: URL Share Link

**Purpose:** Encode all deal inputs into a URL hash so users can share a fully-configured deal via a link. On page load, restore state from the hash.

**Files:**
- Modify: `jolco-v3.jsx` (encode/decode helpers, useEffect on mount, "Copy Link" button in header)

- [ ] **Step 1: Add `encodeState` and `decodeState` helpers**

Insert before `export default function JOLCOv3()`:

```jsx
const STATE_VERSION = 1;
function encodeState(s) {
  try { return "#" + btoa(JSON.stringify({ v: STATE_VERSION, ...s })); }
  catch (_) { return ""; }
}
function decodeState(hash) {
  try {
    if (!hash || hash.length < 2) return null;
    const obj = JSON.parse(atob(hash.slice(1)));
    if (obj.v !== STATE_VERSION) return null;
    return obj;
  } catch (_) { return null; }
}
```

- [ ] **Step 2: Add `copyLinkStatus` state**

After the `showTaxShock` / `shockDeprPct` state, add:

```jsx
  const [copyLinkStatus, setCopyLinkStatus] = useState("idle"); // "idle" | "copied"
```

- [ ] **Step 3: Add URL restore on mount**

After the `copyLinkStatus` state, add a one-time useEffect:

```jsx
  React.useEffect(() => {
    const s = decodeState(window.location.hash);
    if (!s) return;
    if (s.vesselPrice != null) setVesselPrice(s.vesselPrice);
    if (s.debtPct != null) setDebtPct(s.debtPct);
    if (s.amortYrs != null) setAmortYrs(s.amortYrs);
    if (s.sofrRate != null) setSofrRate(s.sofrRate);
    if (s.spreadBps != null) setSpreadBps(s.spreadBps);
    if (s.jpyBaseRate != null) setJpyBaseRate(s.jpyBaseRate);
    if (s.bankSpreadBps != null) setBankSpreadBps(s.bankSpreadBps);
    if (s.swapCostBps != null) setSwapCostBps(s.swapCostBps);
    if (s.saleCommission != null) setSaleCommission(s.saleCommission);
    if (s.bbcCommission != null) setBbcCommission(s.bbcCommission);
    if (s.taxRate != null) setTaxRate(s.taxRate);
    if (s.capGainsTaxRate != null) setCapGainsTaxRate(s.capGainsTaxRate);
    if (s.foreignInterestTaxPct != null) setForeignInterestTaxPct(s.foreignInterestTaxPct);
    if (s.specialDeprPct != null) setSpecialDeprPct(s.specialDeprPct);
    if (s.treasuryYield != null) setTreasuryYield(s.treasuryYield);
    if (s.vesselTypeId != null) setVesselTypeId(s.vesselTypeId);
    if (s.flagId != null) setFlagId(s.flagId);
    if (s.vesselAgeYrs != null) setVesselAgeYrs(s.vesselAgeYrs);
    if (s.leaseTerm != null) setLeaseTerm(s.leaseTerm);
    if (s.exerciseYear != null) setExerciseYear(s.exerciseYear);
    if (s.poPremium != null) setPoPremium(s.poPremium);
    if (s.poFirstYear != null) setPoFirstYear(s.poFirstYear);
    if (s.poLastYear != null) setPoLastYear(s.poLastYear);
    if (s.jpyUsdRate != null) setJpyUsdRate(s.jpyUsdRate);
    if (s.poOverrides != null && typeof s.poOverrides === "object") setPoOverrides(s.poOverrides);
  }, []);
```

- [ ] **Step 4: Add "Copy Link" button to the header**

In the header `<div>` (the `display: "flex", alignItems: "center", justifyContent: "space-between"` div), find the right-side block with "Created By Sriniwas Ghate". After that block, add:

```jsx
<button
  onClick={() => {
    const hash = encodeState({
      vesselPrice, debtPct, amortYrs, sofrRate, spreadBps,
      jpyBaseRate, bankSpreadBps, swapCostBps, saleCommission, bbcCommission,
      taxRate, capGainsTaxRate, foreignInterestTaxPct, specialDeprPct,
      treasuryYield, vesselTypeId, flagId, vesselAgeYrs, leaseTerm,
      exerciseYear, poPremium, poFirstYear, poLastYear, jpyUsdRate,
      poOverrides,
    });
    const url = window.location.origin + window.location.pathname + hash;
    navigator.clipboard.writeText(url).then(() => {
      setCopyLinkStatus("copied");
      setTimeout(() => setCopyLinkStatus("idle"), 2000);
    });
  }}
  style={{ padding: "7px 14px", borderRadius: 6, border: "1px solid #3b4261", background: copyLinkStatus === "copied" ? "rgba(158,206,106,0.15)" : "#24283b", color: copyLinkStatus === "copied" ? "#9ece6a" : "#a9b1d6", fontSize: 11, fontFamily: F, fontWeight: 600, cursor: "pointer", letterSpacing: "0.04em", whiteSpace: "nowrap" }}
>
  {copyLinkStatus === "copied" ? "✓ Copied!" : "🔗 Copy Link"}
</button>
```

- [ ] **Step 5: Build and verify**

```bash
npx esbuild entry.jsx --bundle --outfile=jolco-bundle.js --format=iife
```

Open in browser → change some inputs → click "🔗 Copy Link" → paste the URL in a new tab → verify all inputs are restored.

- [ ] **Step 6: Commit**

```bash
git add jolco-v3.jsx jolco-bundle.js
git commit -m "feat: URL share link — encode/restore all deal inputs via URL hash"
```

---

### Task 6: Export Investment Memo (HTML Print)

**Purpose:** One-click "Export Memo" that opens a new browser tab with a print-friendly investment memo showing all key metrics, stream breakdown, cashflow table, and deal economics. User can Cmd+P to get a PDF.

**Files:**
- Modify: `jolco-v3.jsx` (add `buildMemoHTML` helper, add "Export Memo" button)

- [ ] **Step 1: Add `buildMemoHTML` function**

Insert before `export default function JOLCOv3()`:

```jsx
function buildMemoHTML(R, inputs) {
  const { vesselPrice, spreadBps, debtPct, amortYrs, exerciseYear, specialDeprPct } = inputs;
  const pctFmt = (v) => v != null ? (v * 100).toFixed(2) + "%" : "—";
  const moneyFmt = (v) => v != null ? "$" + (v / 1e6).toFixed(2) + "M" : "—";
  const bpsFmt = (v) => v != null ? (v * 10000).toFixed(0) + " bps" : "—";

  const rows = R.years.map(y => `
    <tr>
      <td>${y.yr}</td>
      <td>${moneyFmt(y.totalHire)}</td>
      <td>${moneyFmt(y.hireSpread)}</td>
      <td>${moneyFmt(y.taxShieldThisYear)}</td>
      <td>${moneyFmt(y.residualGain || 0)}</td>
      <td>${moneyFmt(y.netCF)}</td>
      <td style="color:${y.cumulativeEquityCF >= 0 ? '#2d7a2d' : '#a83232'}">${moneyFmt(y.cumulativeEquityCF)}</td>
    </tr>`).join("");

  return `<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>JOLCO Investment Memo — ${new Date().toLocaleDateString()}</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: 'Helvetica Neue', Arial, sans-serif; font-size: 12px; color: #1a1a2e; background: #fff; padding: 32px; max-width: 960px; margin: 0 auto; }
    h1 { font-size: 22px; font-weight: 700; margin-bottom: 4px; }
    h2 { font-size: 14px; font-weight: 600; margin: 20px 0 8px; color: #333; border-bottom: 1px solid #ddd; padding-bottom: 4px; }
    .subtitle { font-size: 11px; color: #666; margin-bottom: 24px; }
    .kpi-grid { display: grid; grid-template-columns: repeat(4, 1fr); gap: 12px; margin-bottom: 20px; }
    .kpi { border: 1px solid #ddd; border-radius: 6px; padding: 12px; }
    .kpi-label { font-size: 10px; text-transform: uppercase; letter-spacing: 0.05em; color: #666; }
    .kpi-val { font-size: 22px; font-weight: 700; margin-top: 4px; }
    .stream-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 12px; margin-bottom: 20px; }
    .stream { border: 1px solid #ddd; border-radius: 6px; padding: 12px; text-align: center; }
    table { width: 100%; border-collapse: collapse; font-size: 11px; }
    th { background: #f0f0f0; padding: 6px 8px; text-align: right; font-weight: 600; font-size: 10px; text-transform: uppercase; }
    th:first-child { text-align: center; }
    td { padding: 5px 8px; text-align: right; border-bottom: 1px solid #f0f0f0; }
    td:first-child { text-align: center; font-weight: 600; }
    .green { color: #2d7a2d; } .red { color: #a83232; } .blue { color: #1a4fa8; }
    @media print { body { padding: 16px; } .no-print { display: none; } }
  </style>
</head>
<body>
  <div class="no-print" style="margin-bottom:20px;padding:10px;background:#f5f5f5;border-radius:4px;font-size:11px;color:#666">
    Press <strong>Cmd+P</strong> (Mac) or <strong>Ctrl+P</strong> (Windows) to save as PDF.
  </div>
  <h1>JOLCO TK Equity — Investment Memo</h1>
  <div class="subtitle">
    Generated ${new Date().toLocaleString()} · Vessel $${(vesselPrice).toFixed(1)}M · ${debtPct}% Leverage · ${amortYrs}yr amort · ${spreadBps}bps equity spread · Exit Yr ${exerciseYear}
  </div>

  <h2>Key Performance Indicators</h2>
  <div class="kpi-grid">
    <div class="kpi">
      <div class="kpi-label">Blended IRR (All Streams)</div>
      <div class="kpi-val ${R.blendedIRR > 0 ? 'green' : 'red'}">${pctFmt(R.blendedIRR)}</div>
    </div>
    <div class="kpi">
      <div class="kpi-label">Charter Economics IRR</div>
      <div class="kpi-val" style="color:#996600">${pctFmt(R.equityIRR)}</div>
    </div>
    <div class="kpi">
      <div class="kpi-label">vs US Treasury (post-tax)</div>
      <div class="kpi-val ${R.spread > 0 ? 'green' : 'red'}">${bpsFmt(R.spread)}</div>
    </div>
    <div class="kpi">
      <div class="kpi-label">Equity Deployed</div>
      <div class="kpi-val blue">${moneyFmt(R.totalEquityDeployed)}</div>
    </div>
    <div class="kpi">
      <div class="kpi-label">Total Profit</div>
      <div class="kpi-val ${R.jolcoProfit > 0 ? 'green' : 'red'}">${moneyFmt(R.jolcoProfit)}</div>
    </div>
    <div class="kpi">
      <div class="kpi-label">MoIC</div>
      <div class="kpi-val">${((R.totalEquityDeployed + R.jolcoProfit) / R.totalEquityDeployed).toFixed(2)}×</div>
    </div>
    <div class="kpi">
      <div class="kpi-label">Special Depreciation</div>
      <div class="kpi-val">${specialDeprPct}%</div>
    </div>
    <div class="kpi">
      <div class="kpi-label">Bank Leverage</div>
      <div class="kpi-val">${debtPct}%</div>
    </div>
  </div>

  <h2>Return Streams</h2>
  <div class="stream-grid">
    <div class="stream">
      <div class="kpi-label">① Hire Spread (pre-tax)</div>
      <div class="kpi-val green">${moneyFmt(R.totalStream1)}</div>
      <div style="font-size:10px;color:#666;margin-top:4px">Charter cash yield after debt service</div>
    </div>
    <div class="stream">
      <div class="kpi-label">② Tax Shield</div>
      <div class="kpi-val" style="color:#6a2d9a">${moneyFmt(R.totalStream2)}</div>
      <div style="font-size:10px;color:#666;margin-top:4px">NPV of depreciation tax savings</div>
    </div>
    <div class="stream">
      <div class="kpi-label">③ Residual / PO</div>
      <div class="kpi-val" style="color:#a06000">${moneyFmt(R.totalStream3)}</div>
      <div style="font-size:10px;color:#666;margin-top:4px">PO exercise net of debt &amp; cap gains tax</div>
    </div>
  </div>

  <h2>Year-by-Year Equity Cashflows</h2>
  <table>
    <thead>
      <tr>
        <th>Yr</th><th>Total Hire</th><th>Hire Spread</th><th>Tax Shield</th>
        <th>Residual</th><th>Net CF</th><th>Cumulative CF</th>
      </tr>
    </thead>
    <tbody>${rows}</tbody>
  </table>
</body>
</html>`;
}
```

- [ ] **Step 2: Add "Export Memo" button next to "Copy Link" in the header**

Right after the "Copy Link" button (Task 5, Step 4), add:

```jsx
<button
  onClick={() => {
    const html = buildMemoHTML(R, {
      vesselPrice, spreadBps, debtPct, amortYrs,
      exerciseYear: effectiveExerciseYear, specialDeprPct,
    });
    const w = window.open("", "_blank");
    w.document.write(html);
    w.document.close();
  }}
  style={{ padding: "7px 14px", borderRadius: 6, border: "1px solid #3b4261", background: "#24283b", color: "#a9b1d6", fontSize: 11, fontFamily: F, fontWeight: 600, cursor: "pointer", letterSpacing: "0.04em", whiteSpace: "nowrap" }}
>
  📄 Export Memo
</button>
```

- [ ] **Step 3: Build and verify**

```bash
npx esbuild entry.jsx --bundle --outfile=jolco-bundle.js --format=iife
```

Open browser → click "📄 Export Memo" → new tab opens with clean white investment memo → verify KPIs, streams, and cashflow table are populated correctly → Cmd+P looks print-ready.

- [ ] **Step 4: Commit**

```bash
git add jolco-v3.jsx jolco-bundle.js
git commit -m "feat: investment memo HTML export with print-to-PDF support"
```

---

### Task 7: Deal Optimizer Panel (Sensitivity Tab)

**Purpose:** In the Sensitivity tab, add a panel that runs bisection to answer: "What is the maximum PO premium I can offer while keeping blended IRR ≥ X%?" and "What leverage maximizes blended IRR?" Shows the answer and an "Apply" button.

**Files:**
- Modify: `jolco-v3.jsx` (add optimizer state, add optimizer panel inside `tab === "sensitivity"`)

- [ ] **Step 1: Add optimizer state**

After `copyLinkStatus` state, add:

```jsx
  const [optTarget, setOptTarget] = useState(6.0);  // target IRR %
  const [optResult, setOptResult] = useState(null);  // { maxPoPct, maxSpread }
```

- [ ] **Step 2: Add `runOptimizer` function (inside JOLCOv3 component, uses closure)**

After the `taxShockR` useMemo, add:

```jsx
  const runOptimizer = React.useCallback(() => {
    const baseInputs = {
      vesselPrice, debtPct, amortYrs, sofrRate, spreadBps,
      jpyBaseRate, bankSpreadBps, swapCostBps, saleCommission, bbcCommission,
      taxRate, capGainsTaxRate, foreignInterestTaxPct, usefulLife, specialDeprPct,
      treasuryYield, effectiveExerciseYear, poPriceMil: R.poPriceMil,
      fxFactor: JPY_USD_BASE / jpyUsdRate,
    };
    const target = optTarget / 100;
    const maxPoPct = bisectBreakeven(baseInputs, "poPremiumPct", target);
    const maxSpread = bisectBreakeven(baseInputs, "spreadBps", target);
    setOptResult({ maxPoPct, maxSpread, target });
  }, [vesselPrice, debtPct, amortYrs, sofrRate, spreadBps, jpyBaseRate, bankSpreadBps,
      swapCostBps, saleCommission, bbcCommission, taxRate, capGainsTaxRate,
      foreignInterestTaxPct, usefulLife, specialDeprPct, treasuryYield,
      effectiveExerciseYear, R.poPriceMil, optTarget, jpyUsdRate]);
```

- [ ] **Step 3: Add Optimizer panel in Sensitivity tab**

In the `tab === "sensitivity"` JSX block, after the `<SensitivityTab ... />` component, add:

```jsx
<div style={{ background: "#1a1b26", borderRadius: 10, padding: 18, border: "1px solid #292e42", marginTop: 14 }}>
  <div style={{ fontSize: 11, fontWeight: 700, color: "#565f89", textTransform: "uppercase", letterSpacing: "0.06em", marginBottom: 12, fontFamily: F }}>
    ⚙ Deal Optimizer — find inputs for a target IRR
  </div>
  <div style={{ display: "flex", gap: 12, alignItems: "flex-end", flexWrap: "wrap" }}>
    <div>
      <label style={{ display: "block", fontSize: 10, color: "#7aa2f7", textTransform: "uppercase", marginBottom: 4, fontFamily: F, fontWeight: 600 }}>Target IRR (%)</label>
      <input type="number" value={optTarget} onChange={e => setOptTarget(parseFloat(e.target.value) || 0)}
        min={0} max={20} step={0.5}
        style={{ width: 100, padding: "7px 8px", borderRadius: 5, border: "1px solid #3b4261", background: "#16161e", color: "#c0caf5", fontSize: 14, fontFamily: F }} />
    </div>
    <button onClick={runOptimizer}
      style={{ padding: "8px 18px", borderRadius: 6, border: "1px solid #7aa2f7", background: "rgba(122,162,247,0.15)", color: "#7aa2f7", fontSize: 12, fontFamily: F, fontWeight: 700, cursor: "pointer" }}>
      Find Limits
    </button>
    {optResult && (
      <>
        <div style={{ background: "#16161e", borderRadius: 8, padding: "10px 16px", border: "1px solid #292e42", minWidth: 180 }}>
          <div style={{ fontSize: 10, color: "#565f89", textTransform: "uppercase", fontFamily: F }}>Max PO Premium</div>
          <div style={{ fontSize: 18, fontWeight: 700, color: "#e0af68", fontFamily: F }}>
            {typeof optResult.maxPoPct === "number" ? optResult.maxPoPct.toFixed(1) + "%" : String(optResult.maxPoPct)}
          </div>
          <div style={{ fontSize: 9, color: "#565f89", marginTop: 2 }}>highest PO premium keeping IRR ≥ {optTarget}%</div>
          {typeof optResult.maxPoPct === "number" && (
            <button onClick={() => {
              const remBal = Math.max(0, vesselPrice * 1e6 - (vesselPrice * 1e6 / amortYrs) * effectiveExerciseYear);
              const newPoPriceMil = remBal * (1 + optResult.maxPoPct / 100);
              setPoPremium((newPoPriceMil - remBal) / 1e6);
            }}
              style={{ marginTop: 6, padding: "3px 10px", borderRadius: 4, border: "1px solid #e0af68", background: "transparent", color: "#e0af68", fontSize: 10, fontFamily: F, cursor: "pointer" }}>
              Apply →
            </button>
          )}
        </div>
        <div style={{ background: "#16161e", borderRadius: 8, padding: "10px 16px", border: "1px solid #292e42", minWidth: 180 }}>
          <div style={{ fontSize: 10, color: "#565f89", textTransform: "uppercase", fontFamily: F }}>Min Equity Spread</div>
          <div style={{ fontSize: 18, fontWeight: 700, color: "#7aa2f7", fontFamily: F }}>
            {typeof optResult.maxSpread === "number" ? optResult.maxSpread.toFixed(0) + " bps" : String(optResult.maxSpread)}
          </div>
          <div style={{ fontSize: 9, color: "#565f89", marginTop: 2 }}>minimum spread needed to hit IRR ≥ {optTarget}%</div>
          {typeof optResult.maxSpread === "number" && (
            <button onClick={() => setSpreadBps(Math.round(optResult.maxSpread))}
              style={{ marginTop: 6, padding: "3px 10px", borderRadius: 4, border: "1px solid #7aa2f7", background: "transparent", color: "#7aa2f7", fontSize: 10, fontFamily: F, cursor: "pointer" }}>
              Apply →
            </button>
          )}
        </div>
      </>
    )}
  </div>
</div>
```

- [ ] **Step 4: Build and verify**

```bash
npx esbuild entry.jsx --bundle --outfile=jolco-bundle.js --format=iife
```

Open browser → Sensitivity tab → enter target IRR 7% → click "Find Limits" → verify results appear with correct PO premium % and spread bps → click "Apply →" on one of them → verify Deal Inputs update.

- [ ] **Step 5: Commit**

```bash
git add jolco-v3.jsx jolco-bundle.js
git commit -m "feat: deal optimizer panel on Sensitivity tab"
```

---

### Task 8: Monte Carlo Tab

**Purpose:** A new "Monte Carlo" tab. The user sets uncertainty ranges for 4 key variables (spreadBps, poPremiumPct, vesselPrice, sofrRate). Runs 1000 random draws from uniform distributions over those ranges. Renders a canvas histogram of blended IRR outcomes with P10/P50/P90 markers and probability stats.

**Files:**
- Modify: `jolco-v3.jsx` (add MonteCarloTab component, add state, add tab)

- [ ] **Step 1: Add `MonteCarloTab` component**

Insert before `export default function JOLCOv3()`:

```jsx
function MonteCarloTab({ baseInputs, R }) {
  const [running, setRunning] = React.useState(false);
  const [results, setResults] = React.useState(null);
  const [n, setN] = React.useState(1000);
  const [ranges, setRanges] = React.useState({
    spreadBps:    { lo: Math.max(0, (baseInputs.spreadBps ?? 200) - 100), hi: (baseInputs.spreadBps ?? 200) + 100, enabled: true },
    poPremiumPct: { lo: -10, hi: 20, enabled: true },
    vesselPrice:  { lo: Math.max(5, (baseInputs.vesselPrice ?? 29) * 0.85), hi: (baseInputs.vesselPrice ?? 29) * 1.15, enabled: false },
    sofrRate:     { lo: Math.max(0.5, (baseInputs.sofrRate ?? 4.3) - 1), hi: (baseInputs.sofrRate ?? 4.3) + 1, enabled: false },
  });
  const canvasRef = React.useRef(null);
  const outerRef = React.useRef(null);  // attached to always-rendered outer div
  const [canvasW, setCanvasW] = React.useState(700);

  React.useEffect(() => {
    if (!outerRef.current) return;
    setCanvasW(outerRef.current.offsetWidth);
    const ro = new ResizeObserver(([e]) => setCanvasW(e.contentRect.width));
    ro.observe(outerRef.current);
    return () => ro.disconnect();
  }, []);

  const runMC = () => {
    setRunning(true);
    setResults(null);
    const irrs = [];
    let done = 0;
    const batch = 50;

    const rand = (lo, hi) => lo + Math.random() * (hi - lo);

    const processBatch = () => {
      for (let i = 0; i < batch && done < n; i++, done++) {
        const inp = { ...baseInputs };
        if (ranges.spreadBps.enabled)    inp.spreadBps    = rand(ranges.spreadBps.lo, ranges.spreadBps.hi);
        if (ranges.sofrRate.enabled)     inp.sofrRate     = rand(ranges.sofrRate.lo, ranges.sofrRate.hi);
        if (ranges.vesselPrice.enabled)  inp.vesselPrice  = rand(ranges.vesselPrice.lo, ranges.vesselPrice.hi);
        if (ranges.poPremiumPct.enabled) {
          const remBal = Math.max(0, inp.vesselPrice * 1e6 - (inp.vesselPrice * 1e6 / inp.amortYrs) * inp.effectiveExerciseYear);
          const pct = rand(ranges.poPremiumPct.lo, ranges.poPremiumPct.hi);
          inp.poPriceMil = remBal * (1 + pct / 100);
        }
        try {
          const r = computeDealOutputs(inp);
          if (r.blendedIRR != null) irrs.push(r.blendedIRR * 100);
        } catch (_) {}
      }
      if (done < n) {
        setTimeout(processBatch, 0);
      } else {
        irrs.sort((a, b) => a - b);
        const p = (q) => irrs[Math.floor(q * irrs.length)] ?? null;
        setResults({
          irrs,
          p10: p(0.10), p50: p(0.50), p90: p(0.90),
          mean: irrs.reduce((a, b) => a + b, 0) / irrs.length,
          pAbove0: irrs.filter(v => v > 0).length / irrs.length * 100,
          pAbove5: irrs.filter(v => v > 5).length / irrs.length * 100,
          pAbove7: irrs.filter(v => v > 7).length / irrs.length * 100,
          n: irrs.length,
        });
        setRunning(false);
      }
    };
    setTimeout(processBatch, 0);
  };

  // Draw histogram
  React.useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas || !results) return;
    const ctx = canvas.getContext("2d");
    const W = canvas.width, H = canvas.height;
    const padL = 45, padR = 20, padT = 30, padB = 36;
    ctx.clearRect(0, 0, W, H);
    ctx.fillStyle = "#16161e";
    ctx.fillRect(0, 0, W, H);

    const { irrs, p10, p50, p90 } = results;
    const lo = Math.floor(irrs[0] - 0.5), hi = Math.ceil(irrs[irrs.length - 1] + 0.5);
    const binSize = Math.max(0.25, (hi - lo) / 40);
    const bins = {};
    irrs.forEach(v => {
      const k = Math.floor((v - lo) / binSize);
      bins[k] = (bins[k] || 0) + 1;
    });
    const maxCount = Math.max(...Object.values(bins));
    const xScale = (v) => padL + ((v - lo) / (hi - lo)) * (W - padL - padR);
    const yScale = (c) => padT + (1 - c / maxCount) * (H - padT - padB);

    // Bars
    const totalBins = Math.ceil((hi - lo) / binSize);
    for (let k = 0; k < totalBins; k++) {
      const v = lo + k * binSize;
      const count = bins[k] || 0;
      if (count === 0) continue;
      const x = xScale(v), x2 = xScale(v + binSize);
      const y = yScale(count);
      const color = irrToColor(v / 100);
      ctx.fillStyle = color;
      ctx.fillRect(x + 1, y, Math.max(1, x2 - x - 2), H - padB - y);
    }

    // Percentile lines
    [[p10, "#f7768e", "P10"], [p50, "#9ece6a", "P50"], [p90, "#7aa2f7", "P90"]].forEach(([v, color, label]) => {
      if (v == null) return;
      const x = xScale(v);
      ctx.strokeStyle = color;
      ctx.lineWidth = 1.5;
      ctx.setLineDash([4, 3]);
      ctx.beginPath(); ctx.moveTo(x, padT); ctx.lineTo(x, H - padB); ctx.stroke();
      ctx.setLineDash([]);
      ctx.fillStyle = color;
      ctx.font = "bold 10px 'JetBrains Mono', monospace";
      ctx.textAlign = "center";
      ctx.fillText(label + " " + v.toFixed(1) + "%", x, padT - 6);
    });

    // X-axis labels
    ctx.fillStyle = "#565f89";
    ctx.font = "9px 'JetBrains Mono', monospace";
    ctx.textAlign = "center";
    for (let v = Math.ceil(lo); v <= Math.floor(hi); v += 1) {
      const x = xScale(v);
      ctx.fillText(v + "%", x, H - padB + 14);
    }
    // Y-axis label
    ctx.save();
    ctx.translate(12, (H - padT - padB) / 2 + padT);
    ctx.rotate(-Math.PI / 2);
    ctx.fillStyle = "#565f89";
    ctx.font = "9px 'Inter', sans-serif";
    ctx.textAlign = "center";
    ctx.fillText("Count", 0, 0);
    ctx.restore();
  }, [results, canvasW]);

  const C = { background: "#1a1b26", borderRadius: 10, padding: 18, border: "1px solid #292e42", marginBottom: 14 };
  const F = "'JetBrains Mono', monospace";

  const varLabels = {
    spreadBps: "Equity Spread (bps)",
    poPremiumPct: "PO Premium (%)",
    vesselPrice: "Vessel Price ($M)",
    sofrRate: "SOFR Rate (%)",
  };

  return (
    <div ref={outerRef}>
      <div style={C}>
        <div style={{ fontSize: 11, fontWeight: 700, color: "#565f89", textTransform: "uppercase", letterSpacing: "0.06em", marginBottom: 14, fontFamily: F }}>
          Monte Carlo — IRR Distribution Simulator
        </div>
        <div style={{ display: "grid", gridTemplateColumns: "repeat(2, 1fr)", gap: 12, marginBottom: 14 }}>
          {Object.entries(ranges).map(([key, cfg]) => (
            <div key={key} style={{ background: "#16161e", borderRadius: 8, padding: 12, border: `1px solid ${cfg.enabled ? "#3b4261" : "#1e2030"}`, opacity: cfg.enabled ? 1 : 0.5 }}>
              <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginBottom: 8 }}>
                <span style={{ fontSize: 11, fontWeight: 600, color: cfg.enabled ? "#c0caf5" : "#565f89", fontFamily: F }}>{varLabels[key]}</span>
                <label style={{ display: "flex", alignItems: "center", gap: 5, cursor: "pointer" }}>
                  <input type="checkbox" checked={cfg.enabled}
                    onChange={e => setRanges(r => ({ ...r, [key]: { ...r[key], enabled: e.target.checked } }))} />
                  <span style={{ fontSize: 10, color: "#565f89" }}>vary</span>
                </label>
              </div>
              <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr", gap: 8 }}>
                {["lo", "hi"].map(field => (
                  <div key={field}>
                    <div style={{ fontSize: 9, color: "#565f89", marginBottom: 2, fontFamily: F, textTransform: "uppercase" }}>{field === "lo" ? "Min" : "Max"}</div>
                    <input type="number" value={cfg[field]} step={0.5}
                      onChange={e => setRanges(r => ({ ...r, [key]: { ...r[key], [field]: parseFloat(e.target.value) || 0 } }))}
                      disabled={!cfg.enabled}
                      style={{ width: "100%", padding: "5px 7px", borderRadius: 4, border: "1px solid #3b4261", background: "#24283b", color: "#c0caf5", fontSize: 12, fontFamily: F }} />
                  </div>
                ))}
              </div>
            </div>
          ))}
        </div>
        <div style={{ display: "flex", gap: 12, alignItems: "center" }}>
          <div>
            <div style={{ fontSize: 9, color: "#565f89", marginBottom: 2, fontFamily: F, textTransform: "uppercase" }}>Simulations</div>
            <select value={n} onChange={e => setN(parseInt(e.target.value))}
              style={{ padding: "6px 10px", borderRadius: 5, border: "1px solid #3b4261", background: "#1a1b26", color: "#c0caf5", fontSize: 12, fontFamily: F }}>
              {[500, 1000, 2000, 5000].map(v => <option key={v} value={v}>{v.toLocaleString()}</option>)}
            </select>
          </div>
          <button onClick={runMC} disabled={running}
            style={{ padding: "9px 22px", borderRadius: 7, border: "1px solid #7aa2f7", background: running ? "#16161e" : "rgba(122,162,247,0.15)", color: running ? "#565f89" : "#7aa2f7", fontSize: 13, fontFamily: F, fontWeight: 700, cursor: running ? "not-allowed" : "pointer" }}>
            {running ? "Running…" : "▶ Run Monte Carlo"}
          </button>
        </div>
      </div>

      {results && (
        <>
          <div style={{ display: "grid", gridTemplateColumns: "repeat(4, 1fr)", gap: 12, marginBottom: 14 }}>
            {[
              { l: "P10 (pessimistic)", v: results.p10?.toFixed(2) + "%", c: "#f7768e" },
              { l: "P50 (median)", v: results.p50?.toFixed(2) + "%", c: "#9ece6a" },
              { l: "P90 (optimistic)", v: results.p90?.toFixed(2) + "%", c: "#7aa2f7" },
              { l: "Mean", v: results.mean?.toFixed(2) + "%", c: "#c0caf5" },
            ].map(({ l, v, c }) => (
              <div key={l} style={{ ...C, marginBottom: 0, textAlign: "center" }}>
                <div style={{ fontSize: 10, color: "#565f89", textTransform: "uppercase", fontFamily: F }}>{l}</div>
                <div style={{ fontSize: 22, fontWeight: 700, color: c, fontFamily: F, marginTop: 4 }}>{v}</div>
              </div>
            ))}
          </div>
          <div style={{ display: "grid", gridTemplateColumns: "repeat(3, 1fr)", gap: 12, marginBottom: 14 }}>
            {[
              { l: "Pr(IRR > 0%)", v: results.pAbove0.toFixed(1) + "%", c: results.pAbove0 > 90 ? "#9ece6a" : "#e0af68" },
              { l: "Pr(IRR > 5%)", v: results.pAbove5.toFixed(1) + "%", c: results.pAbove5 > 70 ? "#9ece6a" : "#e0af68" },
              { l: "Pr(IRR > 7%)", v: results.pAbove7.toFixed(1) + "%", c: results.pAbove7 > 50 ? "#9ece6a" : "#f7768e" },
            ].map(({ l, v, c }) => (
              <div key={l} style={{ ...C, marginBottom: 0, textAlign: "center" }}>
                <div style={{ fontSize: 10, color: "#565f89", textTransform: "uppercase", fontFamily: F }}>{l}</div>
                <div style={{ fontSize: 22, fontWeight: 700, color: c, fontFamily: F, marginTop: 4 }}>{v}</div>
              </div>
            ))}
          </div>
          <div style={C}>
            <div style={{ fontSize: 11, fontWeight: 600, color: "#565f89", textTransform: "uppercase", fontFamily: F, marginBottom: 10 }}>Blended IRR Distribution ({results.n.toLocaleString()} simulations)</div>
            <canvas ref={canvasRef} width={canvasW} height={220} style={{ width: "100%", height: "auto", display: "block", borderRadius: 6 }} />
          </div>
        </>
      )}
    </div>
  );
}
```

- [ ] **Step 2: Add Monte Carlo state**

After `optResult` state, add:
```jsx
  // Monte Carlo — no state needed at parent level; MonteCarloTab is self-contained
```
(No parent state needed — the tab manages its own state.)

- [ ] **Step 3: Add "Monte Carlo" tab to the tab bar**

Find:
```jsx
        {T("deal", "Deal Inputs")}{T("sensitivity", "Sensitivity")}{T("scenarios", "Scenarios")}{T("depr", "Depreciation Scale")}{T("cf", "Equity Cashflows")}{T("vs", "vs Treasury")}
```

Replace with:
```jsx
        {T("deal", "Deal Inputs")}{T("sensitivity", "Sensitivity")}{T("scenarios", "Scenarios")}{T("mc", "Monte Carlo")}{T("depr", "Depreciation Scale")}{T("cf", "Equity Cashflows")}{T("vs", "vs Treasury")}
```

- [ ] **Step 4: Render MonteCarloTab**

After the Scenarios tab block (`{tab === "scenarios" && ...}`), add:

```jsx
        {/* ═══ MONTE CARLO ═══ */}
        {tab === "mc" && (
          <MonteCarloTab
            baseInputs={{
              vesselPrice, debtPct, amortYrs, sofrRate, spreadBps,
              jpyBaseRate, bankSpreadBps, swapCostBps, saleCommission, bbcCommission,
              taxRate, capGainsTaxRate, foreignInterestTaxPct, usefulLife, specialDeprPct,
              treasuryYield, effectiveExerciseYear,
              poPriceMil: R.poPriceMil,
              fxFactor: JPY_USD_BASE / jpyUsdRate,
            }}
            R={R}
          />
        )}
```

- [ ] **Step 5: Build and verify**

```bash
npx esbuild entry.jsx --bundle --outfile=jolco-bundle.js --format=iife
```

Open browser → Monte Carlo tab → enable 2 variables → run 1000 sims → verify histogram renders with P10/P50/P90 lines and probability stats cards.

- [ ] **Step 6: Commit**

```bash
git add jolco-v3.jsx jolco-bundle.js
git commit -m "feat: Monte Carlo IRR distribution simulator tab"
```

---

### Task 9: FX Sensitivity Axis on Heatmap

**Purpose:** Add "JPY/USD Rate (¥)" as a selectable axis variable on the sensitivity heatmap, so users can see how IRR changes as the yen strengthens or weakens.

**Files:**
- Modify: `jolco-v3.jsx` (add to AXIS_VARS, AXIS_CFG, handle in SensitivityTab cell computation, add jpyUsdRate to baseInputs, add JPY/USD input to Deal Inputs)

- [ ] **Step 1: Add jpyUsdRate to AXIS_VARS**

Find:
```jsx
const AXIS_VARS = [
  { key: "spreadBps",      label: "Equity Spread (bps)",         unit: "bps" },
  { key: "poPremiumPct",   label: "PO Premium over Balance (%)",  unit: "%"  },
  { key: "debtPct",        label: "Leverage — Debt %",            unit: "%"  },
```

Add `jpyUsdRate` entry:
```jsx
const AXIS_VARS = [
  { key: "spreadBps",      label: "Equity Spread (bps)",         unit: "bps" },
  { key: "poPremiumPct",   label: "PO Premium over Balance (%)",  unit: "%"  },
  { key: "debtPct",        label: "Leverage — Debt %",            unit: "%"  },
  { key: "jpyUsdRate",     label: "JPY/USD Rate (¥)",             unit: "¥"  },
```

- [ ] **Step 2: Add jpyUsdRate to AXIS_CFG**

Find the `AXIS_CFG` object and add:
```jsx
  jpyUsdRate: { step: 5, count: 9, mode: "range", min: 120, max: 180 },
```

- [ ] **Step 3: Handle jpyUsdRate in SensitivityTab cell computation**

In `SensitivityTab`'s useEffect where cells are computed:

Find the block that handles `poPremiumPct` specially:
```jsx
          if (heatXVar === "poPremiumPct") {
            const remBal = Math.max(0, inp.vesselPrice * 1e6 - (inp.vesselPrice * 1e6 / inp.amortYrs) * inp.effectiveExerciseYear);
            inp.poPriceMil = remBal * (1 + xVal / 100);
          } else { inp[heatXVar] = xVal; }

          if (heatYVar === "poPremiumPct") {
            const remBal = Math.max(0, inp.vesselPrice * 1e6 - (inp.vesselPrice * 1e6 / inp.amortYrs) * inp.effectiveExerciseYear);
            inp.poPriceMil = remBal * (1 + yVal / 100);
          } else { inp[heatYVar] = yVal; }
```

Replace with:
```jsx
          if (heatXVar === "poPremiumPct") {
            const remBal = Math.max(0, inp.vesselPrice * 1e6 - (inp.vesselPrice * 1e6 / inp.amortYrs) * inp.effectiveExerciseYear);
            inp.poPriceMil = remBal * (1 + xVal / 100);
          } else if (heatXVar === "jpyUsdRate") {
            inp.fxFactor = (inp.jpyUsdBase ?? 150) / xVal;
          } else { inp[heatXVar] = xVal; }

          if (heatYVar === "poPremiumPct") {
            const remBal = Math.max(0, inp.vesselPrice * 1e6 - (inp.vesselPrice * 1e6 / inp.amortYrs) * inp.effectiveExerciseYear);
            inp.poPriceMil = remBal * (1 + yVal / 100);
          } else if (heatYVar === "jpyUsdRate") {
            inp.fxFactor = (inp.jpyUsdBase ?? 150) / yVal;
          } else { inp[heatYVar] = yVal; }
```

- [ ] **Step 4: Pass jpyUsdBase and fxFactor in baseInputs for SensitivityTab**

Find the `<SensitivityTab` invocation in the JSX:
```jsx
            baseInputs={{
              vesselPrice, debtPct, amortYrs, sofrRate, spreadBps,
              jpyBaseRate, bankSpreadBps, swapCostBps, saleCommission, bbcCommission,
              taxRate, capGainsTaxRate, foreignInterestTaxPct, usefulLife, specialDeprPct,
              treasuryYield, effectiveExerciseYear,
              poPriceMil: R.poPriceMil,
            }}
```

Replace with:
```jsx
            baseInputs={{
              vesselPrice, debtPct, amortYrs, sofrRate, spreadBps,
              jpyBaseRate, bankSpreadBps, swapCostBps, saleCommission, bbcCommission,
              taxRate, capGainsTaxRate, foreignInterestTaxPct, usefulLife, specialDeprPct,
              treasuryYield, effectiveExerciseYear,
              poPriceMil: R.poPriceMil,
              jpyUsdRate, jpyUsdBase: JPY_USD_BASE,
              fxFactor: JPY_USD_BASE / jpyUsdRate,
            }}
```

- [ ] **Step 5: Add JPY/USD Rate input to Deal Inputs tab**

In the Deal Inputs tab, find the "Equity Cashflow Parameters" or "Charter / Hire" card. After `sofrRate`'s `<Inp>` component, add a JPY/USD rate input. Find a good place — it logically belongs near the swap cost input:

```jsx
<Inp label="JPY/USD Rate (¥)" value={jpyUsdRate} onChange={setJpyUsdRate}
  unit="¥" step={1} min={80} max={250}
  help={`Tax shield scales by ${JPY_USD_BASE}/${jpyUsdRate} = ${(JPY_USD_BASE/jpyUsdRate).toFixed(3)}× vs base`} />
```

- [ ] **Step 6: Build and verify**

```bash
npx esbuild entry.jsx --bundle --outfile=jolco-bundle.js --format=iife
```

Open browser → Sensitivity tab → set Y-Axis to "JPY/USD Rate (¥)" → heatmap computes with FX axis → verify changing JPY/USD in Deal Inputs updates blended IRR correctly.

- [ ] **Step 7: Commit**

```bash
git add jolco-v3.jsx jolco-bundle.js
git commit -m "feat: JPY/USD FX rate axis on heatmap + deal input"
```

---

## Final Build and Push

- [ ] **Final verification**

```bash
cd "/Users/sriniwasghate/Programs/JOLCO Calculator V4"
npx esbuild entry.jsx --bundle --outfile=jolco-bundle.js --format=iife
```

Expected: clean build, ~1.3mb, no errors.

Open `https://sriniwas512.github.io/JOLCO-Calculator-V4/` and verify:
1. Deal Inputs → IRR waterfall chart visible, tax shock panel works
2. Sensitivity tab → range inputs work, optimizer panel works
3. Monte Carlo tab → runs and renders histogram
4. Header → Copy Link and Export Memo buttons work
5. Equity Cashflows → Download CSV button works
6. Sensitivity tab → JPY/USD Rate selectable as axis

- [ ] **Final push**

```bash
git add jolco-v3.jsx jolco-bundle.js
git commit -m "chore: final build — JOLCO V4 extended features complete"
git push
```
