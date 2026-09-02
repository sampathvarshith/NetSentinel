# NetSentinel

A lightweight, responsive network operations center (NOC) dashboard and security event monitoring platform.

---

## 1. Project Overview

### Problem Statement
Managing network availability and security in local or edge environments is often complex, requiring heavy server resources and complex agent configurations. Small-to-medium offices or local labs lack lightweight, zero-dependency options to monitor device reachability, track historical response times, detect outages, send Slack/SMTP alerts, and maintain secure, credentials-masked admin logs without exposing credentials.

### Solution
NetSentinel provides a lightweight, secure network monitoring platform running on SQLite (WAL mode) and FastAPI. It schedules periodic, non-shell ICMP checks, stores device metrics, evaluates threat rules (e.g., packet loss spikes), dispatches email/webhook alerts with cooldown limits to prevent storming, and records a recursive audit log. An intuitive, single-page NOC dashboard visualizes telemetry charts and facilitates incident response.

---

## 2. Key Capabilities

* **Subnet Scanner**: Multi-threaded CIDR discovery verifying IP status, hostnames, and MACs safely.
* **Persistent Inventory**: Identity-aware device tracking utilizing MAC addresses and fallback IPs to handle dhcp changes.
* **Continuous Monitoring**: An asynchronous background thread polling device status, latency, and packet loss.
* **Defensive Event Detection**: Rule evaluators generating alerts for outages, latency spikes, and packet loss.
* **SMTP & Webhook Alerts**: Dynamic alert dispatch with built-in sliding window cooldowns to prevent spam.
* **Production Hardening**: Enforces SQLite WAL mode, IP-based rate limiting, system health diagnostics, and recursive JSON audit log redaction.

---

## 3. Architecture & Tech Stack

```
                 ┌─────────────────┐
                 │  Web Dashboard  │
                 │  (HTML5/CSS/JS) │
                 └────────┬────────┘
                          │
                    HTTP REST API
                          │
        ┌─────────────────┴─────────────────┐
        │                                   │
 Device Inventory                     Security Events
 (Inventory API)                      (Security API)
        │                                   │
 Monitoring Engine                  Detection Engine
 (Worker Loop)                        (Rule Evaluator)
        │                                   │
 Metrics Repository                 Alerting Engine
 (DeviceMetric ORM)                 (SMTP/Webhooks)
        │                                   │
        └──────────────┬────────────────────┘
                       │
               SQLAlchemy ORM
                       │
                 SQLite (WAL)
```

### Technology Stack
* **Backend**: FastAPI (Python 3.12+), SQLAlchemy 2.0 ORM, SQLite.
* **Frontend**: HTML5, Vanilla CSS3 (NOC Dark Theme), Modern JavaScript, Chart.js.
* **Deployment**: Docker, Docker Compose.
* **Testing**: Pytest & HTTPX.

---

## 4. Feature Breakdown

### Security & Hardening Features
* **Rate Limiting**: Tiered Token-Bucket sliding window limiters (e.g., 5 RPM for system and security settings, 100 RPM default). Internal daemon workers bypass limits to avoid self-blocking.
* **Audit Logs**: State-changing API operations write to `logs/audit.log` recursively masking sensitive keys (`password`, `token`, `secret`, `webhook_url`, etc.).
* **Non-Shell Execution**: Pings are executed without system shell parsing (`shell=False`) to eliminate command-injection risks.
* **Safe Diagnostics**: Hardware resource endpoints verify system diagnostics without leaking filesystems or configuration paths.

### Monitoring & Telemetry Features
* **Asynchronous Polling**: A daemon worker polls latency (ms) and packet loss (%) periodically.
* **Time-Series metrics**: Recorded telemetry is stored in the database for analytics.

### Threat Detection & Alerting
* **Outage Trigger**: Generates security events upon complete outage (`DEVICE_UNREACHABLE`).
* **Auto-Resolution**: Spawns recovery alerts and automatically closes open incidents once devices return online.
* **Alert Storm Suppression**: Sliding window cooldown limits alerts for the same event types.

---

## 5. Getting Started & Local Runs

### Prerequisites
* Python 3.11+
* Git

### Step-by-Step Installation
1. **Clone the repository**:
   ```bash
   git clone <repo-url> NetSentinel
   cd NetSentinel
   ```
2. **Setup virtual environment**:
   ```bash
   python -m venv .venv
   source .venv/bin/activate  # On Windows: .\.venv\Scripts\activate
   ```
3. **Install dependencies**:
   ```bash
   pip install -r requirements.txt
   ```
4. **Configure environment parameters**:
   ```bash
   cp .env.example .env
   ```

### Seeding Mock Demonstration Data
Populate the database with historical latency records, mock devices, events, alerts, and audit entries:
```bash
python scripts/seed_security.py
```

### Run Server Locally
Start the backend ASGI worker:
```bash
python -m uvicorn backend.app.main:app --host 127.0.0.1 --port 8000
```
Open your browser and navigate to:
* **Web Dashboard**: `http://127.0.0.1:8000`
* **API Documentation**: `http://127.0.0.1:8000/docs`

---

## 6. Docker Container Deployment

To launch NetSentinel in production with database persistence:
```bash
docker-compose up -d --build
```
This starts the service container, maps port `8000`, and attaches persistent volumes for data (`/app/data`) and logs (`/app/logs`).

---

## 7. Running Automated Tests

Run the full suite using pytest:
```bash
pytest -v
```
All tests execute using deterministic mocks and run against an isolated in-memory SQLite backend.

---

## 8. API Overview

* `GET /health` - Root application health.
* `GET /api/v1/system/diagnostics` - CPU, RAM, Disk telemetry and SQLite WAL mode status.
* `GET /api/v1/system/audit-logs` - Paginated admin audit trail.
* `POST /api/v1/network/scan` - Trigger subnet scans.
* `GET /api/v1/devices` - Retrieve inventory devices.
* `GET /api/v1/security/events` - Retrieve security events.
* `POST /api/v1/security/events/{id}/acknowledge` - Acknowledge incidents.
* `POST /api/v1/security/events/{id}/resolve` - Resolve incidents.

---

## 9. Project Structure

```
NetSentinel/
├── backend/
│   └── app/
│       ├── api/          # FastAPI routers and endpoints
│       ├── core/         # Audit, config, rate limiter, security helper logic
│       ├── db/           # SQLAlchemy models and session configuration
│       └── network/      # Monitoring worker thread and discovery scanner
├── docs/                 # Architecture, CV items, interview and backup docs
├── frontend/             # Single-page dashboard client code
├── logs/                 # Active audit logs directory (git ignored)
├── scripts/              # Seed scripts
├── tests/                # Automated Pytest suite
├── Dockerfile            # Container build spec
└── docker-compose.yml    # Local container orchestration
```

---

## 10. Limitations & Future Roadmap

### Limitations
* **SQLite Scope**: Ideal for small-to-medium local subnet monitoring; high-scale enterprise environments with thousands of concurrent devices may require migrating to PostgreSQL.
* **In-Memory Cache**: Outbound alert suppression cache is currently local in-memory; scaling across multiple app containers would require a shared Redis-like cache.

### Future Roadmap
* **IPv6 Scanning Support**: Add IPv6 subnet parsing and reachability checks.
* **Role-Based Access Control (RBAC)**: Support multiple administrator/operator accounts with distinct access permissions.
* **SNMP Telemetry Integration**: Extend network metrics collection beyond ICMP checks.
