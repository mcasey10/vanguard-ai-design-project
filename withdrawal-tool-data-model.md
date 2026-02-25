# Sell &amp; Rebalance Tool — Data Model
*Generated from product brief: Vanguard investor sell &amp; rebalance tool*
*Solving for: portfolio rebalancing + tax loss harvesting on periodic sales*
*Revised: data model grounded against actual Vanguard unrealized gain/loss reporting (02/17/2026)*

---

## Entities Overview

```
Investor
  └── Account (one or more)
        └── FundHolding (one per fund per account)
              └── TaxLot (one per purchase event)
                    └── TaxLotDisposition (created when lot is sold)

PortfolioTarget (investor's desired allocation)
  └── TargetAllocation (one per asset class)

SellRequest (workflow A entry point)
  └── SellRecommendation (output of optimization engine)
        └── RecommendedFundSale (one per fund)

ManualSellScenario (workflow B entry point)
  └── ManualFundSale (one per fund, user-entered)
        └── ScenarioTaxImpact (calculated output)
```

---

## Entity Detail

---

### Investor
The authenticated user. Relatively simple — most complexity lives downstream.

| Field | Type | Notes |
|---|---|---|
| investor_id | UUID | Primary key |
| name | string | Display name |
| tax_filing_status | enum | single, married_joint, married_separate, head_of_household |
| federal_tax_bracket | decimal | Marginal rate, e.g. 0.22 |
| state_tax_rate | decimal | Combined state + local rate |
| ltcg_tax_rate | decimal | Long-term capital gains rate (0, 0.15, or 0.20 federally) |
| stcg_tax_rate | decimal | Short-term rate — same as ordinary income bracket |
| niit_applies | boolean | Net Investment Income Tax (3.8%) applies above certain thresholds |
| ytd_realized_gains | decimal | Running total of realized gains this tax year — affects bracket |
| ytd_realized_losses | decimal | Running total of realized losses this tax year |
| preferred_accounting_method | enum | fifo, lifo, hifo, mintax, specific_identification — **mintax added**: Vanguard's proprietary method that algorithmically selects lots to minimize tax impact; likely the default for this tool's target users |

> ⚠️ **Data challenge:** Tax rates require either user input or integration with a tax estimation service. They also change intra-year if significant income events occur. This field should be treated as an estimate, not a guarantee, and the UI must communicate that clearly.

---

### Account
Vanguard investors often hold funds across multiple account types with very different tax treatments.

| Field | Type | Notes |
|---|---|---|
| account_id | UUID | Primary key |
| investor_id | UUID | Foreign key → Investor |
| account_type | enum | taxable_brokerage, traditional_ira, roth_ira, 401k, 403b, inherited_ira |
| account_name | string | e.g. "Joint Brokerage", "Rollover IRA" |
| is_tax_advantaged | boolean | Derived from account_type |
| requires_rmd | boolean | Required Minimum Distributions apply (traditional IRA, 401k over age 73) |
| rmd_amount_current_year | decimal | If applicable — constrains or drives sell decisions |
| total_value | decimal | Calculated: sum of all FundHolding.current_value |

> ⚠️ **Data challenge:** Account type is critical because capital gains rules only apply to taxable accounts. Sales from traditional IRAs are taxed as ordinary income. Sales from Roth IRAs may be tax-free. The optimization engine must treat these very differently — this is one of the most complex aspects of the tool.

---

### FundHolding
One record per fund per account. This is the core entity the user sees in workflow B.

| Field | Type | Notes |
|---|---|---|
| holding_id | UUID | Primary key |
| account_id | UUID | Foreign key → Account |
| fund_id | UUID | Foreign key → Fund (see below) |
| total_shares | decimal | Sum of all open tax lots |
| current_nav | decimal | Net asset value per share, current |
| current_value | decimal | Calculated: total_shares × current_nav |
| total_cost_basis_fifo | decimal | Calculated from TaxLots using FIFO method |
| total_cost_basis_lifo | decimal | Calculated from TaxLots using LIFO method |
| total_cost_basis_hifo | decimal | Calculated: Highest In, First Out — minimizes gains |
| total_cost_basis_mintax | decimal | Vanguard MinTax method — lot selection suppresses a single cost-per-share display (shown as "—" in Vanguard UI) |
| total_cost_basis_average | decimal | Average cost method — simpler but less flexible |
| cost_per_share_display | decimal | **Nullable** — suppressed when accounting method is mintax or specific_identification, as no single average is meaningful |
| stcg_unrealized | decimal | Short-term unrealized gain/loss — sum of lots held < 1 year |
| ltcg_unrealized | decimal | Long-term unrealized gain/loss — sum of lots held ≥ 1 year |
| total_unrealized_gain_loss | decimal | stcg_unrealized + ltcg_unrealized — matches "Total capital gain/loss" in Vanguard UI |
| total_unrealized_pct | decimal | Percent gain/loss — matches "Percent gain/loss" in Vanguard UI |
| has_any_unrealized_loss | boolean | True if any lot has an unrealized loss — key flag for tax loss harvesting |
| has_stcg_loss_lots | boolean | True if any short-term lots are currently at a loss — highest priority for harvesting |
| has_ltcg_loss_lots | boolean | True if any long-term lots are at a loss |
| asset_class | enum | us_equity, intl_equity, us_bond, intl_bond, reit, cash_equivalent |
| target_allocation_pct | decimal | From PortfolioTarget — used for rebalancing calc |
| current_allocation_pct | decimal | Calculated: current_value / portfolio_total_value |
| allocation_drift | decimal | current_allocation_pct − target_allocation_pct |

> ⚠️ **Data challenge:** Vanguard's API (or data export) must provide lot-level detail for cost basis to be accurate. Many years of reinvested dividends create dozens or hundreds of lots per fund. If lot data isn't available programmatically, users may need to import it manually — a significant UX friction point that could undermine the tool's value proposition.

---

### TaxLot
The most granular and most important entity for tax calculations. One record per purchase event (including dividend reinvestments).

| Field | Type | Notes |
|---|---|---|
| lot_id | UUID | Primary key |
| holding_id | UUID | Foreign key → FundHolding |
| acquisition_date | date | Purchase or reinvestment date — the single most important field for determining short vs long term |
| shares_acquired | decimal | Shares purchased in this lot |
| shares_remaining | decimal | Shares not yet sold — decremented as sold |
| cost_per_share | decimal | Price paid at acquisition (e.g. $633.96 for 12/22/2025 lot in screenshot) |
| total_cost_basis | decimal | Calculated: shares_remaining × cost_per_share |
| current_nav | decimal | Current net asset value per share — same for all lots of same fund |
| current_value | decimal | Calculated: shares_remaining × current_nav |
| unrealized_gain_loss | decimal | current_value − total_cost_basis — positive or negative |
| unrealized_gain_loss_pct | decimal | unrealized_gain_loss / total_cost_basis — matches "Percent gain/loss" per lot in Vanguard UI |
| holding_period | enum | **short_term** (< 1 year from acquisition_date) or **long_term** (≥ 1 year) — mutually exclusive per lot, never both. A lot is short_term or long_term, determined purely by acquisition_date vs today. |
| holding_period_days | integer | Days since acquisition_date |
| days_until_long_term | integer | **Null if already long_term**. If short_term: days remaining until 1-year mark. Strategically important — a lot 5 days from long-term status should not be sold short-term. |
| stcg_amount | decimal | **Populated only if short_term** — the gain/loss if sold today, taxed at ordinary income rate. Null if long_term. Mirrors Vanguard's "Short term capital gain/loss" column which shows "—" for long-term lots. |
| ltcg_amount | decimal | **Populated only if long_term** — the gain/loss if sold today, taxed at preferential LTCG rate. Null if short_term. Mirrors Vanguard's "Long term capital gain/loss" column which shows "—" for short-term lots. |
| is_harvestable_loss | boolean | True if unrealized_gain_loss < 0 AND wash sale rules don't restrict sale |
| is_wash_sale_restricted | boolean | Cannot harvest loss if substantially identical fund purchased within 30 days |
| wash_sale_restriction_expires | date | If applicable |
| lot_type | enum | purchase, dividend_reinvestment, gift, transfer |

> ⚠️ **Data challenge:** Wash sale rules make this significantly more complex. If an investor sells a fund at a loss and buys a "substantially identical" fund within 30 days before or after, the loss is disallowed. Determining what counts as "substantially identical" for mutual funds is a grey area that the tool must handle carefully — and probably with a legal disclaimer.

> ⚠️ **Data challenge:** Dividend reinvestments over many years can produce hundreds of lots. The UI for workflow B must present this in an aggregated, comprehensible way while still allowing lot-level inspection.

---

### Fund
Reference data about each mutual fund. Relatively static.

| Field | Type | Notes |
|---|---|---|
| fund_id | UUID | Primary key |
| ticker | string | e.g. VTSAX |
| name | string | e.g. "Vanguard Total Stock Market Index Fund" |
| asset_class | enum | Maps to asset_class on FundHolding |
| expense_ratio | decimal | Informational |
| fund_family | string | e.g. "Vanguard" |
| is_index_fund | boolean | Relevant for wash sale — index funds less likely to trigger wash sale rules |
| benchmark_index | string | e.g. "CRSP US Total Market Index" |

---

### PortfolioTarget
The investor's desired allocation. Used to calculate drift and drive rebalancing recommendations.

| Field | Type | Notes |
|---|---|---|
| target_id | UUID | Primary key |
| investor_id | UUID | Foreign key → Investor |
| created_date | date | Targets can change over time |
| rebalancing_threshold_pct | decimal | How much drift triggers a rebalancing flag, e.g. 5% |
| is_active | boolean | Current active target |

---

### TargetAllocation
One record per asset class within a PortfolioTarget.

| Field | Type | Notes |
|---|---|---|
| allocation_id | UUID | Primary key |
| target_id | UUID | Foreign key → PortfolioTarget |
| asset_class | enum | Must match FundHolding.asset_class values |
| target_pct | decimal | e.g. 0.55 for 55% US equity |

---

### SellRequest
Workflow A entry point. The user enters a dollar amount and asks for a recommendation.

| Field | Type | Notes |
|---|---|---|
| request_id | UUID | Primary key |
| investor_id | UUID | Foreign key → Investor |
| requested_amount | decimal | Dollar amount the investor wants to sell |
| accounting_method | enum | Override for this request — defaults to investor preference |
| optimization_priority | enum | minimize_gains, maximize_rebalancing, balanced |
| requested_date | timestamp | When request was made |
| status | enum | pending, recommended, partially_executed, executed |

---

### SellRecommendation
The output of the optimization engine for workflow A. Also serves as the pre-populated starting point for workflow B if the user wants to inspect or adjust.

| Field | Type | Notes |
|---|---|---|
| recommendation_id | UUID | Primary key |
| request_id | UUID | Foreign key → SellRequest |
| total_recommended_amount | decimal | May differ slightly from requested due to share constraints |
| total_estimated_tax_impact | decimal | Estimated tax owed on recommended sales |
| total_estimated_ltcg | decimal | Long-term capital gains portion |
| total_estimated_stcg | decimal | Short-term capital gains portion |
| estimated_portfolio_drift_after | decimal | How balanced the portfolio will be post-sale |
| rebalancing_improvement_score | decimal | How much better balanced vs current state |
| generated_at | timestamp | |
| explanation_summary | string | Plain-language explanation of why these funds were chosen |

> 💡 **Design note:** The explanation_summary field is important for trust. Users need to understand *why* the tool recommended selling fund X before fund Y. AI-generated natural language explanations are a strong candidate here.

---

### RecommendedFundSale
One record per fund within a recommendation. The line-item detail.

| Field | Type | Notes |
|---|---|---|
| line_id | UUID | Primary key |
| recommendation_id | UUID | Foreign key → SellRecommendation |
| holding_id | UUID | Foreign key → FundHolding |
| recommended_shares | decimal | Shares to sell |
| recommended_dollar_amount | decimal | Estimated value at current NAV |
| estimated_stcg | decimal | Short-term capital gain/loss component of this sale |
| estimated_ltcg | decimal | Long-term capital gain/loss component of this sale |
| estimated_net_gain_loss | decimal | estimated_stcg + estimated_ltcg |
| estimated_stcg_tax | decimal | estimated_stcg × investor's short-term rate (ordinary income) |
| estimated_ltcg_tax | decimal | estimated_ltcg × investor's long-term rate (0%, 15%, or 20%) |
| estimated_total_tax | decimal | estimated_stcg_tax + estimated_ltcg_tax |
| lots_selected | array[lot_id] | Specific lots chosen — critical for mintax and specific_identification methods |
| days_until_lt_on_next_lot | integer | If any selected lots are short-term and near the 1-year boundary — surfaces the "wait N days to save $X" opportunity |
| rebalancing_rationale | string | e.g. "Over-allocated to US equity by 4.2%" |

---

### ManualSellScenario
Workflow B entry point. User manually enters amounts per fund and sees the tax impact.

| Field | Type | Notes |
|---|---|---|
| scenario_id | UUID | Primary key |
| investor_id | UUID | Foreign key → Investor |
| source_recommendation_id | UUID | Foreign key → SellRecommendation — null if started fresh, populated if pre-filled from workflow A |
| accounting_method | enum | User's choice for this scenario |
| created_at | timestamp | |
| last_modified | timestamp | |
| scenario_name | string | Optional user label, e.g. "Option A — minimize tax" |

---

### ManualFundSale
One record per fund in a manual scenario. User-editable.

| Field | Type | Notes |
|---|---|---|
| manual_line_id | UUID | Primary key |
| scenario_id | UUID | Foreign key → ManualSellScenario |
| holding_id | UUID | Foreign key → FundHolding |
| user_entered_amount | decimal | Dollar amount entered by user |
| derived_shares | decimal | Calculated: user_entered_amount / current_nav |
| estimated_stcg | decimal | Short-term portion of gain/loss based on lots that would be sold |
| estimated_ltcg | decimal | Long-term portion of gain/loss based on lots that would be sold |
| estimated_stcg_tax | decimal | Tax on short-term portion at ordinary income rate |
| estimated_ltcg_tax | decimal | Tax on long-term portion at preferential LTCG rate |
| estimated_total_tax | decimal | estimated_stcg_tax + estimated_ltcg_tax |
| is_tax_loss | boolean | True if this sale harvests a net loss |
| allocation_after_sale | decimal | Portfolio weight after this sale |

---

### ScenarioTaxImpact
Rolled-up tax impact for the entire manual scenario. Recalculated on every change.

| Field | Type | Notes |
|---|---|---|
| impact_id | UUID | Primary key |
| scenario_id | UUID | Foreign key → ManualSellScenario |
| total_sale_amount | decimal | Sum of all ManualFundSale amounts |
| total_stcg | decimal | Sum of short-term gains across all funds — taxed at ordinary income rate |
| total_ltcg | decimal | Sum of long-term gains across all funds — taxed at preferential rate |
| total_st_losses | decimal | Short-term losses harvested — offset short-term gains first, then long-term |
| total_lt_losses | decimal | Long-term losses harvested — offset long-term gains first, then short-term |
| net_stcg_after_losses | decimal | Short-term gains net of loss offsets |
| net_ltcg_after_losses | decimal | Long-term gains net of loss offsets |
| estimated_federal_st_tax | decimal | net_stcg_after_losses × ordinary income rate |
| estimated_federal_lt_tax | decimal | net_ltcg_after_losses × LTCG rate (0%, 15%, or 20%) |
| estimated_niit | decimal | 3.8% surcharge if investor.niit_applies and gains exceed threshold |
| estimated_state_tax | decimal | State rate applied to net gains |
| estimated_total_tax | decimal | All of the above combined |
| effective_tax_rate_on_sale | decimal | estimated_total_tax / total_sale_amount — the single most important summary number for the user |
| portfolio_drift_after | decimal | How balanced the portfolio will be |
| rebalancing_delta_vs_target | json | Per asset class drift after sale |

---

## Workflow Convergence Point

The connection between workflow A and workflow B is the `source_recommendation_id` field on `ManualSellScenario`. When a user completes workflow A and receives a `SellRecommendation`, the system can instantiate a `ManualSellScenario` pre-populated with the recommended amounts. The user can then edit individual fund amounts and see the tax impact recalculate in real time.

This means the UI can present a single "Review & Adjust" screen that serves as both the output of workflow A and the input of workflow B — the convergence is a data handoff, not a navigation event.

---

## Key Data Challenges Summary

| Challenge | Severity | Mitigation |
|---|---|---|
| Lot-level cost basis data availability | High | Require Vanguard API access or CSV import |
| Tax rate accuracy | Medium | User inputs + clear disclaimer that estimates are not tax advice |
| Wash sale rule determination | High | Conservative flagging + legal disclaimer |
| RMD calculations | Medium | Age + account type + IRS table lookup |
| Intra-year tax bracket changes | Low | Annual snapshot with note to revisit |
| Account type tax treatment differences | High | Must be core to optimization logic, not an afterthought |

---

## Observations from Vanguard's Existing UI (VFIAX Example, 02/17/2026)

The Vanguard unrealized gain/loss report informed several refinements to this data model:

**Short-term and long-term are mutually exclusive per lot.** Vanguard's UI shows either a short-term value or a long-term value per lot row — never both. The other column displays "—". This confirmed that `stcg_amount` and `ltcg_amount` on TaxLot must be mutually exclusive, determined solely by whether `acquisition_date` is within or beyond one year of today.

**MinTax is a real accounting method users already use.** The screenshot shows "MinTax" selected for VFIAX. This is Vanguard's proprietary method and should be the default for this tool since its users are already optimizing for tax minimization. It was missing from the original data model.

**Cost-per-share is suppressed under MinTax.** The holding-level "Cost per share" field shows "—" when MinTax is selected. This informs the UI design: `cost_per_share_display` must be nullable and the UI should handle its absence gracefully.

**Individual lots within a large-gain fund can have losses.** The 12/22/2025 lot shows −$0.24 short-term loss despite the fund having +$26,142 total long-term gain. This is a meaningful design implication: the tool should surface harvestable losses at the lot level even when the overall fund position is highly profitable.

**Dividend reinvestments create the lot complexity.** The quarterly lot dates (12/22, 09/29, 06/30, 03/27...) are VFIAX dividend reinvestment dates. A long-term investor may have dozens of these lots per fund — confirming that lot-level data access is a hard requirement, not a nice-to-have.

**The short-term/long-term boundary creates a "wait and save" opportunity.** Lots from 12/22/2025 are currently short-term (taxed at ordinary income rates). They become long-term on 12/22/2026. The tool should flag when a short-term lot is within a configurable window (e.g. 60 days) of converting to long-term — surfacing the question "would you save meaningful tax by waiting?"

---

*This data model is a starting point for design and engineering alignment. It should be reviewed by a tax professional before any calculations are represented to users as financial advice.*
