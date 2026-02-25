# Scenario Analysis — Interaction Specifications
**Vanguard Sell & Rebalance Tool · Screen S4 (Scenario Comparison)**
*Reconstructed February 22, 2026 · Derived from hifi wireframe v3, flow diagram v4, and data model*

---

## 1. Screen Entry — State Differences by Workflow

### Workflow A (Recommendation-first)
- User arrives via "Adjust" CTA on the Recommendation Summary screen
- **Scenario A pre-populated** from the optimization engine's output (fund amounts, lot selections, G/L)
- **Helper text visible:** "Pre-filled from Recommendation · Edit any amount to recalculate"
- **"↺ Reset to Recommendation" button** is shown in the top action bar
- `source_recommendation_id` is populated on the `ManualSellScenario` record → this is the conditional that drives the button's visibility

### Workflow B (Manual-first)
- User arrives via "Analyze Scenario →" CTA on the Fund List Manual screen
- **Scenario A pre-populated** from the user's manually entered amounts
- **Helper text:** "Edit any amount to recalculate" (no "Pre-filled from Recommendation" prefix)
- **"Reset to Recommendation" button is hidden** — no implied "correct" answer
- `source_recommendation_id` is null on the `ManualSellScenario` record

> **Engineering note:** The button's visibility is a single conditional check on `source_recommendation_id !== null`. Do not create two separate screen states — render the same component and show/hide the button based on this flag.

---

## 2. Top Action Bar

| Element | Wf-A | Wf-B | Behavior |
|---|---|---|---|
| Helper text | "Pre-filled from Recommendation · Edit any amount to recalculate" | "Edit any amount to recalculate" | Static label, updates copy only |
| ↺ Reset to Recommendation | Visible (ghost button) | **Hidden** | On click: restore all fund amounts from the linked `SellRecommendation` record; recalculate all G/L and tax fields |
| Save All Scenarios | Visible | Visible | Saves all in-progress scenario columns to the user's session; does not execute |
| Execute Best Scenario | Visible (primary CTA) | Visible (primary CTA) | Label updates dynamically to name the scenario with the lowest `estimated_total_tax`; see §6 |

---

## 3. Scenario Column Structure

Each scenario is rendered as a vertically stacked column. Up to **3 scenarios max** in v1 (side-by-side layout breaks beyond this without horizontal scroll).

### Column Header
- **Scenario name** — bold, 12px. Dashed underline style (indicates hover target)
  - On hover → **tooltip** appears with a sell breakdown summary: funds, dollar amounts, net G/L, estimated tax, and any user note
- **Badge** — shown when scenario was seeded from the optimization engine: `Recommended · VTSAX $6,000 + VBTLX $4,000 · harvests full available loss`
  - User-created or user-modified scenarios show `User-modified · [fund list]` or no badge
- **⋮ Kebab menu** → actions: Rename, Duplicate, Delete
  - Rename: inline edit of the scenario name field
  - Duplicate: copies all amounts into a new column (if < 3 scenarios exist)
  - Delete: removes the column; disabled if only 1 scenario remains

### Column Row Header
| Fund | Sell | G/L | Est. Tax |
|---|---|---|---|

### Fund Rows
- **Fund name** — ticker + account type label (e.g. "VTIAX (IRA)")
- **Sell amount** — editable input field, right-aligned, currency format (`$X,XXX`)
  - On change: debounce 400ms → recalculate `derived_shares`, `estimated_stcg`, `estimated_ltcg`, `estimated_stcg_tax`, `estimated_ltcg_tax`, `estimated_total_tax` for this row, then roll up to column totals and tax summary
  - Input border turns green when the amount triggers a net loss (tax-loss harvest)
  - IRA fund rows: sell amount input is still editable, but the G/L column shows "N/A — IRA" (no cap gains impact)
- **G/L cell** — clickable; shows net gain/loss value in green (positive) or red (negative)
  - On click → **G/L tooltip/popup** expands inline showing:
    - Short-term G/L
    - Long-term G/L
    - Divider
    - Net gain/loss
  - Only one G/L tooltip open at a time; clicking another cell closes the previous one; clicking outside closes all
- **Est. Tax** — calculated from `estimated_total_tax`; shown in green for losses (negative tax impact = harvested loss credit)

### Column Footer — Tax Summary
Per-scenario summary block below the fund rows:

| Line | Value |
|---|---|
| ST Capital Gains | Sum of `estimated_stcg` across all taxable rows |
| LT Capital Gains | Sum of `estimated_ltcg` across all taxable rows |
| Losses Harvested | Sum of rows where `is_tax_loss = true` |
| Net Taxable Gain | ST + LT − Losses |
| Est. Federal Tax | Applied at investor's federal rate |
| Est. State Tax | Applied at investor's state rate |
| **Est. Total Tax** | **Bold total** |
| Effective Rate on Sale | Total tax ÷ total sell amount |
| Comparison note | "costs $X more/less than Scenario A" — only shown on non-A scenarios |

---

## 4. Asset Mix Impact Section (per column)

Displayed below the tax summary. Shows the portfolio rebalancing impact of executing this specific scenario.

### Stacked Bar Charts
- **Current after sale** — projected allocations if this scenario is executed
- **Target** — investor's target allocation from `InvestorTargetAllocation`
- Segments: US Equity, Intl Equity, Fixed Income (color-coded)

### Asset Class Table
Columns: Asset Class | Before | After | Target | Diff.

- Each asset class row has an **expand button (▶)** → reveals fund-level drill-down rows showing ticker, account type, and asset class label
- `toggleDrill()` function handles open/close; button arrow flips ▶/▼

### Overall Drift Notice
- Text summary at the bottom: e.g. "US Equity moves further from target · consider selling bonds instead"
- Shown in alert color (`--negative`) when the scenario moves any asset class further from target

---

## 5. Add Scenario Column

- Shown as the rightmost "column" — dashed border, `+` icon, "Add Scenario" label
- On click: adds a new blank scenario column with all sell amounts = $0
- Disabled (visually suppressed, no click action) when 3 scenarios already exist
- Annotation note (b5): "A 'more scenarios' pattern (tabs or dropdown) should be considered for v2 if user research shows demand for more than 3"

---

## 6. Execute Actions

### Top Bar CTA: "Execute Best Scenario"
- Label is dynamic: identifies the scenario with the lowest `estimated_total_tax` and names it
  - e.g. "Execute Best Scenario" → resolves to "Execute Scenario A" if A has the lowest tax
- On click: routes to order confirmation / trade review flow

### Footer CTAs
Sticky footer bar at the bottom of the page, outside the scrollable scenario area:

| Button | Style | Behavior |
|---|---|---|
| Execute Scenario A ← saves $582 in tax | Primary | Executes Scenario A; routes to confirmation |
| Execute Scenario B · better rebalancing | Secondary | Executes Scenario B; routes to confirmation |

- Footer CTA copy is dynamic: tax savings difference is calculated vs. the most expensive scenario
- Helper text to the left of the CTAs: "Select a scenario to execute, or edit amounts above to create a custom scenario"

---

## 7. Interaction: Reset to Recommendation (Wf-A only)

1. User clicks "↺ Reset to Recommendation"
2. Confirm dialog (inline or modal): "This will restore Scenario A to the original recommendation amounts. Any edits you've made will be lost."
3. On confirm: restore `ManualFundSale.user_entered_amount` values from the linked `RecommendedFundSale.recommended_dollar_amount` records
4. Recalculate all G/L, tax summary, and Asset Mix Impact values
5. Re-apply the "Recommended" badge to Scenario A header

---

## 8. Interaction: G/L Tooltip

- **Trigger:** click on any G/L cell
- **Dismiss:** click the same cell again, click any other G/L cell (which opens its own), or click anywhere outside `.gl-cell` elements
- **Mutual exclusion:** `document.querySelectorAll('.gl-cell.open')` — only one open at a time
- **Content:**
  - Short-term gain/loss (colored)
  - Long-term gain/loss (colored)
  - Divider rule
  - Net gain/loss (colored)
- IRA rows: G/L cell is replaced with "N/A — IRA" text; no click behavior

---

## 9. Interaction: Kebab Menu (⋮)

Actions per scenario header:

**Rename**
- Inline: replace scenario name `<div>` with a text `<input>` pre-filled with current name
- On blur or Enter key: save, revert to display mode
- On Esc: cancel, revert to display mode without saving

**Duplicate**
- If fewer than 3 scenarios: create a new column to the right with all the same amounts, name = "[Original Name] (copy)"
- If 3 scenarios already exist: show tooltip "Maximum of 3 scenarios. Delete one to duplicate."

**Delete**
- Remove the column
- If only 1 scenario remains: Rename and Delete are disabled (Duplicate allowed as long as < 3 total)
- Footer CTAs update: remove the button for the deleted scenario

---

## 10. Data Recalculation Flow

On any `user_entered_amount` change (debounced):

```
Input change →
  Recalculate ManualFundSale (this row):
    derived_shares = user_entered_amount / current_nav
    estimated_stcg, estimated_ltcg  (lot-based calculation)
    estimated_stcg_tax, estimated_ltcg_tax, estimated_total_tax
    is_tax_loss = (estimated_stcg + estimated_ltcg) < 0
    allocation_after_sale
  →
  Roll up to ScenarioTaxImpact:
    total_sell_amount
    total_stcg, total_ltcg, total_losses_harvested
    net_taxable_gain
    est_federal_tax, est_state_tax, est_total_tax
    effective_rate
  →
  Update Asset Mix Impact:
    Recalculate per-asset-class Before/After/Target/Diff
    Update stacked bar widths
    Update overall drift notice
  →
  Update footer CTA copy and "best scenario" designation
```

---

## 11. Annotations Carried Over from Wireframe

| ID | Text |
|---|---|
| b1 | "Review & Adjust" heading. The label changes to "Scenario Comparison" in the sub-tab and page header, but the action bar inside uses "Review & Adjust" to signal editability. These must stay in sync. |
| b2 | "Reset to Recommendation" is only shown when `source_recommendation_id` is populated (Wf-A). It re-anchors the user to the system's recommendation after experimentation. |
| b3 | "Execute Best Scenario" highlights the lowest-tax scenario but allows override. Button label updates dynamically. |
| b4 | Scenario A carries the "Recommended" badge when seeded from the optimization engine. User-created scenarios do not. This preserves the recommendation's authority without preventing deviation. |
| b5 | Max 3 scenarios for v1. A tabs or dropdown pattern should be considered for v2 if user research shows demand for more. |
| gl1 | G/L cell click reveals the ST vs LT breakdown. This is the primary way users understand the tax character of each sale without cluttering the row with extra columns. |

---

## 12. Out of Scope for v1

- More than 3 simultaneous scenarios (tabs/dropdown pattern deferred)
- Saving scenarios to a persistent named list across sessions
- Sharing scenarios with a financial advisor
- Printing individual scenario columns

---

*Document status: Reconstructed from hifi wireframe v3, flow diagram v4, withdrawal-tool-data-model.md, and Vanguard design system · Ready for Figma annotation layer*
