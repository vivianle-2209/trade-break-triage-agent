# Production Hardening

The current implementation is a portfolio-grade operational prototype built to demonstrate post-trade reconciliation, exception management, controlled AI assistance, and failure-safe workflow design.

The following changes would be required before deployment in a production trading environment.

## 1. Transactional Incident Registry

The current implementation uses n8n Data Tables for persistent incident state.

Sequential and in-run deduplication are supported, but two workflow executions running concurrently could potentially evaluate the same new `dedup_key` before either execution persists it.

### Production approach

Use a transactional database such as PostgreSQL with:

- `UNIQUE(dedup_key)`
- atomic insert/upsert operations
- transaction-aware incident reservation
- explicit concurrency handling

This would provide database-level idempotency across overlapping executions.

---

## 2. External-State Reconciliation

The workflow persists a new incident only after Jira successfully returns an issue key.

A remaining failure window exists if Jira CREATE succeeds but registry persistence fails immediately afterwards.

Blindly replaying CREATE would be unsafe because the Jira issue may already exist.

### Production approach

Introduce a reconciliation process that:

1. searches Jira using deterministic incident identity;
2. determines whether the external side effect already occurred;
3. recovers the Jira issue key;
4. repairs the local incident registry;
5. creates a new issue only when no existing incident can be found.

---

## 3. Production Data Ingestion

The portfolio implementation uses synthetic execution and clearing records.

### Production approach

Replace mock loaders with controlled ingestion from approved sources such as:

- execution-management systems
- clearing or broker feeds
- message queues
- governed databases
- internal APIs

Production ingestion would also require schema validation, source-level lineage, authentication, access controls, and appropriate handling of late or incomplete records.

---

## 4. AI Evaluation and Calibration

AI is restricted to advisory root-cause triage, while reconciliation and operational controls remain deterministic.

Before expanding automated triage, the model should be evaluated against a labelled historical break dataset.

Useful measures would include:

- root-cause classification accuracy
- confidence calibration
- false automation rate
- manual-review rate
- performance by break category

The current `0.80` automatic-triage threshold should therefore be treated as a configurable control rather than a permanently fixed production value.

---

## 5. Operational Monitoring

The current workflow records per-trade audit outcomes and run-level summaries.

A production implementation should additionally expose metrics such as:

- reconciliation volume
- break rate
- break rate by category and severity
- Jira CREATE/COMMENT success rate
- cap-skipped incidents
- retry queue size
- processing latency
- AI fallback rate
- incident age and SLA status

These metrics could feed dashboards and alerting for operations teams.

---

## 6. Secrets and Environment Management

Credentials and environment-specific configuration should remain outside the workflow definition.

Production deployment should use:

- managed credentials or secret storage
- environment-specific configuration
- least-privilege Jira/API permissions
- credential rotation
- separate development, test, and production environments

---

## Production Target Architecture

A production evolution of the project would therefore retain the current deterministic reconciliation and control model while replacing prototype infrastructure around it:

`Governed Data Sources → Reconciliation → Deterministic Controls → AI Advisory Triage → Transactional Incident Registry → Jira → Monitoring & Audit`

The objective is not to give AI more control, but to make the surrounding system more concurrent, recoverable, observable, and operationally robust.
