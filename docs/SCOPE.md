# NOVA Scope

## Purpose

This document separates the smallest credible end-to-end NOVA product from optional extensions. The MVP must demonstrate the full decision path—from reproducible source data through trusted metrics and models to an operator-facing application—without depending on live external systems or advanced optimization.

The governing rule is:

> **Advanced work may extend a completed MVP, but it may not redefine, delay, or become a dependency of MVP completion.**

NOVA remains a portfolio project for the fictional Northstar Charge Network and uses synthetic data unless an advanced, clearly isolated public source is added. It is a decision-support product, not an autonomous charger-control, pricing, maintenance, procurement, or safety system.

## Tier 1: MVP

The MVP is one locally reproducible vertical slice of the weekly site-intervention decision. It must connect network performance, near-term demand, charger failure risk, and a bounded pricing scenario in a simple but polished interface. Breadth is intentionally limited; coherence, traceability, and execution quality matter more than the number of features.

### 1. Deterministic data generation

- Generate a coherent synthetic network including sites, chargers, charging attempts or sessions, charger status or telemetry, failures or work orders, and prices and energy costs sufficient for the required metrics and models.
- Use an explicit random seed and versioned generation parameters so the same inputs produce the same records and identifiers.
- Encode a small number of documented operating patterns, such as seasonality, peak demand, price response, degraded charger signals, failures, and missing or late data.
- Produce enough history for time-aware training and evaluation without claiming that synthetic performance represents a real charging network.
- Provide a single documented command to regenerate the data from a clean checkout.

### 2. PostgreSQL

- Use PostgreSQL as the analytical system of record for generated source data and transformed outputs.
- Define repeatable schema creation and loading with stable primary keys, relationships, timestamps, and station time-zone fields.
- Keep raw or source-shaped data distinct from dbt-managed analytical relations.
- Support a clean local setup with documented configuration and no committed secrets.

### 3. dbt staging, intermediate, and marts

- Implement all three layers with clear responsibilities:
  - **Staging:** source renaming, typing, deduplication, and minimal normalization.
  - **Intermediate:** reusable business logic, event reconciliation, hourly capacity, feature preparation, and other cross-source joins.
  - **Marts:** decision-ready facts, dimensions, network KPIs, forecast outputs, failure-risk outputs, pricing-scenario inputs or results, and data-quality summaries used by the API.
- Follow the definitions in `docs/METRIC_CONTRACT.md`; downstream surfaces must not silently redefine a contracted metric.
- Declare model grain, primary keys, source lineage, and appropriate dbt tests.
- Build the project successfully from the documented command.

### 4. Data-quality report

- Produce a human-readable report from automated checks covering at least freshness, completeness, uniqueness, accepted values, and key relationships.
- Show passed, failed, and unknown or not-evaluated states separately.
- Identify affected sources, entities or time ranges, and relevant downstream decisions where practical.
- Include at least one deterministic synthetic defect that demonstrates a visible warning in the report and application.
- Never interpret missing telemetry or source data as evidence of healthy equipment or zero activity.

### 5. Network KPIs

- Expose a coherent minimum KPI set from dbt marts through the API and UI. At minimum include successful sessions, session completion rate, serviceable capacity or availability, station downtime, gross charging revenue, energy cost, and contribution proxy.
- Use the defined grains, exclusions, time handling, and provisional-data behavior from the metric contract.
- Display observation windows, calculation or refresh time, and data-fitness context.
- Provide network summary and site-level drill-down sufficient to identify a site-week requiring attention.

### 6. One demand forecast

- Train and score one reproducible near-term site-demand forecasting approach at a declared grain and horizon, such as site-hour demand over the next seven days.
- Use temporal train, validation, and test splits; prevent future information from entering features.
- Store immutable forecast issue time, target time, model version, feature cutoff, point forecast, and an uncertainty interval or range.
- Report WAPE, MAE, and signed bias on eligible, completed out-of-sample targets, with coverage and exclusions.
- Include a documented baseline so the model's value and failure modes are interpretable.

### 7. One failure-risk model

- Train and score one reproducible charger failure-risk model for a declared horizon and eligible charger population.
- Use temporally out-of-sample evaluation and prevent label leakage.
- Store prediction time, horizon, model version, threshold version, probability or calibrated band, and outcome maturity status.
- Report precision, recall, probability calibration, and coverage at the selected operating threshold; show cohort behavior where the synthetic sample supports it.
- Present risk as decision evidence only. A score must not dispatch maintenance or create a work order.

### 8. One constrained pricing scenario

- Provide one bounded site-and-time pricing scenario comparing the current-price baseline with a user-selected increase, decrease, or time shift.
- Constrain the allowed price range and eligible hours, and account for forecast demand, serviceable capacity, queue-risk proxy, revenue, energy cost, and contribution proxy.
- Show assumptions, scenario range or sensitivity, and warnings for extrapolation or weak price-response evidence.
- Label the result as a scenario, not a causal estimate or autonomous recommendation.
- Do not publish or change a customer price.

### 9. FastAPI

- Provide a documented FastAPI service over the curated PostgreSQL/dbt outputs.
- Expose the minimum endpoints needed by the UI for network KPIs, site detail, demand forecasts, charger failure risk, pricing scenarios, and data-quality status.
- Validate request and response schemas, return useful error states, and avoid leaking database-specific structures into the client contract.
- Include health or readiness behavior and automated tests for the critical read and scenario paths.
- Keep model training and dbt transformation concerns outside request-time API logic.

### 10. Simple but polished UI

- Deliver a responsive, coherent operator interface rather than a collection of raw charts.
- Include a network overview or Command Center, a site detail view, demand forecast, failure-risk evidence, constrained pricing scenario controls and comparison, and visible data-quality status.
- Preserve shared site and time context across views.
- Distinguish observed facts, predictions, scenarios, and warnings through labels and visual treatment.
- Show loading, empty, error, stale, and unknown states intentionally.
- Use accessible hierarchy, legible charts and tables, consistent formatting, and a small documented design system.
- Never imply that a recommendation was executed or that a model output is an approved operating action.

## Tier 2: Advanced

Advanced work begins only after the MVP completion checklist passes. Each addition must be isolated behind a module, configuration flag, separate pipeline, or additive schema so the deterministic MVP continues to run without it. An unavailable public endpoint, orchestration service, optimizer, or advanced model must not break the core data build, API, or UI.

### Public-source ingestion

Add one or more reproducible public datasets that materially improve context, such as weather, holidays, electricity-market context, or geographic attributes. Cache or snapshot inputs where licensing permits, record provenance and retrieval time, validate contracts, and retain the synthetic fallback. No real customer, employee, or confidential operator data belongs in the project.

### Supplier and inventory modeling

Extend the synthetic domain with parts compatibility, inventory states, reservations, shipments, purchase orders, supplier lead-time distributions, and maintenance demand. Add stockout-risk and intervention-coverage views that support transfer, reorder, expedite, or supplier-escalation analysis. All purchasing, substitution, inventory movement, and supplier actions remain human-authorized and external to NOVA.

### Drift monitoring

Add scheduled feature, prediction, calibration, and data-distribution monitoring with explicit reference windows, minimum sample sizes, missingness bins, alert thresholds, and review ownership. Drift is a diagnostic signal, not proof of degraded model performance and not permission to retrain or promote a model automatically.

### Richer scenario optimization

Compare combinations of pricing, maintenance timing, capacity constraints, or inventory actions under explicit budgets and guardrails. Report a feasible set, objective tradeoffs, sensitivity, and reasons an option is infeasible. Optimization remains prescriptive decision support; it does not execute the selected plan.

### Automated orchestration

Schedule and observe data generation or ingestion, loading, dbt builds and tests, model scoring, report publication, and service refreshes. Include retries, idempotency, lineage, run metadata, failure notification, and safe recovery. A manual, documented local path must remain available for the MVP demonstration.

## Tier boundary and dependency rules

- MVP components may use simple, transparent implementations if they satisfy their contracts and evaluation requirements.
- No MVP acceptance criterion may require a public API key, internet access, supplier or inventory tables, drift infrastructure, an optimizer, or an orchestration platform.
- Advanced schemas and endpoints are additive; core API response contracts remain usable when advanced features are absent.
- Advanced UI panels must handle unavailable features without hiding or disabling the MVP workflow.
- Advanced pipeline failures cannot invalidate a successful core build unless they expose corruption in shared MVP inputs.
- Refactoring for an advanced item must keep the deterministic seed, core fixtures, tests, and documented MVP run path working.
- Advanced performance does not compensate for a failing MVP gate. A richer model, live feed, or scheduler is not a substitute for correct metrics, visible uncertainty, and a working end-to-end decision story.

## MVP complete checklist

MVP is complete only when every item below is true:

- [ ] A clean checkout can follow the documented setup without relying on undeclared local state or secrets.
- [ ] One documented command deterministically generates the complete synthetic input dataset from a fixed seed.
- [ ] PostgreSQL schemas can be created and loaded repeatably, with stable keys and valid source relationships.
- [ ] The dbt staging, intermediate, and mart layers build successfully from the generated data.
- [ ] dbt model grains, keys, tests, and lineage are documented, and contracted metric definitions match `docs/METRIC_CONTRACT.md`.
- [ ] The automated data-quality report covers freshness, completeness, uniqueness, accepted values, and relationships, and visibly reports a known synthetic defect.
- [ ] The required network KPIs are queryable at network and site level with time window, refresh time, and fitness context.
- [ ] The demand forecast is reproducible, temporally evaluated against a baseline, stores uncertainty, and reports WAPE, MAE, bias, and coverage.
- [ ] The failure-risk model is reproducible, temporally evaluated without leakage, and reports precision, recall, calibration, and coverage.
- [ ] The constrained pricing scenario compares baseline and scenario outcomes, enforces bounds, exposes assumptions and uncertainty, and cannot publish a price.
- [ ] FastAPI serves validated KPI, site, forecast, risk, scenario, and data-quality contracts and passes critical endpoint tests.
- [ ] The UI provides a polished end-to-end site-week workflow with deliberate loading, empty, error, stale, and unknown states.
- [ ] The UI clearly distinguishes observations, predictions, scenarios, and data-quality warnings and preserves operator approval boundaries.
- [ ] Automated tests cover the critical deterministic-data, transformation, model, API, and UI paths at a level appropriate to a portfolio demonstration.
- [ ] A scripted or documented demo can start from generated data, identify a site at risk, inspect demand and charger risk, compare a pricing scenario, and see the relevant data-quality warning.
- [ ] The core system can run and demonstrate successfully with every advanced feature disabled or absent.

Passing this checklist freezes the MVP boundary. Later advanced work may improve implementation quality, but discovery of an attractive extension does not reopen MVP scope unless it reveals a correctness defect in an existing requirement.

## Portfolio complete checklist

Portfolio complete means the MVP is credible, reviewable, and easy for another person to understand and run. Advanced features are optional evidence of depth; they are not required for this gate.

- [ ] Every item in the MVP complete checklist passes from a clean checkout.
- [ ] The README explains the problem, architecture, setup, test commands, demo path, synthetic-data disclaimer, and product limitations.
- [ ] Architecture documentation shows the path from generated source data through PostgreSQL, dbt, models, FastAPI, and UI, including ownership boundaries.
- [ ] Product documentation links the metric contract, decision loop, and this scope so reviewers can trace UI outputs to definitions and operator decisions.
- [ ] The repository contains no secrets, real company or personal data, generated bulk artifacts that should be reproducible, or unsupported production-readiness claims.
- [ ] A concise demo dataset and narrative reliably exhibit both a meaningful operating decision and a relevant uncertainty or data-health warning.
- [ ] Important model results include baselines, temporal evaluation, limitations, and reproducibility metadata rather than only headline accuracy.
- [ ] Screens and API outputs use consistent terminology and do not present modeled estimates as observed facts or autonomous decisions.
- [ ] Installation, build, test, and demo instructions have been verified in the intended local environment.
- [ ] Known limitations and intentionally deferred work are documented without disguising advanced backlog items as defects in the MVP.
- [ ] Any implemented advanced feature is labeled, documented, tested in proportion to its risk, and removable or disableable without breaking the MVP workflow.
- [ ] A final walkthrough demonstrates the observation-to-feedback decision chain and explains what NOVA must never decide automatically.

The portfolio may be declared complete with no advanced items implemented. If advanced work is included, it should be chosen for the clearest incremental evidence of engineering or analytical depth, not to satisfy an implied requirement that every possible extension be built.

## Explicitly deferred from MVP

The following do not block MVP or portfolio completion unless separately adopted as an advanced deliverable:

- Live or public-source ingestion
- Supplier, purchasing, spare-parts, and inventory optimization
- Automated drift alerts or model retraining
- Multi-action or mathematical scenario optimization
- Workflow schedulers, cloud deployment, and production observability
- Real-time charger control, work-order dispatch, price publication, procurement execution, or supplier communication
- Production security, scalability, availability, or regulatory certification claims

These items remain candidates for Tier 2, not hidden acceptance criteria for Tier 1.
