# NORMAL-001 - Legitimate OCPP Charging Session

## Scenario

This scenario represents a normal EV charging session between the EVSE simulator and the OCPP CSMS.

## Sequence

1. Authorize
2. StartTransaction
3. MeterValues
4. MeterValues
5. MeterValues
6. StopTransaction

## Expected Result

- Authorization accepted
- Transaction created successfully
- Meter values transmitted
- Transaction terminated normally

## Purpose

This scenario establishes a legitimate behavioral baseline for later
rule-based and machine-learning anomaly detection experiments.
