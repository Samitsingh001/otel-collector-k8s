# otel-collector-k8s

Production-ready OpenTelemetry Collector deployment on Kubernetes (EKS/AWS).

## What's included

- OTLP HTTP & gRPC receivers
- Prometheus metrics exporter
- Debug exporter
- HTTPS via nginx ingress + cert-manager (Let's Encrypt)
- 2 replicas for high availability

## Architecture

```
Your App (SDK)
     │
     │  OTLP HTTP (https://otel.ustpace.com/v1/traces)
     ▼
AWS ELB → Nginx Ingress (443 → 4318)
     │
     ▼
otel-collector Service (ClusterIP)
     │
     ▼
otel-collector Pods (x2)
     ├── debug exporter     → stdout logs
     └── prometheus exporter → :8889/metrics (in-cluster)
```

## Prerequisites

- Kubernetes cluster (EKS or any)
- `kubectl` configured
- Nginx Ingress Controller installed
- `cert-manager` with a `letsencrypt-prod` ClusterIssuer
- A domain pointed to your ingress ELB

## Deploy

```bash
git clone https://github.com/Samitsingh001/otel-collector-k8s.git
cd otel-collector-k8s
```

> Before applying, update your domain in `manifests/04-ingress.yaml`

```bash
kubectl apply -f manifests/00-namespace.yaml
kubectl apply -f manifests/01-configmap.yaml
kubectl apply -f manifests/02-deployment.yaml
kubectl apply -f manifests/03-service.yaml
kubectl apply -f manifests/04-ingress.yaml
```

## Verify

```bash
# Check pods
kubectl get pods -n otel-system

# Check ingress
kubectl get ingress -n otel-system

# Check TLS certificate
kubectl get certificate -n otel-system

# Check logs
kubectl logs -n otel-system deployment/otel-collector --tail=30
```

## Test

```bash
curl -X POST https://otel.ustpace.com/v1/traces \
  -H "Content-Type: application/json" \
  -d '{
    "resourceSpans": [{
      "resource": {
        "attributes": [{"key": "service.name", "value": {"stringValue": "test-service"}}]
      },
      "scopeSpans": [{
        "spans": [{
          "traceId": "00000000000000000000000000000001",
          "spanId": "0000000000000001",
          "name": "test-span",
          "kind": 1,
          "startTimeUnixNano": "1000000",
          "endTimeUnixNano": "2000000",
          "status": {}
        }]
      }]
    }]
  }'
```

Expected: `HTTP 200` — then check logs for the `test-span`.

## Ports

| Port  | Protocol | Purpose                  | Access         |
|-------|----------|--------------------------|----------------|
| 4317  | gRPC     | OTLP gRPC receiver       | In-cluster     |
| 4318  | HTTP     | OTLP HTTP receiver       | Public (Ingress)|
| 8889  | HTTP     | Prometheus metrics scrape| In-cluster     |
| 13133 | HTTP     | Health check             | In-cluster     |

## App Configuration

```bash
OTEL_EXPORTER_OTLP_ENDPOINT=https://otel.ustpace.com
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_SERVICE_NAME=your-service-name
```

## Upgrade

```bash
kubectl set image deployment/otel-collector \
  otel-collector=otel/opentelemetry-collector-contrib:0.159.0 \
  -n otel-system

kubectl rollout status deployment/otel-collector -n otel-system

# Rollback if needed
kubectl rollout undo deployment/otel-collector -n otel-system
```

## Teardown

```bash
kubectl delete -f manifests/04-ingress.yaml
kubectl delete -f manifests/03-service.yaml
kubectl delete -f manifests/02-deployment.yaml
kubectl delete -f manifests/01-configmap.yaml
kubectl delete -f manifests/00-namespace.yaml
```
