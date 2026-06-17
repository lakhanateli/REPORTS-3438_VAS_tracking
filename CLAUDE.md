# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

VAS (Value Added Services) tracking and revenue attribution for OneAssist — **REPORTS-3438**, part of the `REPORTS-1623_Analysis` parent project. Sits alongside `REPORTS-3381` (SR-level risk scoring), `REPORTS-3382` (BU risk document), and `REPORTS-3439` (cross-sell). VAS tracking outputs feed directly into the cross-sell Awareness scoring model (`vas_redeemed` parameter, weight = 3).

**Scope: All OneAssist categories** — PE, HA, Finance, Motor, Furniture. Never scope to a single category.

## Running Analysis

No build system or test suite. Execute scripts directly:

```bash
python3 <script_name>.py
```

Dependencies: `pandas`, `numpy`, `openpyxl`.

**No Python scripts exist yet.** The Athena-based dashboard query was built directly in the AWS console (May 18–21). When Python scripts are added, follow the naming convention `vas_<descriptor>_<version>.py` and output `vas_<descriptor>_<version>.csv`.

## VAS Classification & Revenue Attribution Logic

The four-step pipeline that defines this project. Any script written here must implement all four steps.

### Step 1 — Service Tagging

Every service on a membership is pre-tagged Core or VAS.

| Type | Examples |
|------|----------|
| Core | ADLD, Extended Warranty, Theft Cover, HA Breakdown |
| VAS | Roadside Assistance, Dark Web Monitoring, Personal Accident, OTT Vouchers |

Source of tags: the working spreadsheet Sheet 1. *(No local copy of this spreadsheet exists in the repo — it is maintained externally by the business team.)*

### Step 2 — Voucher COGS Mapping

For VAS that includes a voucher/OTT benefit, effective COGS depends on distribution mode:

| Distribution Mode | COGS Calculation |
|------------------|-----------------|
| AUTOMATIC | 100% of defined cost (always counts) |
| ONDEMAND | Defined cost × actual redemption % |

Example: Cost = ₹50, Redemption = 20% → Effective COGS = ₹10.

Source: Sheet 2.

### Step 3 — Membership Classification

Each membership is classified into one bucket:

| Tag | Condition |
|-----|-----------|
| ONLY CORE | Core services only, no VAS |
| VAS | VAS or Voucher only, no Core |
| CORE + VAS | Both Core and VAS present |

Source: Sheet 3 — Grid 1.

### Step 4 — Revenue Attribution

Revenue is split across services by COGS weight:

```
Revenue Share (service) = (COGS of service / Total COGS) × Total Plan Revenue
```

Example — ₹500 plan:

| Service | COGS | Cost % | Revenue Share |
|---------|------|--------|---------------|
| ADLD | ₹50 | 62.5% | ₹313 |
| Amazon Voucher (ONDEMAND, 20% redemption) | ₹20 | 25.0% | ₹125 |
| RSA | ₹10 | 12.5% | ₹62 |
| Total | ₹80 | 100% | ₹500 |

Source: Sheet 3 — Grid 2.

## Relationship to Sibling Projects

- **Canonical logic doc:** `../REPORTS-3439_cross_sell/docs/vas_classification_revenue_logic.md` — the markdown source for the logic above.
- **Cross-sell context:** `../REPORTS-3439_cross_sell/CLAUDE.md` — explains how `vas_redeemed` fits into the Awareness × Affluence scoring model. Read before adding VAS behavioral dimensions.
- **Coding patterns:** `../REPORTS-3381_sr_level_risk_scoring/CLAUDE.md` — canonical reference for Python script architecture, Excel state-machine parsing, normalization formula, and deduplication patterns used across this analysis family.

## Athena Data Sources

The VAS dashboard query runs in AWS Athena against these tables:

| Table | Role |
|-------|------|
| `oa_membership_policy_service_product_agg_dtl` | Service-level membership data; `ic_premium_system` is the COGS column for Core/VAS services |
| `vas_voucher_cost_mapping` | Voucher cost and redemption rate lookup; `voucher_cost × (redemption% ÷ 100)` = effective voucher COGS |

COGS from both sources are unioned per `mem_id`. Revenue attribution (Step 4) uses a window sum of total COGS over `mem_id` to compute each service's cost percentage, then multiplies by `final_pre_tax_revenue`.

## Data File Hygiene

- **Excel lock files** (`~$*.xlsx`) are Office temp files — always read the versions without the `~$` prefix.
- Raw data CSVs may contain duplicate rows from join expansion — always deduplicate on the membership or SR key before analysis.
- Commit scripts and CLAUDE.md updates; do not commit large CSVs or PDF reports unless explicitly needed.

## Cross-Sell Integration — Awareness Score Findings (2026-06-09)

*This section records findings from the cross-sell scoring review that are directly relevant to VAS tracking output. The full scoring model lives in `../REPORTS-3439_cross_sell/CLAUDE.md`.*

### Bucketing Logic (Low / Medium / High)
- Percentile-based terciles: bottom 33% = Low, 33–66% = Medium, >66% = High.
- Majority of base (~66%) sits at scores 0–4 → the High bucket starts at score 5, which is a low absolute bar.
- Score 5 customers are dominated by `tenure_over_10m` (weight = 3, ~13L customers) + one passive attribute (e.g., `active_membership`).

### Open Issue — ~40% Base Falling in "High Awareness"
- Flagged as counterintuitive for a non-customer-facing brand.
- Root cause: `tenure_over_10m` (weight = 3) alone pushes customers to score 3; one additional passive attribute reaches score 5 = High.
- `active_membership` is present for virtually all customers, inflating scores across the board.

### Proposed Parameter Tweaks
1. **Reduce `tenure_over_10m` weight** from 3 → 1; tenure is a loyalty signal, not an awareness signal.
2. **Add a minimum absolute score threshold** for High (e.g., ≥ 8 or ≥ 10) rather than relying purely on percentile cutoff.
3. **Require at least one active engagement attribute** (login, comm clicked, app used) for a customer to qualify as High — passive attributes alone should not be sufficient.
4. **Re-evaluate `active_membership`** as an awareness parameter — it is present by default and should be removed or down-weighted.

### COGS Logic (confirmed in query review)
- Service COGS = `ic_premium_system` summed per service from `oa_membership_policy_service_product_agg_dtl`.
- Voucher COGS = `voucher_cost × (redemption% ÷ 100)` from `vas_voucher_cost_mapping`.
- Both are unioned per membership; each service type's cost % = its COGS ÷ total COGS (window sum over `mem_id`).
- Cost % × `final_pre_tax_revenue` = revenue attributed to Core / VAS / Voucher per membership.

### Membership Tagging (confirmed)
- When both Core and VAS services exist → tagged **CORE + VAS** (no further sub-tagging; revenue split handled downstream via COGS weight).

## Work Log

- **2026-05-18** — Lakhan Gupta: VAS Athena query development.
- **2026-05-19** — Lakhan Gupta: Continuing Athena query work with VAS logic discussed and aligned with business team.
- **2026-05-21** — Lakhan Gupta: VAS tracking query and dashboard completed; currently in data validation stage by business team.
- **2026-06-03** — Lakhan Gupta: Working on Juhi query. *(context unknown — update this entry with query purpose and output)*
- **2026-06-09** — Lakhan Gupta: VAS & Core revenue logic finalised for dashboard; awareness score bucketing logic reviewed; ~40% High awareness issue flagged with proposed parameter tweaks (tenure_over_10m weight, min score threshold, active engagement gate).

## Script Architecture (when code is added)

Follow the pattern from `../REPORTS-3381_sr_level_risk_scoring/CLAUDE.md`:

1. **Load & deduplicate** — read raw CSV, drop duplicate rows on `mem_id` (the primary key here, not `sr_no`).
2. **Excel threshold/tag parsing** — state-machine over rows: column B matches a header → set current context; subsequent rows are `(key, value)` pairs.
3. **Per-row classification/scoring** — apply Steps 1–4 of VAS logic above.
4. **Output CSV** — naming convention `vas_<descriptor>_<version>.csv`; percentage columns as `raw × 100` for display.
