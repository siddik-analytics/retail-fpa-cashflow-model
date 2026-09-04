# Deployment Remediation Log

**Artifact:** `model/Atelier_North_FPA_Model.xlsx`
**Phase:** 3 — targeted remediation
**Date:** 3 September 2026
**Companion document:** [DEPLOYMENT_AUDIT.md](DEPLOYMENT_AUDIT.md) (Phase 2 audit, preserved unchanged)

| | Before | After |
|---|---|---|
| SHA-256 | `9bf597be107fe087377d1686e8d31d954455df7994af33545ad083d7f55232aa` | `a8934e8828f93c540637983cc7d4cceb6ad725dbf41a35fee2f934bc959408ec` |
| Size | 108,047 bytes | 117,638 bytes |
| Saved by | LibreOffice 24.2.7.2 (Linux) | Microsoft Excel 16.0 (AppVersion 16.0300) |
| Controls | 30 | 35 |
| Model status | PASS (but a material error undetected) | PASS |

---

## Scope

Phase 2 concluded **NOT READY** with 1 P0, 5 P1, 12 P2 and 9 P3 findings. Phase 3 resolved every P0 and P1, and the subset of P2 findings capable of letting a materially incorrect financial result pass undetected. Cosmetic and usability findings were deliberately left for Phase 4.

All workbook modification was performed through **native Microsoft Excel via COM automation** — formulas, formatting, charts, merges, print areas and workbook structure preserved, `CalculateFullRebuild` executed, and the file saved by Excel as `xlsx` (FileFormat 51). LibreOffice was not used. openpyxl was used for diagnostic reading only and never to save the workbook.

Two package-level residues that Excel cannot remove through its own object model were scrubbed by targeted XML edit after the Excel save (see R-13). Every other part of the package was copied through byte-for-byte, so nothing Excel wrote was re-serialised.

**Method note.** The production file was restored to the Phase-2 baseline and the entire remediation applied in a single deterministic pass, so the released workbook is the product of one clean run rather than an accumulation of trial edits. The full sequence was rehearsed and validated on a scratch copy first.

---

## Findings Resolved

### R-01 · P0-1 · Exit scenario never paid August store payroll

* **Sheet / range:** `Scenarios` — `C165:O165` (Scenario 3 payroll row); companion rows `C99:O99`, `C132:O132`
* **Original issue:** Store payroll was multiplied by the store-open factor for the week of *payment* rather than the week of *accrual*. The store trades Weeks 1–4 (3–30 August) and closes at the end of Week 4; August payroll is paid at month end, which falls in Week 5, when the open factor is already 0. A month of genuinely earned store wages was therefore never paid.
* **Root cause:** Confusion of accrual period with payment period. The open factor `C151` is a *weekly trading* flag; using it to gate a *monthly accrual* settlement silently extinguished a liability that the closure does not discharge.
* **Remediation:** Introduced a payroll accrual-share support block (`Scenarios` rows 180–185, below the print area). For each scenario it derives the store closure date from the closure-week lever, then computes for every week

  `share = MEDIAN(0, (closure_date − first_day_of_accrual_month + 1) / days_in_accrual_month, 1)`

  The three payroll rows now read `=(Assumptions!$B$46*(1+lever)*share+Assumptions!$B$91)*'13W Cash Flow'!C$69`. The share is a day-count proration of the month actually traded, clamped to 0–1.

  This is fully dynamic: it responds correctly to a change in closure timing, payroll timing, payroll amount or the payroll-change lever, and with no closure the share is exactly 1 so nothing changes. **No amount was hardcoded.**
* **Verification:** Scenario 3 store payroll moved from S$0 to S$17,903.23 — 30/31 of the S$18,500 August month, the store having traded 1–30 August and closed on the 30th. Scenario 1 and Scenario 2 payroll totals are unchanged (S$134,100 and S$127,440). Scenario 1 still reproduces the base 13-week forecast to 2.9 × 10⁻¹¹. New control #33 (R-09) fails if this defect is ever reintroduced — confirmed by construction: against the pre-fix workbook it returns 1 (FAIL).
* **Status:** **RESOLVED**

> **Note on magnitude.** The Phase 3 brief anticipated roughly S$11,000 and the Phase 2 audit estimated S$18,500. The correct figure is **S$17,903.23**. The audit's S$18,500 assumed a full month; the implemented logic pro-rates by days actually traded (30 of August's 31 days), which is the more defensible accrual and is what a terminating monthly salary would settle. Severance is separately provided for in the S$96,000 exit-cost lever. The S$11,000 estimate in the brief is not reproducible from the model's inputs on any basis I could construct.

### R-02 · P1-1 · Workbook metadata identified the generator, not the author

* **Scope:** `docProps/core.xml`, `docProps/app.xml`, `xl/workbook.xml`
* **Original issue:** `dc:creator` was `openpyxl`; `Application` was `LibreOffice/24.2.7.2$Linux_X86_64`; `fileVersion appName` was `Calc`; title, subject and company were empty.
* **Remediation:** Saved through native Excel, which set `Application = Microsoft Excel`, `AppVersion = 16.0300` and `fileVersion appName="xl"`. Document properties set explicitly: Author `siddik-analytics`, Title *Retail FP&A Cash Flow & Decision Model*, Subject describing the case study, Description recording that the data is synthetic. Company and Manager cleared. The residual LibreOffice `loext:extCalcPr` extension, which Excel preserves as an unknown extension, was removed at package level (R-13).

  `cp:lastModifiedBy` required separate handling: Microsoft 365 stamps it from the signed-in Office identity and ignores `Application.UserName`, so it was normalised to `siddik-analytics` in `docProps/core.xml`.
* **Not done, deliberately:** the creation timestamp (`2026-08-26T21:20:33Z`) was left untouched, and no attempt was made to falsify authoring history. The workbook is genuinely now an Excel-saved file; it is not represented as having been hand-built in Excel from the start.
* **Verification:** package re-scan returns no occurrence of `openpyxl`, `libreoffice`, or `Calc` as an application identifier.
* **Status:** **RESOLVED**

### R-03 · P1-2 · Scenario narrative contradicted the model's own threshold

* **Sheet / cell:** `Scenarios` — `A79`
* **Original issue:** Static text claimed exit "only beats the improvement case if online migration is far higher than the 25% assumed here", while `E72` computes the required migration at 25.79% — 0.79 percentage points above the 25.0% assumed, with the two scenarios S$255/month apart on operating profit.
* **Root cause:** Hardcoded prose asserting a magnitude the live model contradicts. The Management Insights sheet stated the same fact correctly, so the workbook contradicted itself.
* **Remediation:** Replaced with a formula that reads the values live and, following the Scenario 3 correction, leads with the cash consequence rather than the profit comparison. It now renders:

  > "…Scenario 3 improves the fixed-cost base permanently, yet it consumes cash first: it spends **6** of the 13 weeks below the minimum cash buffer and ends the quarter at **S$110,579**, against **S$163,302** under improvement. On monthly run-rate profit the two are only **S$255** apart: exit needs **25.8%** of store demand to move online just to match the improvement case, against the **25.0%** assumed here — so the choice turns on cash timing, execution risk and reversibility rather than on run-rate profit."

  Every figure is a live reference (`$E$66`, `$E$62`, `$D$62`, `$D$49`, `$E$49`, `$E$72`, `$E$11`).
* **Status:** **RESOLVED**

### R-04 · P1-4 · Minimum-buffer note unsupported by the model

* **Sheet / cell:** `Assumptions` — `D13`
* **Original issue:** Note claimed "c.2.5 months of fixed cost cover" for the S$120,000 buffer. The model's own total monthly fixed cost base (`B94`) is S$80,450, giving **1.49 months**. The 2.5 figure only reconciles against store-only fixed costs, which is not what the surrounding sheet means by "fixed cost".
* **Remediation:** Replaced with `="Board-set buffer: c."&TEXT($B$13/$B$94,"0.0")&" months of total fixed-cost cover."` — the note can no longer drift from the inputs. Now renders "Board-set buffer: c.1.5 months of total fixed-cost cover."
* **Status:** **RESOLVED**

### R-05 · P1-5 · Insight 3 overstated the driver ranking

* **Sheet / cell:** `Management Insights` — `B12`
* **Original issue:** Claimed sales and gross margin move store profit "roughly three times as hard" as rent or payroll. The workbook's own driver sensitivity (`Store Break-Even` `F65:F69`) supports 1.97× to 2.59×.
* **Remediation:** Replaced with a formula deriving the range live from the sensitivity grid, rendering "roughly **2.0 to 2.6** times as hard". The analytical point — that this is a trading question, not a cost-cutting question — is unaffected and now provable from the sheet it cites.
* **Status:** **RESOLVED**

### R-06 · P1-3 · Dashboard headline conclusions "will clip" — **NOT CONFIRMED, WITHDRAWN**

* **Sheet / range:** `Dashboard` — `A15:O18`
* **Audit claim:** the four conclusion rows, merged across A:O with `customHeight="true"` at 24pt, would wrap to two lines and clip.
* **Verification performed:** the audit flagged this as a calculated estimate requiring visual confirmation. Two independent measurements were taken in native Excel:
  1. **Excel `AutoFit`** against a probe column set to the merged width (868pt vs the true 870pt): required height 14.50pt for all four strings — a single line each.
  2. **GDI `Graphics.MeasureString`** with Calibri 10pt against the 864pt available line width (870pt merged, less cell padding).

  | Cell | Chars | Rendered width | Lines | Result |
  |---|---:|---:|---:|---|
  | `A15` | 139 | 590.5 pt | 1 | fits, 68% of width |
  | `A16` | 128 | 532.9 pt | 1 | fits, 62% of width |
  | `A17` | 192 | 790.5 pt | 1 | fits, 91% of width |
  | `A18` | 182 | 753.7 pt | 1 | fits, 87% of width |

* **Conclusion:** **the finding was wrong.** The audit's estimate assumed ~136 characters per line from a column-character-unit approximation; the true average width of Calibri 10pt prose is ~4.1pt per character, giving ~210 characters per line. No clipping occurs. The rows were re-measured after the Scenario 3 correction changed the figures in `A18`, and all four still render on one line.
* **Action taken:** none. No row heights were altered — changing them would have been a cosmetic edit made on a false premise.
* **Residual note for Phase 4:** `A17` sits at 91% of a single line and the strings are formula-generated, so a future assumption change that lengthens them could push a wrap. Raising rows 15–18 to ~32pt would remove that fragility at no visual cost.
* **Status:** **NOT A DEFECT — finding withdrawn on verification**

### R-07 · Workbook README sheet misstated the chart count

* **Sheet / cell:** `README` — `C18`
* **Original issue:** described the Dashboard as having "five charts"; there are four.
* **Remediation:** corrected to "four charts". (Listed in the Phase 2 Must Fix table without a P-number.)
* **Status:** **RESOLVED**

### R-08 · P2-1 / P2-2 · Payment timing could silently delete a payment, and the guarding control could not detect it

* **Sheets / ranges:** `13W Cash Flow` — `C64:O68` (timing flags), new coverage block rows 90–101; `Controls` — `D15`, `B15`, `F15`
* **Original issue:** Each flag tested `DATE(YEAR(week_end), MONTH(week_end), payment_day)` against the week window — looking only at the *week-ending* month. For a week spanning a month boundary, a payment day falling in the week-*starting* month was invisible. Measured against the production calendar: payment days 26–27 lost one payment, 28–30 lost two, and day 31 lost all three. Control #9 summed three flag rows and compared the total to 9, so it could not detect a per-item error and would pass while S$44,000 of rent silently vanished.
* **Root cause:** two independent weaknesses — a timing test that examined only one of the two months a week can span, and an aggregate control that admitted compensating errors.
* **Remediation, both layers:**
  * **Logic.** Each flag now evaluates the payment date from *both* the week-start month and the week-end month, and clamps to the month end so a day of 29–31 settles on the last day of a shorter month instead of rolling into the next one:

    `=IF(OR(AND(pd(start)>=C$6,pd(start)<=C$7),AND(pd(end)>=C$6,pd(end)<=C$7)),1,0)`
    where `pd(r) = MIN(DATE(YEAR(r),MONTH(r),day), EOMONTH(r,0))`

    `OR` rather than addition, so a date visible from both months cannot double-count.
  * **Control.** Control #9 was rewritten rather than added to. A new coverage block (`13W Cash Flow` rows 90–101, outside the print area) independently enumerates five candidate accrual months, computes each item's clamped payment date, and counts how many fall inside the forecast window. Control #9 now asserts that the expected count equals the flags actually raised, **per item**, for all six timing rows. The two computations use different methods — a date list versus a week scan — so the control is genuinely independent, not a restatement of the formula it checks.
* **Verification:** payment day swept across 1, 5, 10, 15, 20, 26, 28, 29, 30 and 31. Every value now produces 3 rent payments totalling S$66,000, coverage difference nil, control #9 PASS. Days 26–31 previously produced 2, 1, 1, 1 and 0 payments respectively. Short-month behaviour confirmed: day 31 resolves to 31 Aug, 30 Sep and 31 Oct. The production assumption (day 1) was restored and re-verified afterwards.

  The control does not assume three payments per item — it computes the expectation from the calendar, so a day such as 2 (which legitimately yields two payments inside this window) also passes.
* **Status:** **RESOLVED**

### R-09 · P2-4 + control gap behind P0-1 · No integrity controls on Scenario 2 and 3 blocks

* **Sheet / range:** `Controls` — new rows 37–39
* **Original issue:** Controls 19–21 tested only that Scenario 1 reproduces the base case. The Scenario 2 and 3 weekly blocks — 66 rows carrying the exit case that drives the central recommendation — had no roll-forward, continuity or completeness control. This is the structural gap that allowed P0-1 to pass thirty green controls.
* **Remediation:** three controls added.
  * **#31 Scenario cash roll-forward** — `opening + receipts − outflows = closing`, every week, all three scenario blocks, via `SUMPRODUCT(ABS(...))` across each block.
  * **#32 Scenario cash continuity** — each week opens on the prior week's close, all three blocks.
  * **#33 Store payroll survives closure** — asserts, for each scenario, that store payroll paid across the 13 weeks is nil *only if* the store never traded. This is the direct guard against P0-1 and is independent of the accrual formula it protects: it compares total payroll less the corporate component against the count of weeks the store was open.
* **Verification:** all three PASS with variance 0.000000. Control #33 returns 1 (FAIL) against the pre-fix logic.
* **Status:** **RESOLVED**

### R-10 · P2-12 · Missing receipts-completeness control

* **Sheet / cell:** `Controls` — new row 40
* **Original issue:** Nothing verified that cash collected reconciles to sales and the debtor movement. A change to any settlement percentage or lag could break collection without a single control failing.
* **Remediation:** Control **#34** asserts the identity

  `net sales − consignment commission + opening debtors − closing debtors − total receipts = 0`

  Closing debtors are derived from the model's own lag parameters — the Week 13 store and e-commerce tails, and the trailing consignment weeks selected by `SUMPRODUCT` on the week-number row so the control follows a change in the lag assumption rather than hardcoding four weeks.
* **Verification:** PASS, variance 0.000000, confirming the receipts logic that Phase 2 verified only by external calculation.
* **Status:** **RESOLVED**

### R-11 · P2-5 · Only 3 of 8 Dashboard KPI cards had link controls

* **Sheet / cell:** `Controls` — new row 41
* **Original issue:** The Dashboard footer asserts "no dashboard number is keyed independently", but controls 23–25 substantiated that for three of eight cards.
* **Remediation:** Control **#35** ties the remaining five KPI cards (lowest cash, cash runway, 13-week net revenue, gross margin, store profit) *and* the three scenario cash cards feeding the scenario chart. The runway comparison is guarded with `ISNUMBER` so the card's legitimate text state ("n/a — cash generative") does not raise a false failure.
* **Verification:** PASS, variance 0.000000. All eight KPI cards and all three scenario cards are now controlled.
* **Status:** **RESOLVED**

### R-12 · P2-3 · Error-scan controls partly defeated, and blind to two sheets

* **Sheet / cells:** `Controls` — `D32`, `D35`, `D36`, `B36`
* **Original issue:** The `ISERROR` sweeps did not cover the Assumptions tab at all (29 formulas, including the derived cash groupings that feed every other sheet) nor the Dashboard chart-data block at rows 100–129 that feeds all four charts.
* **Remediation:** Control #26 extended to `A4:Q101` to cover the new coverage block; control #29 extended to `A3:P186` to cover the new accrual block; control #30 extended to include `Dashboard!A100:C129` and `Assumptions!A6:O124`, with its label updated accordingly.
* **Partially addressed:** the deeper half of P2-3 — that 73 `IFERROR(...,0)` wrappers convert an upstream `#REF!` into a plausible zero before any scan sees it — is **not** fully solved by extending ranges. Controls #33, #34 and #35 now provide positive-value assertions on the most consequential of those paths (payroll, receipts, dashboard links), so a masked error on those chains surfaces as a control failure rather than a clean zero. Blanket positive-value assertions across all 73 sites remain open and are recorded below.
* **Status:** **RESOLVED (range coverage) / PARTIAL (IFERROR masking)**

### R-13 · NEW FINDING · Excel stamped the author's local filesystem path into the workbook

* **Scope:** `xl/workbook.xml`
* **Issue:** Saving through Excel introduced

  `<x15ac:absPath url="C:\Users\<account>\...\retail-fpa-cashflow-model\model\"/>`  *(path redacted)*

  Excel writes the workbook's absolute local directory on every save. On a public repository this publishes the operating-system account name and the local folder layout — precisely the class of leak the Phase 2 privacy audit was checking for, introduced by the remediation itself.
* **Why it matters:** This is a genuine deployment/privacy defect in a file intended for public download, and it is invisible from Excel's user interface. It would have shipped unnoticed had the package not been re-inspected after saving.
* **Remediation:** Removed by targeted XML edit after the Excel save, together with the residual LibreOffice `loext` extension block (R-02). Only those elements were altered; every other part was copied through byte-for-byte and the zip re-verified.
* **Verification:** package re-scan finds no absolute path, no `openpyxl`, no `libreoffice`, no email address and no host name anywhere in any part. Excel reopens the scrubbed file with no repair prompt, all 9 sheets, all 4 charts and all 35 controls passing.
* **Caveat for future saves:** Excel will re-add `absPath` on any subsequent save. The scrub must be repeated before each public release. This is recorded as a release-step requirement, not a one-off fix.
* **Status:** **RESOLVED**

### R-14 · P2-7 · Consignment commission used a different base in P&L and cash

* **Sheets / cells:** `Channel Profitability` `F15`; `Assumptions` `C41`, `B73`
* **Original issue:** The P&L charged commission on gross sales (`=-F5*B41`) while the cash forecast netted it from net sales. The two agreed only because the consignment discount input is 0; any non-zero value would silently diverge the P&L from cash with no control detecting it. `Assumptions!B73` (opening consignment receivable) likewise omitted the discount factor.
* **Remediation:** `F15` changed to `=-F7*Assumptions!$B$41` (net basis, consistent with the cash model and with every other variable cost line); the unit label at `C41` corrected from "% of gross" to "% of net"; `B73` changed to `=B19*(1-B25)*B69*(1-B41)` so the opening receivable follows the discount assumption.
* **Verification:** no change to any reported figure (the discount is 0, so gross and net coincide today), and control #34 now covers the resulting receipts identity. The latent divergence is removed.
* **Status:** **RESOLVED**

---

## Control Improvements

**30 → 35 controls**, plus three upgraded in place. The status roll-up (`C4`) and count (`D4`) were made dynamic — `D4` previously hardcoded "of 30" — so adding a control can no longer leave the header wrong.

| # | Control | Type | Guards |
|---|---|---|---|
| 9 | Payment completeness — every configured payment date captured exactly once | **upgraded** | P2-1, P2-2 |
| 26 | Error scan, 13W Cash Flow — range extended to `A4:Q101` | **upgraded** | P2-3 |
| 29 | Error scan, Scenarios — range extended to `A3:P186` | **upgraded** | P2-3 |
| 30 | Error scan — now includes Dashboard chart data and Assumptions | **upgraded** | P2-3 |
| 31 | Scenario cash roll-forward, every week, all three scenarios | **new** | P2-4 |
| 32 | Scenario cash continuity, all three scenarios | **new** | P2-4 |
| 33 | Store payroll survives closure | **new** | **P0-1** |
| 34 | Receipts completeness | **new** | P2-12 |
| 35 | Dashboard links — remaining five KPI cards and three scenario cards | **new** | P2-5 |

Two supporting blocks were added, both **outside their sheet's print area** so neither appears in printed output or screenshots:

* `13W Cash Flow` rows 90–101 — payment date coverage test
* `Scenarios` rows 180–185 — payroll accrual share

Control #9 was rewritten rather than supplemented, in line with the instruction to prefer upgrading an existing control over adding a weak new one. No control was added merely to raise the count.

---

## Before / After Results

Only Scenario 3 changed. Scenario 1 and Scenario 2 are numerically identical to Phase 2, and the monthly run-rate P&L is unchanged for all three scenarios — correctly, since it is a steady-state view and rightly excludes a one-off transitional payment.

### Scenario 3 — Store Exit / Channel Shift

| Measure | Phase 2 | Phase 3 | Change |
|---|---:|---:|---:|
| Store payroll paid over 13 weeks | 0.00 | **17,903.23** | +17,903.23 |
| Total cash outflows | 701,036.28 | **718,939.51** | +17,903.23 |
| Net cash movement | (76,517.42) | **(94,420.65)** | −17,903.23 |
| **Week 13 closing cash** | 128,482.58 | **110,579.35** | −17,903.23 |
| **Lowest projected cash** | 97,330.40 | **79,427.18** | −17,903.23 |
| Headroom at the lowest point | (22,669.60) | **(40,572.82)** | −17,903.23 |
| **Weeks below the minimum buffer** | 4 | **6** | +2 |
| Week of lowest cash | 11 | 11 | — |
| Monthly run-rate operating profit | 27,088.02 | 27,088.02 | — |

### Unchanged (confirming the fix is surgical)

| Measure | S1 | S2 |
|---|---:|---:|
| Week 13 closing cash | 139,650.22 | 163,301.64 |
| Lowest projected cash | 114,528.43 | 130,288.23 |
| Weeks below buffer | 1 | 0 |
| Total payroll over 13 weeks | 134,100.00 | 127,440.00 |
| Monthly run-rate operating profit | 20,195.98 | 27,343.50 |

### Does the corrected result change the decision story?

**Yes — and it strengthens the existing recommendation rather than overturning it.**

Under Phase 2 figures, the exit case ended the quarter at S$128,483, comfortably above the S$120,000 buffer, and spent 4 of 13 weeks below it. Corrected, exit **ends the quarter at S$110,579 — below the minimum cash buffer — and spends 6 of 13 weeks under it.**

The decision therefore no longer rests on the marginal S$255/month operating-profit difference between improvement and exit, which was always too small to carry a recommendation. It now rests on a clear liquidity distinction: improvement never breaches the buffer, as-is breaches it once, and exit breaches it for nearly half the quarter and does not recover by Week 13. That is a materially better-founded basis for "improve the store; keep exit as the fallback" than the model previously had.

The commentary was updated to lead with this (R-03 and the Insight 7 rationale, which was static prose quoting the 25.8%/25.0% migration figures and is now a live reference). The "Preferred scenario" banner still selects Scenario 2 on operating profit; that conclusion is now better supported than before, though the single-metric selector itself (P2-11) remains an open finding.

---

## Remaining Findings

### P1 — none outstanding

P1-1, P1-2, P1-4 and P1-5 resolved. **P1-3 withdrawn** as not a defect on verification (R-06).

### P2 — 7 of 12 resolved; 5 outstanding

| ID | Finding | Disposition |
|---|---|---|
| P2-3 | `IFERROR` masking (range coverage fixed; blanket positive-value assertions not) | **Partial** — most consequential paths now covered by #33/#34/#35 |
| P2-6 | No data validation anywhere | **Deferred to Phase 4** — the silent-deletion risk it protected against is now closed at the logic layer (R-08) and by control #9, so this is a usability improvement rather than a correctness one |
| P2-8 | Scenarios print area `A1:M79` truncates the merged banner in `A3:P3` and note in `A79:O79` | **Deferred to Phase 4** — presentation only; note that `A79` is now longer than before, so this should be fixed before any PDF export |
| P2-9 | Inventory note implies a reconciliation that does not hold (spend is 136% of COGS) | **Deferred to Phase 4** — documentation wording |
| P2-10 | KPI card label/value alignment disagree | **Deferred to Phase 4** — cosmetic |
| P2-11 | "Preferred scenario" decided on a 0.9% difference by a single metric | **Deferred, deliberately** — the corrected cash figures now strongly support the same selection, so this is a framing improvement rather than a wrong answer |

### P3 — all 9 outstanding, all deferred

P3-1 (parametric P&L note), P3-2 (`4.3333` hardcode), P3-3 (sensitivity axis labels), P3-4 (off-sheet hardcodes), P3-5 (tab order), P3-7 (seasonality centring control), P3-8 (store payroll per head), P3-9 (no worksheet protection) — all optional. **P3-6 (hardcoded "of 30" in the control count) was resolved incidentally** when the control count was made dynamic.

### New, carried forward

| Item | Severity | Disposition |
|---|---|---|
| Excel re-adds `x15ac:absPath` on every save | **Release-step requirement** | The package scrub must be re-run before each public release. Recorded in R-13. |
| `xl/printerSettings/printerSettings1.bin` (5.4 KB) added by Excel when the Controls print area was set | Cosmetic | Contains only "Microsoft Print to PDF", a standard Windows virtual printer — no host name, account name or PII. Left in place; removing it would require editing the sheet relationship and `[Content_Types].xml` for negligible benefit. |
| Dashboard `A17` sits at 91% of one line | Phase 4 | Formula-generated text; a future assumption change could push a wrap against the fixed 24pt row height. |

---

## Validation Results

All tests run against the released production file in native Microsoft Excel.

| Test | Result |
|---|---|
| **Scenario 1 reconciliation to base forecast** | Max absolute difference **2.910 × 10⁻¹¹** across receipts, outflows, net movement, closing cash and net sales — floating point only |
| **Cash roll-forward, base sheet** | 13/13 weeks, zero breaks (control #1) |
| **Cash roll-forward, all three scenario blocks** | 39/39 week-checks, zero breaks (control #31) |
| **Cash continuity, all three scenario blocks** | 36/36 checks, zero breaks (control #32) |
| **Scenario 3 payroll** | August payroll now paid: S$17,903.23; store traded 4 weeks, payroll non-nil (control #33 PASS) |
| **Payment timing edge cases** | Days 1, 5, 10, 15, 20, 26, 28, 29, 30, 31 all resolve to 3 rent payments totalling S$66,000, coverage difference nil, control #9 PASS in every case. Short months handled by month-end clamping. Production assumption (day 1) restored and re-verified. |
| **Controls** | **35 of 35 PASS**, zero FAIL, every variance 0.000000. MODEL STATUS reads PASS. |
| **Formula error scan** | **0 errors** across all 9 sheets — no `#REF!`, `#DIV/0!`, `#N/A`, `#VALUE!`, `#NAME?`, `#NUM!`, `#SPILL!` or `#CALC!`. Verified live in Excel by iterating every formula cell via `SpecialCells`, independently of the workbook's own error-scan controls. |
| **Circular references** | None. Calculation completed with `CalculationState = xlDone`; no circular-reference warning raised. |
| **Excel recalculation** | `CalculateFullRebuild` executed before save; calculation mode Automatic; state `xlDone` |
| **Reopen in Excel** | Opens clean — **no repair prompt**, no external-link prompt. 9 sheets, 4 charts, 16 defined names, 0 hidden sheets, all formulas and formatting intact. Closed cleanly. |
| **External links / connections / VBA / pivot caches** | None — `LinkSources` returns NONE; no `externalLink`, `connections.xml`, `vbaProject.bin` or pivot cache parts |
| **Metadata** | Author `siddik-analytics`; Last modified by `siddik-analytics`; Application `Microsoft Excel` (16.0300); `fileVersion appName="xl"`; Title and Subject populated; Company and Manager empty; creation date unaltered |
| **Privacy re-scan (package XML)** | **Clean** — no absolute paths, no `openpyxl`, no `libreoffice`, no email addresses, no host or account names in any part |
| **Zip integrity after scrub** | `testzip()` OK; 28 parts |

### Safety record

Working tree was clean before modification. The Phase 2 baseline was hashed (`9bf597be…`) and backed up outside the repository to the session scratchpad; the backup was not committed. The full remediation was rehearsed on scratch copies, then the production file was restored to baseline and the entire change applied in one deterministic pass. No temporary or trial file was written into the repository.
