# Pilot Acceptance Checklist (UAT) — Phase 10

Source of criteria: `config/pilot.yaml` → `acceptance_criteria`

| ID | SRS | Criterion | Verification method | Result | Signature |
|----|-----|--------|-----------|--------|------|
| AC-01 | R-GEN-01 | ≥1 record/second | `GET /scale/ingestion` or ingestion metric | ☐ Pass ☐ Fail ☐ Waive | |
| AC-02 | FR-ML-01 / NFR-PER-01 | Prediction P95 ≤2s | `GET /metrics/latency` | ☐ Pass ☐ Fail ☐ Waive | |
| AC-03 | FR-ML-02 | Critical alert to the control room | Control room tab / Redis alerts | ☐ Pass ☐ Fail ☐ Waive | |
| AC-04 | FR-ML-03 | Optimize with safety constraints | `POST /optimize` → constraints_satisfied | ☐ Pass ☐ Fail ☐ Waive | |
| AC-05 | NFR-SEC-01 | Login only with 2FA | `POST /auth/login` → requires_otp | ☐ Pass ☐ Fail ☐ Waive | |
| AC-06 | NFR-PER-02 | downtime ≤ SLO budget | Prometheus / `/observability/slo` | ☐ Pass ☐ Fail ☐ Waive | |
| AC-07 | R-GEN-03 | Live dashboard | `/dashboard` + WebSocket | ☐ Pass ☐ Fail ☐ Waive | |
| AC-08 | R-GEN-04 | Shift report PDF/Excel | `/reports/shift/export` | ☐ Pass ☐ Fail ☐ Waive | |

## Mandatory user scenarios
- [ ] The operator logs in, enters the OTP, and sees the dashboard
- [ ] A test alert is acknowledged and recorded in the audit
- [ ] What-if is run before apply; apply only by supervisor+
- [ ] Manual model training only by admin
- [ ] Logout revokes the token

## Safety
- [ ] Automatic application to the DCS is disabled (`auto_apply_setpoints` off)
- [ ] The safe range of temperature/pressure/flow is not violated in the recommendation
- [ ] The Rollback path has been practiced in the Runbook

**Date:** __________  **Site:** __________  **Final decision:** ☐ Go ☐ No-Go
