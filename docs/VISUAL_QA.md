# Visual & UX QA

**Artifact:** `model/Atelier_North_FPA_Model.xlsx`
**Phase:** 4 — presentation and usability pass
**Date:** 3 September 2026
**Companion documents:** [DEPLOYMENT_AUDIT.md](DEPLOYMENT_AUDIT.md) · [REMEDIATION_LOG.md](REMEDIATION_LOG.md)

| | Before | After |
|---|---|---|
| SHA-256 | `a8934e8828f93c540637983cc7d4cceb6ad725dbf41a35fee2f934bc959408ec` | `52afb30c1c20f3a96bc5e9bb15ae01a9f74bf3bfef1b4238988e8824983b7677` |
| Size | 117,633 bytes | 118,686 bytes |
| Controls | 35 of 35 PASS | 35 of 35 PASS |
| Formula errors | 0 | 0 |

**No financial logic was changed.** Every scenario, cash, profitability and break-even output is numerically identical to the Phase 3 release — verified cell-by-cell after formatting (see Final QA).

All modification was performed through native Microsoft Excel COM automation, recalculated with `CalculateFullRebuild`, and saved by Excel as `xlsx`. LibreOffice was not used; openpyxl was used only for read-only package inspection.

### Method note — this pass was driven by rendered screenshots, not by reading cells

Every management-facing sheet was exported to PNG from native Excel and inspected visually before and after the changes. That is how the two genuine defects below were found; neither is visible from formulas or cell values, and both would have shipped otherwise.

---

## Design System

The workbook already had a coherent visual language. This pass enforced it where it had drifted rather than replacing it.

### Typography

| Role | Treatment |
|---|---|
| Workbook title | Calibri 16pt bold, navy `#142F4B` |
| Sheet subtitle | Calibri 11pt, slate `#5B6B7B` |
| Section header | Calibri 10pt bold, navy on a pale blue-grey band `#E8EEF4` |
| Column header | Calibri 9.5pt bold, navy, bottom rule |
| Row label | Calibri 9–10pt, near-black `#1A1A1A`, indented by hierarchy level |
| KPI label | Calibri 8.5pt bold, slate, uppercase |
| KPI value | Calibri 15pt bold, navy |
| KPI caption | Calibri 8.5pt, slate |
| Note / footnote | Calibri 8.5–9pt, slate |

**Font decision — Calibri retained, Aptos not adopted.** The brief recommended Aptos. I kept Calibri deliberately:

* the workbook is already **100% Calibri** across all nine sheets, so the actual requirement — one coherent typeface with a clear hierarchy — is already met;
* Aptos has different metrics from Calibri, and every column width and row height in this workbook was tuned to Calibri. Switching would silently re-wrap text across nine sheets. This pass **found two live text-clipping defects**, so introducing a workbook-wide metric change at the same time would have been reckless;
* Calibri renders identically on every Excel version a reviewer might open the file with; Aptos falls back on pre-2024 installs.

This is a reversible preference. If you want Aptos, it should be its own pass with a full re-measure of column widths and row heights afterwards.

### Colour system

| Purpose | Colour | Where |
|---|---|---|
| Primary headers, key values | Navy `#142F4B` / `#1F4E79` | titles, KPI values, totals |
| Secondary structure | Pale blue-grey `#E8EEF4` | section bands, subtotal rows |
| Secondary information | Slate `#5B6B7B` | captions, notes, weekly-average column |
| Editable input | Pale amber `#FFF6DA` fill + navy text | Assumptions column B, Scenario levers |
| Cross-sheet link | Muted green `#2A6B4F` text | values pulled from another sheet |
| Genuine risk | Dark red `#9C2B2B` on pale `#F8E4E4` | below-buffer cash, negative KPI values |
| Caution | Amber `#8A5D0F` on `#FBF1DC` | within 15% of the buffer, weeks below threshold |
| Genuine positive | Green `#2E6B38` on `#E4EFE4` | best scenario, PASS status |

No gradients, no icon sets, no data bars, no colour scales. Conditional formatting totals 25 rules across 18 ranges — proportionate rather than blanket.

### Input convention

Pale amber fill with navy text marks **the only cells intended to be edited** — 140 on Assumptions, 45 scenario levers, and a small number of local sensitivity-axis inputs. Verified programmatically in Phase 2: zero hardcoded inputs are styled as formulas and zero formulas are styled as inputs. Historical source data is *not* amber, because it is not intended to be edited.

### Number format convention

| Type | Format | Renders |
|---|---|---|
| Currency (headline) | `"S$"#,##0;"(S$"#,##0);-` | `S$139,650` / `(S$1,105)` / `-` |
| Currency (in-table) | `#,##0;(#,##0);-` | `139,650` / `(1,105)` / `-` |
| Percentage | `0.0%;(0.0%);-` | `52.4%` / `(2.5%)` |
| Multiple | `0.00x` | `1.30x` |
| Weeks / months | `0.0" wks"` / `0.0" mths"` | `27.8 wks` |
| Date | `dd mmm yy` | `03 Aug 26` |
| Sensitivity axis | `"Sales "+0%;"Sales "-0%;"Sales unchanged"` | `Sales +10%` / `Sales unchanged` |

Negatives are shown in parentheses throughout and zeros as a hyphen. No cents anywhere. Red is applied by conditional formatting on the rows where a negative is a *warning* (cash movement, headroom, KPI results) and deliberately **not** on P&L cost lines, where every figure is negative by construction and colouring them all would be noise.

### Warning convention

Restrained and targeted. Cash below the buffer is a pale red cell, not a filled row. Cash within 15% of the buffer is amber. The lowest-cash week is marked by tinting only that week's column header. A negative KPI is a red number on a neutral card — never a red card.

---

## Sheets Reviewed

Sheet order was changed to follow the management story rather than the calculation order:

`README → Dashboard → Management Insights → 13W Cash Flow → Scenarios → Channel Profitability → Store Break-Even → Assumptions → Controls`

The file opens on the Dashboard. Tab colours already encoded function (outputs / calculations / inputs / controls) and were left alone.

| Sheet | Purpose | Changes made | Remaining concerns |
|---|---|---|---|
| **README** | Model guide, conventions, limitations | None needed — already clean, gridlines off, print area correct | None |
| **Dashboard** | One-page executive summary | KPI value alignment fixed; row/column headings hidden; singular/plural fixed; chart axis collision fixed | None |
| **Management Insights** | Seven findings with consequence and action | **Row clipping fixed** (see below) | Row 24 is now 90pt — tall, but proportionate to the content it holds |
| **13W Cash Flow** | Rolling 13-week forecast | None needed — reviewed in full and already meets the target | Minimum-threshold row repeats 120,000 thirteen times; needed for the chart series, kept greyed |
| **Scenarios** | Levers, run-rate P&L, cash outcomes, thresholds | Print area extended so the banner and trade-off note no longer truncate; singular/plural fixed | Preferred-scenario banner is still a single-metric selector (audit P2-11) |
| **Channel Profitability** | Six-month management P&L by channel | None needed — hierarchy, subtotals, sign conventions and margin placement already correct | Presented as actuals but parametric by construction (audit P3-1) |
| **Store Break-Even** | Store viability and sensitivities | Sensitivity column axes made self-labelling | None |
| **Assumptions** | 16 sections of central inputs | Input fill cleared from 16 empty spacer cells where it bled through | No data validation (audit P2-6, deferred) |
| **Controls** | 35 integrity checks | **Row clipping fixed**, category label un-truncated, PASS de-emphasised (see below) | Minor unevenness in single-line row heights after auto-fit |

---

## Dashboard QA

The Dashboard is the primary portfolio asset and was reviewed element by element against a rendered export.

**What was already right and left alone:** the two-band KPI layout (cash first, then trading); eight cards on a consistent grid with 1.8-character spacer columns providing real whitespace; live-linked values with no independently keyed numbers; four charts on a balanced 2×2; restrained conditional formatting; A3 landscape print area fitting to one page; a footer stating that every figure links to the calculation sheets.

**Changes made:**

1. **KPI card alignment.** Card labels and captions were left-aligned while the value was centre-aligned, so the number floated in the middle of a left-aligned card. All eight values moved to left alignment, giving each card a single clean left rail. *(Closes audit P2-10.)*

2. **Row and column headings hidden.** The Dashboard is a report page, not a worksheet to navigate. Hiding the A/B/C and 1/2/3 headings removes spreadsheet furniture from the primary screenshot. Applied to the Dashboard only — every other sheet keeps its headings so the model stays navigable and auditable.

3. **Singular/plural.** The lead conclusion read "below the minimum buffer in 1 week(s)". The `(s)` construction is a tell of generated text. Now `IF(...=1," week"," weeks")`, rendering "in 1 week". The equivalent fix was applied to the Scenarios banner.

4. **Negative KPI treatment confirmed, not changed.** Store Profit `(S$1,105)` and Store Break-Even Gap `(S$2,108)` render as dark red numbers on the standard neutral card. The brief asked specifically that these not become giant red blocks; they already followed the preferred pattern, so they were left as they are. Both were checked for clipping at the rendered size and neither clips — the card is ~29 characters wide against values under 12.

**Confirmed non-issue.** Audit finding P1-3 claimed the four "WHAT THIS MEANS" conclusion rows would clip. Phase 3 measurement disproved it and the Phase 4 render confirms it again: the longest string occupies 790pt of 864pt available and sits on one line. No row heights were changed.

---

## Chart QA

All four charts reviewed against a rendered export.

| Chart | Type | Assessment | Action |
|---|---|---|---|
| **Closing cash vs minimum buffer (SGD)** | Line, legend bottom | Correct type for 13 weekly points. Zero-based value axis, no manipulation of scale. Buffer as a dashed red reference line reads immediately. Week labels W1–W13 unambiguous. | None |
| **Contribution vs direct channel profit, per month (SGD)** | Clustered bar, legend bottom | Carries the model's core argument. Zero baseline present so the negative Physical Store bar reads correctly. **Defect: the `(1,105)` data label for the negative bar collided with the category axis labels.** | **Fixed** — category tick labels moved to `xlTickLabelPositionLow`, placing them below the plot area clear of the negative label. Verified on re-render. |
| **Store monthly sales vs break-even (SGD)** | Bar, no legend | Two bars, legend correctly omitted. The 2.5% gap looks small because it *is* small — the axis is zero-based and was not truncated to exaggerate it. | None |
| **Scenario cash outcomes (SGD)** | Clustered bar, legend bottom | Ending cash and lowest cash per scenario, consistent series colours, no label collisions, scenario names in full. Shows the corrected Phase 3 exit-case figures. | None |

No 3-D effects, no pie charts, no gradients, no redundant legends, no unexplained abbreviations. All four titles carry the currency unit.

---

## Defects Found and Fixed

Both were invisible from cell values and were found only by rendering the sheets.

### 1. Management Insights — text clipped mid-sentence

Content rows were at a fixed 55.5pt with wrapped text that needed more. Three cells were cut off:

* Insight 1, *Why it matters* — ended at "...turns a policy breach" with "into a funding problem." hidden;
* Insight 7, *Why it matters* — ended mid-sentence;
* Insight 7, *Recommended action* — "with evidence." hidden.

Insight 7 is text **I lengthened in Phase 3** when adding the corrected buffer-breach figures, so this was a defect introduced by the previous phase and caught here.

**Fix:** auto-fitted rows 6, 9, 12, 15, 18, 21, 24 with 3pt of breathing room. Row 6 went 55.5 → 65.5pt, row 24 → 90.5pt. Re-rendered and confirmed every insight now reads to completion.

### 2. Controls — new control rows clipped and overlapped

The five controls added in Phase 3 (rows 37–41) inherited a single-line row height from the row they were format-copied from, but their labels wrap to two lines. Control 31's text was clipped and visually overlapping control 30 above it. The category label also rendered as "SCENARIO & COMP", truncated by the column width.

**Fix:** auto-fitted rows 7–41 with a floor of 18pt; set the category cell to wrap within its five-row merge. Control 31's row went 15 → 27pt.

### 3. Controls — PASS was a wall of green

The result column applied a green fill to all 35 rows, so a passing model rendered as a solid green block and a single FAIL would have been easy to miss. Per the brief, PASS now renders as bold green **text with no fill**, and FAIL keeps a solid pale-red fill with bold red text. The single `MODEL STATUS` badge retains its green fill, which is where the reader should look first.

### 4. Scenarios — print area truncated its own headline

Print area was `A1:M79` while the dynamic "Preferred scenario" banner is merged `A3:P3` and the trade-off note `A79:O79` — both extending past column M, and the note is longer since Phase 3. Extended to `A1:P79`. *(Closes audit P2-8.)*

### 5. Store Break-Even — unlabelled sensitivity axes

Both matrices had a labelled row axis but bare percentages on the column axis, leaving the reader to infer the dimension from the section title. Rather than consuming a layout row, the column headers now self-label through their number format: `Sales -20% … Sales unchanged … Sales +20%` and `Rent -20% … Rent unchanged`. *(Closes audit P3-3.)*

### 6. Assumptions — input fill bled through spacer rows

The amber input fill ran continuously down column B including blank separator rows, making the input block look like one long strip rather than grouped sections. Cleared from 16 empty cells.

---

## Screenshot Readiness

No screenshots were created. These are the recommended captures for the public repository.

| # | Sheet | Recommended viewport | Shows | Why it earns its place |
|---|---|---|---|---|
| 1 | **Dashboard** | `A1:O53` at 90% zoom (headings already hidden) | Title, 8 KPI cards, 4 narrative conclusions, all 4 charts | The hero image. Proves in one frame that the builder can turn a model into a decision-ready one-pager: liquidity position, trading position, what it means, and the scenario choice. |
| 2 | **13W Cash Flow** | `A1:Q41` at 85% zoom (freeze panes at C8 already set) | Week headers and dates, receipts by channel, 15 outflow lines, net movement, closing cash, buffer, headroom | The engine. Shows channel-specific settlement lags, calendar-driven payment timing and a visible liquidity breach in Week 9 — the substance behind the dashboard. |
| 3 | **Scenarios** | `A55:P79` at 85% zoom | 13-week cash outcome comparison across three scenarios, decision thresholds, trade-off note | The analytical high point. Shows the migration break-even (25.8% vs 25.0% assumed) and exit payback — genuine decision analysis, not a percentage overlay. |
| 4 | **Channel Profitability** | `A1:H45` at 90% zoom | Revenue → COGS → gross profit → contribution → direct channel profit → shared overhead → operating profit, by channel | Shows the contribution-first architecture and the deliberate choice not to allocate overhead. This is the judgement an FP&A reviewer looks for. |
| 5 | **Management Insights** | `A1:C26` at 95% zoom | All seven findings with consequence and action | Proves the builder can convert a model into management communication. Every number is a live link. |

Optional sixth: **Controls** `A1:G43` at 95% zoom — 35 checks all passing, which answers "how do I know this model is right?" before the reviewer asks.

For each, capture at 100% Windows display scaling. The Dashboard prints to one A3 landscape page; the others fit to width.

---

## Final QA

| Check | Result |
|---|---|
| Workbook reopened in Excel | **OK — no repair prompt, no external-link prompt, no unexpected dialogs** |
| Broken links | None (`LinkSources` returns NONE) |
| Calculation mode | Automatic; `CalculateFullRebuild` run before save; state `xlDone` |
| Controls | **35 of 35 PASS**, zero FAIL |
| Formula error scan | **0 errors** across all 9 sheets — no `#REF!`, `#DIV/0!`, `#N/A`, `#VALUE!`, `#NAME?`, `#NUM!`, `#SPILL!`, `#CALC!`. Verified live by iterating every formula cell via `SpecialCells`. |
| Charts intact | 4 |
| Defined names | 16 (9 print areas + 7 print titles); Scenarios and Controls updated, none added or orphaned |
| Sheet order | README → Dashboard → Management Insights → 13W Cash Flow → Scenarios → Channel Profitability → Store Break-Even → Assumptions → Controls |
| Opens on | Dashboard, cell A1 |
| Metadata | Author `siddik-analytics`; Last modified by `siddik-analytics`; Application `Microsoft Excel`; Company and Manager empty; creation date unaltered |
| Package privacy scan | **Clean** — no absolute paths, no `openpyxl`, no `libreoffice`, no emails, no host or account names |
| `absPath` re-stamp | Excel re-added the author's local path on save, as predicted in the Phase 3 log. **Scrubbed again.** This remains a required release step on every future save. |
| Printer settings blobs | Two present (5,428 bytes each), containing only "Microsoft Print to PDF" — a standard Windows virtual printer, no host or account name |
| Headers / footers | Company label, sheet name and page number only — **no filename, no file path** |
| Financial outputs | **Identical to Phase 3** — S1/S2/S3 Week 13 cash 139,650.22 / 163,301.64 / 110,579.35; lowest cash 114,528.43 / 130,288.23 / 79,427.18; weeks below buffer 1 / 0 / 6; Scenario 1 still reproduces the base forecast to 2.9 × 10⁻¹¹ |

### Outstanding issues

| Item | Severity | Disposition |
|---|---|---|
| Controls single-line row heights slightly uneven after auto-fit | Cosmetic | Visible only on close inspection; rows range roughly 18–21pt. Not worth another formatting pass. |
| Management Insights row 24 is 90pt | Cosmetic | Tall, but proportionate — insight 7 carries the most content. Alternative would be shortening validated commentary, which is not a formatting decision. |
| Assumptions has no data validation | Audit P2-6 | Deferred. The correctness risk it guarded was closed at the logic layer in Phase 3. |
| Preferred-scenario banner is a single-metric selector | Audit P2-11 | Deferred deliberately. The corrected Phase 3 cash figures now strongly support the same selection. |
| Inventory note implies a reconciliation that does not hold | Audit P2-9 | Deferred — wording, not formatting. |
| Channel P&L is parametric but presented as actuals | Audit P3-1 | Deferred — a one-line disclosure, best handled with the README in a later phase. |
| `ScrollArea` not set on any sheet | By design | `ScrollArea` is not persisted in `xlsx` without VBA, so setting it would give a false sense of restriction. Not attempted. |

**No new calculation issue was discovered during the visual pass.** Every figure inspected on screen reconciled to the Phase 3 validated outputs.
