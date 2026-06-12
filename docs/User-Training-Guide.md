# Inventory Optimizer — User Training Guide
## Free Features: ABC/XYZ Classification · MRP Exception · MRP Assessment · Excess & Obsolete Cockpit

**Module:** Inventory Optimizer (ABS)  
**Platform:** Microsoft Dynamics 365 Finance & Operations  
**Audience:** Demand planners, materials planners, inventory controllers, MRP super users  
**Last updated:** June 2026

> **Update (June 2026):** Configuration for MRP Assessment check toggles is maintained in the standard
> **Inventory and warehouse management parameters** form, tab **Inventory optimizer**,
> FastTab **MRP Assessment check configuration**. There is no separate Inventory Optimizer
> parameters form for free features.
>
> **Update (June 2026):** ABC/XYZ demand extraction now reads inventory issue movements from
> `InventTrans` (status Sold or Deducted), so manufacturing component consumption is included.
>
> **Update (June 2026):** The **Excess & Obsolete Cockpit** has been added as a free feature.
> It buckets on-hand stock by movement aging (Active → Dead) and recommends a disposition
> per item. Items with on-hand but no issue history yet (e.g. recently received) are placed
> in a new **No movement history** bucket rather than being flagged dead/scrap.

---

## Table of Contents

1. [How the three features work together](#1-how-the-three-features-work-together)
2. [ABC/XYZ Classification](#2-abcxyz-classification)
   - [What is ABC analysis?](#what-is-abc-analysis)
   - [What is XYZ analysis?](#what-is-xyz-analysis)
   - [The combined 3×3 matrix](#the-combined-33-matrix)
   - [How to run the batch job](#how-to-run-the-abcxyz-batch-job)
   - [How to read the results](#how-to-read-the-abcxyz-results)
3. [MRP Exception](#3-mrp-exception)
   - [What exceptions are detected?](#what-exceptions-are-detected)
   - [How severity and priority are calculated](#how-severity-and-priority-are-calculated)
   - [How to run the batch job](#how-to-run-the-mrp-exception-batch-job)
   - [How to review and action exceptions](#how-to-review-and-action-exceptions)
4. [MRP Assessment](#4-mrp-assessment)
   - [What does the assessment check?](#what-does-the-assessment-check)
   - [Severity scoring and trend tracking](#severity-scoring-and-trend-tracking)
   - [How to run the batch job](#how-to-run-the-mrp-assessment-batch-job)
   - [How to review the results](#how-to-review-the-mrp-assessment-results)
   - [Acting on common findings](#acting-on-common-findings)
5. [Excess & Obsolete Cockpit](#5-excess--obsolete-cockpit)
   - [What does the cockpit show?](#what-does-the-cockpit-show)
   - [Aging buckets and recommendations](#aging-buckets-and-recommendations)
   - [How to run the batch job](#how-to-run-the-excess--obsolete-batch-job)
   - [How to read the results](#how-to-read-the-excess--obsolete-results)
6. [Recommended operating rhythm](#6-recommended-operating-rhythm)
7. [Quick-reference menu paths](#7-quick-reference-menu-paths)
8. [Role-based quick starts](#8-role-based-quick-starts)
9. [Troubleshooting and FAQ](#9-troubleshooting-and-faq)

---

## 1. How the three features work together

The four free features answer four different questions about your supply chain:

| Feature | Analyses | Question answered |
|---|---|---|
| **ABC/XYZ Classification** | Inventory issue demand history (`InventTrans`) | Which items matter most, and how predictable is their demand? |
| **MRP Exception** | Planned order output (`ReqPO`) | What supply actions need to be taken right now? |
| **MRP Assessment** | Plan configuration + master data | Can the MRP output be trusted? |
| **Excess & Obsolete Cockpit** | On-hand stock + issue/receipt movement aging | Which stock is no longer moving, and what should we do with it? |

They are designed to run in sequence:

```
ABC/XYZ Classification  →  MRP Exception  →  MRP Assessment
       (monthly)                (after each MRP run)        (weekly)
```

- **ABC/XYZ** enriches the other two features with value and variability context.  
- **MRP Exception** uses ABC/XYZ class to weight severity scores — a safety stock breach on an A-class item is more urgent than the same breach on a C-class item.  
- **MRP Assessment** uses ABC/XYZ class to identify configuration gaps (e.g. A-class items without a coverage group) and flags items that simultaneously have an assessment finding *and* an open MRP exception as **compounded risk**.

> **Important:** Always run ABC/XYZ Classification at least once before running MRP Exception or MRP Assessment. Without classification data, the ABC/XYZ-weighted priority scores cannot be computed.

### Before you begin (checklist)

Use this checklist before team onboarding, UAT walkthroughs, or go-live handover.

| Checklist item | Why it matters | Owner |
|---|---|---|
| Batch processing is healthy (no recurring failed jobs) | All three features are batch-driven | IT / batch admin |
| Master planning runs are completing on schedule | Exception and assessment results depend on fresh plan output | MRP super user |
| ABC/XYZ job has been run at least once in current legal entity | Priority scoring quality depends on class data | Inventory controller |
| Users have access to periodic tasks + inquiry menus | Training fails if users can only view, not run/review | Security admin |
| One agreed "working plan" for training | Avoids confusion when comparing results | Planning lead |

**Recommended training flow (90-120 min):**

1. Run ABC/XYZ Classification for one site (or all sites in sandbox).
2. Run master planning and then MRP Exception Scan.
3. Review and action top 10 exceptions.
4. Run MRP Assessment and walk through all High/Critical findings.
5. Assign owners and due dates for each unresolved finding.

---

## 2. ABC/XYZ Classification

### What is ABC analysis?

ABC analysis ranks every stocked item at a site by its share of total demand value over the analysis window, applying the Pareto (80/20) principle.

| Class | Cumulative revenue share | Typical inventory strategy |
|---|---|---|
| **A** | 0 – 80 % | Highest attention. Short replenishment cycles, tight safety stock review. |
| **B** | 80 – 95 % | Moderate attention. Periodic review, standard safety stock. |
| **C** | 95 – 100 % | Minimal effort. Long review cycles, bulk ordering where possible. |

**Example:** If your site has 1,000 items, the top ~50 items likely generate 80 % of your revenue — those are your A-class items and deserve the most planning attention.

### What is XYZ analysis?

XYZ analysis measures how *predictable* an item's monthly demand is using the **Coefficient of Variation (CV)** — the ratio of the standard deviation to the mean of monthly demand quantities.

$$CV = \frac{\sigma_{monthly}}{\mu_{monthly}}$$

A low CV means demand is consistent from month to month, making it easy to plan. A high CV means demand swings wildly — sometimes very high, sometimes near zero — making it much harder to hold the right quantity of stock at the right time.

**Why CV and not just standard deviation?** Standard deviation alone is misleading because it is scale-dependent. An item selling 10,000 units per month with a standard deviation of 1,000 is far more predictable than an item selling 200 units per month with a standard deviation of 400, even though the first item's absolute variation is larger. CV normalises the variation relative to average demand, making items comparable across very different volume levels.

| Class | CV threshold | Demand pattern | Replenishment approach |
|---|---|---|---|
| **X** | CV < 0.50 | Steady, predictable | Statistical reorder points work well |
| **Y** | 0.50 ≤ CV < 1.00 | Moderately variable, seasonal | Regular forecasting with safety stock |
| **Z** | CV ≥ 1.00 | Sporadic or intermittent | Demand-driven or just-in-time ordering |

**Understanding each class in practice:**

- **X items** have highly stable demand. The standard deviation is less than half of the mean — you can forecast these items with high confidence using statistical methods (e.g. reorder point = mean demand × lead time + safety stock based on service level). Small safety stock buffers are sufficient. Examples: consumables with a steady usage rate, spare parts replaced on fixed maintenance schedules.

- **Y items** show meaningful variation but not chaos. Demand may follow a seasonal pattern (peaks in summer, troughs in winter), respond to promotions, or reflect a broader business cycle. Forecasting is worthwhile but must account for the variability — safety stock needs to be sized more generously than for X items. Examples: seasonal products, items tied to project-based customer demand, items with promotional uplift.

- **Z items** have highly irregular demand — months with very high orders interspersed with months of zero or near-zero demand. Statistical reorder points perform poorly on Z items because there is no reliable mean to anchor them. The preferred approach is either make-to-order (carry no stock; source only when an order is placed) or to hold a relatively large safety stock buffer if stock-out risk is unacceptable. Examples: spare parts for ageing equipment, slow-moving specialty items, items used only in specific customer projects.

**How XYZ affects safety stock sizing:** The safety stock formula used in the Safety Stock Recommender is $SS = Z \times \sigma_d \times \sqrt{LT}$, where $\sigma_d$ is the standard deviation of demand and $LT$ is the replenishment lead time. For Z-class items with high $\sigma_d$, this formula produces very large safety stock values — often signalling that make-to-order is economically preferable to stocking. The XYZ class shown in the exception list and assessment results gives planners an immediate signal about whether a safety stock recommendation is realistic.

**Interpreting CV values:**

| CV example | Monthly demand example | Interpretation |
|---|---|---|
| 0.10 | Mean 500 units, std dev 50 units | Highly stable. X class — ideal for statistical replenishment. |
| 0.45 | Mean 200 units, std dev 90 units | Borderline X/Y. Minor variability; statistical methods still work well. |
| 0.75 | Mean 80 units, std dev 60 units | Y class. Noticeable swings; investigate seasonal or promotional drivers. |
| 1.20 | Mean 30 units, std dev 36 units | Z class. Sporadic. Consider make-to-order or consignment arrangement. |
| 3.50 | Mean 5 units, std dev 17.5 units | Deeply Z. Very low volume with large spikes. Safety stock is likely uneconomical. |

**Items with zero demand months:** When an item has months with zero demand within the analysis window, those zero values are included in the CV calculation. A product sold in 4 out of 12 months will have a very high CV even if the months it does sell show consistent quantities. The **Zero periods** column on the item list shows how many months had zero demand — use this alongside the CV to distinguish *truly erratic* demand from *seasonal/intermittent* demand.

> **Minimum data requirement:** XYZ analysis requires at least **3 complete calendar months** of issue-demand history within the date range. This most commonly affects newly introduced items and items that were recently added to a site's stocking policy.

> **No demand history? Items default to C/Z.** After the demand-based pass, the job runs a completeness sweep: any item that has item-coverage settings at the site but no qualifying demand history is classified **C (lowest value) / Z (sporadic)** — the most conservative planning quadrant. The **History months** column shows 0 for these items, so you can tell a swept default apart from a statistics-based classification. This means planned items no longer show up as *Unclassified* in the MRP Assessment or Exception features.

> **Cross-site fallback in downstream features:** Many downstream features (MRP Exception, Stock Level Monitor, BOM Analysis, Backlog Monitor, Kanban Monitor, Lead Time Monitor, Material Document Analysis) read ABC/XYZ class for display. When a row exists for an item at site A but not at site B (the site currently being processed), these features will display the class from the item's *highest-DemandMean* site as a fallback so the planner always sees an ABC/XYZ class instead of a blank. Only the **classifications** (A/B/C and X/Y/Z) are inherited &mdash; site-specific statistics such as `DemandMean` are never borrowed across sites, because that would distort calculations such as days-of-stock coverage or the safety stock formula.

### The combined 3×3 matrix

Combining ABC and XYZ produces nine planning segments:

|  | **X** (Stable) | **Y** (Variable) | **Z** (Sporadic) |
|---|---|---|---|
| **A** (High value) | **AX** | **AY** | **AZ** |
| **B** (Mid value) | **BX** | **BY** | **BZ** |
| **C** (Low value) | **CX** | **CY** | **CZ** |

**Segment guidance:**

| Segment | Recommended approach |
|---|---|
| **AX** | Continuous review, vendor-managed inventory, kanban. Your most critical items. |
| **AY** | Regular forecasting with safety stock. Investigate seasonality drivers. |
| **AZ** | Make-to-order where possible; high safety stock if MTS. Close planner attention required. |
| **BX** | Periodic review with statistical reorder point. |
| **BY** | Moderate safety stock. Review forecast quarterly. |
| **BZ** | Demand-driven replenishment. Consider consignment stocking. |
| **CX** | Min-max ordering. Low admin effort. |
| **CY** | Bulk purchasing, infrequent review. |
| **CZ** | Consider stocking out; source on demand. |

**What to look for in the workspace:**

| Signal | Interpretation |
|---|---|
| Large **AX inventory value** | Healthy — significant capital in your most predictable, high-value items. |
| Large **AZ inventory value** | Risk — high capital tied up in sporadic A-items. Consider make-to-order or consignment. |
| Large **CZ inventory value** | Working-capital concern — low-value, erratic items consuming cash. |
| Many items with **no class** (all tiles = 0) | The classification job has not been run yet for this site. (Planned items with no demand history are auto-classified C/Z by the sweep, so a blank class genuinely means "not yet run".) |

### How to run the ABC/XYZ batch job

1. Navigate to **Inventory Management › Periodic tasks › Inventory optimizer › ABC/XYZ Classification**.
2. Fill in the parameters:

   | Parameter | Description | Recommendation |
   |---|---|---|
   | **Site** | The site to classify. Leave blank to classify **all active sites** in one job. | Leave blank (all sites) |
   | **From date** | Start of the analysis window. | First day of the month, 12 months ago |
   | **To date** | End of the analysis window. | Last day of the previous month |
   | **Delete and regenerate** | Deletes existing classification records before re-inserting — **only for the selected site** when a site is specified; all sites only when Site is blank. Safe to run site-by-site. | **Tick this** for monthly runs |

3. Click **OK** to run immediately, or use the **Batch** tab to schedule overnight.
4. Monitor progress at **System administration › Inquiries › Batch jobs** (description: *ABC/XYZ Classification*).
5. When status shows **Ended**, open the workspace to review results.

> **Best practice:** Schedule monthly on the first working day of each month with Site blank, a 12-month look-back, and *Delete and regenerate* ticked. This refreshes all sites in one run.

> **Troubleshooting — no data found:** If a site reports no data, verify that issue transactions exist for the selected site and period in inventory transactions (`InventTrans`) with issue statuses such as Sold or Deducted. Sites with no issue-demand data are silently skipped.

### Parameter presets (ABC/XYZ)

| Scenario | Site | From date | To date | Delete and regenerate |
|---|---|---|---|---|
| Monthly production refresh | Blank (all sites) | First day, 12 months ago | Last day, previous month | Yes |
| Mid-month validation run | One site | First day, 12 months ago | Yesterday | Yes |
| Quick UAT test | One site | First day, 3 months ago | Last day, previous month | Yes |

> **Tip:** In production, prefer a fixed monthly cadence over ad-hoc reruns so class changes are easier to explain to planners and finance.

### How to read the ABC/XYZ results

**Workspace (Inventory health dashboard)**  
Navigate to **Inventory Management › Inquiries and reports › Inventory optimizer › Inventory health dashboard**.

- The **9-tile matrix** at the top shows item counts per segment. Click any tile to open the item list filtered to that segment.
- The **summary card grid** shows item count, 12-month sales value, and inventory value per segment.

**Item list (Item ABC/XYZ classification)**  
Navigate to **Inventory Management › Inquiries and reports › Inventory optimizer › Item ABC/XYZ classification**.

Key columns:

| Column | Description |
|---|---|
| **ABC class** | A / B / C based on revenue share |
| **XYZ class** | X / Y / Z based on demand CV |
| **12M sales amount** | Item revenue over the analysis window |
| **Sales amount %** | Item share of total site revenue |
| **Demand mean** | Average monthly demand quantity |
| **Demand std dev** | Standard deviation of monthly demand |
| **Demand CV** | Coefficient of variation (σ / μ) |
| **Zero periods** | Months in the window with zero demand |
| **Calculated date** | Date of the last classification run |

---

## 3. MRP Exception

MRP Exception scans the output of a completed master planning run and surfaces supply-side problems that need planner action. It enriches each exception with ABC/XYZ context so planners can prioritise the most critical issues first.

### What exceptions are detected?

Four exception types are detected automatically:

| Exception type | What it means | Suggested action |
|---|---|---|
| **Overdue order** | A planned purchase order's requirement date is in the past (`ReqDate < today()`) | Expedite the purchase order |
| **Safety stock breach** | Current physical stock is below the item's minimum on-hand (safety stock) | Raise an emergency PO to restore safety stock |
| **Lead time violation** | Days remaining before the requirement date is less than the item's purchase lead time | Place the order immediately — normal lead time is already insufficient |
| **Oversized order** | A single planned order quantity exceeds the configured months of average demand (a lot-sizing / overstock risk) | Split the planned order into smaller batches |

> **Excess inventory is now handled by the Excess & Obsolete cockpit**, not by an MRP exception. The cockpit values, ages, and recommends a disposition for surplus stock — richer than a bare quantity exception. The `Excess inventory` exception type is retained only so historical rows still display.

> **Detection scope (prerequisites):**
> - Only **purchase** planned orders are scanned. Overdue/oversized **production and transfer** orders are not flagged in this version.
> - **Safety stock breach** only covers items that already have an ABC/XYZ classification row — run **ABC/XYZ Classification** first, or a newly stocked item with safety stock will not be checked until it is classified.

### How severity and priority are calculated

Each exception is assigned a **severity** based on how critical it is:

| Exception type | High | Medium | Low |
|---|---|---|---|
| Overdue order | > High-days threshold (default 7) late | > Medium-days threshold (default 3) late | ≤ Medium-days threshold late |
| Safety stock breach | A-class item | B-class item | C-class item |
| Lead time violation | A-class item | B-class item | C-class item |
| Oversized order | A-class item | B-class item | C-class item |

> **Configurable thresholds:** the overdue High/Medium day bands and the oversized-order months-of-demand multiplier are set on **Inventory management › Setup › Inventory and warehouse management parameters › Inventory optimizer tab › MRP exception thresholds**. Leave a value at 0 to use its default (7 days / 3 days / 3 months).

A **priority score** combines severity, ABC class, and XYZ class:

$$\text{Priority score} = \text{Severity score} \times \text{ABC score} \times \text{XYZ score}$$

| Dimension | Score 3 | Score 2 | Score 1 |
|---|---|---|---|
| Severity | High | Medium | Low |
| ABC class | A | B | C |
| XYZ class | Z | Y | X |

**Maximum score = 27** (High severity + A-class + Z demand — an unpredictable, high-value item with a live supply problem). Work exceptions from highest to lowest priority score.

### How to run the MRP Exception batch job

> **Pre-requisite:** Complete a full or regenerative master planning run *before* running exception detection. The engine reads `ReqPO` data produced by master planning — stale planning output produces stale exceptions.

1. Navigate to **Inventory Management › Periodic tasks › Inventory optimizer › MRP Exception Scan**.
2. Fill in the parameters:

   | Parameter | Description | Recommendation |
   |---|---|---|
   | **Plan** | The master plan to scan. Leave blank to scan **all master plans**. | Leave blank, or specify your working plan |
   | **Site** | The site to scan. Leave blank to scan **all sites**. | Leave blank (all sites) |

3. Click **OK** or schedule via the **Batch** tab.

> **Best practice:** Chain the exception scan as a batch dependency *after* your nightly master planning run so exception data is always fresh by the time planners start work in the morning.

> **Note:** Only `Open` status exceptions for the scanned plan/site combination are deleted and regenerated on each run. Exceptions you have manually set to `Confirmed` or `Dismissed` are preserved and provide an audit trail.

### Parameter presets (MRP Exception)

| Scenario | Plan | Site | When to use |
|---|---|---|---|
| Daily operations | Working plan | Blank (all sites) | Standard start-of-day queue |
| Site war room | Working plan | Specific site | Local shortage or escalation event |
| Post-cutover stabilization | Blank (all plans) | Blank (all sites) | Catch broad setup issues in first 1-2 weeks |

### How to review and action exceptions

Navigate to **Inventory Management › Inquiries and reports › Inventory optimizer › MRP exception**.

**Key columns:**

| Column | Description |
|---|---|
| **Priority score** | Sort descending by this column to work highest-risk items first |
| **Exception type** | Category of problem detected |
| **Severity** | High / Medium / Low — colour-coded |
| **Item ID / Site** | The affected item and site |
| **ABC class / XYZ class** | From the last ABC/XYZ classification run |
| **Requirement date** | Date the planned order is needed |
| **Exception days** | Days overdue, lead time shortfall, or 0 |
| **Exception qty** | Quantity at risk |
| **Suggested action** | System-generated recommended action |
| **Status** | Open / Confirmed / Dismissed |

**Working the exception list — suggested process:**

1. Filter to **Status = Open** and sort by **Priority score** descending.
2. For each high-priority exception, click **View requirement** to open the underlying planned order, and take the appropriate step in D365FO (place/expedite a PO, adjust safety stock, etc.). *(Safety stock breaches have no planned order, so View requirement is not available for them.)*
3. Once actioned, set the exception **Status** to **Confirmed** (action taken) or **Dismissed** (not applicable). Select multiple rows first to **Confirm** or **Dismiss them in bulk** — handy for clearing many low-severity rows at once.
4. `Confirmed` and `Dismissed` exceptions are retained across re-runs as an audit trail. Re-running detection refreshes the quantitative fields (days, quantity, severity) of still-Open exceptions, so a worsening condition is reflected rather than frozen at its first-detected values.

---

## 4. MRP Assessment

The MRP Assessment answers: *Can the MRP output be trusted?* Where MRP Exception analyses *what MRP produced*, the Assessment analyses the *inputs and configuration* that determine whether that output is valid.

### What does the assessment check?

The assessment runs up to **27 diagnostic checks** grouped into nine categories:

| Category | What it checks | Check codes |
|---|---|---|
| **Plan configuration** | Time fence settings, negative days, static/dynamic plan conflict | PC-001, PC-002, PC-003, PC-005 |
| **Data quality** | Orphaned/inactive plan data, BOM lines without components, unit of measure consistency, max order quantity | DQ-001, DQ-002, DQ-004 to DQ-009 |
| **Safety stock** | Safety stock vs ABC class, on-hand alignment, coverage group assignment | SS-001 to SS-004 |
| **Item coverage** | Coverage group completeness, lead time setup, period length | CV-001 to CV-004 |
| **Performance risk** | Finite capacity on main plan, data volume, BOM depth | PR-001 to PR-003 |
| **Firming window** | Firming fence vs lead time, stale planned orders, negative on-hand with no inbound supply | FW-001 to FW-003 |
| **BOM & route quality** | Multiple active BOM versions, future-dated BOMs, vendor lead time drift | BR-001 to BR-003 |
| **Planner behaviour** | Dead stock with open supply, coverage drift after reclassification, open planned supply without recent PO history | PB-001 to PB-003 |
| **Planning Optimization health** | Service integration flag, company exclusion, combined availability, fit analysis compatibility issues, fit runtime availability | PO-001 to PO-006 |

*Legacy-MRP-only checks (DQ-001, DQ-002, PC-003) are automatically skipped and marked **Not applicable** when Planning Optimization is active. PO-004 to PO-006 are only executed when Planning Optimization is active for the current company.*

> **Pre-requisite:** Run ABC/XYZ Classification at least once before running the MRP Assessment. Several checks (SS-001, CV-001, CV-003, PB-001, PB-003) enrich their findings using ABC/XYZ class data. Without classification data, those checks will report zero affected items even if issues exist.

### Severity scoring and trend tracking

**Overall severity** is the highest severity of any check with `Affected count > 0`:

| Severity | Meaning |
|---|---|
| **Critical** | MRP output cannot be trusted — resolve before making any planning decisions |
| **High** | Significant issue — resolve within the current sprint |
| **Medium** | Sub-optimal configuration — review within the current month |
| **Low** | Best-practice recommendation — informational |
| **No issue** | All checks passed |

**Trend** compares each check's affected count to the previous run for the same plan:

| Trend | Meaning |
|---|---|
| New | Firing for the first time — no prior run for this check |
| Worsening ↑ | Affected count increased since the last run |
| Stable → | Affected count unchanged (issue persists) |
| Improving ↓ | Affected count decreased but issue is still present |
| Resolved ✓ | Issue was present before but is no longer detected |
| Unchanged | No issue in either run — normal passing state |

**Compounded risk** is flagged when an item appears in an assessment finding *and* has a related open MRP exception of the same root cause. Its priority score is multiplied by 1.5, making these items the highest priority in the detail grid.

| Assessment check | Related signal |
|---|---|
| SS-001, SS-004 | Safety stock breach (MRP exception) |
| FW-003 | Overdue order (MRP exception) |
| CV-002, CV-003 | Lead time violation (MRP exception) |
| PB-001 | Dead/Dormant stock with open supply (Excess & Obsolete cockpit) |

### How to run the MRP Assessment batch job

1. Navigate to **Inventory Management › Periodic tasks › Inventory optimizer › Run MRP assessment**.
2. Fill in the parameters:

   | Parameter | Description | Recommendation |
   |---|---|---|
   | **Plan** | The master plan to assess. Leave blank to assess **all plans**. | Your primary working plan |
   | **Site** | Limit item-level checks to one site. Leave blank for all sites. | Leave blank (all sites) |
   | **Include item-level detail** | When Yes, per-item rows are written for drill-down checks. No gives a summary-only run. | **Yes** |
   | **Delete runs older than (days)** | Purges completed runs older than this many days after the new run completes. 0 = retain all. | `90` |

3. Click **OK** or schedule via the **Batch** tab.
4. When the job completes, open **Inventory Management › Inquiries and reports › Inventory optimizer › MRP assessment**.

### Parameter presets (MRP Assessment)

| Scenario | Plan | Site | Include item-level detail | Delete runs older than |
|---|---|---|---|---|
| Weekly governance review | Working plan | Blank | Yes | 90 |
| Fast health pulse | Working plan | Blank | No | 90 |
| Site-level deep dive | Working plan | Specific site | Yes | 90 |

### Planning Optimization health checks (PO-001 to PO-006)

Use this quick interpretation guide during review meetings:

| Check | What it indicates | First action |
|---|---|---|
| PO-001 | Global PO integration flag is off | Validate and enable global Planning Optimization integration setting |
| PO-002 | Current company excluded from PO | Remove exclusion for the legal entity if PO should be active |
| PO-003 | PO unavailable for current company (combined signal) | Triage PO-001 and PO-002 first, then environment/service status |
| PO-004 | Fit analysis found compatibility issues | Review fit analysis output and resolve unsupported setup patterns |
| PO-005 | Fit analysis runtime is unavailable | Check fit-analysis prerequisites, flights/toggles, and service state |
| PO-006 | Explicit fit analysis issue count is non-zero | Work issue list to zero before trusting PO output for critical decisions |

### How to review the MRP Assessment results

Navigate to **Inventory Management › Inquiries and reports › Inventory optimizer › MRP assessment**.

The form has three levels:

**Run header grid (top):** Lists completed assessment runs with their overall severity. A `Critical` or `High` overall severity means planners should not act on MRP output until the root cause is resolved.

**Result grid (centre):** One row per check for the selected run. Key columns:

| Column | Description |
|---|---|
| **Check code** | e.g. `SS-001` — use this as a reference when escalating to IT |
| **Category** | The check group |
| **Severity** | Colour-coded badge |
| **Affected count** | Number of records that triggered the check. Zero = check passed. |
| **Severity trend** | Worsening / Stable / Improving / Resolved. Worsening or New require immediate attention. |
| **Suppressed** | Check excluded from the overall severity score by an administrator |
| **Not applicable** | Check does not apply to the active planning engine |
| **Check description** | What the check detected |
| **Recommendation** | Suggested corrective action |
| **Affected objects** | Sample of the first 20 affected item IDs, plan IDs, or coverage group IDs |

**Item detail grid (bottom):** Per-item rows for checks run with *Include item-level detail = Yes*. Filter by **Compounded risk = Yes** to focus on the highest-risk items first.

### Acting on common findings

| Finding | Who to involve | Typical resolution |
|---|---|---|
| **PC-001 to PC-005** (Plan configuration) | MRP super user / IT | Review and correct master plan settings in **Master planning › Master plans** |
| **DQ-001, DQ-002** (Stale plan data) | IT / batch administrator | Schedule the *Delete MRP data* periodic task to run nightly |
| **DQ-004** (BOM without lines) | Engineering / product management | Add active BOM lines or deactivate the BOM version |
| **SS-001, SS-003, SS-004** (Safety stock gaps) | Demand planner | Use the Safety Stock & ROP Simulator to recalculate safety stock levels, then update item coverage |
| **CV-002, CV-003** (Coverage gaps) | Demand planner / buyer | Assign coverage groups and set lead times in **Master planning › Setup › Item coverage** |
| **FW-003** (Negative on-hand, no supply) | Buyer / warehouse | Create an emergency purchase order or investigate a warehouse posting error |
| **PB-001** (Dead stock with open supply) | Demand planner / buyer | Cancel or reduce open purchase orders for dead-stock items |
| **PO-001 to PO-003** (PO availability/integration) | IT / D365 platform admin | Enable global PO integration, remove company exclusion, and confirm current-company PO availability before running planning |
| **PO-004 to PO-006** (Fit-analysis diagnostics) | IT / planning process owner | Run fit analysis, resolve unsupported configuration patterns, and enable fit-analysis runtime toggle if disabled |
> **Enable/disable checks:** Administrators can turn checks on or off on **Inventory management › Setup › Inventory and warehouse management parameters** (tab **Inventory optimizer**, FastTab **MRP Assessment check configuration**). Check codes are grouped by family (PC, DQ, SS, CV, PR, FW, BR, PB, PO). Disabled checks are not executed and are excluded from the overall severity score.

---

## 5. Excess & Obsolete Cockpit

The Excess & Obsolete (E&O) Cockpit answers: *Which on-hand stock is no longer moving, and what should we do with it?* It buckets every stocked item at a site by how long ago it last moved, overlays a provision value and an excess-coverage flag, and recommends a disposition per item.

### What does the cockpit show?

For each item/site with on-hand stock, the cockpit computes:

| Output | Description |
|---|---|
| **Aging bucket** | Movement-aging band derived from months since the last issue (sale/deduction) |
| **Recommendation** | Suggested disposition (No action, Run out, Reduce orders, Transfer, Discount, Scrap) |
| **On-hand quantity / value** | Current physical stock and its posted + physical value |
| **Provision value** | On-hand value × the bucket's provision percentage — your suggested write-down exposure |
| **Months since last issue** | How long since the item last had a Sold/Deducted movement at the site |
| **Months of coverage** | On-hand divided by mean monthly demand (999 when demand is zero) |
| **Excess flag** | Yes when days-of-stock exceed the ABC class-specific coverage limit (A 30 / B 60 / C 120 days) |
| **Open supply flag** | Yes when an open purchase order still exists for a Dormant/Dead item ("still buying what doesn't move") |
| **ABC / XYZ class** | From the last ABC/XYZ classification run (enriches the view; not required to run) |

A **stacked column chart** at the top shows provision value and remaining (uncovered) on-hand value per aging bucket, so you can see at a glance where write-down exposure is concentrated.

### Aging buckets and recommendations

| Aging bucket | Trigger | Default provision % | Recommendation |
|---|---|---|---|
| **Active** | Last issued < 3 months ago | 0 % | No action (or Reduce orders if flagged excess) |
| **Slow moving** | Last issued 3 – 6 months ago | 10 % | Run out |
| **Very slow moving** | Last issued 6 – 12 months ago | 25 % | Run out |
| **Dormant** | Last issued 12 – 24 months ago | 50 % | Discount |
| **Dead stock** | Last issued ≥ 24 months ago | 100 % | Scrap |
| **No movement history** | On-hand but never issued, and not received long ago | 0 % | No action |

> **No movement history is *not* dead stock.** An item that has on-hand but no issue history yet — e.g. recently received purchases or a newly stocked item — is placed in the **No movement history** bucket with 0 % provision and **no scrap recommendation**, because it simply has not had a chance to move. Only never-issued stock that has *also sat since receipt* for a long time ages into Dormant (12 – 24 months since receipt) and then Dead (≥ 24 months since receipt). This prevents brand-new stock from being condemned to scrap.

> **Pre-requisite:** Run ABC/XYZ Classification at least once before running the cockpit. The excess flag and coverage calculations rely on each item's `DemandMean` and ABC class. Without classification data, items show as *Unclassified* with `Months of coverage = 999` and `Excess flag = No`.

**Row colour coding:** Dead = red, Dormant = orange, Very slow = amber, Slow = blue, No movement history = grey. To avoid drowning the list in colour, the bands only fire for items with at least 3 months of demand history (or buckets that are already Dormant/Dead).

### How to run the Excess & Obsolete batch job

1. Navigate to **Inventory Management › Periodic tasks › Inventory optimizer › Run excess and obsolete analysis**.
2. Fill in the parameters:

   | Parameter | Description | Recommendation |
   |---|---|---|
   | **Site** | The site to analyse. Leave blank to analyse **all active sites**. | Leave blank (all sites) |
   | **As of date** | The reference date for the aging calculation. | Today |
   | **Provision % (Slow / Very slow / Dormant / Dead)** | Write-down percentage applied per bucket. | Defaults: 10 / 25 / 50 / 100 |
   | **Delete and regenerate** | Clears existing rows before re-inserting — for the selected site only when a site is given. | **Tick this** for periodic runs |

3. Click **OK** to run immediately, or use the **Batch** tab to schedule.
4. When complete, the job posts a summary showing total provision value for the analysed site(s).

> **Tip:** The cockpit is also included in the **Optimize all** master job (it runs straight after ABC/XYZ classification), so a full Optimize All run refreshes it automatically.

### How to read the Excess & Obsolete results

Navigate to **Inventory Management › Inquiries and reports › Inventory optimizer › Excess and obsolete cockpit**.

- The **chart** shows provision value vs remaining value by aging bucket.
- The **grid** lists each item/site with its bucket, recommendation, values, and flags. Sort by **Provision value** descending to tackle the biggest write-down exposure first.
- Use the **quick filter** to focus on a bucket (e.g. *Dead stock*) or a recommendation (e.g. *Scrap*).
- The **Item ABC/XYZ** button opens the classification list for context.

**Working the list — suggested process:**

1. Filter to **Aging bucket = Dead stock** and review the *Scrap* candidates with the highest provision value.
2. Check the **Open supply flag** — Dormant/Dead items still on open PO are the priority ("stop buying what isn't moving").
3. For **Dormant** items, evaluate discount/clearance; for **Slow / Very slow**, plan to run the stock down rather than reorder.
4. Treat **No movement history** rows as "watch" items — recently received stock that needs time, not action.

---

## 6. Recommended operating rhythm

| Frequency | Task | Who |
|---|---|---|
| **Monthly** (first working day) | Run **ABC/XYZ Classification** for all sites, 12-month look-back, Delete and regenerate ticked | Inventory controller / batch admin |
| **Nightly** (scheduled batch) | Run **master planning** → then **MRP Exception Scan** as a chained dependency | Batch admin |
| **Weekly** | Run **MRP Assessment** and review the overall severity. Resolve any Critical or High findings before the weekly planning meeting. | MRP super user |
| **Monthly** | Run **Excess & Obsolete analysis** (or rely on Optimize all). Review Dead/Dormant provision value and act on Scrap/Discount candidates with open supply. | Inventory controller / demand planner |
| **Daily** (start of day) | Open **MRP exception** list, filter Open, sort by Priority score descending. Work exceptions top-down. | Demand planner / buyer |
| **Monthly** | Review ABC/XYZ segment shifts. Update coverage groups and safety stock for items that have changed class. | Demand planner |

---

## 7. Quick-reference menu paths

### Batch jobs (Periodic tasks)

| Task | Menu path |
|---|---|
| ABC/XYZ Classification | **Inventory Management › Periodic tasks › Inventory optimizer › ABC/XYZ Classification** |
| MRP Exception Scan | **Inventory Management › Periodic tasks › Inventory optimizer › MRP Exception Scan** |
| Run MRP Assessment | **Inventory Management › Periodic tasks › Inventory optimizer › Run MRP assessment** |
| Run excess and obsolete analysis | **Inventory Management › Periodic tasks › Inventory optimizer › Run excess and obsolete analysis** |

### Inquiry forms (Inquiries and reports)

| Form | Menu path |
|---|---|
| Inventory Optimizer workspace | **Inventory Management › Inquiries and reports › Inventory optimizer › Inventory health dashboard** |
| Item ABC/XYZ classification | **Inventory Management › Inquiries and reports › Inventory optimizer › Item ABC/XYZ classification** |
| MRP exception | **Inventory Management › Inquiries and reports › Inventory optimizer › MRP exception** |
| MRP assessment | **Inventory Management › Inquiries and reports › Inventory optimizer › MRP assessment** |
| Excess and obsolete cockpit | **Inventory Management › Inquiries and reports › Inventory optimizer › Excess and obsolete cockpit** |

### Setup

| Task | Menu path |
|---|---|
| MRP Assessment check configuration | **Inventory management › Setup › Inventory and warehouse management parameters › Inventory optimizer tab** |

---

## 8. Role-based quick starts

Use these playbooks to reduce onboarding time and make daily routines explicit by role.

### Demand planner quick start

**Daily (20-30 min):**

1. Open **MRP exception** list and filter `Status = Open`.
2. Sort by **Priority score** descending.
3. Action top exceptions first:
   - `OverdueOrder` and `SafetyStockBreach` first
   - `CapacityOverload` escalated immediately
4. Mark actioned rows as `Confirmed`; mark non-actionable rows `Dismissed` with a reason.

**Weekly (45-60 min):**

1. Run **MRP Assessment** for the working plan.
2. Review all `Critical` and `High` findings.
3. Assign owner + due date for unresolved items.
4. Check trend direction (`Worsening`, `New`) and escalate where needed.

### Buyer quick start

**Daily (20-30 min):**

1. Focus on exceptions tied to procurement actions:
   - `OverdueOrder`
   - `LeadTimeViolation`
   - `SafetyStockBreach`
2. Validate supply response (expedite, convert, or create PO).
3. Update exception status after action is completed.

**Weekly (30-45 min):**

1. Review recurring items in `LeadTimeViolation`.
2. Align lead times and coverage setup with planning team.
3. Flag suppliers/items requiring service-level or contractual review.

### Inventory controller quick start

**Monthly (60-90 min):**

1. Run **ABC/XYZ Classification** with 12-month look-back and `Delete and regenerate = Yes`.
2. Review segment shifts (`AX`, `AY`, `AZ`, etc.).
3. Identify high-capital risk clusters (`AZ`, `CZ`).
4. Share class-change list with planners for policy updates.
5. Run **Excess & Obsolete analysis** and review total provision value. Action Dead/Dormant items with an **Open supply flag** first, then Scrap/Discount candidates by descending provision value.

**Monthly governance check (30 min):**

1. Confirm classification job completed for all active sites.
2. Verify no critical data gaps (`no class` items from missing sales history).

### MRP super user / process owner quick start

**Weekly governance meeting (45-60 min):**

1. Review latest assessment `Overall severity`.
2. Walk through unresolved `Critical`/`High` findings by category.
3. Track closure of previous-week actions.
4. Decide temporary suppressions only with documented business reason.

### IT / batch administrator quick start

**Daily (10-15 min):**

1. Check overnight batch chain health:
   - Master planning completed
   - MRP Exception Scan completed
2. Review failed jobs and rerun where appropriate.

**Weekly (30 min):**

1. Validate MRP Assessment run cadence.
2. Confirm retention cleanup behaves as expected (`Delete runs older than`).
3. Monitor recurring PO health failures (`PO-001` to `PO-006`) and coordinate remediation.

---

## 9. Troubleshooting and FAQ

### Common symptoms and fixes

| Symptom | Likely cause | What to check first |
|---|---|---|
| ABC/XYZ matrix is empty | Job has not run or no issue-demand data in range | ABC/XYZ batch history, inventory transactions, selected date range |
| Exception list suddenly drops to near-zero | Master planning did not complete, or wrong plan/site was scanned | Latest MRP batch status, exception job parameters |
| Assessment shows many `Not applicable` checks | Active engine changed (Legacy vs PO) | Planning engine status, PO applicability of checks |
| E&O cockpit shows everything as Dead stock | ABC/XYZ not run, or demo/old data with no recent movement | Run ABC/XYZ classification first; check that recent Sold/Deducted issues exist in the window |
| E&O cockpit is empty | Analysis not run, or no on-hand stock at the site | Run **Run excess and obsolete analysis**; verify on-hand exists |
| PO checks remain High/Critical | PO integration/config not resolved | PO-001 to PO-006 rows and recommended actions |
| Severity trend keeps worsening | Underlying process ownership gap | Assign owner + due date per check and review weekly |

### FAQ

**Q: Should planners run all three jobs every day?**  
A: No. ABC/XYZ is typically monthly. Exception Scan is usually daily after MRP. Assessment is typically weekly (or after major setup changes).

**Q: Can we disable a noisy check?**  
A: Yes. Use **MRP Assessment check configuration** in Inventory and warehouse management parameters. Keep a short reason log for governance.

**Q: Why does a check show zero affected count but still appear in results?**  
A: The framework records all checks for transparency. Zero means pass, while still showing trend/suppression/applicability context.

**Q: When should we trust MRP output?**  
A: When Assessment has no unresolved Critical findings, High findings are understood/owned, and the top exceptions are actively managed.

**Q: Why is a brand-new item showing in the Excess & Obsolete cockpit?**  
A: It will appear in the **No movement history** bucket (grey, 0 % provision, *No action*) because it has on-hand but no issue history yet. It is *not* flagged dead or scrap. Only never-issued stock that has also sat since receipt for 12+ months ages into Dormant, and 24+ months into Dead.

---

*For questions or support, contact your Inventory Optimizer system administrator or refer to the full [user manual](user-manual.html).*
