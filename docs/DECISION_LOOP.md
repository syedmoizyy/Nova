# NOVA Decision Loop

## Purpose

NOVA is a decision-support layer for Northstar Charge Network's weekly operating cycle. It connects observations about demand, capacity, charger health, parts, suppliers, and data fitness to a named operator action and then measures what happened. It does not turn a model score into an operating command.

The primary decision unit is a **site-week**, with charger-level, site-hour, part-location, and supplier-order detail where needed. Every committed decision must retain the observation cutoff, assumptions, model and metric versions, uncertainty, operator and approval time, expected consequence, guardrails, and realized feedback. Unknown is not treated as zero, and a missing signal is not treated as evidence that conditions are healthy.

## Decision types

- **Descriptive:** Establishes what is known now or what happened, including data coverage and operational constraints. It can prioritize attention but does not estimate a future outcome by itself.
- **Predictive:** Estimates what is likely to happen under a stated baseline, horizon, and model version. It must provide a range or calibrated risk, not only a point estimate.
- **Prescriptive:** Compares feasible actions and their expected consequences. The output is advice for a named operator; it is not authorization to execute.
- **Experimental:** Changes a controllable input within explicit guardrails to learn its causal effect. The design, stopping rules, exposure, and evaluation window must be approved before launch.

A surface can contain more than one type. Each output should be labeled at the point of use so an observed fact, forecast, recommendation, and experiment estimate cannot be mistaken for one another.

## Common loop contract

Every surface implements the same chain:

> **Observation -> analysis/model -> uncertainty -> operator action -> expected consequence -> feedback signal**

An observation must be time-stamped and tied to its source coverage. The analysis must name its baseline, horizon, and version. Uncertainty must change how the output is used: widen a range, lower confidence, constrain the available actions, require corroboration, or withhold the output. The operator action must have a named accountable owner. Expected consequences must include both benefit and foreseeable harm. Feedback must be compared with the frozen expectation used at decision time, not a forecast rewritten after actuals arrive.

## 1. Command Center: prioritize a site-week

**Decision types:** Descriptive, predictive, and prescriptive.

**Observation:** For every candidate site-week, assemble the demand forecast cutoff, successful sessions and attempts, serviceable connectors, utilization and queue-risk proxy, open maintenance events, charger-risk bands, contribution proxy, and parts feasibility. Show freshness, completeness, conflicting identifiers, planned closures, and already committed work. Preserve both the as-known-at-decision snapshot and any later-restated view.

**Analysis/model:** Estimate the baseline fulfilled sessions, lost sessions, contribution proxy, and operational risk if Northstar does nothing. Attribute the risk to demand growth, constrained usable capacity, probable charger failure, price timing, and supply limitations. Rank sites by recoverable sessions and contribution under feasible intervention bundles, not by theoretical upside alone. Avoid double-counting when pricing, maintenance, and parts actions address the same expected loss.

**Uncertainty:** Display forecast intervals, failure-risk calibration, scenario sensitivity, parts-confidence status, and data-fitness warnings beside the rank. Identify which driver dominates the range. A site with missing recent status, incomplete attempts, unknown parts feasibility, or out-of-population equipment cannot appear confidently healthy. Materially unfit inputs should suppress the numerical rank and move the site to a review queue.

**Operator action:** Maya selects which site-weeks receive scarce engineering attention or regional budget. Luis then accepts the risk, requests analysis, or commits a feasible intervention plan with an owner, deadline, dependencies, and escalation path. NOVA records, but does not execute, that choice.

**Expected consequence:** The committed plan states the expected recovered sessions and contribution range, avoided outage or queue exposure, direct intervention cost, capacity affected, and principal downside. It also states the do-nothing baseline and the assumptions that must remain true.

**Feedback signal:** During the week, monitor assumption breaches: forecast demand deviation, serviceable capacity, new failures, parts arrival, intervention completion, and data-health changes. After the site-week closes and actuals mature, compare fulfilled sessions, lost-session proxy, contribution proxy, downtime, and intervention cost with the frozen baseline and scenario. Record overrides and reasons; feed forecast error, action completion, and realized-effect error into model and operating reviews.

## 2. Pricing Studio: change or test a price

**Decision types:** Descriptive, predictive, prescriptive, and experimental. A scenario remains predictive; it becomes experimental only when an approved intervention is designed to learn an effect.

**Observation:** For the selected site and hours, show posted prices, successful sessions and attempts, energy delivered, revenue, energy cost, serviceable capacity, utilization or queue proxy, nearby time-period patterns, known outages, prior interventions, seasonality, and treatment coverage. Separate observed inputs from assumed cost or response parameters.

**Analysis/model:** Forecast baseline demand and economics, then simulate hold, increase, decrease, or time-shift scenarios. Estimate ranges for fulfilled sessions, peak demand, utilization, queue risk, gross revenue, energy cost, and contribution proxy. For a test, define eligible hours or sites, comparison strategy, exposure, primary metric, guardrails, minimum evaluation window, and stopping rules before publishing the change. Observational association must not be labeled causal.

**Uncertainty:** Show forecast and price-response intervals, sample size, treatment-history sparsity, comparison credibility, confounding outages, and extrapolation beyond observed price levels. Highlight sensitivity to elasticity, capacity, and cost assumptions. If a credible counterfactual is unavailable, label the result as a scenario range rather than an expected causal lift.

**Operator action:** Priya chooses to hold price, reject the proposal, request more evidence, or approve a bounded change or experiment with amount, hours, duration, customer-impact guardrails, rollback conditions, and an accountable publisher. Pricing publication occurs in the appropriate external system after human approval; NOVA only records the decision.

**Expected consequence:** State the expected change from baseline in sessions, peak load, queue risk, revenue, energy cost, and contribution proxy, plus the customers and periods exposed. Include the risk that demand is suppressed, shifted into another constrained period, or routed toward unreliable capacity.

**Feedback signal:** Monitor realized price, exposure integrity, attempts, sessions, utilization, queue proxy, charger availability, contribution proxy, and guardrail breaches during the intervention. At the predeclared evaluation date, estimate effect against the frozen comparison design and report confidence, spillovers, and exclusions. Update response evidence only after review; do not silently promote a single test result into a permanent rule.

## 3. Reliability: dispatch or defer maintenance

**Decision types:** Descriptive, predictive, and prescriptive.

**Observation:** For each charger, show recent errors, resets, status intervals, charging attempts and outcomes, firmware and hardware cohort, age, relevant telemetry, prior work, open events, site demand consequence, and telemetry coverage. Distinguish corroborated failure evidence, inactivity, planned work, and missing data.

**Analysis/model:** Estimate calibrated failure probability over a stated horizon and likely failure mode. Combine that probability with the consequence of losing the charger during forecast demand and compare monitor, remote reset, inspect, derate, repair, or temporary removal from service. Check technician timing, compatible parts, and site redundancy before calling an action feasible. Risk scores set review priority; they do not create work orders.

**Uncertainty:** Provide risk bands and calibration by relevant cohort, mature sample counts, false-positive and false-negative context, feature drift, unseen equipment or firmware warnings, label maturity, and telemetry gaps. Missing telemetry must widen uncertainty or trigger investigation, never reduce risk. If safety-relevant evidence conflicts, route to the established safety process rather than averaging it into a score.

**Operator action:** Jordan recommends a proportionate action and records the evidence and threshold version. Luis approves, changes, or defers the plan and schedules it through the maintenance system, with a reason for any override. Urgent safety or regulatory procedures take precedence over NOVA's ranking.

**Expected consequence:** Record the expected avoided failure or downtime range, protected sessions and contribution, truck-roll and parts cost, time out of service, and risk of unnecessary work or a missed failure. For deferral, state the accepted exposure and next review trigger.

**Feedback signal:** Track whether the work occurred, parts and labor used, findings, verified restoration, repeat visit, subsequent attempts, failure within the prediction horizon, downtime, and forecast demand affected. Once labels mature, classify the frozen prediction and action outcome, update precision, recall, calibration, and cohort gaps, and review whether intervention changed the observable outcome before using it as a training label.

## 4. Supply Desk: reposition inventory or escalate a supplier

**Decision types:** Descriptive, predictive, and prescriptive.

**Observation:** Show on-hand, reserved, in-transit, and quality-hold quantities by part and location; approved compatibility and substitutions; forecast maintenance demand; committed intervention demand; purchase-order due dates; lead-time history; fill rate; landed cost; minimum orders; shipment confirmations; and inventory freshness. Preserve unit of measure and the as-known snapshot used for allocation.

**Analysis/model:** Forecast part demand and lead-time distributions over the intervention horizon. Calculate available-to-promise inventory, stockout risk, intervention coverage, and supplier concentration. Compare doing nothing with transfer, reorder, expedite, approved substitute, intervention resequencing, and supplier escalation. A transfer recommendation must include the risk created at the sending location. A supplier escalation should be triggered by a specific threatened commitment or persistent performance pattern, not a context-free score.

**Uncertainty:** Show demand and lead-time ranges, unconfirmed shipments, stale counts, small supplier samples, compatibility gaps, unit mismatches, reservation conflicts, and the probability that inventory arrives before need. Unapproved substitutes and inventory of uncertain physical availability are infeasible, not low-confidence supply.

**Operator action:** Elena approves, rejects, or modifies a transfer, reorder, or expedite; requests confirmation; selects only an already approved substitute; or escalates a supplier through the responsible commercial owner. Luis may choose a different maintenance plan when coverage is inadequate. Purchase orders, warehouse movements, supplier communications, and contract remedies remain external, human-authorized actions.

**Expected consequence:** State protected interventions and site-weeks, expected reduction in stockout exposure and charger downtime, landed and expedite cost, working-capital increase, coverage lost elsewhere, and supplier concentration or schedule risk. For an escalation, state the requested recovery commitment and the operational consequence if it is missed.

**Feedback signal:** Track authorization, pick and ship events, physical receipt and acceptance, reservation fulfillment, actual part consumption, intervention completion, emergency freight, supplier response, frozen-due-date performance, and unused transferred stock. Compare realized demand and lead time with the frozen distributions; update forecasting and supplier evidence without rewriting the historical decision snapshot.

## 5. Data Trust: investigate and qualify a data-health failure

**Decision types:** Descriptive and prescriptive. Predictive methods may help detect anomalies or estimate blast radius, but anomaly scores are not proof that data is wrong.

**Observation:** Capture the failed contract or anomaly, first observed time, expected cadence, last successful arrival, affected rows and entities, lineage, source owner, downstream surfaces and model versions, incident history, waivers, and corroborating sources. Report unknown, failed, waived, and passed as distinct states.

**Analysis/model:** Determine whether the signal is a late delivery, schema or semantic break, referential failure, volume anomaly, stale dimension, model drift, or expected operational event. Trace its blast radius to specific sites, horizons, metrics, forecasts, and recommendations. Compare with independent evidence and assess whether the defect changes the direction, range, or feasibility of a pending decision.

**Uncertainty:** State detection limitations, observation coverage, anomaly-model false positives, unresolved root cause, and what cannot yet be bounded. A global green status cannot override a relevant local failure. When fitness cannot be established, surface `unknown` and identify the evidence needed to resolve it.

**Operator action:** Sam opens or links an investigation, assigns an owner, and marks each affected output as certified, qualified, or withheld for its specific use. Other operators may proceed with documented corroboration, defer the decision, or use an approved fallback. Waivers require an owner, rationale, scope, expiry, and visible residual risk.

**Expected consequence:** Record which bad decisions or false alarms the response is intended to prevent, which outputs become unavailable or less precise, the operational cost of delay, and the expected restoration or next-update time. Qualification should reduce the blast radius without implying that unaffected uses are unsafe.

**Feedback signal:** Monitor source recovery, backfill completion, contract-test results, lineage reconciliation, affected-model recomputation, and operator acknowledgement. After closure, measure detection delay, time to containment, time to trusted restoration, decisions delayed or changed, recurrence, and whether the original blast-radius estimate was accurate. Retain the incident and waiver history for later audits.

## Cross-surface learning and governance

The weekly review should join the five loops rather than score them independently. A price outcome is uninterpretable without actual capacity; a maintenance result depends on part arrival; supplier performance depends on frozen due dates; and all results depend on data fitness. The review therefore compares the complete committed intervention bundle with its do-nothing baseline and records material external events.

Feedback has three cadences:

- **In-flight monitoring:** Detect guardrail breaches, invalid assumptions, missed dependencies, and new data-health incidents while the operator can still adapt the plan.
- **Outcome review:** After metric finality and label maturity, compare realized results with the frozen expectation and document overrides, execution fidelity, confounders, and unintended consequences.
- **Model and policy review:** On an appropriate accumulated sample, evaluate forecast error and bias, failure-model calibration and coverage, scenario error, supplier lead-time error, recommendation acceptance, override outcomes, and cohort gaps. Threshold or policy changes require versioning; they are not tuned from a single anecdote.

The system must distinguish **model error** from **execution failure** and **changed conditions**. A sound forecast with a missed parts delivery is not the same failure as a bad demand model. Likewise, an operator override is evidence to review, not automatically an error or a training label.

## What the model must never decide automatically

NOVA may rank, forecast, simulate, recommend, warn, and monitor. It must never autonomously:

- Publish, change, or roll back a customer price, fee, discount, or commercial term.
- Dispatch a technician, create or close a work order, remotely reset or control equipment, derate a charger, or remove it from service.
- Override a safety procedure, engineering judgment, regulatory requirement, accessibility obligation, or emergency response.
- Approve a repair, declare equipment safe, verify physical restoration, or infer safety from missing telemetry.
- Transfer or reserve inventory, place or cancel a purchase order, approve an unverified substitute, authorize expedite spend, or change a supplier.
- Contact, threaten, penalize, or contractually escalate a supplier, invoke a remedy, or make a legal or commercial commitment.
- Certify suspect data, waive a failed data contract, close an incident, hide a warning, or treat unknown data as passed.
- Allocate budgets, approve spend, rank employee performance, or assign blame for a failure or override.
- Expand an experiment's population, duration, or price bounds; suppress a guardrail breach; or declare causal impact without the approved design and review.
- Retrain, promote, change thresholds, or replace a production-facing model solely from live outcomes without validation, versioning, and accountable approval.
- Convert a recommendation into action merely because a score crosses a threshold. Every consequential action requires a named human owner and the appropriate system of record.

When an urgent condition is detected, the model's automatic behavior is limited to making the warning prominent, propagating it to dependent surfaces, preserving evidence, and notifying the designated human workflow where that integration has been explicitly approved. Silence, default acceptance, and hidden fallback are not approvals.
