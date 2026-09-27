# Wazuh SIEM

This directory documents the Wazuh SIEM component of the EV charging
cybersecurity monitoring platform.

## Role

The Wazuh server provides centralized security monitoring and analysis
for the OCPP charging infrastructure.

## Lab Address

- Wazuh SIEM: 192.168.100.30
- OCPP CSMS: 192.168.100.20
- EVSE Simulator: 192.168.100.10

## Components

- Wazuh Manager
- Wazuh Indexer
- Wazuh Dashboard
- Filebeat

## Integration

Suricata network-security events generated on the OCPP-CSMS host will
be forwarded to Wazuh for centralized analysis, correlation, alerting,
and visualization.

## Security

Credentials, certificates, private keys, keystores, indexes, and other
sensitive/generated data are intentionally excluded from this repository.
