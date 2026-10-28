# Khalij POS — Kubernetes (phase 9)

Declarative manifests with **Kustomize** for deployment on a cluster (kind / minikube / EKS / AKS / GKE).

## Structure

```
infra/k8s/
  base/                 # production-like base environment
    apps/               # api-gateway, ingestion, preprocessing, prediction, alerting
    infra/              # postgres, redis, kafka, influxdb
    networking/         # Ingress TLS + NetworkPolicy
    autoscaling/        # HPA
    observability/      # Prometheus with kubernetes_sd
  overlays/dev/         # fewer replicas for development
```

## Prerequisites

- Docker
- `kubectl`
- A local cluster (e.g., `kind create cluster`) or a cloud cluster
- Ingress Controller (nginx): `kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml`

## Quick deployment

```bash
# Linux/macOS
chmod +x scripts/deploy-k8s.sh
./scripts/deploy-k8s.sh          # base
./scripts/deploy-k8s.sh dev      # development overlay

# Windows PowerShell
.\scripts\deploy-k8s.ps1
.\scripts\deploy-k8s.ps1 -Overlay dev
```

Or manually:

```bash
cp infra/k8s/base/secrets.example.yaml infra/k8s/base/secrets.yaml
# edit secrets.yaml

kubectl apply -k infra/k8s/base
# or
kubectl apply -k infra/k8s/overlays/dev
```

## Images

```
khalij/api-gateway:latest
khalij/data-ingestion:latest
khalij/preprocessing:latest
khalij/prediction:latest
khalij/alerting:latest
```

For kind: after the build, `kind load docker-image khalij/api-gateway:latest` (and the rest).

## Probes

All microservices:

- liveness: `GET /health/live`
- readiness: `GET /health/ready`

## Scaling

HPA on CPU for gateway / ingestion / preprocessing / prediction / alerting.

Ingestion uses the pod `HOSTNAME` for sharding (phase 7); align `INGESTION_SHARD_COUNT` with the replica count.

## Ingress

- Host: `pos.khalij.local`
- TLS secret: `khalij-pos-tls`
- In `/etc/hosts` or CoreDNS: `pos.khalij.local` → ingress IP

## Security

- NetworkPolicy: data-plane only from inside the namespace
- Secrets separate from ConfigMap
- `ENFORCE_PRODUCTION_SECRETS=true` in the ConfigMap

⚠️ Never commit `secrets.example.yaml` with real values.
