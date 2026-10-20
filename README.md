# Trade Break Detection & Triage Agent

An automated post-trade reconciliation and exception-management pipeline that detects discrepancies between execution and clearing records, prioritises operational risk, performs AI-assisted root-cause triage, and manages incidents through Jira with persistent deduplication, audit logging, and failure recovery.

Built as a portfolio project exploring the intersection of **trading operations, data engineering, workflow automation, and controlled AI assistance**.

---

## Overview

Post-trade discrepancies occur when internally recorded executions do not align with confirmations received from clearing or brokerage systems.

Operations teams may need to investigate exceptions such as:

- quantity mismatches
- price mismatches
- account mapping errors
- missing confirmations
- duplicate bookings
- late confirmations
- side mismatches
- multiple simultaneous discrepancies

This project implements an end-to-end workflow for detecting, triaging, escalating, and auditing these exceptions.

The architecture follows one central principle:

> **Use deterministic logic for operational controls and AI only where interpretation adds value.**

AI does not determine whether a trade is matched, classify deterministic severity, control deduplication, or decide Jira capacity.

---

## Architecture

```text
Execution Records ──┐
                    ├──> Normalisation
Clearing Records ───┘         │
                              ▼
                        Reconciliation
                         │          │
                      MATCH       BREAK
                         │          │
                         │          ▼
                         │    Severity Scoring
                         │          │
                         │          ▼
                         │      AI Triage
                         │          │
                         │          ▼
                         │    Confidence Gate
                         │          │
                         │          ▼
                         │     Deduplication
                         │      │         │
                         │   Existing     New
                         │      │         │
                         │   COMMENT      ▼
                         │          Severity Priority
                         │                 │
                         │            Ticket Cap
                         │                 │
                         └──────────┬──────┘
                                    ▼
                              Jira Actions
                                    │
                          ┌─────────┴─────────┐
                          ▼                   ▼
                    Incident Registry     Audit Log
                                    │
                                    ▼
                              Run Summary
```

---

## Key Features

- Deterministic execution vs clearing reconciliation
- Trade-break classification
- Deterministic severity scoring
- AI-assisted root-cause analysis
- Confidence-gated human review
- Persistent cross-run incident deduplication
- In-run duplicate protection
- Severity-prioritised Jira ticket allocation
- Jira CREATE and COMMENT workflows
- Fail-closed per-run ticket cap
- Explicit failure and retry classification
- Persistent incident registry
- Per-trade audit logging
- Run-level operational summaries

---

## Break Taxonomy

| Break Type | Example |
|---|---|
| `QUANTITY_MISMATCH` | Execution and clearing quantities differ |
| `PRICE_MISMATCH` | Price difference exceeds configured tolerance |
| `ACCOUNT_MISMATCH` | Execution and clearing accounts differ |
| `CURRENCY_MISMATCH` | Currency differs between records |
| `SIDE_MISMATCH` | BUY/SELL direction differs |
| `MISSING_CONFIRMATION` | No accepted clearing confirmation exists |
| `DUPLICATE_TRADE` | Multiple records exist for the same trade |
| `LATE_CONFIRMATION` | Clearing confirmation exceeds configured SLA |
| `MULTIPLE_BREAKS` | Multiple discrepancies affect the same trade |
| `UNKNOWN` | Malformed or insufficient data prevents reliable classification |

---

## Deterministic Severity

Breaks are classified as:

```text
CRITICAL
HIGH
MEDIUM
LOW
```

Severity is calculated before AI triage.

Rules consider factors such as:

- break type
- quantity mismatch percentage
- clearing delay
- multiple material discrepancies
- approximate trade notional

This ensures operational priority remains reproducible and testable.

---

## AI-Assisted Triage

Once a deterministic break has been identified, an LLM analyses the supplied evidence and returns structured triage information:

```json
{
  "trade_id": "TRD-100002",
  "break_type": "QUANTITY_MISMATCH",
  "severity": "HIGH",
  "likely_root_cause": "PARTIAL_FILL_OR_ALLOCATION",
  "confidence": 0.55,
  "recommended_action": "Verify the allocated quantity with the relevant operations source.",
  "evidence": [
    "OBSERVED: execution quantity differs from clearing quantity",
    "HYPOTHESIS: allocation may not be reflected in the internal booking"
  ]
}
```

The model cannot override deterministic break type or severity.

### Confidence Gate

| Confidence | Routing |
|---|---|
| `>= 0.80` | Auto-triaged |
| `0.50–0.79` | Review required |
| `< 0.50` | Manual investigation required |
| `UNKNOWN` root cause | Manual investigation required |

The automatic threshold was deliberately not lowered simply to increase automation during testing.

---

## Incident Deduplication

Each break receives a deterministic incident identity:

```text
TRADE_ID:BREAK_TYPE
```

Example:

```text
TRD-100008:SIDE_MISMATCH
```

Two levels of protection are implemented.

**Persistent deduplication** detects incidents already registered during previous workflow executions and routes them to Jira COMMENT rather than CREATE.

**In-run deduplication** prevents repeated observations within the same workflow execution from independently creating multiple Jira tickets.

AI output is deliberately excluded from incident identity.

---

## Severity-Prioritised Ticket Allocation

Jira creation is protected by a configurable per-run ticket cap.

When eligible CREATE candidates exceed available capacity, incidents are ranked:

```text
CRITICAL > HIGH > MEDIUM > LOW
```

Equal-severity incidents retain their original processing order.

Existing incidents routed to COMMENT do not consume CREATE capacity.

Invalid ticket-cap configuration fails closed to zero new tickets rather than becoming unlimited.

---

## Failure Recovery

Jira failures are explicitly classified using:

```text
retry_required
retry_stage
```

Supported retry stages include:

```text
JIRA_CREATE
JIRA_COMMENT
NONE
```

A failed CREATE does not write a false incident-registry entry.

A failed COMMENT preserves the existing incident mapping.

AI fallback is observable but does not automatically make a successful Jira operation retryable.

---

## Validation

The workflow was tested against integration, deduplication, failure-recovery, capacity-control, and regression scenarios.

Key tests included:

| Scenario | Result |
|---|---|
| Cross-run deduplication | Existing incidents routed to COMMENT |
| In-run duplicate | 2 observations → 1 Jira CREATE |
| Ticket cap | 3 candidates → 2 CREATE + 1 cap skip |
| Jira CREATE failure | Failure audited; no false registry entry |
| Jira CREATE recovery | Retry created exactly one incident |
| AI structured-output failure | Deterministic workflow continued safely |
| Jira COMMENT failure | Explicit `JIRA_COMMENT` retry classification |
| Jira COMMENT recovery | Existing incident updated without duplicate CREATE |
| Severity-priority regression | CRITICAL incidents received available CREATE capacity |

### Final V1 Regression

The final regression processed:

| Metric | Result |
|---|---:|
| Unique trades | 13 |
| Matched trades | 2 |
| Trade breaks | 11 |
| CRITICAL | 3 |
| HIGH | 5 |
| MEDIUM | 2 |
| LOW | 1 |
| Jira tickets created | 2 |
| Existing incidents updated | 4 |
| Ticket-cap skips | 4 |
| Jira failures | 0 |
| Workflow errors | 0 |
| Retry-required items | 0 |

A previous regression identified a design issue where ticket capacity was allocated by processing order. The routing algorithm was subsequently redesigned so CREATE candidates are prioritised by deterministic severity before the cap is applied.

See [`docs/TEST_MATRIX.md`](docs/TEST_MATRIX.md) for the complete validation history.

---

## Sample Data

The repository contains synthetic execution and clearing datasets:

```text
data/
├── execution_records_sample.csv
└── clearing_confirmations_sample.csv
```

The fixtures intentionally include formatting inconsistencies, duplicate records, missing confirmations, delayed confirmations, mismatches, and malformed values to exercise the workflow's normalisation and reconciliation logic.

No real trading, client, brokerage, or clearing data is included.

---

## Repository Structure

```text
trade-break-triage-agent/
│
├── README.md
│
├── assets/
│   └── README.md
│
├── data/
│   ├── README.md
│   ├── execution_records_sample.csv
│   └── clearing_confirmations_sample.csv
│
├── docs/
│   ├── DESIGN_DECISIONS.md
│   ├── PRODUCTION_HARDENING.md
│   └── TEST_MATRIX.md
│
└── workflow/
    ├── README.md
    └── trade-break-triage-agent.json
```

---

## Workflow Import

The repository includes a sanitised n8n workflow export:

```text
workflow/trade-break-triage-agent.json
```

Environment-specific credentials and resource identifiers have been removed.

Before running the workflow, configure the required Jira credentials, Data Tables, AI model credentials, and environment-specific resource identifiers.

The included scheduled trigger is intentionally disabled for safe demonstration.

---

## Production Hardening

The current implementation is an operational prototype rather than a production trading system.

A production implementation would introduce:

- transactional database-level uniqueness for concurrent executions
- atomic incident reservation/upsert
- Jira/local-state reconciliation
- governed production data ingestion
- AI evaluation against labelled historical breaks
- operational monitoring and alerting
- managed secrets and environment separation

See [`docs/PRODUCTION_HARDENING.md`](docs/PRODUCTION_HARDENING.md).

---

## Design Documentation

Additional technical documentation:

- [`Design Decisions`](docs/DESIGN_DECISIONS.md)
- [`Validation & Test Matrix`](docs/TEST_MATRIX.md)
- [`Production Hardening`](docs/PRODUCTION_HARDENING.md)

---

## Tech Stack

**Workflow orchestration:** n8n  
**Incident management:** Jira  
**AI triage:** structured LLM workflow  
**State management:** n8n Data Tables  
**Core logic:** JavaScript  
**Version control:** Git / GitHub

---

## Project Status

**V1 — Feature frozen and integration tested**

Final regression validated deterministic reconciliation, severity-prioritised Jira capacity, persistent and in-run deduplication, Jira CREATE/COMMENT handling, audit persistence, retry semantics, and run-level reporting.

The workflow is maintained in a controlled portfolio/demo configuration with automatic scheduling disabled.
