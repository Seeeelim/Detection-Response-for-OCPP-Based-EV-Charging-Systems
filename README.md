# Detection and Response for OCPP-Based EV Charging Systems

A reproducible cybersecurity lab for monitoring, detecting, and later responding to suspicious activity in OCPP-based EV charging infrastructure.

This project combines:

- an EVSE charging-station simulator,
- a SteVe OCPP Central System Management System (CSMS),
- Suricata network intrusion detection,
- Wazuh SIEM,
- custom OCPP-focused detection logic,
- packet captures and structured baselines,
- repeatable normal and malicious scenarios.

The goal is to build a realistic testbed where normal charging behavior can be documented first, then compared with suspicious or malicious OCPP activity using both network-level and SIEM-level telemetry.

---

## 1. Project Architecture

```text
                     VirtualBox Host-Only Network
                           192.168.100.0/24

┌───────────────────────────────┐
│ VM1 - EVSE Simulator          │
│ 192.168.100.10                │
│                               │
│ SAP e-Mobility Simulator      │
│ evse-cli                      │
└──────────────┬────────────────┘
               │
               │ OCPP 1.6J / WebSocket
               ▼
┌───────────────────────────────┐
│ VM2 - OCPP CSMS               │
│ 192.168.100.20                │
│                               │
│ SteVe                         │
│ Suricata IDS                  │
│ Custom OCPP rules             │
└──────────────┬────────────────┘
               │
               │ eve.json / alerts
               ▼
┌───────────────────────────────┐
│ VM3 - Wazuh SIEM              │
│ 192.168.100.30                │
│                               │
│ Wazuh Manager                 │
│ Wazuh Indexer                 │
│ Wazuh Dashboard               │
│ Filebeat                      │
└───────────────────────────────┘
```

---

## 2. Lab Network

| VM | Role | Host-only IP |
|---|---|---|
| VM1 | EVSE Simulator | `192.168.100.10` |
| VM2 | SteVe CSMS + Suricata | `192.168.100.20` |
| VM3 | Wazuh SIEM | `192.168.100.30` |

Each VM uses:

- `enp0s3` for NAT / Internet access
- `enp0s8` for the isolated EV charging lab network

The host-only interface does not need a default gateway.

Verify networking on each VM:

```bash
ip -br a
```

Expected lab addresses:

```text
EVSE-SIM    192.168.100.10/24
OCPP-CSMS   192.168.100.20/24
WAZUH-SIEM  192.168.100.30/24
```

Basic connectivity tests:

```bash
ping -c 4 192.168.100.20
ping -c 4 192.168.100.30
```

---

## 3. Repository Structure

```text
Detection-Response-for-OCPP-Based-EV-Charging-Systems/
│
├── evse-sim/
│   └── EVSE simulator source and project files
│
├── ocpp-csms/
│   ├── steve/
│   ├── ocpp-captures/
│   ├── ocpp-logs/
│   ├── suricata-baselines/
│   └── ev-ocpp-threat-detection/
│
├── suricata/
│   └── rules/
│       └── ev-ocpp.rules
│
├── wazuh-siem/
│   ├── config/
│   ├── rules/
│   ├── evidence/
│   └── README.md
│
├── scenarios/
│   └── NORMAL-001.md
│
├── docs/
│   └── screenshots/
│
├── .gitignore
└── README.md
```

---

# 4. Recommended Lab Startup Order

Use this order whenever you start the full environment:

1. Start VM3 - Wazuh SIEM
2. Verify Wazuh services
3. Start VM2 - OCPP CSMS
4. Start SteVe
5. Start Suricata
6. Start VM1 - EVSE Simulator
7. Start the EVSE simulator application
8. Open the EVSE-to-CSMS connection
9. Generate OCPP traffic
10. Verify Suricata events
11. Verify Wazuh ingestion and alerts

---

# 5. VM3 - Wazuh SIEM

## Verify Wazuh services

```bash
sudo systemctl status wazuh-indexer --no-pager
sudo systemctl status wazuh-manager --no-pager
sudo systemctl status filebeat --no-pager
sudo systemctl status wazuh-dashboard --no-pager
```

Expected result for each component:

```text
Active: active (running)
```

## Start services if required

```bash
sudo systemctl start wazuh-indexer
sudo systemctl start wazuh-manager
sudo systemctl start filebeat
sudo systemctl start wazuh-dashboard
```

## Enable automatic startup

```bash
sudo systemctl enable wazuh-indexer
sudo systemctl enable wazuh-manager
sudo systemctl enable filebeat
sudo systemctl enable wazuh-dashboard
```

## Verify listening ports

```bash
sudo ss -lntp | grep -E '443|1514|1515|55000|9200'
```

## Wazuh Dashboard

```text
https://192.168.100.30
```

Credentials are intentionally not stored in this repository.

## Wazuh manager log

```bash
sudo tail -f /var/ossec/logs/ossec.log
```

## Wazuh resource checks

```bash
df -h /
free -h
```

---

# 6. Wazuh Storage Note

During the initial deployment, the Wazuh dashboard installation failed because the Ubuntu root logical volume filled during package extraction.

The virtual disk was already 50 GB, but Ubuntu was only using approximately 24 GB of the LVM volume.

Useful checks:

```bash
lsblk
sudo pvs
sudo vgs
sudo lvs
```

If free LVM space is available, the root logical volume can be expanded with:

```bash
sudo lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
```

Then verify:

```bash
df -h /
```

This lab expanded the usable root filesystem before completing the Wazuh deployment.

---

# 7. VM2 - Start SteVe

Connect to the CSMS VM:

```bash
ssh selim@192.168.100.20
```

Open the SteVe project:

```bash
cd ~/steve
```

Start the Docker Compose environment:

```bash
docker compose up -d
```

Check containers:

```bash
docker compose ps
```

Follow SteVe logs:

```bash
docker compose logs -f
```

Verify the service port:

```bash
sudo ss -lntp | grep 8180
```

SteVe management interface:

```text
http://192.168.100.20:8180/steve/manager
```

Stop SteVe:

```bash
docker compose down
```

---

# 8. VM2 - Start Suricata

Validate configuration first:

```bash
sudo suricata -T -c /etc/suricata/suricata.yaml
```

Update community rules when required:

```bash
sudo suricata-update
```

Restart Suricata:

```bash
sudo systemctl restart suricata
```

Check service status:

```bash
sudo systemctl status suricata --no-pager
```

Suricata monitors:

```text
enp0s8
```

and observes traffic between:

```text
192.168.100.10 <-> 192.168.100.20
```

---

# 9. Suricata Configuration

Main configuration file:

```text
/etc/suricata/suricata.yaml
```

Custom project rule:

```text
/etc/suricata/rules/ev-ocpp.rules
```

Repository copy:

```text
suricata/rules/ev-ocpp.rules
```

Inspect the active custom rule:

```bash
sudo cat /etc/suricata/rules/ev-ocpp.rules
```

After changing a rule:

```bash
sudo suricata -T -c /etc/suricata/suricata.yaml
sudo systemctl restart suricata
```

Never restart Suricata before the configuration test succeeds.

---

# 10. Suricata EVE JSON

Main event file:

```text
/var/log/suricata/eve.json
```

Watch all Suricata events:

```bash
sudo tail -F /var/log/suricata/eve.json
```

Watch only alerts:

```bash
sudo tail -F /var/log/suricata/eve.json | \
jq -c 'select(.event_type=="alert")'
```

Show compact alert information:

```bash
sudo tail -F /var/log/suricata/eve.json | \
jq -c 'select(.event_type=="alert") |
{
  timestamp,
  src_ip,
  src_port,
  dest_ip,
  dest_port,
  signature:.alert.signature,
  severity:.alert.severity
}'
```

Filter traffic between EVSE and CSMS:

```bash
sudo jq -c '
select(
  (.src_ip=="192.168.100.10" and .dest_ip=="192.168.100.20") or
  (.src_ip=="192.168.100.20" and .dest_ip=="192.168.100.10")
)
| {
  timestamp,
  event_type,
  src_ip,
  src_port,
  dest_ip,
  dest_port,
  proto,
  app_proto
}' /var/log/suricata/eve.json | tail -20
```

---

# 11. Trigger the Custom Suricata Rule

First inspect the exact rule:

```bash
sudo cat /etc/suricata/rules/ev-ocpp.rules
```

Then open an alert-monitoring terminal on VM2:

```bash
sudo tail -F /var/log/suricata/eve.json | \
jq -c 'select(.event_type=="alert") |
{
  timestamp,
  src_ip,
  dest_ip,
  signature:.alert.signature
}'
```

On VM1, generate the OCPP event matched by the rule.

For the current Heartbeat test used in the lab:

```bash
evse-cli ocpp heartbeat
```

If the rule matches the Heartbeat traffic, an alert should appear in `eve.json`.

If no alert appears:

```bash
sudo cat /etc/suricata/rules/ev-ocpp.rules
sudo suricata -T -c /etc/suricata/suricata.yaml
sudo systemctl status suricata --no-pager
```

Also confirm traffic exists:

```bash
sudo tcpdump -i enp0s8 host 192.168.100.10
```

---

# 12. VM1 - Start the EVSE Simulator

Connect to VM1:

```bash
ssh selim@192.168.100.10
```

Open the simulator project:

```bash
cd ~/e-mobility-charging-stations-simulator
```

Start the simulator using the project command configured in the SAP e-Mobility simulator repository.

For the current lab environment:

```bash
pnpm start
```

Use another terminal connected to VM1 for `evse-cli`.

---

# 13. EVSE Connection and Basic OCPP Traffic

Open the connection:

```bash
evse-cli connection open
```

Send a Heartbeat:

```bash
evse-cli ocpp heartbeat
```

Authorize an ID tag:

```bash
evse-cli ocpp authorize --id-tag NORMAL001
```

---

# 14. Normal OCPP Charging Scenario

The normal baseline uses:

```text
NORMAL001
```

Sequence:

```text
Authorize
   ↓
StartTransaction
   ↓
MeterValues
   ↓
MeterValues
   ↓
MeterValues
   ↓
StopTransaction
```

Authorize:

```bash
evse-cli ocpp authorize --id-tag NORMAL001
```

Start a transaction:

```bash
evse-cli transaction start \
  --connector-id 1 \
  --id-tag NORMAL001
```

Send meter values:

```bash
evse-cli ocpp meter-values --connector-id 1
```

Repeat meter values as required to simulate an active charging session.

Stop the transaction using the transaction ID returned by the CSMS:

```bash
evse-cli transaction stop \
  --transaction-id <TRANSACTION_ID> \
  --connector-id 1
```

Verify the transaction in:

```text
SteVe
Data Management -> Transactions
```

---

# 15. NORMAL-001 Baseline

Documentation:

```text
scenarios/NORMAL-001.md
```

Purpose:

- establish legitimate OCPP behavior,
- document expected charging traffic,
- create repeatable evidence,
- compare later suspicious activity against a known baseline,
- support rule-based and machine-learning detection experiments.

Expected behavior:

- authorization accepted,
- transaction created successfully,
- meter values transmitted,
- transaction stopped normally,
- no unexpected IDS alerts.

---

# 16. Packet Capture

Capture EVSE traffic on VM2:

```bash
sudo tcpdump -i enp0s8 \
  host 192.168.100.10 \
  -w ~/ocpp-captures/ocpp-session.pcap
```

Stop capture:

```text
Ctrl+C
```

Example project evidence already includes:

```text
evse001-first-connection.pcap
evse001-normal-transaction-01.txt
```

Large PCAP files should normally remain outside Git unless intentionally curated.

---

# 17. Useful SteVe Commands

Open project:

```bash
cd ~/steve
```

Start:

```bash
docker compose up -d
```

Status:

```bash
docker compose ps
```

Logs:

```bash
docker compose logs -f
```

Stop:

```bash
docker compose down
```

Port check:

```bash
sudo ss -lntp | grep 8180
```

---

# 18. Useful Suricata Commands

Version:

```bash
suricata -V
```

Configuration validation:

```bash
sudo suricata -T -c /etc/suricata/suricata.yaml
```

Status:

```bash
sudo systemctl status suricata --no-pager
```

Restart:

```bash
sudo systemctl restart suricata
```

Rules:

```bash
ls -lh /var/lib/suricata/rules/
```

Custom rule:

```bash
sudo cat /etc/suricata/rules/ev-ocpp.rules
```

Recent alerts:

```bash
sudo jq -c \
'select(.event_type=="alert") |
{timestamp,src_ip,dest_ip,alert}' \
/var/log/suricata/eve.json | tail -20
```

---

# 19. Useful Wazuh Commands

Status:

```bash
sudo systemctl status wazuh-manager --no-pager
sudo systemctl status wazuh-indexer --no-pager
sudo systemctl status filebeat --no-pager
sudo systemctl status wazuh-dashboard --no-pager
```

Restart manager:

```bash
sudo systemctl restart wazuh-manager
```

Restart indexer:

```bash
sudo systemctl restart wazuh-indexer
```

Restart Filebeat:

```bash
sudo systemctl restart filebeat
```

Restart dashboard:

```bash
sudo systemctl restart wazuh-dashboard
```

Manager log:

```bash
sudo tail -f /var/ossec/logs/ossec.log
```

Disk:

```bash
df -h /
```

Memory:

```bash
free -h
```

Listening ports:

```bash
sudo ss -lntp | grep -E '443|1514|1515|55000|9200'
```

---

# 20. Evidence Collection

Recommended evidence for each project milestone:

- terminal output,
- clean screenshots,
- relevant configuration excerpts,
- custom rules,
- compact JSON evidence,
- selected logs,
- normal baseline documentation,
- small curated packet captures,
- Git commits.

Suggested screenshot naming:

```text
01-evse-network.png
02-steve-connected.png
03-ocpp-heartbeat.png
04-normal-transaction.png
05-suricata-running.png
06-suricata-custom-alert.png
07-wazuh-dashboard.png
08-wazuh-suricata-alert.png
```

---

# 21. Security and Repository Hygiene

Do not commit:

```text
Passwords
API keys
Private keys
Certificates
Wazuh installation password archives
Keystores
Environment files containing secrets
Database credentials
Large generated logs
Wazuh index data
node_modules
build artifacts
```

Never commit files such as:

```text
wazuh-install-files.tar
*.pem
*.key
*.p12
*.pfx
*.jks
client.keys
.env
```

Recommended `.gitignore` entries:

```gitignore
# Secrets
*.pem
*.key
*.p12
*.pfx
*.jks
*keystore*
*credentials*
*password*
*secret*

# Wazuh sensitive files
wazuh-install-files.tar
client.keys

# Environment files
.env
.env.*

# Dependencies
node_modules/

# Build output
target/
build/
dist/

# Logs
*.log

# Wazuh generated data
**/wazuh-indexer/data/
**/var/lib/wazuh-indexer/
**/var/ossec/logs/
```

Before committing:

```bash
git status
```

Inspect staged files:

```bash
git diff --cached --name-only
```

Search for obvious accidental secrets:

```bash
grep -RniE \
'password|passwd|secret|token|api[_-]?key|private[_-]?key|credential' .
```

Review every match before pushing.

---

# 22. Git Workflow

Check status:

```bash
git status
```

Stage:

```bash
git add .
```

Review staged files:

```bash
git diff --cached --name-only
```

Commit:

```bash
git commit -m "Describe the completed project milestone"
```

Push:

```bash
git push origin main
```

---

# 23. Current Project Progress

Completed:

- Three-VM isolated laboratory network
- EVSE simulator deployed
- SteVe OCPP CSMS deployed
- EVSE-to-CSMS connectivity validated
- OCPP Heartbeat validated
- Authorization tested
- OCPP transaction workflow tested
- Normal OCPP baseline created
- Packet capture workflow created
- Suricata installed on the CSMS
- Suricata configured on `enp0s8`
- EVE JSON logging enabled
- OCPP traffic observed by Suricata
- Custom Suricata rule created
- Wazuh SIEM deployed
- GitHub repository created for the complete project

Current phase:

- Trigger and validate the custom Suricata rule
- Preserve alert evidence
- Forward Suricata events to Wazuh
- Build Wazuh detection and visualization
- Add additional attack scenarios
- Compare malicious activity against the NORMAL-001 baseline

---

# 24. Detection Roadmap

Planned scenarios include:

- network reconnaissance,
- authentication abuse,
- charging-station identity anomalies,
- abnormal OCPP message frequency,
- OCPP flooding,
- unexpected OCPP message sequences,
- availability attacks,
- resource-exhaustion behavior,
- network anomaly detection,
- host/SIEM correlation,
- multi-source attack detection,
- automated response experiments.

---

# 25. Project Goal

The final platform combines:

```text
EV charging simulation
        +
OCPP protocol behavior
        +
Network intrusion detection
        +
Centralized SIEM monitoring
        +
Custom detection logic
        +
Reproducible scenarios
        +
Baseline comparison
        =
EV charging cybersecurity detection and response testbed
```

The project is designed to evolve from a basic working OCPP environment into a reproducible multi-source cyberattack detection and response platform for EV charging infrastructure.

---

# 26. Quick Full-Lab Startup Checklist

## VM3 - Wazuh

```bash
sudo systemctl start wazuh-indexer
sudo systemctl start wazuh-manager
sudo systemctl start filebeat
sudo systemctl start wazuh-dashboard
```

Verify:

```bash
sudo systemctl status wazuh-manager --no-pager
```

## VM2 - SteVe

```bash
cd ~/steve
docker compose up -d
docker compose ps
```

## VM2 - Suricata

```bash
sudo suricata -T -c /etc/suricata/suricata.yaml
sudo systemctl restart suricata
sudo systemctl status suricata --no-pager
```

## VM1 - EVSE Simulator

```bash
cd ~/e-mobility-charging-stations-simulator
pnpm start
```

In another VM1 terminal:

```bash
evse-cli connection open
evse-cli ocpp heartbeat
```

## VM2 - Watch Alerts

```bash
sudo tail -F /var/log/suricata/eve.json | \
jq -c 'select(.event_type=="alert") |
{
  timestamp,
  src_ip,
  dest_ip,
  signature:.alert.signature
}'
```

At this point the complete monitoring lab is running.

---

## Disclaimer

This repository is intended for controlled cybersecurity research and educational testing in an isolated laboratory environment.
