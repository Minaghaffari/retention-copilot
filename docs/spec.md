# Retention Copilot: Frozen Specification (v3.1)

Status: **frozen for v1 implementation**, 2026-10-21 (v3.1 applies the final review; no further scope expansion). Changes after this point go through an ADR in `docs/adr/`.
All data and results in this project are **synthetic**.

## 1. Research and product question

Under a fixed weekly review capacity and action budget, does assigning at-risk paid accounts to a **risk-ranked LLM recommendation policy** improve 60-day paid retention compared with **no intervention**, and compared with a **rules-based policy** that has the same constraints?

Prediction, explanation and intervention are kept separate:

| Question | Answered by | Claim allowed |
|---|---|---|
| Who is likely to churn? | Churn model | Calibrated risk |
| What evidence raised the risk? | SHAP and usage summary | Descriptive only, never causal |
| Which action improves retention? | Randomized experiment | Causal effect of *assigning a policy* |

Recommendations are grounded in an **approved, eligible playbook**. They never claim that changing a SHAP driver will prevent churn.

## 2. Setting and definitions

**Product.** "Palette" is a fictional freemium design SaaS with Free, Pro (monthly or annual) and Teams plans.

**Scoring.** Every Monday *t*, each paid account is scored using only data whose `event_time ≤ t` **and** `ingested_time ≤ t`.

**Churn label (model target).** `churn_30 = 1` if the account's paid subscription ends (cancelled or not renewed) in (t, t+30], and it is not paid again by end date + 7 days. Involuntary churn (payment failure) is labelled separately. A label only exists when the full t+37 window has been observed.

**Primary outcome: 60-day paid retention.** For an account enrolled on Monday t₀, `Y = 1` if, on day t₀+60, it holds an active paid subscription (Pro or Teams, either billing period, including a grace period of 7 days or less past due). Free or cancelled-and-ended accounts count as `Y = 0`. An account that churned and re-subscribed by t₀+60 counts as retained. Discounted and downgraded (still paid) accounts count as retained; their cost shows up in net revenue. There is no pause option in v1.

**Secondary outcome: customer preservation.** `P = 1` if, on day t₀+60, the account is paid, **or** it is on Free with at least one session in days t₀+31 to t₀+60. This captures relationships that are kept without paid revenue.

**Net revenue retained.** Measured per enrolled account over the window [t₀, t₀+60):

- Recognised subscription revenue is accrued daily at the price actually charged, net of any discount. Annual plans are prorated daily, and plan changes are prorated from the day they take effect.
- Refunds are subtracted.
- A fixed cost per CSM contact is subtracted.
- LLM inference cost is reported as a separate line.

The same accounting applies in all three arms, and the result is analysed under ITT as a difference from control.

## 3. Timeline

| Period | Dates | Purpose |
|---|---|---|
| Acquisition | 2024-01-01 → 2025-11-30 | New sign-ups arrive |
| Simulated history | 2024-01-01 → **2026-03-31** | Includes follow-up buffer |
| Model train (scoring dates) | 2024-07-01 → 2025-01-27 | |
| Model validation | 2025-02-03 → 2025-04-28 | Includes mobile export bug (Mar 2025) |
| Model test | 2025-05-05 → 2025-07-21 | Includes price rise (1 Jun 2025) |
| Production model freeze | 2025-08-31 | Trained on all labels observed by then |
| Experiment enrollment | 12 Mondays, 2025-09-01 → 2025-11-17 | Extendable by the power gate (§6) |
| Outcome follow-up | last t₀+60 = 2026-01-16 | ≈ 10-week buffer to data end |

A competitor launch slowly raises the baseline hazard from 2025-01. Model monitoring also runs on the experiment period, using labels observed up to 2026-03.

## 4. Capacity, eligibility and caps

These are four different quantities:

1. **Candidate pool** (per week): paid accounts with calibrated risk ≥ the 85th percentile of the paid base. They must also have no retention contact in the last 30 days, never have been enrolled before, and have no pending cancellation already processed. Typically a few thousand accounts.
2. **Enrollment** (per week): **E = 600** accounts, sampled at random from the candidate pool *before* any recommendation is generated. They are then randomized 1:1:1, giving 200 per arm.
3. **Review capacity** (per week): **400 human reviews**, which is exactly the two treatment arms. Control needs no review.
4. **Action caps** (per treatment arm, per week, identical in both arms): A3 discount ≤ 50, A5 CSM outreach ≤ 20 (Teams only). Every policy outputs a **ranked top-2** of eligible actions. When a cap binds, the cap goes to the highest-risk accounts, and the rest receive their second choice, or A0 if that is also capped. Overflow is logged.

**One enrollment per account.** Once randomized, an account keeps its arm and is never re-enrolled. It is excluded from any other retention contact during its 60-day window, so there is no overlapping exposure.

## 5. Policies

| Policy | Action selection | Inputs |
|---|---|---|
| π_control | A0 (business as usual) | — |
| π_rules | Deterministic mapping: top drivers × segment → ranked eligible playbooks | Usage summary, score, drivers, eligibility |
| π_LLM | One structured Bedrock call that returns a ranked top-2 from the eligible set, plus an explanation, cited evidence, uncertainty and a draft message | Same inputs, plus playbook text |

**Playbooks.** A0 none · A1 feature-education email · A2 in-app nudge · A3 20% off for 3 months · A4 paid-plan right-sizing offer (Pro monthly → Pro annual at a lower monthly equivalent, or fewer Teams seats; the account always stays on a paid plan) · A5 CSM outreach (Teams only). Eligibility is enforced in code.

**LLM output contract.** The output is validated with Pydantic. Any action outside the eligible set, or any cited number that doesn't match the inputs, is rejected. After one retry, a failure becomes **A0, logged as a policy failure**. It never falls back to the rules policy, because that would contaminate the comparison. Failures count against the LLM arm under ITT.

**Same operations in both arms.** Both treatment arms use the same review queue, and reviewers are blind to the arm. The simulated reviewer uses one shared model, in which the probability of approve, edit or reject depends on recommendation features, not on the arm. Approval, edits, rejection, delivery and opens are logged per account in both arms.

## 6. Experiment analysis and the power gate

**Hypotheses.**

- *Primary*: H1 LLM vs control and H2 rules vs control, each two-sided at α = 0.025 (Bonferroni across two comparisons), with 80% power.
- *Secondary*: LLM vs rules at α = 0.05, reported with its minimum detectable effect and not over-claimed.

**Estimand.** ITT is the **only causal estimand in v1**: the effect of *assigning* a policy, including rejections, delivery failures and LLM failures. Approval, edit, delivery and open rates are reported per arm and by action as **descriptive** results only. There is no IV or treatment-on-treated estimate in v1, because "treated" is not a single binary treatment across different actions.

**Estimator.** A regression-adjusted difference in proportions: OLS of Y on arm dummies, centred pre-period covariates (log1p of sessions in the 28 days before t₀, and the risk score at t₀), and their interactions with the arm (Lin's estimator, the regression form of CUPED). Standard errors are HC2-robust. The unadjusted difference is reported alongside. **Checks:** an SRM χ² test on arm counts, 95% confidence intervals, and a breakdown by cohort week.

**Power gate.** This runs *before* the experiment simulation. Using baseline 60-day retention from historical accounts that would have been eligible:

- **p₀ comes from the simulation.** It is measured on accounts that would have been eligible (same eligibility rules, frozen model) during a no-treatment dry run of the enrollment period. No number is chosen independently.
- *Planning value only*: a monthly paid churn of 3–6% overall implies roughly 88–94% 60-day retention for the average account. For the top-15% risk pool, the V1 target is **p₀ ≈ 0.75–0.85**. At an absolute MDE of +4 pp, α = 0.025 and power 0.8, that needs **≈ 1,340–2,100 per arm**. The planned 12 weeks × 200 = 2,400 per arm covers the whole range. LLM vs rules at that size has an MDE of ≈ 2.9–3.5 pp.
- **Supply check.** The dry run also counts **unique** eligible, never-enrolled accounts across the whole enrollment window. It must be at least 1.5 × the total enrollment needed.
- If either check falls short, extend to at most 20 weeks, or increase the population size. If it's still short, the gate fails and the decision is recorded in an ADR.
- The report gives the *achieved* power for the realised sample.

**Metrics.**

- *Primary*: 60-day paid retention.
- *Secondary*: net revenue retained (revenue minus discount cost).
- *Guardrails*: unsubscribe rate, complaint rate, discount given to accounts that would have retained anyway (from ground truth, in evaluation only), LLM cost per recommendation, review edit and reject rate.

## 7. Synthetic data-generating process

**Design principle.** Outcomes are driven by latent state that product code never sees. Observed features are noisy proxies for it.

**Population.** Default size is **300k sign-ups**. About 12% convert to paid, giving an active paid base of roughly 20k. A `small` profile with 30k sign-ups is used for development and CI.

**Latent state.**

- θᵢ: engagement propensity
- vᵢ(t): value realisation, an AR(1) process lifted by activation milestones
- πᵢ: price sensitivity
- fᵢ(t): frustration, raised by bugs, failed exports and over-contact

**Churn mechanism.** A discrete-time hazard at renewal:

> logit h = β₀ + β₁v + β₂θ + β₃f + β₄π·Δprice − β₅log(seats) + g(tenure) + persona effects + interactions

Payment failure is a separate process.

**Treatment effects come from the mechanism.** No effect sizes are injected into outcomes. Actions change latent state, and churn follows from that:

| Action | Mechanism |
|---|---|
| A1 education | Raises the probability of reaching an activation milestone for low-activation accounts, which lifts v |
| A2 nudge | Points to one unused feature. For 14 days, sessions carry an extra probability of reaching that activation milestone (the same channel as A1, but weaker), which lifts v. The effect is zero for fully activated accounts. A unit test checks the effect on v is > 0 when a milestone is unreached and = 0 when activated, and that the A2 ground-truth effect falls within its configured range. |
| A3 discount | Lowers effective price, so it only matters through the π·Δprice term (price-sensitive accounts) |
| A4 right-sizing | Lowers the price paid (through the π term) and resets the renewal date, but lowers revenue; the plan stays paid |
| A5 CSM | Raises v and the probability of adding seats for Teams accounts |
| Any contact | Raises f slightly, which models over-contact fatigue |

True effects are computed in **evaluation only**, by simulating every policy with common random numbers. They are stored in `_ground_truth/`.

**Generator layers.** Each layer has to pass its exit checks before the next one starts.

| Layer | Adds | Exit check |
|---|---|---|
| V1 | Accounts, subscriptions, usage, churn hazard, ground truth | Marginals hit targets: 3–6% monthly paid churn, DAU/MAU 20–40%, heavy-tailed sessions, and 60-day retention of 0.75–0.85 in the top-15% risk pool |
| V2 | Latent engagement, noisy correlated proxies, missingness | Model beats baselines but not the oracle that sees latent state |
| V3 | Seasonality, drift events, late and duplicate events, outages, leakage traps (`cancellation_flow_started`, `scheduled_end_date`) | Leakage tests catch the traps; drift is visible in the backtest |
| V4 | Mechanism-based treatment effects, reviewer and delivery non-compliance, experiment simulation | Repeated-seed study (§8) passes |

Everything is driven by a YAML config with fixed seeds. Data is written as Parquet partitioned by month: `accounts`, `subscriptions`, `daily_usage`, `support_tickets`, `interventions`, `assignments`, `review_log`.

## 8. Evaluation layers

1. **Generator validity**: marginal targets per layer, plus sensitivity runs across signal strength, noise, missingness and drift.
2. **Predictive**: rolling-origin temporal backtest. Report PR-AUC, recall and precision at the candidate threshold, and Brier score with a reliability curve. Compare against two baselines (days since last active, and logistic regression) and an oracle ceiling.
3. **Recommendation quality (offline, LLM vs rules)**: action validity, grounding of cited numbers, policy compliance, simulated-reviewer approval rate, and a message-quality LLM judge. About 100 human-labelled examples serve as an **initial calibration set**; agreement (Cohen's κ, with a bootstrap CI) is reported, and the judge is not presented as definitive proof of quality.
4. **Causal and decision**: ITT estimates compared with the ground-truth effects. A **repeated-seed study** re-randomizes and re-simulates outcomes from a fixed pre-experiment state, with R = 200 on the small profile. It reports bias, RMSE, 95% CI coverage, power (rejection rate) and the correct-decision rate.
5. **System**: latency, tokens, cost per call, failure and retry rates.

## 9. Ground-truth isolation (enforced by tests)

- **Separate modules.** `sim/` generates the data. `rc/` (features, model, policies, analysis) is the product code. `evaluation/` is the **only** module allowed to read `_ground_truth/`. An import-linter contract in CI blocks `rc` → `sim.ground_truth` imports.
- **Forbidden columns.** Feature tables are checked against a schema allowlist. CI fails on any ground-truth column or leakage-trap column.
- **Point in time.** Every feature row is checked so that the events it depends on satisfy `event_time ≤ t` and `ingested_time ≤ t`.
- **Invariance test.** Perturb or shuffle the `_ground_truth/` files while the observed data stays fixed. Feature tables, model artefacts and predictions must be hash-identical.

## 10. Phase plan

| Phase | Deliverable | Needs |
|---|---|---|
| 0 | Public GitHub repo with this spec, ADRs, package skeleton, CI (lint, type check, tests) | GitHub only |
| 1 | Generator V1–V2, validation and isolation tests | Local |
| 2 | Features and churn model (baselines, backtest, calibration, model card), then generator V3 | Local |
| 3 | Rules policy, LLM policy (one structured call), offline eval layer 3 | AWS account, with budgets and Bedrock in ap-southeast-2 (Australia cross-region inference) |
| 4 | Generator V4, power gate, experiment simulation, repeated-seed study, readout | Local and Bedrock |
| 5 | Thin AWS slice: Terraform, GitHub Actions with OIDC and eval gate, recommendation endpoint (Lambda and API Gateway), S3 artefacts, CloudWatch dashboard and alarms | AWS |
| 6 | Portfolio polish: README, architecture diagram, results, ADRs, demo video | — |

A tool-calling agent is considered only after Phase 4, and only if it beats the single-call policy on evaluation layer 3.

## 11. Non-goals and limitations

**Non-goals for v1.** Real customer data, real message delivery, real-time scoring, and uplift or heterogeneous-treatment-effect targeting models (a possible extension after Phase 4).

**Limitation, stated in the README.** The data and the reviewers are simulated. The project demonstrates engineering, evaluation and experimental discipline, and it shows whether the analysis recovers known simulated effects. It **cannot** show that an LLM policy would improve retention for a real SaaS business.
