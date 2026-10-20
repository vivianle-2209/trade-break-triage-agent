# Workflow

This directory contains the n8n workflow definition for the Trade Break Detection & Triage Agent.

## Workflow Design

The workflow implements an end-to-end post-trade exception-management process:

1. Ingest execution and clearing records
2. Normalise source data
3. Reconcile trades using deterministic rules
4. Classify trade breaks
5. Calculate deterministic severity
6. Perform AI-assisted root-cause triage
7. Apply confidence-based review routing
8. Resolve persistent and in-run duplicates
9. Prioritise new incidents by severity
10. Create or update Jira incidents
11. Persist incident state and audit records
12. Compile a run-level operational summary

## Safety Boundary

AI is used as an advisory triage layer only.

Reconciliation, break classification, severity scoring, deduplication, Jira eligibility, ticket-cap allocation, and retry behaviour remain deterministic.

## Public Workflow Export

A sanitised n8n workflow export will be provided in this directory.

Credentials, secrets, environment-specific identifiers, and private configuration must not be included in the public export.
