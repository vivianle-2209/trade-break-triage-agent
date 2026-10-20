# Design Decisions

This document explains the major architectural decisions behind the Trade Break Detection & Triage Agent.

The system was designed around a central principle:

> Use deterministic logic for operational controls and use AI only where interpretation adds value.

This separation reduces the risk of allowing probabilistic model output to directly control reconciliation results or external operational side effects.

---

## 1. Deterministic Reconciliation

Trade matching and break detection are rule-based rather than AI-driven.

Execution and clearing records are compared across fields including:

- trade ID
- symbol
- side
- quantity
- price
- currency
- account
- confirmation timing

The reconciliation layer determines whether a trade is matched or broken and assigns a deterministic break type.

### Why

Reconciliation decisions should be:

- reproducible
- explainable
- testable
- auditable

The same pair of records should produce the same reconciliation result regardless of model behaviour.

AI therefore does not determine whether a trade is matched.

---

## 2. Deterministic Severity

Severity is calculated before AI triage.

The severity engine considers deterministic break characteristics and approximate trade exposure to classify incidents as:

`CRITICAL`, `HIGH`, `MEDIUM`, or `LOW`

### Why

Incident priority affects downstream operational actions.

Allowing an LLM to determine severity would introduce unnecessary variability into a control that should remain predictable and testable.

AI output cannot override deterministic severity.

---

## 3. AI as an Advisory Layer

AI is used after reconciliation and severity scoring.

Its role is to interpret an already-detected exception and provide structured investigation guidance such as:

- likely root cause
- confidence
- recommended action
- supporting evidence

Example root-cause categories include:

- `BOOKING_ERROR`
- `PARTIAL_FILL_OR_ALLOCATION`
- `CLEARING_DELAY`
- `REFERENCE_DATA_MISMATCH`
- `DUPLICATE_BOOKING`
- `PRICE_CAPTURE_ERROR`
- `ACCOUNT_MAPPING_ERROR`
- `UNKNOWN`

### Why

Root-cause analysis often requires interpretation across multiple pieces of evidence, making it a more appropriate use of AI than deterministic matching.

The model therefore assists investigation without becoming the system of record for reconciliation.

---

## 4. Confidence-Gated Human Review

AI recommendations are routed through a confidence gate.

| Confidence | Treatment |
|---|---|
| `>= 0.80` | Auto-triaged |
| `0.50–0.79` | Review required |
| `< 0.50` | Manual investigation required |
| Root cause = `UNKNOWN` | Manual investigation required |

### Why

The workflow should become more conservative when the model is uncertain.

The threshold was not reduced merely to increase automation during testing. In the final regression dataset, AI confidence remained below the automatic threshold, so all breaks remained subject to human review.

---

## 5. Deterministic Incident Identity

Each incident is assigned a normalised deduplication key:

`TRADE_ID:BREAK_TYPE`

For example:

`TRD-100008:SIDE_MISMATCH`

The components are normalised before comparison.

### Why

External incident creation should not depend on probabilistic AI output.

Using deterministic fields means the same operational exception can be recognised across workflow executions even if AI wording, confidence, or recommended action changes.

---

## 6. Persistent and In-Run Deduplication

The workflow protects against duplicate incidents at two levels.

### Persistent deduplication

An incident registry stores the relationship between a `dedup_key` and its Jira issue.

When the same incident appears in a later workflow execution, the workflow updates the existing Jira issue using COMMENT instead of creating another ticket.

### In-run deduplication

The workflow also tracks incident identities within the current execution.

Repeated observations of the same incident therefore cannot independently create multiple Jira tickets during one run.

### Why

Cross-run and within-run duplication are different failure modes and require separate controls.

---

## 7. Deduplication Happens Before Capacity Allocation

Incident identity is resolved before the Jira CREATE cap is applied.

Conceptually:

`DEDUPLICATION → ACTION CANDIDATES → SEVERITY PRIORITY → CREATE CAP`

### Why

A duplicate observation should never compete with a genuinely new incident for scarce ticket-creation capacity.

Likewise, recurring incidents that resolve to COMMENT should not consume CREATE capacity.

---

## 8. Severity-Prioritised Jira Capacity

Jira ticket creation is capped per workflow execution.

When new CREATE candidates exceed available capacity, they are ranked:

`CRITICAL > HIGH > MEDIUM > LOW`

Items with equal severity retain their original processing order.

### Why

The original implementation applied the cap in processing order.

Regression testing showed that this could allow HIGH-severity incidents to consume available ticket slots before later CRITICAL incidents were processed.

The logic was redesigned so scarce CREATE capacity is allocated according to deterministic operational severity.

AI confidence is deliberately excluded from this ranking.

---

## 9. COMMENT Does Not Consume CREATE Capacity

Previously registered incidents are routed to COMMENT.

COMMENT actions do not consume the per-run Jira CREATE allowance.

### Why

The ticket cap exists to control creation of new external incidents.

Updating an already-known incident does not introduce the same duplicate-ticket risk and should not prevent a higher-priority new exception from being opened.

---

## 10. Fail-Closed Ticket Cap

The workflow uses a configurable maximum number of Jira tickets per execution.

If the configuration is:

- missing
- null
- empty
- boolean
- non-numeric
- negative

the effective CREATE capacity becomes zero.

### Why

Invalid safety configuration should reduce external side effects rather than silently remove the limit.

A malformed cap must never be interpreted as unlimited ticket creation.

---

## 11. Registry Persistence Only After Successful CREATE

A new incident is written to the persistent registry only after Jira returns a successful issue key.

### Why

Writing the registry before Jira succeeds could create false state:

`registry says incident exists → Jira ticket does not exist`

This could cause later executions to suppress a legitimate CREATE.

The external side effect therefore succeeds first, followed by persistence of its returned identity.

---

## 12. Jira Failures and Retry Semantics

Retry eligibility is explicit rather than inferred from every workflow error.

The workflow records:

- `retry_required`
- `retry_stage`

Supported Jira retry stages are:

- `JIRA_CREATE`
- `JIRA_COMMENT`
- `NONE`

### Why

Not every processing degradation means an external operation should be replayed.

For example, an AI structured-output failure may trigger deterministic fallback while Jira creation still succeeds.

Repeating Jira CREATE in that situation could create a duplicate incident.

Retry therefore requires explicit opt-in from the relevant Jira failure path.

---

## 13. AI Failure Does Not Control Deterministic Processing

If structured AI output fails validation, the workflow can fall back to conservative manual-review behaviour.

The AI failure remains observable, but deterministic reconciliation and incident handling can continue.

### Why

An advisory subsystem should not become a single point of failure for deterministic operational controls.

---

## 14. Jira Success + Registry Failure Is Not Blindly Retried

A remaining edge case occurs if Jira succeeds but registry persistence subsequently fails.

This is intentionally not treated as a normal Jira CREATE retry.

### Why

The external side effect may already exist.

Blindly repeating CREATE could produce a duplicate Jira incident.

A production recovery process should instead:

1. reconcile Jira using deterministic incident identity;
2. recover the existing issue key;
3. repair the incident registry;
4. avoid repeating CREATE.

---

## 15. Auditability

Each processed trade produces an audit outcome containing operational state such as:

- trade and break identity
- severity
- AI status
- action taken
- processing status
- Jira issue key where applicable
- retry requirement
- retry stage
- workflow execution identifier

### Why

Operational automation should make it possible to reconstruct what happened and why.

Audit records also make workflow behaviour independently comparable with Jira and registry state during regression testing.

---

## Design Principle Summary

The architecture intentionally separates four responsibilities:

**Deterministic controls**

Reconciliation, break classification, severity, incident identity, deduplication, capacity allocation, and retry eligibility.

**AI interpretation**

Root-cause hypothesis, evidence interpretation, and recommended investigation action.

**External side effects**

Controlled Jira CREATE and COMMENT operations.

**Operational state**

Incident registry, audit history, and run-level observability.

This separation allows AI to add investigative value without giving probabilistic model output authority over core reconciliation or incident-control decisions.
