---
name: feasibility-compliance
description: Assesses whether an idea can actually be built and legally operated by an indie/startup founder — data sensitivity, regulatory regime (GDPR, health/financial/public-sector), the hosting model the data forces (public API vs EU-region managed vs self-hosted), the resulting infra/capital cost, and other gating dependencies (certifications, integrations, licenses, third-party lock-in). Finds the cheapest compliant path and judges it against the founder's budget tier. Run during validation for any idea touching personal/sensitive data, regulated sectors, AI applied to user data, or heavy infrastructure — it catches fatal feasibility problems cheaply, before deep market scoring.
---

<!-- version: 0.1.0 | outputs: memory/ideas/<slug>/feasibility.json -->

# Skill: feasibility-compliance

## Purpose

A great market doesn't matter if the founder cannot legally operate or affordably build the thing. Market-only analysis (demand, distribution, monetization) systematically misses this class of killer, so it has to be checked explicitly.

The most common indie/startup trap in this category: an idea handles **sensitive data** (health, biometric, financial, children, public-sector records) and therefore cannot route that data through a cheap public LLM/cloud API without a compliant data-processing path. The naive reaction — "then self-host the model" — assumes expensive GPU infrastructure a bootstrapped founder can't fund, and quietly kills otherwise-good ideas. But self-hosting is rarely the *only* compliant option: **EU-region managed services with a signed DPA** often satisfy data residency at a fraction of the cost. The job here is to find the **cheapest compliant path** and decide whether it's viable for *this* founder's budget — not to wave the idea through, and not to reflexively kill it because the strictest option is expensive.

## Input

- Idea slug; `memory/ideas/<slug>/idea.md`
- `memory/user_profile.md` — budget tier, geography, technical level. Feasibility is **relative to this founder**; the same idea can be feasible for a growth-tier team and infeasible for a hobbyist.
- Optional: `competitors.json` (how incumbents host/comply is evidence of the real requirement), `market_insights/` (region-specific rules).

## Step 1 — Classify data sensitivity

| Class | Examples | Compliance weight |
|---|---|---|
| none / public | aggregated stats, public listings, anonymous content | minimal |
| personal (GDPR ordinary) | names, emails, usage data, location | DPA + lawful basis |
| special-category | health, biometric, ethnicity, sexual orientation, union, children's data | high — explicit consent / strict basis, data minimization |
| financial | payment, account, credit data | PCI-DSS / PSD2 considerations |
| public-sector | municipal records, case files | procurement rules + often national residency / contractual on-prem |

## Step 2 — Identify regulatory regime & data residency

List the regimes that apply (GDPR/EU is the default for a Norway/EEA founder; add sectoral rules: health → national patient-data law + sector norms; finance → PSD2/PCI; public sector → procurement + residency). State the **data residency requirement**: none / EU-EEA / national / contractual on-prem. Anchor to evidence (law name, a competitor's stated hosting, a public tender's requirements) rather than assuming the strictest tier.

## Step 3 — Determine the required hosting/infra model (the cheapest compliant path)

Work **down** this ladder and stop at the first rung that is genuinely compliant for the data class — do not default to the bottom.

| Hosting model | Compliant for | Cost tier | When you're forced lower |
|---|---|---|---|
| Public LLM/cloud API (OpenAI, Anthropic, etc.) | none/public; sometimes personal w/ DPA + no-training terms | `$` usage-based | data is sensitive and provider lacks EU region/DPA |
| **EU-region managed API + signed DPA** (Azure OpenAI EU, AWS Bedrock EU, Vertex EU, Mistral EU) | personal & most special-category (with consent + minimization) | `$$` usage + small premium | law/contract forbids any external processor |
| EU-hosted open-weight model on managed GPU (EU inference providers) | special-category where a named EU processor is required | `$$$` low-thousands/mo | only on-prem is contractually allowed |
| Self-hosted / on-prem open model (own or managed GPU) | contractually on-prem-only data | `$$$$` tens-of-thousands capex + ops | — |
| On-device / edge inference | data never leaves the device | `$` (shifts cost to user HW) | model too large for target devices |

Cost legend: `$` trivial · `$$` hundreds/mo · `$$$` low-thousands/mo · `$$$$` tens-of-thousands capex.

**Key correction to the common error:** "we handle health data" does **not** automatically mean self-hosting. Many compliant health products run on EU-region managed services with a DPA + pseudonymization. Verify the actual legal/contractual requirement before assuming the `$$$$` rung — assuming it kills viable ideas.

## Step 4 — Estimate infra/capital cost and map to founder budget

Estimate monthly opex + any upfront capex for the cheapest compliant path. Compare against the founder's budget tier from `user_profile.md`:

| infra_cost_vs_budget | Meaning |
|---|---|
| within | cheapest compliant path fits the founder's budget tier comfortably |
| stretch | fits only at the top of the tier, or eats most of the experiment budget |
| exceeds | requires capital beyond the founder's tier (e.g., `$$$$` self-host on a hobby budget) → strong kill signal |

## Step 5 — Other gating dependencies

Flag anything else that blocks build/launch: required certifications (ISO 27001, HIPAA-equivalent, etc.), mandatory integrations (e.g., journal/ERP systems with closed APIs), licenses/authorizations, or hard third-party dependence/lock-in. Rate each `blocking` and effort `low|med|high`. A blocking dependency with no tractable path is as fatal as cost.

## Step 6 — Feasibility verdict

| feasibility_score | feasibility_verdict | Meaning |
|---|---|---|
| 75–100 | **feasible** | no special infra, or sensitive data fully served by an affordable EU-managed path; no blocking certs; integrations tractable |
| 50–74 | **feasible-with-constraints** | a viable compliant path exists at moderate cost/effort (DPA + pseudonymization, one cert, one integration); within tier budget |
| 25–49 | **capital-intensive** | requires self-host/on-prem, heavy certs, or large integration lift — viable only at growth tier / with funding; a kill for low budget tiers |
| 0–24 | **infeasible-for-indie** | no compliant path within this founder's budget/skills (e.g., must self-host large models on special-category data on a hobby budget; required license unattainable) |

Set `viability_killer = true` when `feasibility_verdict` is **infeasible-for-indie**, OR when `infra_cost_vs_budget = exceeds`, OR when a blocking dependency has no tractable path. A `viability_killer` should stop validation early — there is no point modeling pricing and CAC for something that cannot be built or operated.

## Process

1. Load `idea.md` and `user_profile.md`.
2. Classify data sensitivity (Step 1).
3. Identify regulatory regime + residency, anchored to evidence (Step 2).
4. Walk the hosting ladder to the cheapest compliant rung (Step 3).
5. Estimate cost and compare to budget tier (Step 4).
6. Flag other gating dependencies (Step 5).
7. Score feasibility, set verdict and `viability_killer` (Step 6).
8. Write `feasibility.json`. If `viability_killer`, state it loudly and recommend stopping or pivoting before further analysis.

## Output

Write to `memory/ideas/<slug>/feasibility.json`:

```json
{
  "data_sensitivity": "none | personal | special-category | financial | public-sector",
  "regulatory_regime": ["GDPR"],
  "data_residency_required": "none | EU-EEA | national | on-prem",
  "hosting_options": [
    { "model": "", "compliant_for_data_class": true, "monthly_cost_tier": "$ | $$ | $$$ | $$$$", "upfront_capex": "", "notes": "" }
  ],
  "cheapest_compliant_path": { "model": "", "monthly_cost_estimate": "", "upfront_estimate": "", "why": "" },
  "infra_cost_vs_budget": "within | stretch | exceeds",
  "gating_dependencies": [
    { "type": "certification | integration | license | third-party-api", "item": "", "blocking": true, "effort": "low | med | high" }
  ],
  "feasibility_score": 0,
  "feasibility_verdict": "feasible | feasible-with-constraints | capital-intensive | infeasible-for-indie",
  "viability_killer": false,
  "key_risks": [],
  "recommended_mitigation": ""
}
```

## Notes

- **Relative to the founder, not absolute.** Always re-read `user_profile.md`. "Capital-intensive" is a kill for a hobby budget but a normal cost line for a funded/growth-tier team.
- **Don't assume the strictest rung.** The single most expensive mistake is reflexively choosing self-hosting when an EU-region managed service is compliant. Equally, don't wave through a public-API plan for special-category data.
- **Feeds downstream skills.** `pricing-and-wtp` and `cac-modeler` should subtract the cheapest-compliant-path opex from margins; `idea-scoring` uses `feasibility_score` as a weighted dimension and `viability_killer` as a hard gate; `decision-memo` surfaces the path + cost and treats a killer as a top risk.
- **Geography matters.** Residency and available managed regions differ by country. For an EEA founder, EU-region managed services are usually the pragmatic compliant path.
