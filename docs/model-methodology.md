# Model Methodology

How the Atelier North model is built, and why it is built that way. All figures quoted here come from the published workbook.

---

## Business Context

Atelier North Pte. Ltd. is a synthetic independent fashion retailer in Singapore. It is owner-managed, sells through four channels — a flagship store, e-commerce, pop-up events and consignment partners — and turns over roughly S$1.48m of gross sales in a six-month period.

The business is profitable. It earns a 59.8% gross margin and an 8.8% operating margin. That is not the problem.

Two things are:

**A near-term cash squeeze.** A S$235,000 autumn/winter buy has to be paid across the same 13 weeks in which quarterly GST falls due. On the current forecast, cash drops to S$114,528 in Week 9 — S$5,472 below the board's S$120,000 minimum operating buffer — before recovering to S$139,650 by Week 13.

**A flagship store that does not cover its own costs.** The store generates the highest contribution margin of any channel at 52.4%, but after the rent and payroll it must clear before the first sale counts, it loses S$1,105 a month. It sits 2.5% below its own break-even.

The decision the model exists to answer:

> **How should management navigate a near-term liquidity constraint while addressing the economics of an underperforming physical store?**

Those two questions interact. Closing the store improves the run-rate cost base but consumes cash immediately — at precisely the moment the business has least of it. A model that answers one question without the other gives the wrong advice.

---

## Model Architecture

Nine worksheets, arranged so that information flows in one direction.

```
Assumptions
    ↓
Channel economics  ──→  Store break-even
    ↓                        ↓
13-week cash forecast   ────┘
    ↓
Scenario engine  (rebuilds both P&L and cash, three times)
    ↓
Dashboard  ·  Management Insights
    ↓
Controls  (tests every layer above)
```

**Assumptions** holds every hardcoded input in sixteen labelled sections — run-rates, margins, cost ratios, settlement terms, payment dates, the purchase schedule and the historical sales table. Each input carries its unit and, where the number is not self-evident, a note explaining where it came from. This is the only sheet a user needs to edit.

**Channel economics** converts six months of historical sales into a management P&L by channel, ending at contribution profit and then direct channel profit.

**The 13-week cash forecast** is the operating engine. It builds weekly sales from run-rates and seasonality, converts them to receipts through channel-specific settlement lags, and lays payments onto the calendar dates they actually fall on.

**Store break-even** takes the store's monthly economics and asks how far it is from covering its own fixed costs, then tests which drivers actually move that answer.

**The scenario engine** rebuilds the monthly run-rate P&L *and* a full 13-week weekly cash path for each of three courses of action. It does not apply percentage adjustments to the base case; it reconstructs it.

**Dashboard and Management Insights** are the output layer. Every number on both is a live link — no figure is keyed independently, and every sentence of commentary is generated from the cells it describes.

**Controls** tests the layers above it: 35 checks covering cash integrity, channel totals, break-even logic, scenario reconciliation, dashboard agreement and formula errors.

---

## 13-Week Cash Forecast

The forecast runs from Monday 3 August 2026 to Sunday 1 November 2026, opening on a bank balance of S$205,000.

**Revenue and cash are modelled separately.** Weekly net sales are built first — gross run-rate × a weekly seasonality index, less discounts and returns. Cash receipts are then derived from those sales through each channel's own settlement behaviour:

| Channel | Cash conversion |
|---|---|
| Physical store | 92% banked in the week of sale, the balance the following week (card settlement) |
| E-commerce | 85% in the week of sale, the balance the following week (gateway payout cycle) |
| Events / pop-ups | 100% at the event |
| Consignment | 4-week lag, and received **net of the 30% partner commission** |

Opening receivables carry the tail of pre-forecast trading into Weeks 1–4, so the first weeks are not artificially light. Consignment commission never appears as a cash outflow because it is deducted at source — modelling it as a payment would double-count it.

**Payments are placed by calendar date, not by week number.** Each cost line carries a payment day, and the model works out which forecast week that date lands in:

| Line | Timing |
|---|---|
| Rent and monthly sundries | 1st of the month |
| Term loan instalment | 5th |
| Software and brand marketing | 10th |
| Utilities and service charge | 15th |
| Professional fees | 20th |
| Payroll | Last day of the month |
| GST | Single quarterly settlement |
| Inventory | Weekly replenishment plus four scheduled purchase-order payments |

This matters more than it sounds. Because timing is derived from the week dates rather than hardcoded week numbers, rolling the forecast forward is a change to one date. The payment tests also handle weeks that straddle a month boundary and clamp a payment day of 29–31 to the last day of a shorter month, so a payment cannot silently disappear from the forecast.

Each week then resolves as:

```
opening cash + receipts − payments = closing cash
closing cash → next week's opening cash
```

Closing cash is compared with the S$120,000 minimum operating buffer every week, and the sheet reports headroom, the lowest point, the week it occurs, and the number of weeks below the threshold.

---

## Channel Profitability

Channels are judged in two stages, because a single profit number would answer the wrong question.

```
Gross sales
  − discounts and returns
= Net revenue
  − cost of goods sold
= Gross profit
  − variable channel costs      (merchant fees, fulfilment, commission,
                                 variable labour, channel marketing, event costs)
= CONTRIBUTION PROFIT
  − direct fixed costs          (rent, base payroll, utilities, platform costs —
                                 only what disappears if that channel closes)
= DIRECT CHANNEL PROFIT
  − shared overhead             (held back, in total only)
= Operating profit
```

For the six months to 31 July 2026:

| Channel | Net revenue | Gross margin | Contribution margin | Direct channel profit / month |
|---|---:|---:|---:|---:|
| Physical store | 496,320 | 58.0% | 52.4% | **(1,105)** |
| E-commerce | 501,600 | 60.0% | 41.6% | 28,478 |
| Events / pop-ups | 174,600 | 64.0% | 44.1% | 12,833 |
| Consignment | 204,000 | 60.0% | 28.5% | 9,690 |
| **Total** | **1,376,520** | **59.8%** | **43.9%** | **49,896** |

**Why the two-stage view matters.** The store has the best contribution margin in the business and the worst result. Ranking channels on gross margin would keep the store and starve e-commerce. Ranking them on a fully-absorbed profit — after pushing a share of head-office cost into each channel — would make the store look far worse than it is and would change a decision without changing a single cash flow.

Direct channel profit is the number that answers "what happens if this channel closes?", because it charges each channel only the fixed costs that would genuinely go away. Shared overhead is therefore held back from the primary view. A fully-absorbed version is shown separately as a memo, with a check proving the allocation sums back to the total.

---

## Store Break-Even

The store's break-even is built from the costs that actually behave the way the label claims.

Contribution margin is gross margin (58.0%) less the costs that move with store sales — merchant fees, casual labour and other direct costs (5.6% combined) — giving **52.4%**. Fixed store costs are rent, base payroll, utilities, POS and occupancy: **S$44,450 a month**.

```
Break-even net sales = fixed store costs ÷ contribution margin %
                     = 44,450 ÷ 52.4%
                     = S$84,828 per month
```

Against current net sales of S$82,720, that leaves a gap of **S$2,108 a month — 2.5% below break-even**, and a monthly result of **(S$1,105)**.

The sheet also expresses the same gap three other ways, because different levers are available to different managers: a **2.5% sales uplift**, a **1.3 percentage-point gross-margin improvement**, or a **5.0% rent reduction**. All three close it.

Two sensitivity grids test profit against sales and margin, and break-even sales against margin and rent. A driver ranking then compares a proportionate ±10% move in each lever, which is the part that settles the argument: sales and gross margin move store profit roughly **2.0 to 2.6 times** as hard as the same proportionate move in rent or base payroll. The store's problem is a trading problem, not a cost problem — and a forward-looking cross-check against the 13-week forecast run-rate agrees, putting the store 1.9% below break-even on forecast trading.

---

## Scenario Engine

Three courses of action, each driven by fifteen explicit levers.

**1. Current Operations** — the base case. All levers neutral. This scenario exists to prove the engine: with neutral inputs it reproduces the base 13-week forecast exactly, which the control framework tests directly.

**2. Store Improvement** — a trading recovery. Store sales +6%, gross margin +1.5 percentage points, base payroll −12% through reduced hours on the two quietest days, other store fixed costs −8%, e-commerce +2%.

**3. Store Exit / Channel Shift** — closure at the end of Week 4, with 25% of lost store demand migrating online, digital marketing spend increased 1.3× to recapture it, S$96,000 of one-off exit costs settled in two instalments, a S$42,000 inventory clearance inflow, and a smaller forward buy thereafter.

**Scenarios rebuild the mechanics; they do not apply percentages to the base case.** Each carries a full 33-row weekly block that reconstructs sales by channel, cost of goods, receipts through the same settlement lags, and every payment line. The exit case in particular models the things that make closure expensive before it becomes cheap:

- **Closure timing** — a weekly trading flag switches the store off from the week after closure, so store sales, variable labour, rent and occupancy stop on the right date rather than at a month boundary.
- **Payroll obligations** — store payroll is earned in the month worked, not the week it is paid. Closing the store at the end of Week 4 does not extinguish the August wages that fall due in Week 5. The model settles them.
- **Exit costs** — lease break, reinstatement and redundancy, phased across two weeks rather than assumed on day one.
- **Sales migration** — migrated revenue carries e-commerce economics, including the higher marketing cost of recapturing it. It is not free revenue.
- **Inventory release** — a one-off cash inflow from clearing store stock, treated as cash and not as revenue.
- **Fixed-cost savings** — real, permanent, and correctly timed.

### What the scenarios show

| | 1. Current Operations | 2. Store Improvement | 3. Store Exit |
|---|---:|---:|---:|
| Monthly run-rate operating profit | 20,196 | **27,343** | 27,088 |
| Week 13 closing cash | 139,650 | **163,302** | 110,579 |
| Lowest projected cash | 114,528 | **130,288** | 79,427 |
| Weeks below the S$120,000 buffer | 1 | **0** | 6 |

On run-rate profit, improvement and exit are S$255 a month apart — inside the noise. The decision is therefore settled by cash and by risk, not by profit.

The sheet closes with the thresholds that would change the answer. Exit needs **3.6%** of store demand to migrate online merely to match standing still, but **25.8%** to match the improvement case — against the 25.0% the exit case assumes. Net of the inventory release, exit costs S$54,000 in cash and pays back in 7.8 months.

---

## Control Framework

The workbook carries **35 controls**, each with an expected value, a live result, a variance, an explicit tolerance and a pass/fail. A single `MODEL STATUS` cell reads PASS only when every one of them passes. Tolerances are S$0.50 on value reconciliations and 0.001 on counts and flags — a variance outside that is a modelling error, not a rounding artefact.

| Domain | What is proven |
|---|---|
| Cash roll-forward | Opening + receipts − payments = closing, every week; each week opens on the prior week's close; Week 1 ties to the assumed bank balance |
| Payment completeness | Every configured payment date is captured exactly once, tested against an independent calendar enumeration rather than against the same logic that places them |
| Receipts completeness | Cash collected reconciles to net sales less partner commission, adjusted for the movement in debtors |
| Channel totals | Revenue, cost of goods and operating profit each tie to the sum of their parts; gross sales tie back to the historical source table |
| Break-even logic | Sales at the calculated break-even level produce a nil result; contribution margin is positive, so break-even is mathematically meaningful |
| Scenario reconciliation | Scenario 1 reproduces the base forecast; all three scenario blocks roll forward and stay continuous week to week; payroll earned while a channel traded is never extinguished by closing it |
| Dashboard agreement | Every KPI card and scenario figure on the dashboard ties to its source cell |
| Formula integrity | No error values anywhere in the calculation, output or assumption sheets |
| Assumption completeness | Every scenario lever is populated for all three scenarios |

Controls are written to be independent of the thing they check wherever that is possible. The payment-completeness control, for example, enumerates candidate payment dates from the calendar and compares that count with the flags the forecast actually raised — so a fault in the timing logic cannot satisfy the control that is supposed to catch it.

---

## Design Principles

**One place for inputs.** Every hardcoded assumption lives on the Assumptions sheet, in labelled sections, with units. Scenario levers live on the Scenarios sheet and are marked as inputs there. Nothing else in the workbook is typed.

**Cells say what they are.** Pale amber with navy text means an input you may edit. Black means calculated on this sheet. Green means a link to another sheet. The convention is stated on the front sheet and holds without exception across 185 input cells.

**No hidden plugs.** There are no hardcoded numbers buried inside formulas, no hidden sheets, no hidden rows or columns, no macros, and no external links. What you see is the model.

**Scenarios rebuild, they do not overlay.** A scenario that applies a percentage to a result cannot tell you *when* cash moves. One that reconstructs the weekly path can.

**Outputs are linked, not typed.** Every dashboard KPI and every sentence of management commentary is generated from the cells it describes. Change an assumption and the prose changes with it — which is also why the commentary can be trusted not to drift from the numbers.

**The model checks itself.** A model that produces a number without proving it reconciles is asking the reader to take it on faith.

---

## Synthetic Data

All company names, financial data, customers, suppliers and transactions in this workbook are **fictional**, created solely to demonstrate financial modelling and FP&A method.

Atelier North Pte. Ltd. does not exist. No client, employer, customer or confidential information appears anywhere in the file. The structure, conventions and controls reflect the approach used on real engagements; the numbers do not come from one.

This is independent portfolio work.
