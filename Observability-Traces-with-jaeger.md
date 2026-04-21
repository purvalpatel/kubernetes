## Traces Collection:
```
Traces -> Jaeger/Tempo -> UI

App → OTEL Collector → Jaeger
```
There is `jaeger collector` also but there is Limited flexibility, Harder to extend later. so we  will use `OpenTelemetry Collector`. <br>

### Setup jaeger
```
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo update
helm install jaeger jaegertracing/jaeger   --namespace observability
```

### Setup opentelemetry-collector.
1. Create `otel-values.yaml`
```
mode: deployment

image:
  repository: otel/opentelemetry-collector-contrib

config:
  receivers:
    otlp:
      protocols:
        grpc: {}
        http: {}

  processors:
    memory_limiter:
      check_interval: 5s
      limit_mib: 400
      spike_limit_mib: 100
    batch: {}

  exporters:
    otlp:
      endpoint: jaeger.observability.svc.cluster.local:4317
      tls:
        insecure: true

  service:
    pipelines:
      traces:
        receivers: [otlp]
        processors: [memory_limiter, batch]
        exporters: [otlp]

service:
  type: ClusterIP

```
Install:
```
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update
helm install otel-collector open-telemetry/opentelemetry-collector   --namespace observability   -f otel-values.yaml
```

Verify it is working fine or not:
```
kubectl get all -n observability
```
<img width="1909" height="248" alt="image" src="https://github.com/user-attachments/assets/8d1d6f16-bc19-4912-b25f-9b0a4367a6ce" />

Now application should send metrics on opentelemetry - `http://otel-collector.observability.svc.cluster.local:4317` <br>

Allow Jaeger to open in browser:
```
kubectl patch svc jaeger -n observability -p '{"spec": {"type": "NodePort"}}'
```
you can use NodePort of `http://localhost:16686`

### Remove everything: [optional]
```
helm uninstall jaeger -n observability
helm uninstall otel-collector -n observability
```
<img width="1912" height="788" alt="image" src="https://github.com/user-attachments/assets/0ea7b61e-471d-483e-b1f1-a3a5b709ea35" />

### How to Test:
**Step 1 — Start a test pod inside cluster** <br>
```
kubectl run otel-test \
  --image=python:3.11 \
  -n observability \
  -it --rm -- bash
```
Now you're inside the pod. <br>

**Step 2 — Install dependencies** <br>
Inside the pod: <br>
```
pip install opentelemetry-sdk \
            opentelemetry-exporter-otlp \
            opentelemetry-api
```
**Step 3 — Create test script** <br>

Inside the pod: <br>
```
cat <<EOF > test.py
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer(__name__)

exporter = OTLPSpanExporter(
    endpoint="http://otel-collector-opentelemetry-collector:4317",
    insecure=True,
)

trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(exporter)
)

with tracer.start_as_current_span("nuvo-test-span"):
    print("Trace sent successfully")
EOF
```
**Step 4 — Run it** <br>
```
python test.py
```

Step 5 - Check logs in Jaeger UI now <br>
<img width="1912" height="671" alt="image" src="https://github.com/user-attachments/assets/852f4f1d-7ef3-4940-9577-5003ed288ffe" />

🎯 Traces Collection is completed here.


## Traces monitoring with Jaeger and OpenTelemetry Collector:
The best architecture:
```
Istio (auto inject trace headers)
        ↓
Microservices (instrumented or auto)
        ↓
OpenTelemetry Collector
        ↓
Jaeger UI
```

### Setup Jaeger:

```
kubectl create ns observability

helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo update
helm install jaeger jaegertracing/jaeger   --namespace observability
```

### Setup opentelemetry collector:
otel-values.yaml
```
mode: deployment

image:
  repository: otel/opentelemetry-collector-contrib

config:
  receivers:
    otlp:
      protocols:
        grpc: {}
        http: {}

  processors:
    memory_limiter:
      check_interval: 5s
      limit_mib: 400
      spike_limit_mib: 100
    batch: {}

  exporters:
    otlp:
      endpoint: jaeger.observability.svc.cluster.local:4317
      tls:
        insecure: true

  service:
    pipelines:
      traces:
        receivers: [otlp]
        processors: [memory_limiter, batch]
        exporters: [otlp]

service:
  type: ClusterIP
```
install:
```
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update
helm install otel-collector open-telemetry/opentelemetry-collector   --namespace observability   -f otel-values.yaml
```

### Expose Jaeger to NodePort:
```
kubectl patch svc jaeger -n observability -p '{"spec": {"type": "NodePort"}}'
```
### Check tracing is enabled in istio or not?
```
kubectl get configmap istio -n istio-system -o yaml | grep tracing -A 10
```
If not then enable it,

Create, `istio-otel.yaml`
```
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istio-otel
  namespace: istio-system
spec:
  meshConfig:
    enableTracing: true

    defaultConfig:
      tracing:
        sampling: 100

    extensionProviders:
    - name: otel
      opentelemetry:
        service: otel-collector-opentelemetry-collector.observability.svc.cluster.local
        port: 4317

    defaultProviders:
      tracing:
      - otel

```

Apply:
```
/home/merai/istio-1.28.0/bin/istioctl install -f /home/merai/istio-otel.yaml
```
Rollout changes:
```
kubectl rollout restart deployment istiod -n istio-system
kubectl rollout restart deployment istio-ingressgateway -n istio-system
```

Restart deployment of application:
```
kubectl rollout restart deployment sbdd-frontend -n numol
```

check:  https://api.numol.io:30687/

### Traces are showing in jaeger UI

<img width="1913" height="943" alt="image" src="https://github.com/user-attachments/assets/edd9c881-595e-49b7-b20c-2bf636fa94fb" />
