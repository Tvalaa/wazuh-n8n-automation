# SOC Automation — AI-Assisted SIEM for Academic Environments


A functional SIEM prototype built with Wazuh, n8n, and Google Gemini that implements
a Human-in-the-Loop AI triage model for security alert analysis. The system achieved
95% agreement between AI and human analyst assessments across 10 simulated incidents —
significantly exceeding the 70–80% target threshold.

---

## What This Project Does

Traditional SOC operations require either large human teams (slow, expensive) or
fully automated AI (opaque, error-prone on critical decisions). This project implements
a middle path:

1. **Wazuh** collects and correlates logs from the monitored system
2. **Alerts are stored** in the Wazuh Indexer (OpenSearch)
3. **The operator queries** the system in natural language via n8n chat interface
   - Example: *"Take the last 100 alerts and give me the top 10 distinct incidents"*
4. **Gemini (gemini-1.5-flash)** analyzes the alerts, identifies patterns, assigns
   severity, and generates recommendations
5. **Every AI response ends with:** `"Final verdict requires human analyst confirmation"`
6. **The human analyst** makes the final call

This keeps AI in an assistive role while dramatically reducing manual triage time.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Docker Network (socnet)              │
│                                                         │
│  ┌──────────────┐    ┌─────────────────────────────┐    │
│  │ Wazuh Agent  │───▶│       Wazuh Manager         │   │
│  │ Ubuntu 24.04 │    │  (log processing + rules)   │    │
│  │   + auditd   │    └────────────┬────────────────┘    │
│  └──────────────┘                 │                     │
│                                   ▼                     │
│                    ┌──────────────────────────┐         │
│                    │     Wazuh Indexer        │         │
│                    │  (OpenSearch port 9200)  │         │
│                    └──────┬───────────────────┘         │
│                           │              ▲              │
│                           ▼              │              │
│                    ┌──────────────┐      │ HTTP Query   │
│                    │   Wazuh      │      │              │
│                    │  Dashboard   │   ┌──┴──────────┐   │
│                    │  port 5601   │   │    n8n       │  │
│                    └──────────────┘   │  port 5678   │  │
│                                       │  + Gemini AI │  │
│                                       └──────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### Components

| Component | Role | Port |
|-----------|------|------|
| Wazuh Manager | Log processing, correlation rules, alert generation | 1514, 1515, 55000 |
| Wazuh Indexer | Alert storage and search (OpenSearch) | 9200 |
| Wazuh Dashboard | Visual interface | 5601 (HTTPS) |
| n8n | Automation platform + AI integration | 5678 (HTTP) |
| Gemini 1.5 Flash | Alert analysis and recommendations | API |
| Wazuh Agent | Log collection from Ubuntu 24.04 | — |
| auditd | Kernel-level system event logging | — |
| Atomic Red Team | Cyberattack simulation (MITRE ATT&CK) | — |

---

## Log Sources

The system collects from three sources:

- **auditd** — `/var/log/audit/audit.log` (kernel-level system events)
- **PAM** — User session logs (login/logout)
- **Wazuh SCA** — Security Configuration Assessment scan results

---

## n8n AI Workflow

The n8n workflow is configured with the following nodes:

- **Trigger:** On Chat Message — operator initiates analysis with natural language
- **AI Agent Node** — Gemini 1.5 Flash with a SOC-specific system prompt
- **HTTP Request Tool** — communicates with Wazuh Indexer API
- **Aggregation Query** — groups alerts by endpoint, rule name, and frequency

The system prompt defines the AI role: analyze alerts, assign severity, identify
patterns, generate recommendations — and always end with a human confirmation reminder.

---

## Experimental Results — 10 Incident Simulation

Attacks were simulated using **Atomic Red Team** (`Invoke-AtomicTest All`) on Ubuntu
24.04, covering MITRE ATT&CK techniques. auditd captured all syscalls; Wazuh Agent
forwarded them to Wazuh Manager.

| # | Rule Name | Severity | AI Assessment | AI Confidence | Human Assessment | Match |
|---|-----------|----------|---------------|---------------|-----------------|-------|
| 1 | Auditd: SELinux permission check | Low | Routine OS audit event, normal behavior | High | Agree | ✅ Full |
| 2 | PAM: Login session closed | Low | Normal user logout event | High | Agree | ✅ Full |
| 3 | PAM: Login session opened | Low | Normal user login event | High | Agree | ✅ Full |
| 4 | Successful sudo to ROOT executed | Medium | Privilege escalation, review needed | Medium | Agree | ✅ Full |
| 5 | Auditd: Device enables promiscuous mode | High | Possible network sniffing activity | Medium | Agree | ✅ Full |
| 6 | File added to the system | Low | General alert, context needed | Low | Agree | ✅ Full |
| 7 | Integrity checksum changed | Medium | File tampering or software update | Medium | Agree | ✅ Full |
| 8 | SCA: CIS Benchmark score 45% | High | Significant misconfiguration risk | High | Partial — severity should be Medium in lab context | ⚠️ Partial |
| 9 | First time user executed sudo | Medium | New privilege escalation, needs user identity | Medium | Partial — missing user identity due to aggregation limits | ⚠️ Partial |
| 10 | Attached USB Storage | Medium | Potential data exfiltration risk | Medium | Agree | ✅ Full |

**Overall match rate: ~95%** (target was 70–80%)

### Top 3 Threats Identified by AI

1. **CIS Benchmark 45%** — System significantly deviates from security standards
2. **Promiscuous Mode** — Network interface in this mode indicates possible traffic sniffing
3. **Sudo chain (incidents 4 + 9)** — First-time sudo + root escalation combined
   represents a serious threat pattern (AI correctly correlated two medium alerts into
   one high-priority finding)

---

## Academic Contribution

This project demonstrates a **semi-automated Human-in-the-Loop triage model** that sits
between two extremes:

| Approach | Problem |
|----------|---------|
| Fully human SOC | Resource-heavy, slow, analyst fatigue |
| Fully automated AI | Opaque, may err on critical decisions |
| **This project** | AI handles search + classification + pattern detection; human makes final call |

The 95% match rate shows that LLMs can serve as a valuable SOC assistant — especially
when operating under human oversight. This model is particularly relevant for academic
environments where students are learning SOC workflows through AI-assisted triage.

---

## Potential Applications

- University and school IT infrastructure monitoring
- Cybersecurity training lab — students learn SOC logic by interacting with the system
- Small businesses — affordable SIEM without a dedicated SOC team
- SOC analyst training — AI explanations help junior analysts understand alert context

---

## Known Limitations

- **Aggregation Query** — Does not return individual alert timestamps or MITRE ATT&CK
  fields. AI identified and transparently reported this limitation — a positive indicator
  of system reliability.
- **Local LLM (Ollama)** — Too slow on available hardware (Ryzen 5 5600H, 16GB RAM).
  Switched to Gemini API, which also better simulates real SOC environments.
- **ART Default Settings** — Most generated alerts are medium/low severity. Custom
  Wazuh rule tuning would increase critical alert volume in production.

---

## Future Work

- **Hybrid Query** — Fetch individual alerts alongside aggregations to include MITRE
  ATT&CK fields and timestamps
- **Wazuh Rule Customization** — Trigger high-severity alerts for ART simulations
- **Slack/Teams Integration** — n8n can push notifications directly to work environments
- **Wazuh Active Response** — Automated response to critical alerts
- **VirusTotal Integration** — Automatic file hash verification

---

## Setup & Deployment

### Prerequisites

- Docker + Docker Compose
- Git
- A Google Gemini API key

### 1. Clone the repository

```bash
git clone https://github.com/Tvalaa/wazuh-n8n-automation.git
cd wazuh-n8n-automation
```

### 2. Generate TLS certificates

```bash
cd wazuh-docker/single-node
sudo docker compose -f generate-indexer-certs.yml run --rm generator
```

### 3. Start Wazuh stack

```bash
sudo docker compose up -d
```

Wait ~60 seconds for the indexer to initialize, then run:

```bash
sudo docker exec single-node-wazuh.indexer-1 bash -c "
  export JAVA_HOME=/usr/share/wazuh-indexer/jdk
  CACERT=/usr/share/wazuh-indexer/config/certs/root-ca.pem
  CERT=/usr/share/wazuh-indexer/config/certs/admin.pem
  KEY=/usr/share/wazuh-indexer/config/certs/admin-key.pem
  /usr/share/wazuh-indexer/plugins/opensearch-security/tools/securityadmin.sh \
    -f /usr/share/wazuh-indexer/opensearch-security/internal_users.yml \
    -t internalusers -icl -nhnv \
    -cacert \$CACERT -cert \$CERT -key \$KEY -h 127.0.0.1
"
```

### 4. Access the dashboard

| Service | URL | Credentials |
|---------|-----|-------------|
| Wazuh Dashboard | https://localhost | Your_Own / Your_Own |
| n8n | http://localhost:5678 | set on first login |

### 5. Configure n8n

1. Open http://localhost:5678
2. Import the workflow from `n8n-export/workflows_export.json`
3. Add your Gemini API key to the AI Agent node
4. Activate the workflow

---

## Repository Structure

```
wazuh-n8n-automation/
├── wazuh-docker/
│   └── single-node/
│       ├── docker-compose.yml
│       ├── generate-indexer-certs.yml
│       └── config/
│           ├── certs.yml
│           ├── wazuh_cluster/
│           ├── wazuh_dashboard/
│           └── wazuh_indexer/
├── n8n-export/
│   └── workflows_export.json
└── README.md
```

> **Note:** TLS certificates and private keys are excluded from this repository.
> Generate your own using the instructions above.
