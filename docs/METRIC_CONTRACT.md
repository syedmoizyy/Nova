# NOVA Metric Contract

## Purpose and status

This document defines the semantic contract for NOVA's core network metrics before any transformation or calculation is implemented. It establishes what future ingestion, dbt models, APIs, models, and product surfaces must mean when they use these names.

NOVA represents the fictional Northstar Charge Network and will use synthetic data. A metric labeled **Synthetic-business proxy** is a deliberately simplified portfolio definition, not an industry standard, an accounting measure, or a claim about a real charging operator.

## Shared conventions

- **Canonical time:** Source timestamps are retained in UTC. Operational calendar buckets use the station's IANA time zone, including daylight-saving transitions. Both UTC and local bucket keys must remain available.
- **Event identity:** Source event IDs are preferred. Where absent, ingestion assigns a stable deterministic key from the source system, source record identifier, and event time. Retries must not double-count an event.
- **Metric finality:** Results are provisional until the metric-specific late-arrival window closes. A later valid correction may still restate history and must advance the model's `calculated_at` timestamp.
- **Dimensions:** Network metrics must be sliceable only by dimensions valid at their stated grain. Slowly changing asset, station, price, and supplier attributes are joined as-of the event time.
- **Nulls and unknowns:** Unknown is not zero. Records that cannot enter a numerator or denominator are surfaced in data-quality coverage measures rather than silently coerced.
- **Money:** Monetary metrics use synthetic USD. Source currency, if ever varied, is retained and converted using a documented rate before aggregation.
- **Energy:** Energy is measured in kilowatt-hours (kWh). Power is measured in kilowatts (kW). Negative or physically invalid readings are quarantined rather than clipped without evidence.
- **Duration:** Duration metrics use elapsed clock time, not the count of status samples.
- **Ownership:** Each metric has one eventual dbt mart owner. Upstream staging models may normalize inputs but must not redefine business semantics.
- **Versioning:** Material definition changes require a versioned migration, impact note, and explicit backfill decision.

## Charging and commercial metrics

### Successful charging session

**Classification:** Core metric; **Synthetic-business proxy**.

**Definition:** A distinct charging session that starts energy delivery, delivers at least 1.0 kWh, has no terminal unrecovered fault, and ends with a successful or normal-stop outcome.

**Grain:** One session (`session_id`); aggregations may use station, charger, connector, and session-start time.

**Numerator / denominator:** Numerator: one qualifying distinct session. Denominator: not applicable; this is a Boolean session classification and additive count.

**Time window:** Assigned to the operational day, week, month, or hour containing `session_started_at` in station-local time. Session facts may remain open for up to 24 hours after start.

**Exclusions:** Test and commissioning sessions; employee or technician diagnostics when identified; duplicate source records; sessions below 1.0 kWh; sessions without a trusted station/charger mapping; records quarantined for impossible timestamps or energy values.

**Late-arriving-data behavior:** Provisional until 48 hours after session start. Late terminal status, meter corrections, or deduplication may reclassify and restate the session. Valid later corrections remain eligible for backfill and are audit logged.

**Eventual dbt owner:** `mart_charging_sessions`.

### Session completion rate

**Classification:** Core metric; **Synthetic-business proxy**.

**Definition:** The share of eligible charging attempts that become successful charging sessions.

**Grain:** Station-period by default; drillable to charger-period. The atomic eligibility decision is one charging attempt.

**Numerator / denominator:** Numerator: distinct successful charging sessions. Denominator: distinct eligible charging attempts, including attempts that fail before meaningful energy delivery when a valid attempt event exists.

**Time window:** Hour, operational day, week, or month by attempt start in station-local time; default product view is trailing 7 completed operational days.

**Exclusions:** Test/diagnostic attempts; duplicates; customer-cancelled attempts before authorization when identifiable; abandoned records with no reliable attempt timestamp; stations outside their commissioned service interval.

**Late-arriving-data behavior:** Provisional for 48 hours after attempt start. Late session outcomes can move an attempt between incomplete and successful; newly arrived attempts can change the denominator. Backfills restate affected periods.

**Eventual dbt owner:** `mart_station_performance_daily`.

### Energy delivered

**Classification:** Core metric.

**Definition:** Net metered electrical energy delivered to vehicles during valid sessions.

**Grain:** Session; additive to charger-, station-, and network-period.

**Numerator / denominator:** Numerator: sum of valid session-level delivered kWh, preferably final meter minus initial meter. Denominator: not applicable.

**Time window:** At session grain, the full session energy is assigned to session start. Hourly power-analysis models may allocate interval-metered energy separately and must use a distinct metric name.

**Exclusions:** Negative deltas; meter rollovers or resets not reliably repaired; test/diagnostic sessions; duplicates; records with inconsistent units; estimated values unless explicitly flagged and included in a separately qualified series.

**Late-arriving-data behavior:** Provisional for 48 hours after session start. Final meter values supersede interim readings. Corrections restate affected aggregates while retaining source and prior-value audit fields.

**Eventual dbt owner:** `mart_charging_sessions`.

### Utilization

**Classification:** Core metric; **Synthetic-business proxy**.

**Definition:** The share of serviceable connector-minutes occupied by an active energy-delivering session.

**Grain:** Station-hour by default; optionally charger-hour where connector state is reliable.

**Numerator / denominator:** Numerator: occupied connector-minutes during the bucket, capped at available overlap per connector. Denominator: scheduled serviceable connector-minutes: commissioned connector-minutes minus verified maintenance closures and verified station downtime. Unobserved telemetry does not automatically reduce the denominator.

**Time window:** Station-local hour, operational day, week, or month. Sessions crossing buckets are split by elapsed overlap.

**Exclusions:** Connectors outside commissioning/decommissioning dates; test sessions; overlapping duplicate sessions; planned closures; buckets with insufficient asset-history mapping. Downtime exclusions must be visible alongside utilization.

**Late-arriving-data behavior:** Provisional for 72 hours after bucket end because session ends, status intervals, and maintenance closures may arrive separately. Late interval corrections restate numerator or denominator.

**Eventual dbt owner:** `mart_station_capacity_hourly`.

### Wait-time proxy

**Classification:** **Synthetic-business proxy**.

**Definition:** An estimated congestion signal used when actual arrival and queue timestamps are unavailable. At station-hour grain it is the non-negative excess of inferred charging demand over successful session starts, converted to estimated waiting minutes using the station's observed median service time and serviceable connector count. It is a scenario indicator, not observed customer wait time.

**Grain:** Station-hour.

**Numerator / denominator:** Numerator: estimated unmet/queued attempts multiplied by median valid session duration in minutes. Denominator: serviceable connector count for the hour. If inferred attempts are unavailable, no value is produced.

**Time window:** Station-local hour; default summaries report demand-weighted mean and upper-percentile proxy over trailing 7 completed days or the forecast horizon.

**Exclusions:** Stations without eligible attempt events; hours without a reliable serviceable-connector denominator; planned full closures; test traffic; periods with material attempt-event or charger-status gaps.

**Late-arriving-data behavior:** Provisional for 72 hours after hour end. Late attempts, sessions, status intervals, or duration corrections trigger restatement. Forecast scenarios store the input-data cutoff and are not silently mutated.

**Eventual dbt owner:** `mart_station_capacity_hourly`.

### Gross charging revenue

**Classification:** Core metric; **Synthetic-business proxy** for management reporting only.

**Definition:** The synthetic customer charging amount attributable to completed sessions before electricity cost, maintenance, payment fees, taxes, refunds, credits, and other operating expenses.

**Grain:** Session; additive to station-period and network-period.

**Numerator / denominator:** Numerator: sum of the final synthetic session charge, or documented synthetic tariff components when a final charge is absent. Denominator: not applicable.

**Time window:** Assigned by session start in station-local time; reported by operational day, week, or month.

**Exclusions:** Taxes; tips; deposits and authorization holds; refunds and credits unless reported as a separate adjustment; test/diagnostic sessions; duplicates; sessions with unresolved currency or tariff mapping.

**Late-arriving-data behavior:** Provisional for 7 calendar days after session start to allow charge finalization. Later corrections and refunds are recorded as dated adjustments and may restate management views according to the selected accounting view.

**Eventual dbt owner:** `mart_session_economics`.

### Energy cost

**Classification:** **Synthetic-business proxy**.

**Definition:** Estimated electricity commodity cost attributable to delivered charging energy, using the synthetic site tariff effective during delivery. It excludes demand charges, fixed charges, taxes, renewable credits, hedging, and non-energy operating costs unless later defined as separate metrics.

**Grain:** Session for session economics; station-hour for tariff allocation.

**Numerator / denominator:** Numerator: sum of interval-delivered kWh multiplied by the applicable synthetic USD/kWh tariff. Denominator: not applicable. When interval energy is unavailable, session energy may use a documented weighted tariff estimate and must be flagged.

**Time window:** Energy is allocated to the tariff intervals in which it was delivered, then aggregated by station-local operational period.

**Exclusions:** Invalid energy readings; intervals without a resolvable tariff; non-charging site load; demand and fixed charges; test/diagnostic energy.

**Late-arriving-data behavior:** Provisional until both energy and tariff inputs are final, normally 7 calendar days after delivery. Late tariff versions are applied by effective time and restate affected estimates with lineage.

**Eventual dbt owner:** `mart_session_economics`.

### Contribution proxy

**Classification:** **Synthetic-business proxy**; not GAAP profit or contribution margin.

**Definition:** Gross charging revenue minus energy cost and explicitly modeled synthetic variable transaction costs. The initial contract includes only gross charging revenue minus energy cost; additional deductions require named component metrics and a versioned definition.

**Grain:** Session; additive to station-period and network-period.

**Numerator / denominator:** Numerator: gross charging revenue minus energy cost. Denominator: not applicable. A percentage version, if introduced, must be separately named and use gross charging revenue as denominator.

**Time window:** Follows the session-economics reporting period, with tariff energy allocated to delivery intervals and the result attributed to the session-start period.

**Exclusions:** Sessions lacking valid revenue or energy-cost inputs; taxes; fixed site costs; depreciation; labor; maintenance; rent; network overhead; payment fees; and other expenses not explicitly modeled.

**Late-arriving-data behavior:** Inherits the latest finality of its components and remains provisional for at least 7 calendar days. Any component restatement recomputes the proxy and advances its calculation timestamp.

**Eventual dbt owner:** `mart_session_economics`.

## Reliability metrics

### Station downtime

**Classification:** Core metric; **Synthetic-business proxy**.

**Definition:** Elapsed minutes in which a commissioned station cannot deliver charging because zero connectors are serviceable during scheduled service hours. Partial capacity loss is reported separately and does not count as full station downtime.

**Grain:** Station-status interval; additive as non-overlapping minutes to station-period.

**Numerator / denominator:** Numerator: unioned elapsed minutes meeting the full-downtime condition. Denominator: scheduled service minutes for a downtime-rate derivative; the base downtime metric has no denominator.

**Time window:** Split across station-local hourly and operational-day boundaries. Default reporting is trailing 7 completed days and calendar month to date.

**Exclusions:** Planned site closures; pre-commissioning and post-decommissioning periods; upstream utility outages when reliably classified and reported separately; intervals with missing telemetry but no corroborating failed attempts or work-order evidence.

**Late-arriving-data behavior:** Provisional for 7 calendar days after interval end to allow work-order and outage classification. Missing intervals remain unknown until corroborated; late evidence may create, extend, shorten, or reclassify downtime.

**Eventual dbt owner:** `mart_station_reliability_daily`.

### Failure event

**Classification:** Core metric; **Synthetic-business proxy**.

**Definition:** A deduplicated unplanned incident in which a commissioned charger loses its ability to initiate or sustain charging and requires recovery action or remains unavailable beyond 15 consecutive minutes. Related alerts within the same unresolved episode form one failure event.

**Grain:** Charger failure episode (`failure_event_id`).

**Numerator / denominator:** Numerator: one qualifying deduplicated failure episode. Denominator: not applicable; failure rate derivatives must separately define exposure hours or charging attempts.

**Time window:** Event starts at the earliest corroborated failure time and ends at restored-and-verified time. Counts are assigned to event start; duration is split only in duration marts.

**Exclusions:** Planned maintenance; remote tests; commissioning; customer or vehicle faults when identifiable; isolated transient alerts resolved within 15 minutes without charging impact; duplicate or cascading alerts belonging to an existing episode.

**Late-arriving-data behavior:** Provisional for 14 calendar days after apparent restoration. Late work orders, telemetry, or attempt outcomes may merge, split, relabel, or close an event. Training snapshots must retain the label version and as-of cutoff.

**Eventual dbt owner:** `mart_charger_failure_events`.

### Mean time to repair

**Classification:** Core metric; **Synthetic-business proxy**.

**Definition:** Mean elapsed time from the start of a qualifying failure event to verified restoration of charger service.

**Grain:** Charger failure event for the component duration; aggregated by charger cohort, station, region, failure mode, or supplier-period.

**Numerator / denominator:** Numerator: sum of elapsed repair durations for eligible closed failure events. Denominator: count of those closed failure events.

**Time window:** Cohort by failure-event start during a stated day, week, month, or trailing window. Open events are excluded from the mean but their count and age must be shown beside it.

**Exclusions:** Open events; planned maintenance; events without trustworthy start or restoration timestamps; duplicate episodes; time explicitly classified as customer-requested deferral only if a separate gross-duration measure remains available.

**Late-arriving-data behavior:** Provisional for 14 calendar days after restoration. Late closure or timestamp corrections restate the event cohort. Open events entering later do not rewrite prior closed-only values until they close, but open-event coverage remains visible.

**Eventual dbt owner:** `mart_reliability_kpis`.

## Supply metrics

### Part stockout

**Classification:** Core metric; **Synthetic-business proxy**.

**Definition:** A part-location-day condition in which available-to-promise quantity for an approved required part is zero or negative while confirmed or forecast maintenance demand within the replenishment horizon is greater than zero.

**Grain:** Part, stocking location, operational day.

**Numerator / denominator:** Numerator: one stockout part-location-day, or sum of such days for aggregate counts. Denominator: eligible part-location-days with demand for a stockout-rate derivative.

**Time window:** Daily inventory snapshot in the stocking location's time zone; default outlook spans the part-supplier's current expected replenishment lead time.

**Exclusions:** Obsolete parts; non-stocked items intentionally procured on demand; inventory on quality hold; unapproved substitutes; demand outside the defined replenishment horizon; records with unresolved unit-of-measure conversions.

**Late-arriving-data behavior:** Operational snapshots are as-known-at-the-time and retained. Late receipts, reservations, or corrections produce a restated historical view without overwriting the original snapshot used for a decision.

**Eventual dbt owner:** `mart_part_inventory_daily`.

### Supplier on-time delivery

**Classification:** Core metric; **Synthetic-business proxy**.

**Definition:** The share of eligible purchase-order lines whose complete accepted quantity arrives on or before the supplier-confirmed due date effective at the agreed freeze point.

**Grain:** Purchase-order line; aggregated by supplier, part family, destination, and due-date period.

**Numerator / denominator:** Numerator: eligible lines completely received and accepted by the frozen confirmed due date. Denominator: all eligible lines due in the period, including late and still-open lines after due date.

**Time window:** Calendar month or trailing 90 days by frozen confirmed due date. The initial freeze point is 7 calendar days before due date; changes after freeze do not reset performance.

**Exclusions:** Cancelled lines before freeze; buyer-requested deferments documented before freeze; test orders; returns unrelated to delivery quality; lines without a valid due date or receipt identity, which are reported as coverage failures.

**Late-arriving-data behavior:** Provisional until 7 calendar days after due date to allow receipt posting. A late-posted receipt uses physical receipt time when available. Corrections restate the measure while retaining the original due-date version and posting lag.

**Eventual dbt owner:** `mart_supplier_performance_monthly`.

## Forecast metric

### Forecast error

**Classification:** Model-performance metric.

**Definition:** Error between the frozen demand forecast and observed eligible charging demand for the same site-time target. NOVA reports weighted absolute percentage error (WAPE) as the portfolio summary and mean absolute error (MAE) in sessions as the scale-preserving companion. Signed bias must be shown to distinguish systematic over- and under-forecasting.

**Grain:** Forecast target: site-hour, forecast horizon, model version, and forecast issuance time.

**Numerator / denominator:** WAPE numerator: sum of absolute `actual_sessions - forecast_sessions`; denominator: sum of actual eligible sessions. MAE numerator: the same absolute-error sum; denominator: number of eligible forecast targets. Bias numerator: sum of `forecast_sessions - actual_sessions`; denominator: number of eligible targets.

**Time window:** Evaluated by forecast horizon over completed target weeks, with default rolling 4-week and 12-week views. Only forecasts issued before the documented decision cutoff are eligible.

**Exclusions:** Cancelled or planned-closure periods; targets with materially incomplete actuals; forecasts generated after the decision cutoff; backtests contaminated by future information; WAPE cohorts whose actual denominator is zero. Exclusions and coverage must be reported.

**Late-arriving-data behavior:** Actuals remain provisional for 7 calendar days after target time. Metrics are recomputed when valid actuals are restated, but forecast values, issue times, feature cutoff, and model version remain immutable.

**Eventual dbt owner:** `mart_demand_forecast_performance`.

## Failure-model risk metrics

These measures evaluate NOVA's charger failure prediction model. They measure statistical behavior, not the operational cost of a real deployment. Every evaluation must use temporally out-of-sample predictions frozen before the outcome window.

### Failure-model precision

**Classification:** Model-risk metric.

**Definition:** Among charger-horizon observations classified as high risk at the approved threshold, the share followed by a qualifying failure event within the prediction horizon.

**Grain:** Charger, prediction timestamp, prediction horizon, model version, and threshold version.

**Numerator / denominator:** Numerator: true-positive high-risk predictions. Denominator: all high-risk predictions (`true positives + false positives`).

**Time window:** Default prediction horizon is 7 days; evaluated over completed rolling 4-week and 12-week outcome cohorts.

**Exclusions:** Predictions made after outcome evidence; chargers outside the model's declared population; planned decommissions; observations lacking complete label follow-up; duplicate prediction snapshots within the defined decision cadence.

**Late-arriving-data behavior:** Outcome labels mature 14 calendar days after the prediction horizon ends. Evaluations before maturity are provisional; later failure-event relabeling restates metrics while preserving the original prediction.

**Eventual dbt owner:** `mart_failure_model_performance`.

### Failure-model recall

**Classification:** Model-risk metric.

**Definition:** Among charger-horizon observations with a qualifying failure event, the share that had been classified as high risk before the event.

**Grain:** Charger, prediction timestamp, prediction horizon, model version, and threshold version.

**Numerator / denominator:** Numerator: true-positive high-risk predictions. Denominator: all eligible positive outcomes (`true positives + false negatives`).

**Time window:** Default prediction horizon is 7 days; evaluated over completed rolling 4-week and 12-week outcome cohorts.

**Exclusions:** Same as failure-model precision, plus failures with no eligible pre-event scoring opportunity. Excluded positive outcomes must be counted as coverage gaps.

**Late-arriving-data behavior:** Outcome labels mature 14 calendar days after the horizon ends. Late labels can change true-positive and false-negative counts and trigger restatement.

**Eventual dbt owner:** `mart_failure_model_performance`.

### Failure-model probability calibration

**Classification:** Model-risk metric.

**Definition:** Agreement between predicted failure probability and observed failure frequency, summarized by Brier score and displayed in probability bands. Lower Brier score is better; the banded view must reveal systematic over- or under-confidence.

**Grain:** Charger-horizon prediction; summarized by model version, charger cohort, probability band, and evaluation period.

**Numerator / denominator:** Brier numerator: sum of squared differences between predicted probability and binary outcome. Denominator: count of mature eligible predictions. For each calibration band, observed failures are the numerator and predictions in the band are the denominator.

**Time window:** Default 7-day prediction horizon, evaluated over rolling 12 completed weeks; shorter views are allowed only with sample-size disclosure.

**Exclusions:** Unmatured outcomes; predictions outside the declared model population; planned decommissions; records with invalid probability values; label-incomplete cohorts.

**Late-arriving-data behavior:** Provisional until labels mature 14 calendar days after horizon end. Valid label corrections restate the evaluation; probability, model version, and issuance timestamp never change.

**Eventual dbt owner:** `mart_failure_model_performance`.

### Failure-model cohort performance gap

**Classification:** Model-risk metric; **Synthetic-business proxy** for governance thresholds.

**Definition:** The largest absolute difference between an eligible charger cohort's recall and the overall recall at the same approved operating threshold. It highlights performance concentration across hardware model, firmware family, region, or asset-age band; it is not a legal fairness determination.

**Grain:** Model version, threshold version, evaluation period, cohort dimension, and cohort value.

**Numerator / denominator:** Numerator: absolute `cohort recall - overall recall` in percentage points. Denominator: not applicable to the gap itself; both recalls retain their true-positive and positive-outcome denominators and sample sizes.

**Time window:** Rolling 12 completed weeks for the default 7-day horizon.

**Exclusions:** Cohorts below the documented minimum mature-positive and observation counts; unknown cohort membership, which remains a separately reported coverage category; unmatured labels.

**Late-arriving-data behavior:** Recomputed after the same 14-day label-maturity period and whenever valid labels or as-of cohort mappings are corrected. Small-cohort suppression status may therefore change.

**Eventual dbt owner:** `mart_failure_model_risk`.

### Failure-model feature drift

**Classification:** Model-risk metric.

**Definition:** Population Stability Index (PSI) comparing the scored population's feature distribution with the model's frozen training reference for monitored numeric or categorical features. PSI is a monitoring signal, not proof of model degradation.

**Grain:** Model version, feature, scoring cohort, and evaluation week.

**Numerator / denominator:** Numerator / denominator: not a conventional ratio. PSI is the sum across frozen reference bins of `(current_share - reference_share) * ln(current_share / reference_share)`, with a documented small-value treatment for empty bins. Record counts and missingness rates accompany every value.

**Time window:** Weekly scoring population compared with the immutable training reference; trailing 4-week confirmation view for persistent drift.

**Exclusions:** Features not used by the model; invalid values quarantined upstream; cohorts below the minimum observation count. Missing values are not excluded and must occupy an explicit bin.

**Late-arriving-data behavior:** The weekly scoring cohort is provisional for 7 calendar days. Late feature data may restate the current distribution, but the training reference, bin boundaries, and model version remain immutable.

**Eventual dbt owner:** `mart_failure_model_risk`.

## Contract governance

Before implementation, each eventual owner mart must declare its primary key, required upstream sources, tests, and freshness expectation. dbt exposures should connect these marts to the relevant NOVA surface. No downstream API, chart, model feature, or scenario calculation may reuse one of these names with a different formula.

This contract intentionally defines semantics only. It contains no computed values, performance claims, thresholds inferred from real operators, or production deployment assertions.
