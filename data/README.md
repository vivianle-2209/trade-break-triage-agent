# Sample Data

This directory contains synthetic execution and clearing datasets used to demonstrate and validate the Trade Break Detection & Triage Agent.

## Datasets

The project uses two primary input datasets:

- `execution_records_sample.csv` — simulated internal trade execution records
- `clearing_confirmations_sample.csv` — simulated clearing/broker confirmation records

The datasets are intentionally constructed to exercise reconciliation scenarios including matched trades, quantity and price mismatches, account mismatches, missing confirmations, duplicate trades, late confirmations, side mismatches, multiple breaks, and other exception cases.

## Data Privacy

All records in this directory are synthetic and created solely for demonstration and testing.

No real client, trading, brokerage, clearing, or proprietary market data is included.
