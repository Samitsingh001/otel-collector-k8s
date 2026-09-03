OpenTelemetry Collector on Kubernetes

Production-ready OpenTelemetry Collector deployment on Kubernetes (EKS/AWS) with:

HTTPS ingress via nginx + cert-manager (Let's Encrypt)
OTLP HTTP & gRPC receivers
Prometheus metrics exporter
Debug exporter (for log-based tracing)
Health check extension
2 replicas for high availability
Architecture
Your App (SDK)
     │
     │  OTLP HTTP (https://otel.ustpace.com/v1/traces)
     ▼
AWS ELB (LoadBalancer)
     │
     ▼
Nginx Ingress Controller
     │  port 443 → 4318
     ▼
otel-collector Service (ClusterIP)
     │
     ▼
otel-collector Pods (x2)
     │
     ├── debug exporter     → stdout logs
     ├── prometheus exporter → :8889/metrics (in-cluster)
     └── (add your backend exporter here: Jaeger, Tempo, Datadog etc.)
Prerequisites
Kubernetes cluster (EKS or any)
kubectl configured
Nginx Ingress Controller installed
cert-manager installed with a letsencrypt-prod ClusterIssuer
A domain pointed to your ingress ELB (e.g. otel.ustpace.com)
Deployment Steps
Step 1 — Clone the repo
bash
git clone https://github.com/<your-org>/otel-collector-k8s.git
cd otel-collector-k8s
Step 2 — Update your domain

Before applying, open manifests/04-ingress.yaml and replace otel.ustpace.com with your actual domain in both places.

Step 3 — Create the namespace
bash
kubectl apply -f manifests/00-namespace.yaml

Verify:

bash
kubectl get namespace otel-system
Step 4 — Apply the ConfigMap (collector config)
bash
kubectl apply -f manifests/01-configmap.yaml

Verify:

bash
kubectl get configmap -n otel-system
Step 5 — Deploy the collector
bash
kubectl apply -f manifests/02-deployment.yaml

Verify pods are running (may take 30–60 seconds):

bash
kubectl get pods -n otel-system

Expected output:

NAME                              READY   STATUS    RESTARTS   AGE
otel-collector-xxxxxxxxx-xxxxx    1/1     Running   0          1m
otel-collector-xxxxxxxxx-xxxxx    1/1     Running   0          1m
Step 6 — Create the service
bash
kubectl apply -f manifests/03-service.yaml

Verify:

bash
kubectl get svc -n otel-system

Expected output:

NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                                AGE
otel-collector   ClusterIP   10.100.xx.xxx   <none>        4317/TCP,4318/TCP,8889/TCP,13133/TCP   1m
Step 7 — Apply the ingress
bash
kubectl apply -f manifests/04-ingress.yaml

Verify:

bash
kubectl get ingress -n otel-system

Expected output (ADDRESS may take 2–3 minutes to populate):

NAME                     CLASS   HOSTS              ADDRESS                                     PORTS     AGE
otel-collector-ingress   nginx   otel.ustpace.com   xxxx.elb.amazonaws.com                      80, 443   1m
Step 8 — Verify TLS certificate

cert-manager will automatically issue a Let's Encrypt certificate. Check it:

bash
kubectl get certificate -n otel-system

Expected:

NAME                 READY   SECRET               AGE
otel-collector-tls   True    otel-collector-tls   2m

If READY is False, wait a minute and check again. If still failing:

bash
kubectl describe certificate otel-collector-tls -n otel-system
Step 9 — End-to-end test

Send a test trace to confirm everything is working:

bash
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

Expected response: HTTP 200

Then check the collector logs to confirm the span was received:

bash
kubectl logs -n otel-system deployment/otel-collector --tail=30

You should see the test-span printed in the logs.

Verify Deployment
bash
# Run the verify script
chmod +x scripts/verify.sh
./scripts/verify.sh otel.ustpace.com

# Or manually check
kubectl get pods -n otel-system
kubectl logs -n otel-system deployment/otel-collector --tail=50
Ports & Endpoints
Port	Protocol	Endpoint	Access
4317	gRPC	OTLP gRPC receiver	In-cluster only
4318	HTTP	OTLP HTTP receiver (/v1/traces, /v1/metrics, /v1/logs)	Public via Ingress
8889	HTTP	Prometheus metrics scrape	In-cluster only
13133	HTTP	Health check	In-cluster only (used by k8s probes)
Application Configuration

Set these env vars in your application:

bash
# Required
OTEL_EXPORTER_OTLP_ENDPOINT=https://otel.ustpace.com
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_SERVICE_NAME=your-service-name

# Optional but recommended
OTEL_TRACES_EXPORTER=otlp
OTEL_METRICS_EXPORTER=otlp
OTEL_LOGS_EXPORTER=otlp
Python Example
python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter

provider = TracerProvider()
exporter = OTLPSpanExporter(
    endpoint="https://otel.ustpace.com/v1/traces"
)
provider.add_span_processor(BatchSpanProcessor(exporter))
trace.set_tracer_provider(provider)
Node.js Example
javascript
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-http');

const exporter = new OTLPTraceExporter({
  url: 'https://otel.ustpace.com/v1/traces',
});
Upgrade Collector Version
bash
# Update image tag in manifests/02-deployment.yaml
# Then apply rolling update:
kubectl set image deployment/otel-collector \
  otel-collector=otel/opentelemetry-collector-contrib:0.159.0 \
  -n otel-system

# Watch rollout
kubectl rollout status deployment/otel-collector -n otel-system

# Rollback if needed
kubectl rollout undo deployment/otel-collector -n otel-system
Adding a Real Backend Exporter

Edit manifests/01-configmap.yaml to add your backend (e.g. Jaeger, Grafana Tempo, Datadog):

yaml
exporters:
  otlp/jaeger:
    endpoint: jaeger-collector:4317
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, resource, batch]
      exporters: [debug, otlp/jaeger]   # add your exporter here
Teardown
bash
chmod +x scripts/teardown.sh
./scripts/teardown.sh
File Structure
otel-collector-k8s/
├── README.md
├── manifests/
│   ├── 00-namespace.yaml       # otel-system namespace
│   ├── 01-configmap.yaml       # OTel Collector config
│   ├── 02-deployment.yaml      # Deployment (2 replicas)
│   ├── 03-service.yaml         # ClusterIP service
│   └── 04-ingress.yaml         # Nginx ingress + TLS
