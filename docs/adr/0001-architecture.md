# ADR 0001: Local-first NOVA architecture

- **Status:** Accepted
- **Date:** 2026-09-04
- **Decision owners:** NOVA project maintainers
- **Scope:** MVP architecture and extension boundaries

## Context

NOVA is a portfolio decision-support product for the fictional Northstar Charge Network. Its MVP must demonstrate one trustworthy, reproducible path from synthetic operational data to network KPIs, a demand forecast, charger failure risk, a constrained pricing scenario, and a polished operator interface. The primary decision unit is a site-week, with site-hour and charger-level detail.

This architecture optimizes for a single developer or reviewer running the complete project locally. It must make semantic ownership, model reproducibility, uncertainty, and data-quality failures visible. It does not need production-scale distributed systems, real-time charger control, or integration with real commercial systems.

The expected MVP scale is modest: synthetic records for tens to low hundreds of sites, hundreds to low thousands of chargers, and months to a few years of hourly or event-level history. This fits comfortably in one PostgreSQL instance and in-memory model training on a development machine. The workflow is batch-oriented; a successful local run may take minutes rather than seconds.

## Decision

Use a local-first, batch architecture built from:

- Python for deterministic data generation, loading helpers, model training and scoring, report assembly, and narrow command-line entry points.
- PostgreSQL as the durable local system of record for source-shaped data, dbt relations, predictions, scenario inputs or results, and run metadata.
- `dbt-postgres` for SQL transformation from staging through intermediate models to decision-ready marts, including semantic and data-quality tests.
- scikit-learn and, where statistical forecasting or inference is clearer, statsmodels for the demand forecast and failure-risk model.
- FastAPI as the typed application boundary over curated database outputs and constrained scenario logic.
- Next.js for the operator-facing web interface.
- Docker Compose for reproducible local service dependencies and, optionally, containerized application services.
- A Makefile as the documented task interface for setup, generation, loading, transformation, training, testing, startup, and the full demo pipeline.

The MVP will not include a workflow orchestrator. The Makefile and small Python commands will express the dependency order, while PostgreSQL run metadata and command exit codes will make failures inspectable. Automated orchestration remains an advanced option if a concrete recurring or recoverable workflow emerges.

## Logical architecture

```text
Versioned seed + generator configuration
                  |
                  v
       Python synthetic generator
                  |
                  v
      PostgreSQL source schemas
                  |
                  v
      dbt staging -> intermediate -> marts
          |                 |          |
          |                 |          +--> KPI and data-quality outputs
          |                 +-------------> model-ready feature relations
          +-------------------------------> dbt tests and lineage
                                      |
                         Python train/score jobs
                                      |
                                      v
                      versioned prediction tables
                                      |
                                      v
                     FastAPI read/scenario contracts
                                      |
                                      v
                         Next.js operator UI
```

All durable analytical facts flow through PostgreSQL. Files may be used as deterministic generated artifacts or model artifacts, but they are not an alternative semantic store. The UI does not query PostgreSQL directly. FastAPI does not recreate contracted dbt metrics. Model code consumes documented feature relations and writes versioned outputs; it does not mutate historical source facts.

## Technology ownership

### Python

**Owns:**

- Deterministic synthetic data generation from an explicit seed and versioned parameters.
- Repeatable loading commands and input validation at the ingestion boundary.
- Feature extraction interfaces that read dbt-produced relations.
- Model training, temporal evaluation, serialization, batch scoring, and model-run metadata.
- Data-quality report rendering when presentation beyond dbt test output is needed.
- Small, composable CLI commands with non-zero failure exits.

**Intentionally does not own:**

- Business KPI definitions that belong in dbt and the metric contract.
- A second transformation graph implemented with dataframes.
- API request handling or UI presentation.
- Scheduling, distributed processing, or long-running daemon behavior in the MVP.
- Automatic operational actions, model promotion, or threshold changes.

Python packages should be organized by responsibility—for example generation, loading, forecasting, reliability, and reporting—rather than collected into one pipeline script. Randomness must flow from explicit seeded generators; global or implicit random state is insufficient.

### PostgreSQL

**Owns:**

- Durable local storage for source-shaped synthetic data and analytical relations.
- Relational integrity, database types, transaction boundaries, and query execution.
- Separate schemas for source data, dbt-managed outputs, and application or model metadata as needed.
- Immutable or append-oriented forecast and risk predictions keyed by issuance time and model version.
- The stable data boundary shared by dbt, model jobs, and FastAPI.

**Intentionally does not own:**

- Hidden business logic in ad hoc stored procedures or triggers.
- Model training, model artifact storage strategy, or scenario policy.
- Authentication as a product feature beyond local-development credentials.
- Production high availability, replication, or warehouse-scale concurrency.

Database constraints should protect structural invariants. They should not duplicate all dbt tests or conceal transformation semantics in triggers.

### dbt-postgres

**Owns:**

- Source declarations, freshness expectations, and lineage.
- Staging models for naming, typing, deduplication, and minimal source normalization.
- Intermediate models for reusable joins, event reconciliation, temporal logic, capacity calculations, and model-ready features.
- Marts for contracted KPIs, decision-ready grains, data-quality summaries, and stable API/model inputs.
- SQL-level tests for uniqueness, non-null requirements, accepted values, relationships, and documented business assertions.
- Generated documentation of the analytical graph.

**Intentionally does not own:**

- Source-data generation or transport.
- Statistical model fitting or artifact serialization.
- Request-time scenario execution.
- UI formatting or application authorization.
- Scheduling its own runs.

The three-layer convention is a semantic boundary, not a requirement to maximize model count. A transformation belongs in the lowest reusable layer that expresses its responsibility clearly.

### scikit-learn and statsmodels

**Own:**

- A reproducible demand forecast and one charger failure-risk model.
- Temporal training, validation, and test procedures; baseline comparisons; metrics; calibration where applicable; and uncertainty estimates appropriate to the method.
- Pipelines that freeze preprocessing with the estimator and preserve version, feature cutoff, training window, and evaluation metadata.

Use scikit-learn for composable preprocessing, classification, calibration, and general regression. Use statsmodels when an interpretable time-series or statistical formulation provides clearer intervals and diagnostics. The MVP need not use both libraries merely because both are approved; select the simplest credible implementation for each model.

**Intentionally do not own:**

- KPI computation, source cleaning already expressed in dbt, or application presentation.
- Online learning or request-time retraining.
- Autonomous model selection, deployment, promotion, or threshold tuning.
- Claims that synthetic out-of-sample performance will transfer to a real network.

### FastAPI

**Owns:**

- Versioned, validated request and response contracts for network KPIs, site detail, demand forecasts, charger risk, pricing scenarios, and data-quality status.
- Query composition over curated marts and prediction tables.
- Bounded request-time pricing scenario calculations where interactive inputs are required.
- Input bounds, clear error responses, health or readiness checks, and application-level logging.
- Dependency injection that makes database and scenario behavior testable.

**Intentionally does not own:**

- Reimplementing dbt metrics in Python.
- Model training, large batch scoring, source ingestion, or pipeline scheduling.
- Direct operational integration that changes prices, chargers, work orders, inventory, or suppliers.
- Serving as a generic SQL proxy.
- Hiding stale, unknown, or failed data-quality state behind successful HTTP responses.

Read endpoints should query stable relations. The pricing endpoint may calculate a constrained scenario from validated user inputs and versioned inputs, but it must not publish a price or label observational response as causal.

### Next.js

**Owns:**

- The polished operator experience: Command Center, site context, KPI and model views, constrained pricing controls, and data-trust warnings.
- Client-side interaction and presentation state, including loading, empty, error, stale, and unknown states.
- Consistent distinction between observations, forecasts, risk estimates, scenarios, and human decisions.
- Accessible layout, formatting, charts, tables, and responsive behavior.
- Calls to documented FastAPI endpoints through a small typed client boundary.

**Intentionally does not own:**

- Metric formulas, model inference logic, database access, or durable analytical state.
- Server-side duplication of the Python API as a second business backend.
- Authentication, multi-tenancy, or production content management in the MVP.
- Pretending that a displayed recommendation has been executed.

Next.js server capabilities may proxy configuration-safe requests if local deployment needs it, but the Python API remains the business application boundary.

### Docker Compose

**Owns:**

- A reproducible local PostgreSQL service, network, health check, and named development volume.
- Consistent service configuration and startup order for the local stack.
- Optional container definitions for FastAPI and Next.js when a full-container path improves reviewer setup.

**Intentionally does not own:**

- Pipeline semantics, retries, scheduling, or model promotion.
- Production deployment, cluster management, secret management, backups, or high availability.
- Masking application readiness problems with container startup order alone.

The preferred inner development loop may run PostgreSQL in Compose while Python, dbt, FastAPI, and Next.js run on the host for faster reloads. A full-container demonstration path can coexist, but it must not become a separate behaviorally inconsistent architecture.

### Makefile

**Owns:**

- Stable, discoverable task names such as `setup`, `up`, `generate`, `load`, `dbt-build`, `train`, `report`, `api`, `ui`, `test`, `demo`, and `down`.
- The documented ordering of finite local tasks and propagation of non-zero exit status.
- A thin compatibility layer over Python, dbt, Docker Compose, and package-manager commands.

**Intentionally does not own:**

- Business or transformation logic embedded in shell recipes.
- Stateful scheduling, distributed execution, automatic retry policies, or run-history storage.
- Platform detection so elaborate that the Makefile becomes an application.

Recipes should delegate to checked-in scripts or native tool commands. Because Windows does not include `make` by default, the README must name a supported installation path and expose the underlying commands. A small PowerShell equivalent may be added if Windows setup friction proves material, but it must call the same task entry points rather than implement a second pipeline.

## Execution model

The canonical MVP build is a finite dependency chain:

1. Start PostgreSQL and wait for its health check.
2. Generate deterministic synthetic source files or records.
3. Load source-shaped tables transactionally.
4. Run `dbt build` for staging, intermediate, marts, and tests.
5. Train and evaluate the demand and failure-risk models from versioned feature relations.
6. Batch-score the declared horizon and write versioned predictions.
7. Run any post-score dbt models or tests needed to join predictions into presentation marts.
8. Render the data-quality report.
9. Start FastAPI and Next.js for the demonstration.

Each finite command must be safe to rerun for the same seed and run identifier, or must fail with a clear conflict instead of duplicating facts. A failed stage stops the aggregate `make demo` or `make pipeline` command. The previous successful data remains inspectable; consumers must use readiness metadata rather than assuming that the newest attempted run completed.

## Orchestrator decision

An orchestrator is **not necessary for the MVP**. The project has one local batch dependency chain, one operator, no service-level deadline, no parallel backfill fleet, and no requirement for unattended recovery. A Makefile provides adequate dependency ordering and developer ergonomics. Python and dbt emit useful exit codes, while a small run table can record start time, end time, input version, status, and error summary.

Adding Airflow, Dagster, Prefect, or a cloud scheduler now would introduce another service, metadata store or daemon, dependency graph, version surface, and UI without solving an MVP requirement. Docker Compose startup dependencies are also not orchestration; they only help establish local service readiness.

Reconsider an orchestrator only when at least one concrete job requires it, for example:

- Unattended public-source ingestion and dependent daily refreshes with deadlines.
- Independent retries and backfills across multiple partitions or dates.
- Concurrent pipelines whose dependencies cannot be expressed safely as one finite local command.
- Durable scheduling, alerting, run history, or manual rerun controls used by someone other than the developer.
- A need to prevent overlapping runs or coordinate promotion across environments.

If that threshold is reached, write a separate ADR against measured workflow needs. The orchestrator should call the same idempotent Python and dbt commands used locally; it must not absorb their business logic.

## Rejected additions

### Apache Spark

Rejected for the current scope. The expected data fits in PostgreSQL and local memory, and the workload is dominated by relational transformations and modest model training. Spark would add a JVM runtime, distributed-data abstractions, file-format decisions, and local resource cost without a measured volume or runtime problem. Reconsider only if observed data size or transformation duration exceeds the practical limits of PostgreSQL and a single machine, and after simpler indexing, incremental models, and query tuning are exhausted.

### Kubernetes

Rejected. NOVA has no production cluster, multi-service scaling target, high-availability objective, or team deployment boundary. Compose is sufficient for local service lifecycle. Kubernetes manifests would simulate operational sophistication rather than satisfy a requirement. Reconsider only with a real multi-environment deployment, independent scaling or availability needs, and an owner prepared to operate the cluster.

### Kafka or another event broker

Rejected. The MVP is batch-based and has no real-time source, low-latency consumer, event fan-out, or durable streaming requirement. PostgreSQL transactions and deterministic batch loads provide the required consistency. Reconsider only if a measured latency objective and multiple independent consumers require ordered event delivery or replay that periodic ingestion cannot provide.

### Cloud services

Rejected as MVP dependencies. Managed databases, object storage, hosted orchestration, model registries, and deployment platforms would add credentials, cost, network dependency, and provider-specific setup to a local portfolio demonstration. They may later provide a public demo or advanced operational example, but the local deterministic path must remain complete and authoritative. Any cloud adoption requires a separate deployment decision covering cost limits, secrets, data lifecycle, and teardown.

These technologies are rejected because the workload does not require them, not because they are universally unsuitable.

## Failure modes and responses

### PostgreSQL is unavailable or not ready

Compose health checks gate dependent commands, and FastAPI readiness reports failure rather than returning empty success. Commands use bounded connection retries. A missing database never becomes a zero-valued network.

### A partial or repeated source load creates duplicates

Loads use stable deterministic keys, transactions, and explicit replace-by-run or idempotent upsert behavior. Row-count and uniqueness tests fail the run. The loader records the seed and generator version so mismatched reruns are visible.

### Schema drift or an invalid generated record breaks transformation

Source contracts and dbt staging tests fail early. Invalid records may be quarantined with counts and reasons, but a critical loss of coverage blocks dependent marts. SQL errors must not be swallowed to keep the UI green.

### dbt succeeds partially but a mart or test fails

The aggregate command exits non-zero and does not mark the analytical run ready. Previously completed relations may physically exist, so the API checks a successful run marker or data cutoff and exposes stale status rather than assuming relation existence means freshness.

### Model training leaks future information or becomes irreproducible

Temporal split assertions, immutable feature cutoffs, fixed seeds, pipeline serialization, baseline evaluation, and stored code or configuration versions are required. Predictions are append-oriented and tied to model and issuance versions. A model failing evaluation does not replace the last accepted demonstration model.

### Sparse classes or unseen charger cohorts destabilize failure risk

Training validates class counts and cohort coverage. The API exposes unknown or out-of-population status, not a fabricated low probability. Threshold metrics include denominators and mature-label coverage.

### A pricing request exceeds credible bounds

FastAPI rejects values outside declared price and time constraints. Within bounds, the response still flags extrapolation, weak evidence, capacity risk, and assumed costs. The endpoint has no connector to a pricing publication system.

### API and UI contracts drift

FastAPI response models define the contract; generated or checked TypeScript types and contract tests catch incompatible changes. The UI renders explicit unavailable and stale states rather than substituting mock values in normal operation.

### Containers start but applications are not usable

Health checks distinguish process startup from database, migration, and API readiness. Compose ordering is not treated as proof of readiness. Logs remain accessible through ordinary Compose commands.

### Local disk, port, or memory pressure causes confusing behavior

Ports are configurable, volume ownership is documented, and generated datasets have a bounded default size. Setup guidance identifies expected disk and memory use. Cleanup targets distinguish stopping services from deliberately removing data; destructive volume removal is never hidden inside the normal `down` or rebuild path.

### Data is stale, incomplete, or unknown

The data-quality state propagates through marts and API responses into the relevant UI surface. Unknown is distinct from passed. Models do not interpret missing telemetry as health, and the interface retains the last successful cutoff with a visible warning.

## Local-development tradeoffs

- **One PostgreSQL instance is simple but shared.** It gives dbt, models, and API one consistent store, at the cost of weaker isolation between workloads. Separate schemas, least-privilege local roles where practical, and clear run identifiers reduce accidental coupling.
- **Host processes improve iteration but expose environment differences.** Hot reload and debugging are faster when Python and Node run locally, while Compose-only PostgreSQL keeps the heaviest dependency consistent. Pinned Python and Node versions plus lockfiles are required.
- **Full containers improve reproducibility but slow the inner loop.** They add image builds, bind-mount behavior, and cross-platform filesystem quirks. The project should document one authoritative output regardless of launch mode.
- **A Makefile is concise but less native on Windows.** It remains the public task vocabulary because it is readable and CI-friendly. Underlying commands must be documented, and recipes must avoid Unix-only shell cleverness.
- **PostgreSQL is heavier than SQLite but removes a migration later.** The product explicitly requires PostgreSQL and uses its relational behavior; a second SQLite path would create semantic and SQL divergence.
- **Batch scoring is less interactive than online inference but more auditable.** Frozen predictions align with the weekly decision loop and enable temporal evaluation. Only the bounded pricing scenario needs request-time calculation.
- **Synthetic data improves reproducibility but limits external validity.** Generator assumptions must be documented, seeded, and visibly labeled. Model scores demonstrate engineering and evaluation practice, not real-world performance.
- **No orchestrator minimizes moving parts but provides limited unattended recovery.** For the MVP, a developer reruns a failed idempotent command. This is acceptable because there is no operational schedule or production service commitment.

## Consequences

### Positive

- The complete system can run locally with a small, understandable dependency set.
- Technology boundaries align with the metric contract and decision loop.
- SQL transformations, statistical models, API contracts, and presentation remain independently testable.
- Deterministic generation and batch predictions make demonstrations reproducible and auditable.
- The architecture can add public ingestion, supplier modeling, drift monitoring, optimization, or orchestration without making them MVP prerequisites.

### Negative

- Local setup still requires Docker, Python, Node.js, dbt, and a compatible `make` implementation or equivalent underlying commands.
- One-machine execution limits data scale and parallelism.
- Manual pipeline invocation lacks unattended retries, scheduling, and centralized alerting.
- PostgreSQL schema and run-state discipline must substitute for features a managed platform might provide.
- Maintaining Python and TypeScript contracts introduces a cross-language boundary that requires tests or type generation.

### Neutral constraints

- This decision does not select the exact forecasting algorithm, classifier, chart library, Python package manager, or JavaScript package manager. Those choices should favor reproducibility and simplicity and may be recorded when implementation begins.
- This decision does not claim a production deployment architecture. Any move to real data, external users, automated actions, or availability commitments requires new security, privacy, reliability, and deployment decisions.

## Validation criteria

This architecture is validated when the MVP completion checklist in `docs/SCOPE.md` passes through the Makefile task interface, all durable displayed data can be traced to PostgreSQL relations and versioned model outputs, and the entire demonstration works with advanced services absent. If measured scale, latency, scheduling, or recovery requirements invalidate those assumptions, the team must document the evidence and supersede this ADR rather than layering infrastructure onto it informally.
