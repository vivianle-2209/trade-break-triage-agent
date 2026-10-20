# V1 Validation & Test Matrix

This document records the integration, failure-recovery, deduplication, safety-control, and regression tests performed during development of the Trade Break Detection & Triage Agent.

The objective was not only to validate successful execution, but also to test failure modes and ensure that external side effects such as Jira ticket creation remain controlled and auditable.

## Validation Strategy

The workflow was tested across four areas:

1. **Reconciliation correctness** — execution and clearing records produce the expected matched trades and deterministic break classifications.
2. **Idempotency and side-effect control** — repeated or duplicate observations do not create duplicate Jira incidents.
3. **Failure recovery** — Jira failures are explicitly classified, audited, and safely recoverable.
4. **Operational prioritisation** — limited Jira creation capacity is allocated deterministically to the highest-severity incidents.

Where applicable, tests used real Jira requests and persistent n8n Data Tables rather than simulated responses.

---

## Test Matrix

| Execution | Test | Expected Behaviour | Result | Status |
|---:|---|---|---|:---:|
| 18 | Cross-run persisted deduplication | Previously registered breaks should update existing Jira incidents instead of creating new tickets | 2 recurring breaks resolved through the incident registry and generated COMMENT actions; 0 new Jira tickets | PASS |
| 26 | In-run duplicate protection | Two observations with the same incident identity should not create two Jira tickets | 2 identical observations resulted in 1 Jira CREATE and 1 `DUPLICATE_PREVENTED_IN_RUN`; exactly 1 incident was registered | PASS |
| 30 | Jira ticket-cap enforcement | With 3 eligible new incidents and a cap of 2, only 2 Jira tickets should be created | 2 Jira tickets created; third incident audited as `JIRA_SKIPPED_TICKET_CAP_REACHED`; no false registry entry | PASS |
| 36 | Jira CREATE failure | A failed Jira CREATE must not create a registry record and must be marked retryable | Real Jira request failed; 0 tickets and 0 registry rows created; audit recorded `JIRA_TICKET_CREATION_FAILED` with retry required | PASS |
| 38 | Jira CREATE recovery | Retrying the failed incident after restoring Jira configuration should create exactly one ticket | Retry created TOPS-15 exactly once and persisted the incident only after Jira returned a valid issue key | PASS |
| 38 | AI structured-output fallback | AI failure must not prevent deterministic exception handling | Structured-output validation failed; fallback preserved deterministic Jira processing and routed the break for manual review | PASS |
| 42 | Jira COMMENT failure | Failure to update an existing incident should preserve its registry mapping and request a COMMENT retry | Real Jira COMMENT request failed; registry remained mapped to TOPS-15; audit stored `retry_required=true`, `retry_stage=JIRA_COMMENT` | PASS |
| 44 | Jira COMMENT recovery | Retrying the recurring incident should update the same Jira issue without creating another ticket | Comment succeeded on TOPS-15; registry remained unique and was updated with the new last-seen execution | PASS |
| 49 | Full release-candidate regression | Full dataset should preserve reconciliation, deduplication, audit, retry and Jira controls | All functional checks passed; regression exposed severity-blind ticket-cap allocation | PASS / DESIGN ISSUE FOUND |
| 52 | Severity-prioritised ticket allocation | Scarce Jira CREATE capacity must go to the highest-severity eligible incidents | Two available slots went to CRITICAL breaks TRD-100008 and TRD-100009; no HIGH or MEDIUM break displaced a CRITICAL candidate | PASS |
| 52 | Final V1 regression | Full workflow should remain internally consistent after severity-priority change | 13 trades processed, 11 breaks, 2 matches, 2 Jira creates, 4 recurring-incident comments, 0 Jira failures, 0 workflow errors and 0 retry-required items | PASS |

---

## Final Regression Results

The final V1 regression processed:

| Metric | Result |
|---|---:|
| Unique trades | 13 |
| Matched trades | 2 |
| Trade breaks | 11 |
| CRITICAL breaks | 3 |
| HIGH breaks | 5 |
| MEDIUM breaks | 2 |
| LOW breaks | 1 |
| Jira tickets created | 2 |
| Existing Jira incidents updated | 4 |
| Ticket-cap skips | 4 |
| AI fallbacks | 0 |
| Jira CREATE failures | 0 |
| Jira COMMENT failures | 0 |
| Workflow errors | 0 |
| Retry-required items | 0 |

The final regression produced two new CRITICAL Jira incidents:

- `TRD-100008:SIDE_MISMATCH` → TOPS-18
- `TRD-100009:MULTIPLE_BREAKS` → TOPS-19

The third CRITICAL CREATE candidate remained eligible but was held back by the configured two-ticket per-run capacity limit.

---

## Defect Found Through Regression Testing

An earlier implementation applied the Jira creation cap in processing order.

During execution 49, two HIGH-severity incidents consumed the available ticket slots before three later CRITICAL incidents were processed.

Although the cap itself worked correctly, this meant scarce incident capacity was not allocated according to operational severity.

The allocation algorithm was redesigned into two stages:

1. Resolve persistent and in-run deduplication and determine candidate actions.
2. Rank only new CREATE candidates using deterministic severity before applying the ticket cap.

Priority is:

`CRITICAL > HIGH > MEDIUM > LOW`

For incidents with equal severity, original processing order is preserved.

Execution 52 validated the correction: both available Jira ticket slots were allocated to CRITICAL incidents.

---

## Retry Semantics

Retry eligibility is intentionally separated from general workflow errors.

The audit model records:

- `retry_required`
- `retry_stage`

Supported Jira retry stages are:

- `JIRA_CREATE`
- `JIRA_COMMENT`
- `NONE`

For example, an AI fallback can be observable as a workflow degradation without implying that a successful Jira action should be repeated.

This prevents the retry queue from treating every processing warning as a request to replay an external side effect.

---

## Remaining Validation Boundaries

### Concurrent workflow executions

The portfolio implementation uses n8n Data Tables as the incident registry.

Sequential and in-run deduplication were validated, but overlapping workflow executions are not guaranteed to be globally idempotent because the registry does not enforce a database-level unique constraint on `dedup_key`.

A production implementation should use transactional storage with atomic uniqueness enforcement.

### AI auto-triage

The confidence gate supports automatic triage at confidence `>= 0.80`.

Across the final regression dataset, model confidence remained between 0.2 and 0.7, so all breaks correctly remained subject to human review.

The threshold was not lowered merely to exercise the automatic path.

### Jira success followed by registry failure

A Jira action may theoretically succeed while subsequent registry persistence fails.

Blindly replaying the Jira CREATE operation would be unsafe because the external side effect may already exist.

Production recovery should reconcile Jira state first and repair the registry rather than automatically repeating CREATE.
