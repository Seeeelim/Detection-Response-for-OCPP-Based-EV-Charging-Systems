# Hybrid Threat Detection & Response for OCPP-Based EV Charging Systems

Master's Cybersecurity Project at Grand Valley State University.

## Overview

This project develops a multi-source cybersecurity monitoring and response
platform for OCPP-based electric vehicle charging systems.

The environment currently uses the SAP e-Mobility Charging Stations Simulator
to emulate EV charging stations and SteVe as the OCPP 1.6J Charging Station
Management System (CSMS).

The security architecture will integrate Suricata, Wazuh, Python-based event
correlation, and machine-learning behavioral anomaly detection to identify
known and previously unseen attacks affecting EV charging infrastructure.

## Current Architecture

SAP EVSE Simulator
192.168.100.10
        |
        | OCPP 1.6J / WebSocket
        |
        v
SteVe CSMS
192.168.100.20
        |
        +-- MariaDB
        |
        +-- Suricata
              |
              v
           EVE JSON

Planned:
Suricata + Host/Application/OCPP telemetry
        |
        v
      Wazuh
        |
        v
Python Correlation Engine
   |              |
Rules         ML Anomaly
Detection      Detection
   \              /
     Incident Risk
          |
        Response

## Current Progress

- [x] Built an isolated VirtualBox EV charging cybersecurity lab
- [x] Deployed SAP e-Mobility Charging Stations Simulator
- [x] Deployed SteVe OCPP CSMS with MariaDB
- [x] Established OCPP 1.6J communication between EVSE001 and SteVe
- [x] Validated BootNotification and Heartbeat exchanges
- [x] Completed a legitimate OCPP charging transaction
- [x] Captured and documented the NORMAL-001 benign baseline
- [x] Simplified EVSE001 to one controlled charging connector
- [x] Deployed Suricata on the CSMS network interface
- [x] Collected EVSE-to-CSMS network telemetry in EVE JSON
- [x] Implemented the first custom Suricata rule for the OCPP communication path
- [ ] Integrate Suricata telemetry with Wazuh
- [ ] Develop OCPP-specific detection rules
- [ ] Generate controlled cyberattack scenarios
- [ ] Develop Python multi-source event correlation
- [ ] Develop behavioral machine-learning anomaly detection
- [ ] Compare rule-based, ML-based, and hybrid detection performance

## Current Testbed

| Component | Technology |
|---|---|
| Charging Station Simulator | SAP e-Mobility Charging Stations Simulator |
| CSMS | SteVe |
| Protocol | OCPP 1.6J |
| Network IDS | Suricata |
| SIEM | Wazuh (planned next) |
| Correlation | Python |
| Machine Learning | Planned behavioral anomaly detection |

## Baseline Scenario

`NORMAL-001` represents a legitimate OCPP charging session:

Authorize → StartTransaction → MeterValues → StopTransaction

The scenario is preserved as ground-truth benign behavior for later detection
and machine-learning evaluation.

## Security Notice

This repository contains only simulated laboratory infrastructure.
All IP addresses are private lab addresses and no production EV charging
systems are involved.

Credentials, private keys, secrets, and sensitive configuration files are not
included.
