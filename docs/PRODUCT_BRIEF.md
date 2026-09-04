# NOVA Product Brief

## Product premise

NOVA—Network Operations & Value Analytics—is a portfolio project for a fictional nationwide electric-vehicle charging operator, Northstar Charge Network. Northstar owns and operates public fast-charging sites across the United States. It earns revenue when drivers charge, pays for electricity and site operations, and is accountable for keeping advertised chargers available.

Northstar's operating constraint is not a lack of dashboards. Its teams must make connected decisions from imperfect data: where demand will exceed usable capacity, whether price can shift that demand, which chargers are likely to fail, and whether the required parts and suppliers can support an intervention. Today, those decisions are made from separate exports and competing definitions. A pricing change can overload a fragile site; a maintenance plan can fail because a part arrives late; an apparently urgent alert can be an ingestion gap.

NOVA is the shared decision layer for that operating cycle. Each week, network leadership identifies sites at risk of lost charging sessions, specialists evaluate the commercial and operational levers, and regional operators commit a plan. During the week, they monitor whether the assumptions remain valid. NOVA connects those decisions to common forecasts, reliability signals, supply constraints, and visible evidence about data fitness.

NOVA uses only synthetic, fictional data. It does not represent any real company, customer, charging network, or production deployment.

## Business objective

Northstar's goal is to deliver dependable charging while protecting contribution margin. NOVA should help operators recover charging sessions that would otherwise be lost—because of queues, failures, poorly chosen prices, or unavailable parts—without presenting model estimates as facts.

The primary unit of decision is a **site-week**, supported by charger-level health and hourly demand detail. The shared outcome is **expected fulfilled sessions and contribution margin, adjusted for operational risk**.

## Operator personas and decisions

### Maya Chen — VP of Network Operations

Maya owns nationwide availability and operating performance. On Monday morning she decides which sites deserve scarce engineering attention and regional budget. She needs to distinguish a large theoretical opportunity from an intervention the organization can actually execute.

A bad decision sends teams to low-impact sites while high-demand locations lose sessions. Northstar absorbs lost charging margin, service credits, avoidable field costs, and damage to driver trust.

### Luis Ramirez — Regional Operations Manager

Luis commits the weekly intervention plan for several hundred chargers. He decides whether to inspect, remotely reset, derate, or temporarily remove a charger from service, and when to escalate a site. He needs failure risk in the context of expected demand and repair feasibility.

A bad decision can cause an avoidable outage or an unnecessary truck roll. It may also strand a technician without the correct part and leave peak-period capacity unavailable.

### Priya Shah — Pricing and Commercial Lead

Priya decides whether to hold, increase, decrease, or time-shift prices at eligible sites. Her mandate is to improve fulfilled demand and contribution margin without creating unacceptable queueing, customer harm, or load at unreliable chargers.

A bad decision can destroy margin, suppress valuable demand, move demand into a constrained hour, or incorrectly attribute normal seasonality to a price change.

### Jordan Okafor — Reliability Engineering Lead

Jordan decides which charger risks warrant action and which failure modes require systematic engineering work. Jordan also sets risk thresholds with regional operations and reviews whether the model is learning genuine precursors rather than data outages.

A bad decision wastes maintenance capacity, misses a preventable failure, or creates false confidence in a risk score that is poorly calibrated for a charger type or region.

### Elena Park — Parts and Supplier Planner

Elena decides what to stock, where to position it, and when to expedite or change suppliers. She must connect forecast maintenance demand with inventory, compatible substitutes, supplier lead times, and failure-mode uncertainty.

A bad decision ties up cash in idle inventory or leaves a high-value charger offline while a critical part is unavailable. Emergency freight and repeat visits further increase the cost.

### Sam Brooks — Data Product Steward

Sam owns the operational meaning and fitness of NOVA's data products. Sam decides whether a metric, model, or scenario is safe to use, needs qualification, or should be withheld while a source issue is investigated.

A bad decision allows stale, incomplete, or semantically inconsistent data to drive a costly operating action—or blocks a sound decision because a harmless anomaly is treated as critical.

## The connected operating story

On Monday, Maya opens Command Center and sees that the fictional Mesa Gateway site is projected to lose sessions during the Friday evening peak. NOVA separates the drivers: rising demand, two chargers with elevated failure risk, and limited replacement-module availability. Priya tests whether an off-peak price incentive shifts enough flexible demand to reduce the peak. Jordan checks the failure signals and recommends a targeted inspection rather than replacing both chargers. Elena confirms that one compatible module can reach the region before Friday and exposes the supplier lead-time risk. Luis compares the combined plan with doing nothing and commits the feasible intervention.

Throughout that path, Data Trust shows that one telemetry feed was incomplete overnight. NOVA widens the reliability range and prevents users from mistaking missing telemetry for healthy equipment. The final decision therefore includes its expected benefit, cost, feasibility, uncertainty, and evidence quality—not merely a recommendation.

## Product surfaces

### 1. Command Center

**User question:** Where is the network most likely to lose valuable charging sessions, why, and which sites require a decision now?

**Decision supported:** Maya prioritizes site-weeks for review; Luis selects a feasible intervention or accepts the risk. The surface must rank opportunities by expected recoverable sessions and margin while accounting for demand, usable capacity, charger risk, and parts feasibility.

**Minimum data required:**

- Site and charger inventory, geography, connector type, and rated capacity
- Charging sessions with timestamps, energy delivered, price, and session outcome
- Charger status or telemetry sufficient to estimate usable capacity
- Near-term site demand forecast
- Current open maintenance events
- Parts availability or a clear indication that supply feasibility is unknown

**Output:** A ranked site-week intervention queue; a concise explanation of each risk driver; baseline expected sessions, capacity, and margin; feasible actions; accountable owner; and links into Pricing Studio, Reliability, and Supply Desk with the same site and time context.

**Visible uncertainty and data-quality warning:** Forecast intervals and the contribution of model uncertainty must appear beside the ranking. Stale charger status, incomplete session ingestion, unknown parts feasibility, or materially conflicting site identifiers must lower confidence visibly. NOVA must not silently rank a site as healthy when recent observations are missing.

### 2. Pricing Studio

**User question:** If we change price at this site and time, how might demand, queueing, revenue, and contribution margin respond?

**Decision supported:** Priya chooses whether to hold or test a price change and selects its amount, eligible hours, duration, and guardrails. The surface supports a bounded commercial experiment or scenario; it does not autonomously publish prices.

**Minimum data required:**

- Historical site-hour sessions, posted prices, energy delivered, and realized revenue
- Electricity cost or an explicit modeled cost assumption
- Calendar, seasonality, and time-of-day features
- Site capacity and charger availability
- Recorded pricing interventions with start and end times
- Demand forecast for the scenario horizon

**Output:** Baseline-versus-scenario ranges for sessions, utilization, queue-risk proxy, revenue, energy cost, and contribution margin; estimated price response; affected customer periods; proposed test guardrails; and the assumptions needed to reproduce the scenario.

**Visible uncertainty and data-quality warning:** Response estimates must show confidence intervals and flag extrapolation beyond observed prices. The surface must warn when treatment history is sparse, comparison periods are not credible, cost inputs are assumed, or contemporaneous outages could confound the estimate. Association must not be presented as causal impact without an appropriate design.

### 3. Reliability

**User question:** Which chargers are at elevated risk of failing, what evidence drives that risk, and what action is proportionate?

**Decision supported:** Jordan sets review priorities and recommends inspect, monitor, reset, derate, or repair; Luis schedules the approved work in the context of site demand. Risk scores inform judgment and do not directly create work orders.

**Minimum data required:**

- Charger asset hierarchy, model, firmware, age, and commissioning history
- Timestamped status, error codes, resets, and relevant telemetry
- Work orders with failure labels, actions, parts used, and restoration times
- Charging attempts and outcomes to distinguish failure from inactivity
- Near-term site demand or consequence-of-failure estimate

**Output:** Charger risk over a defined horizon; calibrated risk band; likely failure mode; evidence and recent signal history; expected consequence at the site; recommended inspection or maintenance action; and post-event outcome tracking.

**Visible uncertainty and data-quality warning:** NOVA must show calibration and uncertainty by charger cohort where sample size permits. It must flag unseen equipment types, sparse labels, delayed work-order closure, telemetry gaps, firmware changes, and potential label leakage. Missing telemetry cannot be interpreted as absence of risk.

### 4. Supply Desk

**User question:** Can the planned reliability work be completed with available parts and suppliers, and where should inventory or expedites be placed?

**Decision supported:** Elena allocates inventory, places or expedites an order, approves a compatible substitute, or advises operations to choose a different intervention. Luis uses that feasibility result when committing the site plan.

**Minimum data required:**

- Parts catalog, charger compatibility, approved substitutions, and units of measure
- Inventory on hand, reserved, in transit, and location
- Work-order parts consumption and forecast maintenance demand
- Purchase orders, supplier lead times, fill rates, prices, and minimum order constraints
- Shipment and receipt events

**Output:** Projected part requirements by region and horizon; stockout-risk range; intervention coverage; recommended transfer, reorder, or expedite; expected landed cost; supplier exposure; and which site interventions remain unprotected.

**Visible uncertainty and data-quality warning:** Lead-time variability and maintenance-demand uncertainty must be explicit. The surface must warn about stale inventory counts, unconfirmed shipments, missing compatibility mappings, unit mismatches, supplier estimates based on few orders, and stock that is physically present but already reserved.

### 5. Data Trust

**User question:** Is the data behind this decision current, complete, internally consistent, and fit for this specific use?

**Decision supported:** Sam certifies a data product for use, publishes a qualification, or withholds affected outputs. Other personas decide whether to proceed, seek corroboration, or defer an action based on the severity and relevance of the issue.

**Minimum data required:**

- Source inventory, owners, ingestion timestamps, and expected delivery cadence
- Data contracts and business definitions
- Row counts, freshness, completeness, uniqueness, relationship, and accepted-value test results
- Lineage from source through analytics and model outputs
- Incident history, waivers, and affected entities or time ranges
- Model version, training window, evaluation summary, and scenario assumptions

**Output:** Decision-specific fitness status; freshness and completeness indicators; failed tests and blast radius; metric and model lineage; responsible owner; incident status; and a plain-language explanation of how the issue changes or limits the decision.

**Visible uncertainty and data-quality warning:** A single global green score is insufficient. NOVA must distinguish unknown from passed, show coverage and observation time, identify whether a warning affects the selected site and horizon, and propagate critical issues into every dependent surface. Waived failures must remain visible with owner and expiry.

## Product principles

1. **Start with the decision.** Every number must help select, reject, or monitor an operational action.
2. **Share one operating context.** Site, charger, time horizon, baseline, and scenario definitions must remain consistent across surfaces.
3. **Show feasible choices.** Commercial upside that cannot be supported by healthy capacity, parts, or time is not an actionable opportunity.
4. **Expose uncertainty at the point of use.** Ranges, missingness, extrapolation, and model limitations belong beside the output they qualify.
5. **Preserve human accountability.** NOVA provides evidence and scenarios; named operators approve consequential actions.
6. **Make outcomes auditable.** A committed decision must retain its inputs, assumptions, model versions, owner, and realized result.

## Non-goals

NOVA is not:

- A charger-control system, energy-management system, or real-time safety platform
- An autonomous pricing engine or a system that publishes customer prices
- A computerized maintenance management system, work-order dispatcher, ERP, procurement platform, or warehouse-management system
- A driver-facing route planner, mobile application, loyalty product, or payment processor
- A general-purpose business-intelligence catalog intended to reproduce every company metric
- A digital twin or electrical-grid optimization platform
- A promise of causal pricing conclusions from observational data alone
- A replacement for engineering inspection, supplier approval, or operator judgment
- A production deployment, production-ready security claim, or integration with any real charging network
- A repository for real company, employee, supplier, or customer data

The initial product boundary is the weekly site-intervention decision and its near-term monitoring. Features that do not improve the priority, feasibility, expected value, or trustworthiness of that decision should remain outside the first release.
