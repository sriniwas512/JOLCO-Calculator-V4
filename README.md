# JOLCO IRR Calculator v3

> **Japanese Operating Lease with Call Option**, an interactive browser based financial model for Japanese shipping equity investors evaluating the economics of a JOLCO deal.

---

## What is a JOLCO?

A **JOLCO (Japanese Operating Lease with Call Option)** is a structured ship finance product in which:

1. A **Japanese SPC** (Special Purpose Company) acquires a vessel.
2. The SPC is funded by:
   - **~70% JPY bank debt** from a Japanese regional bank or trust bank at low JPY rates (TIBOR + spread), hedged into USD via a cross currency basis swap.
   - **~30% TK equity** from Japanese individual or institutional investors through a **Tokumei Kumiai (匿名組合)**, a silent partnership under Japan's Commercial Code that passes tax losses directly to investors.
3. The SPC **bareboat charters** the vessel back to the original seller / shipping company under a long term BBC.
4. The **BBC hire** has two components:
   - **Fixed hire** = Vessel Price ÷ Amortization Period. Covers principal repayment. Rate insensitive.
   - **Variable hire** = (SOFR + spread) × total outstanding vessel balance. The charter hire rate is applied to the full outstanding balance — both the debt-funded and equity-funded portions. The JPY bank loan is a separate SPC cost structure, independent of the hire, which creates genuine leverage sensitivity: cheap JPY funding means the equity tranche captures the full (hire rate − bank rate) spread on the leveraged portion.
5. The charterer holds **Purchase Options** stepping down over the lease term. The final year is an **obligation** to buy.
6. **Depreciation** on the full vessel cost flows through the TK to investors, creating paper losses that offset their other taxable Japanese income. This is the primary economic driver (Stream ②).

**All percentages and rates are fully user adjustable.** The defaults are a realistic base case only.

---

## The Three Return Streams

| Stream | What it is | Driver |
|--------|-----------|--------|
| **① Hire Spread** | Net charter hire after full bank debt service (principal + JPY interest) and BBC brokerage. The equity tranche captures the leverage arbitrage between the charter hire rate and the cheap JPY bank rate. | Charter hire rate (SOFR + spread) on total balance, minus bank all-in cost on debt portion |
| **② Tax Shield** | Tax saved when SPC depreciation losses flow via TK to investors, offsetting their other Japanese taxable income. Front-loaded in early years due to accelerated DB depreciation. | MOF depreciation schedule × effective JP tax rate |
| **③ Residual / PO** | Net proceeds when charterer exercises the Purchase Option: PO price minus remaining bank debt minus disposal gain tax on (PO price − depreciated book value). | PO price vs book value vs outstanding debt |

The **Blended IRR** is the Internal Rate of Return combining all three streams against the total equity plus sale commission outflow at Year 0.

---

## Financing Structure — JPY Bank Loan

The SPC borrows in JPY from a Japanese bank. Since BBC hire is paid in USD, the SPC executes a **cross currency basis swap** to convert its JPY obligations into USD cashflows. Realistic market parameters:

| Parameter | Typical range | Default in this model |
|-----------|-------------|---------|
| JPY base rate (TIBOR) | 0.5 to 2.0% | 1.30% |
| Bank spread over JPY base | 80 to 130 bps | 100 bps |
| USD/JPY cross currency swap cost | 20 to 45 bps | 35 bps |
| **Effective USD cost of bank debt** | **~1.5 to 3.0%** | **~2.65%** |
| Charter hire rate (SOFR + spread) | SOFR + 50 to 350 bps | SOFR 4.3% + 280 bps = **7.1%** |

The spread between charter hire rate (~7.1%) and bank debt cost (~2.65%), applied to the 70% leveraged portion, is a core economic amplifier for the equity tranche.

---

## Commissions

Two separate brokerage commissions are modelled:

| Commission | Default | When paid | Tax treatment |
|-----------|---------|-----------|--------------|
| **Sale Commission** | 2.0% of VP | Year 0, at closing | **Capitalised into the depreciable asset base** (Japanese tax law treats acquisition costs as part of asset cost). Increases both total equity deployed and the depreciation tax shield. |
| **BBC Commission** | 1.25% of gross hire | Annual, each year | Deductible SPC expense. Reduces net hire, taxable P&L, and equity net cashflow. |

---

## Tax and Depreciation (Japanese Rules)

| Item | Detail |
|------|--------|
| Corporation Tax | 23.2% standard rate |
| Local business tax | ~4.2% |
| Defense surtax | ~3.2% |
| **Effective rate default** | **30.62%** (fully adjustable) |
| Depreciation method | 200% Declining Balance switching to Straight Line when SL ≥ DB (post-FY2012 MOF ordinance) |
| Special depreciation | 18 to 32% of cost in Year 1 for MLIT certified advanced **newbuildings** only. Default **30%** (newbuilding base case). |
| Foreign flag vessels | 12 year flat useful life (MOF 耐用年数省令 Beppyō 1, その他のもの) |
| Japanese flag vessels | Type specific: 11 to 15 years per MOF schedule |
| LPG carriers | Use oil tanker useful life per NTA Circular 2-4-2 |
| **Depreciable base** | **VP + sale commission** — acquisition costs are capitalised per Japanese tax law |

---

## Tax on PO Disposal Gain (Stream ③)

Japan has **no separate capital gains regime** for TK distributions. The gain on vessel disposal at PO exercise is taxed as **ordinary income**, not as a capital gain. This is confirmed by NTA Income Tax Basic Circular 36・37共-21, which classifies TK distributions (including asset disposal proceeds) as follows:

| Investor type | Tax classification | Rate |
|--------------|-------------------|------|
| Japanese resident **corporation** | Ordinary business income | ~30.62% (default) |
| Japanese resident **individual** (passive TK investor) | Miscellaneous income (雑所得) | Progressive up to 55% |
| Japanese resident **individual** (active co-manager) | Business income (事業所得) | Progressive up to 55% |
| **Non-resident** investor (no Japan PE) | Withholding tax as final tax | 20.42% |

The taxable amount is `max(0, PO price − depreciated book value)`. The entire spread is taxed uniformly — there is no bifurcation equivalent to US §1245 recapture. The default rate of **30.62%** matches the corporate investor standard (the primary JOLCO market).

---

## Second-Hand Vessel Provisions

When the vessel age at delivery is greater than zero, the model automatically applies the Japanese NTA rules for used assets under **MOF Ordinance Article 3 (耐用年数省令 第3条)**.

### Remaining Useful Life Formula (中古資産の耐用年数)

| Condition | Remaining Life |
|-----------|---------------|
| Age ≥ statutory new life | `max(2, floor(newLife × 0.2))` |
| Age < statutory new life | `max(2, floor((newLife − age) + age × 0.2))` |

The second formula simplifies to `max(2, floor(newLife − age × 0.8))`. A **minimum of 2 years is always applied** — this is a hard statutory floor (強行規定) regardless of how old the vessel is. When remaining life = 2, the 200% DB rate = 100%, meaning the full cost is written off in Year 1. This is correct per legislation and is flagged with a warning badge in the UI.

**Example — 8-year-old Bulk Carrier ≥2,000 GT (foreign flag, newLife = 12 yr):**
- `max(2, floor((12 − 8) + 8 × 0.2)) = max(2, floor(4 + 1.6)) = max(2, 5) = 5 yr`

### Key Differences for Second-Hand vs Newbuilding

| Item | Newbuilding | Second-Hand |
|------|------------|-------------|
| Useful life | Full statutory (9–15 yr) | Remaining NTA life (min 2 yr) |
| Depreciation schedule | Runs for full new life | Runs only for remaining life |
| Year 1 DB rate | 2 / newLife | 2 / remainingLife (higher — faster write-off) |
| Special depreciation (MLIT) | Eligible (18–32%) | Generally **not eligible** — applies to certified new advanced vessels only. Verify with tax counsel |
| Debt LTV | Banks typically 70% | Banks typically 60–65% for older vessels — adjust `debtPct` accordingly |

### What the calculator does automatically

1. Reads vessel type and flag to determine statutory new life per MOF Beppyō 1.
2. Applies the NTA Art. 3 formula to derive remaining useful life from the entered vessel age.
3. Runs the full 200% DB → SL depreciation schedule on the **remaining life**, starting from **(VP + sale commission)** as the depreciable base.
4. Displays the remaining vs new life in both Tab 1 (Vessel & Structure) and Tab 2 (Depreciation Scale).
5. Warns if special depreciation is set above zero for a second-hand vessel.
6. The MOF Rate Index in Tab 2 shows the NTA-computed remaining life for every vessel type at the entered age.

---

## Codebase Architecture

```
jolco-v3.jsx           source React component (edit this)
entry.jsx              esbuild entry point, imports jolco-v3.jsx and mounts React root
jolco-bundle.js        compiled bundle (generated by npm run build, do not edit directly)
index.html             loads jolco-bundle.js, open in browser after building
updated bg image.png   header logo
favicon.ico            browser tab icon (Mount Fuji, Tokyo Night palette)
```

### Build

```bash
npm install          # install dependencies (first time only)
npm run build        # compile jolco-v3.jsx to jolco-bundle.js via esbuild
# then open index.html in your browser
```

Use `npm run watch` during development. esbuild will automatically rebuild on every save.

### Key functions

| Function | Purpose |
|----------|---------|
| `solveIRR(cf, guess)` | Newton–Raphson IRR solver with bisection fallback and robust sign-flip detection |
| `computeDepr(cost, life, specialPct)` | Full MOF 200% DB → SL schedule with Year 1 special depreciation. `cost` = VP + sale commission |
| `computeUsedAssetLife(newLife, usedYears)` | NTA MOF Art. 3 remaining useful life: `max(2, floor((newLife − usedYears) + usedYears × 0.2))` |
| `JOLCOv3()` | Main React component with all state, memos, and UI |
| `useMemo R{}` | Core financial model — produces all cashflows, IRR, stream totals |

### All user adjustable state variables

| State | Default | Description |
|-------|---------|-------------|
| `vesselPrice` | $29.4M | Vessel purchase price |
| `vesselAgeYrs` | 0 yr | Age at delivery. 0 = newbuilding. Positive value triggers NTA Art. 3 remaining life formula |
| `debtPct` | 70% | Bank debt as % of VP |
| `amortYrs` | 15 yr | Amortization period. Fixed hire = VP ÷ amortYrs. Can differ from leaseTerm |
| `leaseTerm` | 10 yr | BBC lease duration. Syncs poLastYear (last PO / obligation) |
| `jpyBaseRate` | 1.30% | TIBOR JPY base rate |
| `bankSpreadBps` | 100 bps | Bank credit spread over TIBOR |
| `swapCostBps` | 35 bps | USD/JPY cross currency basis swap cost |
| `sofrRate` | 4.30% | USD SOFR reference rate |
| `spreadBps` | 280 bps | Charter hire spread over SOFR (applied to total outstanding balance) |
| `saleCommission` | 2.0% | Vessel purchase brokerage — capitalised into depreciable base |
| `bbcCommission` | 1.25% | Annual bareboat charter brokerage on gross hire |
| `taxRate` | 30.62% | Effective Japanese corporate tax rate (Stream ② and SPC P&L) |
| `capGainsTaxRate` | 30.62% | Tax rate on PO disposal gain (ordinary income per NTA Circular 36・37共-21; same rate as taxRate for corporate investors) |
| `foreignInterestTaxPct` | 27% | JP corporate rate on foreign interest income (for Treasury comparison) |
| `specialDeprPct` | 30% | Year 1 special depreciation % — default 30% for newbuilding base case |
| `treasuryYield` | 4.25% | US Treasury yield for risk-free comparison |
| `poFirstYear` / `poLastYear` | 5 / 10 | PO exercise window |
| `poPremium` | $0.5M | Flat premium added above financing balance at each PO year |
| `exerciseYear` | 10 | Chosen PO exercise year |
| `poOverrides` | {} | Per-year manual PO price overrides |

### Financial model per year (useMemo)

```
DEPRECIABLE BASE:
  depreciableBase  = VP + saleCommCost                          (acquisition cost capitalised per JP tax law)

CASH IN TO SPC:
  Fixed hire       = min(annualPrincipal, outstandingTotal)     (clamped — zeroes out once fully amortised)
  Variable hire    = outstandingTotal × equityAllInRate         (charter hire rate on FULL outstanding balance)
  BBC Commission   = totalHire × bbcCommission%                 (deducted; deductible SPC expense)
  Net hire         = totalHire − bbcCommCost

CASH OUT TO BANK (always senior):
  bankPrincipal    = min(annualPrincipal × debtPct%, outstandingDebt)   (clamped to remaining debt)
  bankInterest     = outstandingDebt × bankAllInRate

NET TO EQUITY:
  equityPrincipal  = min(annualPrincipal × equityPct%, outstandingEquity)
  hireSpread       = netHire − bankPrincipal − bankInterest − equityPrincipal   (Stream ①)
  netCF            = equityPrincipal + hireSpread + taxShield + residual

SPC TAXABLE P&L:
  = netHire − depreciation − bankInterest
  taxShield        = −spcTaxablePL × taxRate%                   (Stream ②: positive = tax saved)

RESIDUAL (exit year only):
  remainingDebt    = outstandingDebt (post this year's bankPrincipal payment)
  grossResidual    = poPriceMil − remainingDebt
  disposalGainTax  = max(0, poPriceMil − bookValue) × capGainsTaxRate%
  residualToEquity = grossResidual − disposalGainTax            (Stream ③)

YEAR 0:
  equityCF[0]      = −(equity + saleCommCost)
  bankAllInRate    = (jpyBaseRate + bankSpreadBps/100)/100 + swapCostBps/10000
  equityAllInRate  = (sofrRate + spreadBps/100)/100
```

---

## UI Tabs

| Tab | Contents |
|-----|---------|
| **Deal Inputs** | KPI summary row at top (all 3 streams plus IRR plus MoIC, all dynamic). Three input columns: Vessel & Structure, Charter & Interest (JPY loan / USD hire), Purchase Options & Tax |
| **Depreciation Scale** | Year-by-year horizontal bar chart of DB → SL depreciation. Clickable MOF rate index for all vessel types |
| **Equity Cashflows** | Full equation with explainers per component. Clickable year-by-year table with drill-down for hire spread, tax shield, and PO residual |
| **vs Treasury** | Side-by-side JOLCO blended IRR vs US Treasury compounded on same equity deployed. Spread in bps |

---

## Purchase Option Schedule

- **Auto generated:** `PO(N) = max(0, VP − (VP / amortYrs) × N) + poPremium`
- **Per year editable inline** (purple border = overridden; "reset" to revert to auto).
- **Final year = Obligation** (charterer must buy regardless).
- **Exercise year** freely selectable within the PO window.

---

## Deployment on GitHub Pages

The app is a **single self-contained HTML file** (`index.html`). No build step, no Node.js, no server required.

```bash
# Open locally
open index.html

# Push to GitHub (Pages serves automatically from main branch root)
git push origin main
```

> **Do not serve `jolco-v3.jsx` directly.** GitHub Pages cannot execute JSX. Always use `index.html`, which loads the pre-compiled bundle.

---

## Legal and Regulatory References

| Reference | Relevance |
|-----------|----------|
| Corporation Tax Act Art. 31 | Depreciation deduction rules for fixed assets |
| MOF Ordinance 耐用年数省令 Beppyō 1 (別表第一) | Vessel statutory useful lives by type and flag |
| Special Measures Taxation Act Art. 67-12 | TK loss restriction — passive investors cannot deduct losses exceeding outstanding capital contribution |
| Special Measures Taxation Act (MLIT special depr.) | Year 1 special depreciation for certified advanced newbuildings |
| NTA Circular 2-4-2 | LPG carriers classified as oil tankers for depreciation purposes |
| MOF Ordinance Art. 3 (耐用年数省令 第3条) | Remaining useful life formula for used (second-hand) assets — 2 yr minimum floor is a hard statutory rule |
| **NTA Income Tax Basic Circular 36・37共-21** | **TK distributions (incl. asset disposal proceeds) are miscellaneous income (雑所得) for individual investors and ordinary income for corporations — NOT capital gains. Governs tax treatment of Stream ③.** |
| Commercial Code Art. 535 to 542 | Tokumei Kumiai (TK) silent partnership structure |
| Ship Act Arts. 4 to 19 | Japanese flag registration requirement (determines useful life category) |

---

*Built for shipping finance professionals. All defaults represent a realistic newbuilding base case. Every parameter is user adjustable.*
