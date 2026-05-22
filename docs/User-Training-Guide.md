# Inventory Optimizer — User Training Guide

**Free Features:** ABC/XYZ Classification · MRP Exception Advisor · MRP Assessment

**Module:** Inventory Optimizer  
**Platform:** Microsoft Dynamics 365 Finance & Operations  
**Audience:** Demand Planners, Materials Planners, Inventory Controllers, MRP Super Users  
**Last updated:** May 2026

---

## Table of Contents

1. [How the three features work together](#1-how-the-three-features-work-together)
2. [ABC/XYZ Classification](#2-abcxyz-classification)
3. [MRP Exception Advisor](#3-mrp-exception-advisor)
4. [MRP Assessment](#4-mrp-assessment)
5. [Recommended Operating Rhythm](#5-recommended-operating-rhythm)
6. [Quick-Reference Menu Paths](#6-quick-reference-menu-paths)

---

## 1. How the three features work together

The three features answer three complementary questions about your supply chain:

| Feature                    | Analyses                        | Core Question Answered |
|---------------------------|---------------------------------|------------------------|
| **ABC/XYZ Classification** | Customer invoice history        | Which items matter most and how predictable is demand? |
| **MRP Exception Advisor**  | Planned orders (`ReqPO`)        | What supply actions need immediate attention? |
| **MRP Assessment**         | Plan configuration + master data| Can we trust the MRP output? |

**Recommended execution sequence:**
ABC/XYZ Classification → MRP Exception Advisor → MRP Assessment
(Monthly)               (After each MRP run)       (Weekly)
text> **Important:** Always run **ABC/XYZ Classification** first. The other two features rely on ABC/XYZ data for priority scoring and contextual analysis.

---

## 2. ABC/XYZ Classification

### What is ABC analysis?

ABC analysis ranks every stocked item at a site by its share of total invoiced sales revenue over the analysis window, applying the Pareto (80/20) principle.

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

> **Minimum data requirement:** XYZ analysis requires at least **3 complete calendar months** of invoice data within the date range. Items with fewer than 3 months of data are skipped for XYZ classification and will show a blank XYZ class in the item list. This most commonly affects newly introduced items and items that were recently added to a site's stocking policy.

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
| Many items with **no class** (all tiles = 0) | The classification job has not been run yet, or no invoice data exists for the selected period. |

### How to run the ABC/XYZ batch job

1. Navigate to **Inventory Management › Periodic tasks › Inventory optimizer › ABC/XYZ Classification**.
2. Fill in the parameters:

   | Parameter | Description | Recommendation |
   |---|---|---|
   | **Site** | The site to classify. Leave blank to classify **all active sites** in one job. | Leave blank (all sites) |
   | **From date** | Start of the analysis window. | First day of the month, 12 months ago |
   | **To date** | End of the analysis window. | Last day of the previous month |
   | **Delete and regenerate** | Deletes all existing classification records before re-inserting. | **Tick this** for monthly runs |

3. Click **OK** to run immediately, or use the **Batch** tab to schedule overnight.
4. Monitor progress at **System administration › Inquiries › Batch jobs** (description: *ABC/XYZ Classification*).
5. When status shows **Ended**, open the workspace to review results.

> **Best practice:** Schedule monthly on the first working day of each month with Site blank, a 12-month look-back, and *Delete and regenerate* ticked. This refreshes all sites in one run.

> **Troubleshooting — no data found:** If a site reports no data, verify that customer invoice transactions exist in **Accounts receivable › Inquiries › Journals › Invoice journal** for that site and period. Sites with no invoice data are silently skipped.

### How to read the ABC/XYZ results

**Workspace (ABC/XYZ Matrix)**  
Navigate to **Inventory Management › Inquiries and reports › Inventory optimizer › ABC/XYZ Matrix**.

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

## 3. MRP Exception Advisor

The MRP Exception Advisor scans the output of a completed master planning run and surfaces supply-side problems that need planner action. It enriches each exception with ABC/XYZ context so planners can prioritise the most critical issues first.

### What exceptions are detected?

Five exception types are detected automatically:

| Exception type | What it means | Suggested action |
|---|---|---|
| **Overdue order** | A planned purchase order's requirement date is in the past (`ReqDate < today()`) | Expedite the purchase order |
| **Safety stock breach** | Current physical stock is below the item's minimum on-hand (safety stock) | Raise an emergency PO to restore safety stock |
| **Lead time violation** | Days remaining before the requirement date is less than the item's purchase lead time | Place the order immediately — normal lead time is already insufficient |
| **Excess inventory** | Days-of-stock on hand exceed the class-specific coverage threshold | Review and cancel or defer planned supply |
| **Capacity overload** | An item simultaneously has an overdue order *and* a safety stock breach | Escalate to production/supply planning |

**Excess inventory thresholds by ABC class:**

| ABC class | Coverage limit |
|---|---|
| A | 30 days |
| B | 60 days |
| C | 90 days |

### How severity and priority are calculated

Each exception is assigned a **severity** based on how critical it is:

| Exception type | High | Medium | Low |
|---|---|---|---|
| Overdue order | > 7 days late | > 3 days late | ≤ 3 days late |
| Safety stock breach | A-class item | B-class item | C-class item |
| Lead time violation | A-class item | B-class item | C-class item |
| Excess inventory | A-class item | B-class item | C-class item |
| Capacity overload | Always High | — | — |

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
2. For each high-priority exception, review the suggested action and take the appropriate step in D365FO (place/expedite a PO, adjust safety stock, etc.).
3. Once actioned, set the exception **Status** to **Confirmed** (action taken) or **Dismissed** (not applicable).
4. `Confirmed` and `Dismissed` exceptions are retained across re-runs as an audit trail.

---

## 4. MRP Assessment

The MRP Assessment answers: *Can the MRP output be trusted?* Where the MRP Exception Advisor analyses *what MRP produced*, the Assessment analyses the *inputs and configuration* that determine whether that output is valid.

### What does the assessment check?

The assessment runs up to **27 diagnostic checks** grouped into nine categories:

| Category | What it checks | Check codes |
|---|---|---|
| **Plan configuration** | Time fence settings, negative days, static/dynamic plan conflict, stale scheduling locks | PC-001 to PC-005 |
| **Data quality** | Orphaned/inactive plan data, BOM lines without components, unit of measure consistency, max order quantity | DQ-001 to DQ-009 |
| **Safety stock** | Safety stock vs ABC class, on-hand alignment, coverage group assignment | SS-001 to SS-004 |
| **Item coverage** | Coverage group completeness, lead time setup, period length | CV-001 to CV-004 |
| **Performance risk** | Finite capacity on main plan, data volume, BOM depth | PR-001 to PR-004 |
| **Firming window** | Firming fence vs lead time, stale planned orders, negative on-hand with no inbound supply | FW-001 to FW-003 |
| **BOM & route quality** | Multiple active BOM versions, future-dated BOMs, vendor lead time drift | BR-001 to BR-003 |
| **Planner behaviour** | Dead stock with open supply, coverage drift after reclassification, expired purchase agreements | PB-001 to PB-003 |
| **Planning Optimization health** | Add-in connectivity, Fit Analysis warnings, run failures, run frequency | PO-001 to PO-004 ✦ |

*✦ Planning Optimization health checks fire only when Planning Optimization is the active engine. Legacy-MRP-only checks (DQ-001 to DQ-003, PC-003, PC-004, PR-004) are automatically skipped and marked **Not applicable** when Planning Optimization is active.*

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

| Assessment check | Related MRP exception |
|---|---|
| SS-001, SS-004 | Safety stock breach |
| FW-003 | Overdue order |
| CV-002, CV-003 | Lead time violation |
| PB-001 | Excess inventory |

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
| **DQ-001 to DQ-003** (Stale plan data) | IT / batch administrator | Schedule the *Delete MRP data* periodic task to run nightly |
| **DQ-004** (BOM without lines) | Engineering / product management | Add active BOM lines or deactivate the BOM version |
| **SS-001, SS-003, SS-004** (Safety stock gaps) | Demand planner | Use the Safety Stock & ROP Simulator to recalculate safety stock levels, then update item coverage |
| **CV-002, CV-003** (Coverage gaps) | Demand planner / buyer | Assign coverage groups and set lead times in **Master planning › Setup › Item coverage** |
| **FW-003** (Negative on-hand, no supply) | Buyer / warehouse | Create an emergency purchase order or investigate a warehouse posting error |
| **PB-001** (Dead stock with open supply) | Demand planner / buyer | Cancel or reduce open purchase orders for dead-stock items |
| **PO-001 to PO-004** (Planning Optimization) | IT / LCS administrator | Check Planning Optimization add-in status in LCS and review the Fit Analysis |

> **Suppressing a check:** If a check is not relevant to your organisation (e.g. PC-003 for companies that intentionally share a static and dynamic plan), an administrator can suppress it via **Master planning › Setup › MRP assessment check configuration**. Suppressed checks are excluded from the overall severity score.

---

## 5. Recommended Operating Rhythm

| Frequency     | Task                                              | Responsible          |
|---------------|---------------------------------------------------|----------------------|
| Monthly       | ABC/XYZ Classification (12-month)                 | Inventory Controller |
| Nightly       | Master Planning → MRP Exception Scan              | Batch Administrator  |
| Weekly        | Run MRP Assessment + resolve Critical/High issues | MRP Super User       |
| Daily         | Review & action MRP Exceptions (by Priority)      | Planners / Buyers    |

---

## 6. Quick-Reference Menu Paths

**Batch Jobs:**
- ABC/XYZ Classification → *Inventory management > Periodic tasks > Inventory optimizer > ABC/XYZ Classification*
- MRP Exception Scan → *Inventory management > Periodic tasks > Inventory optimizer > MRP Exception Scan*
- MRP Assessment → *Inventory management > Periodic tasks > Inventory optimizer > Run MRP assessment*

**Inquiry Forms:**
- ABC/XYZ Matrix → *Inventory management > Inquiries and reports > Inventory optimizer > ABC/XYZ Matrix*
- MRP Exceptions → *Inventory management > Inquiries and reports > Inventory optimizer > MRP exception*
- MRP Assessment → *Inventory management > Inquiries and reports > Inventory optimizer > MRP assessment*

---

**Need help?**  
Contact your Inventory Optimizer administrator or open an issue in the GitHub repository.

---
