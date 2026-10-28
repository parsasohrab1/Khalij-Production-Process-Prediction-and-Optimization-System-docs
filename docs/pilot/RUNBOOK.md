# Pilot Runbook — Khalij POS (Phase 10)

## 1. Pilot goal
A 14-day run on **Olefin Unit 1** in `shadow`/`assisted` mode: prediction and recommendation are active; automatic application to the DCS is forbidden until safety approval.

Reference configuration: [`config/pilot.yaml`](../../config/pilot.yaml)

## 2. Before Go-Live (Gate)
```bash
python scripts/pilot_readiness.py
python scripts/test_phase10.py
```
Mandatory items:
- [ ] Passwords and `JWT_SECRET_KEY` / `DATA_ENCRYPTION_KEY` / `INTERNAL_API_KEY` changed from default values
- [ ] 2FA enabled for shift operators and QR registered
- [ ] Firewall/VLAN between OT and IT checked
- [ ] Control room operator training completed
- [ ] Safety supervisor sign-off

## 3. Pilot deployment

### Docker Compose
```bash
cp .env.example .env   # then set the pilot values
docker compose -f docker-compose.yml -f docker-compose.pilot.yml up -d --build
curl http://localhost:8000/health/services
```

### Kubernetes
```bash
kubectl apply -k infra/k8s/overlays/pilot
kubectl -n khalij-pos get pods,svc,ingress
# Host: https://pilot.pos.khalij.local
```

### Switch to real OPC-UA
In `.env` or ConfigMap:
```
DATA_SOURCE_MODE=opcua
OPCUA_ENDPOINT_URL=opc.tcp://<dcs-host>:4840/...
```
Then restart only `data-ingestion` and monitor the Circuit Breaker.

## 4. Daily operations
1. Log in to the dashboard with 2FA → live monitoring tab
2. Check Critical alerts in the control room
3. Run What-if before every manual application of a recommendation
4. At the end of the shift: generate the report + PDF/Excel export
5. Check `/observability/slo` and Grafana (if enabled)

## 5. Success criteria (summary)
| AC | Requirement | Threshold |
|----|------|--------|
| AC-01 | Data rate | ≥ 1 Hz |
| AC-02 | Prediction latency P95 | ≤ 2000 ms |
| AC-03 | Critical alert | Delivered to the control room |
| AC-04 | Safe Optimize | constraints_satisfied |
| AC-05 | 2FA | Mandatory |
| AC-06 | Uptime | weekly downtime ≤ 60.5 s |
| AC-07 | Dashboard | live snapshot |
| AC-08 | Shift report | export OK |

Details: [`ACCEPTANCE.md`](ACCEPTANCE.md)

## 6. Rollback (maximum 30 minutes)
1. `DATA_SOURCE_MODE=simulator` or cut ingestion from the DCS
2. Scale down prediction/alerting if alert noise is high
3. Restore the previous revision:
   ```bash
   kubectl -n khalij-pos rollout undo deployment/api-gateway
   kubectl -n khalij-pos rollout undo deployment/prediction
   ```
4. Notify the control room and record in the audit / incident report

## 7. Emergency contacts
| Role | Responsibility |
|-----|---------|
| Shift supervisor | Decides to apply/stop recommendations |
| AI product owner | Model stability and drift |
| OT/network | OPC-UA/Modbus connectivity |
| Security | 2FA / keys / audit |

## 8. Exiting the pilot
- Complete the [`REPORT_TEMPLATE.md`](REPORT_TEMPLATE.md) report
- All ACs passed or with a documented waiver
- Update the production runbook and make a Go/No-Go decision for a wider rollout
