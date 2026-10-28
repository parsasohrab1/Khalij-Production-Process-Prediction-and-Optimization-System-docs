# Khalij-Production-Process-Prediction-and-Optimization-System
For the product **"Predictive and Optimizing System for Production Processes"** in petrochemicals, a comprehensive **SRS (Software Requirements Specification)** document is provided, followed by code that generates **10,000 synthetic (simulated) data points per variable** with per-second timing.

---

## 🏗️ Implemented Infrastructure

Based on the technical architecture in section 5 of this document, the complete project infrastructure is implemented as a set of FastAPI microservices together with all required data services (InfluxDB, PostgreSQL, Redis, Kafka, MLflow) via Docker Compose.

### Architecture map

```
                         ┌──────────────────┐
                         │   API Gateway     │  (JWT + 2FA - NFR-SEC-01)
                         │  (FastAPI, :8000)  │
                         └─────────┬─────────┘
                                   │
        ┌───────────────┬─────────┼─────────────┬───────────────┐
        │               │                       │               │
┌───────▼───────┐┌───────▼────────┐   ┌──────────▼───────┐┌──────▼──────┐
│Data Ingestion ││ Preprocessing  │   │    Prediction     ││  Alerting   │
│ OPC-UA/Modbus ││ Outlier/NaN/   │   │ LSTM/XGBoost +     ││ Telegram/   │
│ :8001         ││ Normalize:8002 │   │ Recommender :8003  ││ Email :8004 │
└──────┬────────┘└───────┬────────┘   └─────────┬─────────┘└──────┬──────┘
       │                 │                       │                │
       └────────────┬────┴───────────┬───────────┘                │
                     ▼                ▼                            ▼
              ┌────────────┐  ┌───────────────┐             ┌────────────┐
              │   Kafka    │  │   InfluxDB    │             │ PostgreSQL │
              │ (Streaming)│  │ (Time-Series) │             │(Transaction│
              └────────────┘  └───────────────┘             └────────────┘
                     │                                              │
              ┌────────────┐                               ┌────────────┐
              │   Redis    │                               │   MLflow   │
              │  (Cache)   │                               │  (MLOps)   │
              └────────────┘                               └────────────┘
```

Data flow: `Data Ingestion` sends one sensor record per second (simulated or from OPC-UA/Modbus) to Kafka (`raw-sensor-data`) and raw InfluxDB → `Preprocessing` consumes it, corrects outliers, interpolates missing values, normalizes, and publishes to `processed-sensor-data` → `Prediction` forecasts the temperature 60 seconds ahead, sends an alert to `process-alerts` on critical deviation, and records the optimal feed-flow recommendation in PostgreSQL → `Alerting` sends alerts via Telegram/email, logs them in PostgreSQL, and caches them in Redis for display in the control room.

### Folder Structure

| Path | Description |
| :--- | :--- |
| `shared/` | Pydantic models, shared settings (`.env`), Kafka utilities |
| `services/api-gateway/` | Single API gateway, two-factor authentication (NFR-SEC-01), `/dashboard` dashboard (R-GEN-03) |
| `services/data-ingestion/` | OPC-UA/Modbus connection, local buffer during outages, Circuit Breaker (FR-DATA-01..03) |
| `services/preprocessing/` | Outlier removal, interpolation, normalization (FR-PRE-01..03) |
| `services/prediction/` | Temperature prediction with XGBoost/LSTM + evolutionary recommender (FR-ML-01..03) |
| `services/alerting/` | Telegram/email, alert log, automatic shift report (FR-ALERT + R-GEN-04) |
| `infra/postgres/init.sql` | Initial PostgreSQL schema (operators, sensors, alerts, shift report) |
| `infra/mlflow/` | MLflow server Dockerfile |
| `infra/nginx/` | Reverse proxy with TLS 1.3 (NFR-SEC-02) |
| `shared/schema_registry.py` | Lightweight Schema Registry (phase 2) |
| `shared/dlq.py` | Dead Letter Queue publisher (phase 2) |
| `shared/data_quality.py` | Data quality Expectation Suite (phase 2) |
| `infra/influxdb/` | Init bucket cold + Task downsampling |
| `data/generate_synthetic_data.py` | Generate 10,000 synthetic records for each variable |
| `scripts/test_phase1.py` … `test_phase10.py` | Unit tests for phases 1 to 10 |
| `config/pilot.yaml` / `docs/pilot/` | Pilot configuration and Runbook (phase 10) |
| `docker-compose.pilot.yml` | Pilot Compose profile |
| `infra/k8s/` | Kubernetes manifests + dev/pilot overlays |
| `docker-compose.yml` | Full infrastructure (+ `hardware-sim`, `kafka-init`, `influxdb-init`) |

### Quick Start

```bash
# 1. Copy the sample environment variables file and edit the sensitive values
cp .env.example .env

# 2. Bring up the full infrastructure
docker compose up --build -d

# 3. Check service health
curl http://localhost:8000/health/services
```

The `data-ingestion` service works by default with a built-in simulator (`DATA_SOURCE_MODE=simulator`) so that the whole pipeline (Ingestion → Kafka → Preprocessing → Prediction → Alerting) can be tested without real OPC-UA/Modbus hardware.

To log in to the dashboard (demo operator, default password `ChangeMe123!`):

```bash
curl -X POST http://localhost:8000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"employee_code": "demo-admin", "password": "ChangeMe123!"}'
# The response includes pending_token; generate the OTP code with a tool such as Google Authenticator
# from the Base32 secret in the operators table (otp_secret column).
curl -X POST http://localhost:8000/auth/verify-otp \
  -H "Content-Type: application/json" \
  -d '{"pending_token": "<pending_token>", "otp_code": "<6-digit code>"}'
```

⚠️ The demo operator's password and OTP secret are for the development environment only; remove/change them before operational deployment.

### Generating synthetic data for testing/model training

```bash
pip install -r data/requirements.txt
python data/generate_synthetic_data.py --num-records 10000
```

### Key environment variables

The `.env.example` file includes all required variables (InfluxDB/PostgreSQL/Redis/Kafka/MLflow connections, alert thresholds, Telegram/SMTP settings, JWT key and AES-256 key) with explanations.

### Monitoring dashboard (R-GEN-03 / phase 5)

After bringing up the services:

- HTTP: [http://localhost:8000/dashboard](http://localhost:8000/dashboard)
- HTTPS with TLS 1.3: [https://localhost/dashboard](https://localhost/dashboard)

Log in with the demo operator + OTP (NFR-SEC-01). The dashboard includes live monitoring tabs (WebSocket), control room, optimization, shift report (PDF/Excel) and QR registration for 2FA.

### Hardware connection test (phase 1 / FR-DATA-01)

```bash
# Bring up the OPC-UA/Modbus simulator + the other services
docker compose --profile hardware-sim up --build -d

# Switch data-ingestion to OPC-UA (in .env)
# DATA_SOURCE_MODE=opcua
docker compose up -d data-ingestion

# Or unit tests of phase 1 without Docker
pip install cryptography PyYAML
python scripts/test_phase1.py
```

### Phase 2 — Data pipeline and data quality

| Capability | Implementation |
| :--- | :--- |
| Schema Registry | `shared/schema_registry.py` — Pydantic validation + versioning |
| Dead Letter Queue | `raw-sensor-data-dlq` and `processed-sensor-data-dlq` topics |
| Retention / Downsampling | hot bucket=`30d` + cold=`process_sensors_1m` with 5-year retention; Task every 1 minute |
| Data Quality | `shared/data_quality.py` + CLI in `services/data-quality/cli.py` |

```bash
# Phase 2 unit test
python scripts/test_phase2.py

# Run the Expectation Suite on synthetic CSV
python data/generate_synthetic_data.py --num-records 1000 --output data/sample.csv
pip install -r services/data-quality/requirements.txt
python -m services.data_quality.cli --csv data/sample.csv --limit 1000

# Quality statistics from the API (after login)
curl -H "Authorization: Bearer <token>" http://localhost:8000/quality/stats
```

### Phase 3 — Real modeling and MLOps

| Capability | Implementation |
| :--- | :--- |
| Batch training + backtesting | `training_pipeline.py` — temporal split, MAE/RMSE/MAPE |
| Model comparison | XGBoost vs MLP (and LSTM if TensorFlow is installed) |
| MLflow Registry | Experiment + Staging → Production |
| Drift Detection | PSI on reactor temperature → automatic Re-training |

```bash
# Phase 3 unit test
python scripts/test_phase3.py

# Offline training from CSV
python data/generate_synthetic_data.py --num-records 5000 --output data/train.csv
python scripts/train_model.py --csv data/train.csv --mlflow-uri http://localhost:5000

# APIs
curl -X POST http://localhost:8003/train
curl http://localhost:8003/models/comparison
curl -X POST http://localhost:8003/drift/check
```

### Phase 4 — Advanced optimization and safety constraints

| Capability | Implementation |
| :--- | :--- |
| Multivariable optimization | Feed flow + pressure simultaneously (`MultiVariableEvolutionaryOptimizer`) |
| Hard Constraints | Safe range, rate of change, combined P+T constraint (`safety_constraints.py`) |
| What-if | Simulation before applying (`/optimize/whatif`) |
| Closed loop | Apply recommendation → record actual efficiency → prediction error |

```bash
python scripts/test_phase4.py

# Multivariable optimization
curl -X POST http://localhost:8003/optimize \
  -H "Content-Type: application/json" \
  -d '{"reactor_temp_c":250,"pressure_bar":28,"feed_flow_m3h":380}'

# What-if
curl -X POST http://localhost:8003/optimize/whatif \
  -H "Content-Type: application/json" \
  -d '{"reactor_temp_c":250,"current_pressure_bar":28,"current_feed_flow":380,"proposed_pressure_bar":25,"proposed_feed_flow":320}'

# Apply + feedback
curl -X POST http://localhost:8003/optimize/apply -H "Content-Type: application/json" \
  -d '{"recommendation_id":1,"operator_id":1}'
curl -X POST http://localhost:8003/optimize/feedback -H "Content-Type: application/json" \
  -d '{"recommendation_id":1,"actual_efficiency":72.5}'
```

### Phase 5 — Dashboard and operator experience

| Capability | Implementation |
| :--- | :--- |
| Live WebSocket | `GET/WS /ws/live?token=...` — sensor, prediction, recommendation, alert |
| Multi-page dashboard | Monitoring / control room / optimization / report / 2FA security |
| TOTP registration with QR | `POST /auth/totp/setup` + Base64 PNG image |
| Shift report export | `GET /reports/shift/export?format=pdf\|xlsx` (R-GEN-04) |

```bash
# Phase 5 unit test
pip install qrcode[pil] openpyxl reportlab pyotp python-jose
python scripts/test_phase5.py

# Dashboard
# http://localhost:8000/dashboard

# WebSocket (after login)
# ws://localhost:8000/ws/live?token=<access_token>&unit_id=1

# QR for Authenticator
curl -X POST http://localhost:8000/auth/totp/setup \
  -H "Content-Type: application/json" \
  -d '{"employee_code":"demo-admin","password":"ChangeMe123!"}'

# Excel / PDF export of the shift report
curl -H "Authorization: Bearer <token>" \
  "http://localhost:8000/reports/shift/export?format=xlsx" -o shift.xlsx
curl -H "Authorization: Bearer <token>" \
  "http://localhost:8000/reports/shift/export?format=pdf" -o shift.pdf
```

### Phase 6 — Security and hardening

| Capability | Implementation |
| :--- | :--- |
| RBAC | `operator` < `supervisor` < `admin` — apply/export/train restricted |
| Account lockout | After 5 failed logins, locked for 900s (`LOGIN_*`) |
| Rate limit | Redis on login/API + `limit_req` in nginx |
| Audit trail | `audit_events` table + `GET /security/audit` (admin) |
| Logout / token revocation | `POST /auth/logout` + jti blacklist in Redis |
| Password policy | `POST /auth/change-password` + complexity rules |
| Security headers | CSP / HSTS / X-Frame-Options in Gateway and nginx |
| Restricted CORS | `CORS_ALLOW_ORIGINS` |
| Internal service key | `INTERNAL_API_KEY` → header `X-Internal-Api-Key` |
| Secrets in production | `ENFORCE_PRODUCTION_SECRETS` / `ENVIRONMENT=production` |

```bash
python scripts/test_phase6.py

# Security status (admin)
curl -H "Authorization: Bearer <token>" http://localhost:8000/security/status

# Audit log
curl -H "Authorization: Bearer <token>" http://localhost:8000/security/audit?limit=20
```

Demo roles (password `ChangeMe123!` + the same OTP secret): `demo-admin` / `demo-supervisor` / `demo-operator`

### Phase 7 — Scalability and performance

| Capability | Implementation |
| :--- | :--- |
| Unit sharding | `DATA_UNIT_IDS` + `INGESTION_SHARD_COUNT` / `SHARD_ID` |
| High-throughput Kafka | lz4 compression, linger/batch, default 12 partitions |
| Influx batch | `InfluxRawWriter.write_batch` + async WriteOptions |
| Backpressure | Temporary ingestion pause when the local buffer is full |
| Latency budget | `LatencyTracker` + `/metrics/latency` (NFR-PER-01 <2s) |
| Liveness/Readiness | `/health/live` and `/health/ready` on services |
| Scale compose | `docker-compose.scale.yml` |

```bash
python scripts/test_phase7.py
python scripts/load_test_scale.py --units 1000 --seconds 3

# Horizontal scale (example 3 ingestion + 2 preprocessing + 2 prediction)
docker compose -f docker-compose.yml -f docker-compose.scale.yml up -d \
  --scale data-ingestion=3 --scale preprocessing=2 --scale prediction=2

# Prediction latency metric
curl -H "Authorization: Bearer <token>" http://localhost:8000/metrics/latency
```

To approach the 50,000-sensor target: `DATA_UNIT_IDS=1-50000`, increase `KAFKA_PARTITIONS`, and scale ingestion/preprocessing/prediction.

### Phase 8 — Observability and Uptime (NFR-PER-02)

| Capability | Implementation |
| :--- | :--- |
| Prometheus metrics | `/metrics` on all services + `shared/metrics.py` |
| Grafana dashboard | `infra/observability/grafana/dashboards/khalij-pos-overview.json` |
| Tempo (OTLP) | Distributed tracing with `OTEL_EXPORTER_OTLP_ENDPOINT` |
| Tempo (OTLP) | Distributed tracing with `OTEL_EXPORTER_OTLP_ENDPOINT` |
| Alert rules | ServiceDown / HighErrorRate / HighPredictionLatency / ErrorBudgetBurn |
| SLO helper | `shared/slo.py` + `GET /observability/slo` |

```bash
pip install prometheus-client httpx
python scripts/test_phase8.py

# Observability stack
docker compose -f docker-compose.yml -f docker-compose.observability.yml --profile observability up -d

# UI
# Grafana    http://localhost:3000  (admin/admin)
# Prometheus http://localhost:9090
# Tempo      http://localhost:3200

curl -H "Authorization: Bearer <token>" http://localhost:8000/observability/status
curl -H "Authorization: Bearer <token>" "http://localhost:8000/observability/slo?downtime_seconds=30"
```

A weekly error budget of 99.99% is about **60.5 seconds** of downtime; the `ServiceDown` alert fires after 1 minute of outage.

### Phase 9 — Kubernetes deployment

| Capability | Implementation |
| :--- | :--- |
| Kustomize | `infra/k8s/base` + overlay `dev` |
| Apps | Deployment/Service/PDB for 5 microservices |
| Infra | Postgres / Redis / Kafka / InfluxDB |
| Probes | `/health/live` + `/health/ready` |
| HPA | Automatic scaling on CPU |
| Ingress + TLS | `pos.khalij.local` + NetworkPolicy |
| Observability | Prometheus with kubernetes pod SD |
| Deploy | `scripts/deploy-k8s.sh` / `.ps1` |

```bash
pip install PyYAML
python scripts/test_phase9.py

# Deployment (requires a cluster + docker)
./scripts/deploy-k8s.sh dev
# or
.\scripts\deploy-k8s.ps1 -Overlay dev

kubectl apply -k infra/k8s/base
kubectl -n khalij-pos get pods,svc,ingress,hpa
```

Details: [`infra/k8s/README.md`](infra/k8s/README.md)

### Phase 10 — Operational pilot (Olefin Unit 1)

| Capability | Implementation |
| :--- | :--- |
| Pilot configuration | `config/pilot.yaml` (shadow/assisted mode, ACs) |
| Pilot Compose | `docker-compose.pilot.yml` |
| K8s overlay | `infra/k8s/overlays/pilot` → host `pilot.pos.khalij.local` |
| Runbook / UAT | `docs/pilot/RUNBOOK.md`, `ACCEPTANCE.md`, `REPORT_TEMPLATE.md` |
| Readiness gate | `scripts/pilot_readiness.py` |
| Smoke test | `scripts/pilot_smoke.py` (--offline or --base-url) |

```bash
pip install PyYAML
python scripts/test_phase10.py
python scripts/pilot_readiness.py
python scripts/pilot_smoke.py --offline

# Local pilot deployment
docker compose -f docker-compose.yml -f docker-compose.pilot.yml up -d --build
python scripts/pilot_smoke.py --base-url http://localhost:8000

# Kubernetes
kubectl apply -k infra/k8s/overlays/pilot
```

Automatic application to the DCS is **disabled** until safety sign-off (`auto_apply_setpoints`).

### SRS requirements coverage matrix

| ID | Requirement | Implementation status |
| :--- | :--- | :--- |
| R-GEN-01 | Ingest 1 record/second | ✅ `data-ingestion` + pilot AC-01 |
| R-GEN-02 | Five-year Data Lake | ✅ InfluxDB retention=`1825d` |
| R-GEN-03 | Monitoring dashboard + alerts | ✅ `/dashboard` + AC-07 |
| R-GEN-04 | Automatic end-of-shift energy report | ✅ PDF/Excel + AC-08 |
| FR-DATA-01 | OPC-UA / Modbus | ✅ client + `hardware_sim` + Circuit Breaker |
| FR-DATA-02 | T/P/F/L/GC/RPM | ✅ Tag mapping and data model |
| FR-DATA-03 | Local buffer during outage | ✅ `LocalBuffer` + Fallback |
| FR-PRE-01 | Outlier removal (IQR/Z-Score) | ✅ `preprocessing` |
| FR-PRE-02 | Linear interpolation of NaN | ✅ `preprocessing` |
| FR-PRE-03 | Min-Max normalization | ✅ `preprocessing` |
| FR-ML-01 | 60-second temperature prediction (LSTM/XGBoost) | ✅ + pilot AC-02 |
| FR-ML-02 | Critical alert at >5% deviation | ✅ + AC-03 |
| FR-ML-03 | Evolutionary feed-flow recommender | ✅ + AC-04 (no auto-apply) |
| FR-ALERT-01 | Telegram / email / control room | ✅ `alerting` + Redis |
| FR-ALERT-02 | Alert log + responding operator | ✅ PostgreSQL `alerts` |
| NFR-SEC-01 | Two-factor authentication | ✅ + AC-05 |
| NFR-SEC-02 | AES-256 / TLS 1.3 | ✅ crypto + tls-gateway + Ingress |
| NFR-PER-01 | Prediction response time <2s | ✅ LatencyTracker + AC-02 |
| NFR-SCL-01 | Horizontal scaling up to 50k sensors | ✅ Sharding + HPA |
| NFR-PER-02 | Uptime 99.99% | ✅ SLO + AC-06 + 14-day pilot |

---

## 📄 SRS Document – Production Process Optimization System (POS)

### 1. Introduction
**Purpose:** Develop a system based on data mining and machine learning to predict process anomalies and provide optimal settings to operators in order to reduce energy consumption and increase chemical reaction efficiency.

**Scope:** This system is implemented on the holding's olefin and polymerization units and connects to the DCS (distributed control system).

### 2. General Requirements
| ID | Requirement | Priority |
| :--- | :--- | :--- |
| R-GEN-01 | Receive real-time data from sensors at a rate of at least 1 record per second | High |
| R-GEN-02 | Store raw data in a Data Lake for 5 years | High |
| R-GEN-03 | Provide a monitoring dashboard capable of displaying predicted alerts | High |
| R-GEN-04 | Automatically generate an energy optimization report at the end of each shift | Medium |

### 3. Functional Requirements

#### 3-1. Data Ingestion Module
- **FR-DATA-01:** The system must be able to connect to OPC-UA or Modbus protocols to receive data from sensors.
- **FR-DATA-02:** The received data must include: temperature (T), pressure (P), flow rate (F), tank level (L), output gas composition (GC) and compressor speed (RPM).
- **FR-DATA-03:** If communication is lost, data must be stored in a local buffer and sent after reconnection.

#### 3-2. Preprocessing Module
- **FR-PRE-01:** Detect and remove outliers using the IQR or Z-Score method.
- **FR-PRE-02:** Linear interpolation to replace missing values (NaN).
- **FR-PRE-03:** Data normalization with Min-Max or StandardScaler before entering the model.

#### 3-3. Prediction & Optimization Module
- **FR-ML-01:** Predict the reactor outlet temperature (Target Variable) for the next 60 seconds using an LSTM or XGBoost model.
- **FR-ML-02:** If a deviation of more than 5% from the nominal temperature is predicted, a Critical-level alert must be issued.
- **FR-ML-03:** A Recommender system using evolutionary algorithms must compute the optimal inlet feed flow to achieve maximum efficiency.

#### 3-4. Alerting Module
- **FR-ALERT-01:** Send alerts via Telegram, email and display on the central control room screen.
- **FR-ALERT-02:** Log all alerts with the exact time and the ID of the responding operator.

### 4. Non-Functional Requirements

| ID | Requirement | Target value |
| :--- | :--- | :--- |
| NFR-PER-01 | Prediction response time | Less than 2 seconds |
| NFR-PER-02 | Availability (Uptime) | 99.99% (less than 1 minute of downtime per week) |
| NFR-SEC-01 | Two-factor authentication (2FA) for access to settings | Mandatory |
| NFR-SEC-02 | Data encryption at rest (AES-256) and in transit (TLS 1.3) | Mandatory |
| NFR-SCL-01 | Horizontal scalability to support 50,000 sensors | Expected |

### 5. Technical Architecture
- **Programming language:** Python 3.10
- **Web framework:** FastAPI (for microservices)
- **Database:** InfluxDB (for time-series data) + PostgreSQL (for transactional data)
- **Cache:** Redis
- **Messaging:** Apache Kafka (for data streaming)
- **MLOps:** MLflow for model registry

---

## 🧪 Synthetic Data Generator Code

> 📌 The executable, up-to-date version of this code is in [`data/generate_synthetic_data.py`](data/generate_synthetic_data.py) (run with `python data/generate_synthetic_data.py`). The code below is kept for documentation reference only.

The code below is written in Python and generates **10,000 records** for each of the **7 key variables** at a rate of **1 record per second**. The data includes sinusoidal patterns, seasonal trends and Gaussian noise to simulate the real world, and is finally saved in a CSV file.

```python
import numpy as np
import pandas as pd
from datetime import datetime, timedelta

# ==============================================
# Data generation parameters
# ==============================================
NUM_RECORDS = 10000          # number of records per variable (about 2.7 hours of data)
START_TIME = datetime(2026, 7, 22, 8, 0, 0)   # start time (similar to today)

# ==============================================
# Generate timestamps at 1-second intervals
# ==============================================
timestamps = [START_TIME + timedelta(seconds=i) for i in range(NUM_RECORDS)]

# ==============================================
# Generate synthetic data for each sensor
# ==============================================

# 1. Reactor Temperature - range 150 to 350 degrees Celsius
# includes an upward trend and cyclic fluctuation
t = np.linspace(0, 20 * np.pi, NUM_RECORDS)
reactor_temp = 250 + 30 * np.sin(t * 0.5) + 0.01 * np.arange(NUM_RECORDS) + np.random.normal(0, 2, NUM_RECORDS)
reactor_temp = np.clip(reactor_temp, 150, 350)

# 2. Reactor Pressure - range 10 to 40 bar
pressure = 25 + 5 * np.sin(t * 0.3) + 0.005 * np.arange(NUM_RECORDS) + np.random.normal(0, 0.8, NUM_RECORDS)
pressure = np.clip(pressure, 10, 40)

# 3. Feed Flow - range 100 to 500 cubic meters per hour
feed_flow = 300 + 80 * np.sin(t * 0.2 + 1.2) + np.random.normal(0, 5, NUM_RECORDS)
feed_flow = np.clip(feed_flow, 100, 500)

# 4. Tank Level - range 20 to 85 percent
tank_level = 50 + 20 * np.sin(t * 0.15 + 2.5) + np.random.normal(0, 1.5, NUM_RECORDS)
tank_level = np.clip(tank_level, 20, 85)

# 5. Output gas composition (Ethylene Concentration) - range 12 to 25 percent
ethylene_conc = 18 + 3 * np.sin(t * 0.25 + 0.8) + 0.002 * np.arange(NUM_RECORDS) + np.random.normal(0, 0.5, NUM_RECORDS)
ethylene_conc = np.clip(ethylene_conc, 12, 25)

# 6. Compressor RPM - range 1200 to 1800 revolutions per minute
compressor_rpm = 1500 + 150 * np.sin(t * 0.1 + 3.0) + np.random.normal(0, 10, NUM_RECORDS)
compressor_rpm = np.clip(compressor_rpm, 1200, 1800)

# 7. Production Efficiency - target variable (based on the other variables)
# Efficiency simulated as a function of temperature and pressure (with noise)
efficiency = ( (reactor_temp - 200) / 150 * 20 + (pressure - 20) / 20 * 10 + 60 
               + np.random.normal(0, 2, NUM_RECORDS) )
efficiency = np.clip(efficiency, 40, 98)

# ==============================================
# Build the dataframe
# ==============================================
df = pd.DataFrame({
    'timestamp': timestamps,
    'reactor_temp_c': np.round(reactor_temp, 2),
    'reactor_pressure_bar': np.round(pressure, 2),
    'feed_flow_m3h': np.round(feed_flow, 2),
    'tank_level_percent': np.round(tank_level, 2),
    'ethylene_conc_percent': np.round(ethylene_conc, 2),
    'compressor_rpm': np.round(compressor_rpm, 0).astype(int),
    'efficiency_target': np.round(efficiency, 2)   # dependent (target) variable
})

# ==============================================
# Save to CSV file
# ==============================================
output_file = "petrochemical_sensor_data_10k.csv"
df.to_csv(output_file, index=False)
print(f"✅ Synthetic data was successfully saved to file '{output_file}'.")
print(f"📊 Number of records: {len(df):,} - Number of variables: {len(df.columns)}")

# Show the first 5 records
print("\n🔍 Sample of generated data:")
print(df.head())

# Display summary statistics
print("\n📈 Descriptive statistics of the data:")
print(df.describe())
```

---

### 📊 Sample output (first 5 records):

| timestamp | reactor_temp_c | reactor_pressure_bar | feed_flow_m3h | tank_level_percent | ethylene_conc_percent | compressor_rpm | efficiency_target |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 2026-07-22 08:00:00 | 250.12 | 24.85 | 301.45 | 49.83 | 18.12 | 1502 | 68.45 |
| 2026-07-22 08:00:01 | 250.45 | 24.92 | 302.11 | 50.01 | 18.08 | 1505 | 68.92 |
| ... | ... | ... | ... | ... | ... | ... | ... |

---

### 🔧 Technical implementation notes:
1.  **Sampling rate:** `timedelta(seconds=i)` guarantees exactly 1 record per second.
2.  **Realistic noise:** A normal distribution (`np.random.normal`) is used to simulate sensor error.
3.  **Logical range:** All values are kept within the physically allowed sensor range with `np.clip`.
4.  **Causal relationship:** The `efficiency_target` variable is designed as a nonlinear function of temperature and pressure so the machine learning model can discover the pattern.




📄 SRS Document – Product 1: Production and Operations Process Optimization (Patentable)
1. Introduction
Purpose: Develop a system based on data mining and machine learning to predict process anomalies and provide optimal settings to operators in order to reduce energy consumption, increase chemical reaction efficiency and reduce the production of non-conforming products (Off-Spec).

Patent innovation: Unlike the Jam Petrochemical patent, which focuses solely on predicting product properties, this system provides the capability of "simultaneous optimization of three conflicting objectives (production cost, product quality, energy consumption)" using meta-heuristic algorithms and generating synthetic data to train the model under low-data conditions.

2. Functional Requirements (with emphasis on patent capabilities)
ID	Requirement	Patent capability
FR-ML-01	Real-time prediction of product quality parameters (MFI, density, particle size) using machine-learning-based virtual sensors 	Intelligent virtual sensors (similar to patent 112057)
FR-ML-02	Prediction of reactor outlet temperature and other process variables for the next 60 seconds with an LSTM or XGBoost model	Simultaneous multivariate prediction
FR-OPT-01	Simultaneous optimization of the three objectives cost, quality and energy consumption with Particle Swarm Optimization (PSO) and determination of the Pareto front	Multi-objective optimization (main innovation)
FR-OPT-02	Generation of synthetic data with Monte Carlo and GAN methods to train the model under low historical data	Virtual data generation (new innovation)
FR-ALERT-01	Sending predicted alerts to the operator and providing operational recommendations for adjusting parameters	Real-time recommender system
