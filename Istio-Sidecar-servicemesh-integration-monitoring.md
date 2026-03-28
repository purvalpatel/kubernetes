Observability with servicemesh:
------------------------------
```
metrics -> Prometheus -> Grafana
Container Logs -> Promtail/FluentBit -> Loki -> Grafana
Traces -> Jaeger/Tempo -> UI
```
**Here we are testing for the 1st one. metrics collection with servicemesh.**

### Step 1:  Istio Sidecar injected with the namespace.
```
kubectl label namespace numol istio-injection=enabled
```

### Step 2: Ensure Ingress & Sidecards expose metrics on port 15090.
**List resources:**
```
kubectl get all -n istio-system
```
<img width="1494" height="203" alt="image" src="https://github.com/user-attachments/assets/7efb0f8c-c0c1-4bd3-90d2-22241d6f0c79" />

If it is not showing then add it, <br>

Edit istio-ingressgateway sevrice:
```
kubectl edit service istio-ingressgateway -n istio-system
```
Add below line,
```
  - name: http-envoy-prom
    nodePort: 32170
    port: 15090
    protocol: TCP
    targetPort: 15090  # <-- This is the port of Envoy side car. that needs to be expose. to ingress controller

```
**Rollout restart deployment:**
```
kubectl rollout restart deployment istiod -n istio-system
```

### Step 3:
Scrape sidecards using `podmonitor`. <br>

**create `podmonitor.yaml`**
```
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: istio-proxy
  namespace: monitoring
  labels:
    release: prometheus
spec:
  namespaceSelector:
    any: true
  selector:
### matches pods which have label: app: nginx
    matchLabels:
      app: nginx
  podMetricsEndpoints:
    - port: http-envoy-prom
      path: /stats/prometheus
      interval: 15s
```

**List created `podMonitor`:**
```
kubectl get podmonitor -n monitoring
```
- Here `Selector` match label will match the pod labels named `nginx`. and scrape metrics from the pods directly.
- You can check the End Point from the prometheus.
<img width="1913" height="451" alt="image" src="https://github.com/user-attachments/assets/fefdbcf5-51ef-432b-99b9-425f08e92caa" />
- Here two endpoints are showing, each for the pod.
- `PodMonitor` matches label, not pods, If we use PodMonitor then ServiceMonitor is not required.
- `ServiceMonitor` : Finds the kubernetes service that match this `selector`, then scrape their endpoint.
- `PodMonitor` : finds the pods that match this `selector` directly.

Below is the flow:
```
nginx Pod (devops-test)
 ├── app container
 └── istio-proxy sidecar
        |
        |  exposes :15090/stats/prometheus
        |
Prometheus (monitoring namespace)
  └── reads PodMonitor CR
        |
        |  discovers matching pods
        |
Scrapes metrics directly from pods
        |
Stores time-series data
        |
Grafana queries Prometheus
        |
Displays dashboards
```

### Step 4: Now Create Per Pod Traffic Dashboard in Grafana:
1. Per Pod Traffic
```
sum(rate(istio_requests_total{namespace="devops-test"}[1m])) by (pod)
```
2. P95 Latency
```
histogram_quantile(
  0.95,
  sum(rate(istio_request_duration_milliseconds_bucket{namespace="devops-test"}[5m])) by (le)
)
```
3. Request rate per service
```
sum(rate(istio_requests_total[1m])) by (destination_service)
```
4. Error Rate
```
sum(rate(istio_requests_total{namespace="devops-test",response_code=~"5.."}[1m]))
/
sum(rate(istio_requests_total{namespace="devops-test"}[1m]))
```

5. Traffic Volume:
```
sum(rate(istio_requests_total{namespace="devops-test"}[1m]))
```

Export dashboard JSON : https://github.com/purvalpatel/kubernetes-tutorial/blob/483eb50985bb515a2e1f166c81ad0fbe8079a744/Istio%20Mesh%20Dashboard-1771929073978.json <br>

🎯 Metrics collection is completed here.


## Logs Monitoring with Loki in Grafana (Kubernetes)
- Note : This doesnt requires sidecar to be injected.

- Note Loki `2.x.x` version ins not compatible with Grafana `12.x.x` version. the issue is loki is not connecting grafana from web ui. either connectivity works from backend. so for that need to use new version i.e. `3.x.x`.
- check Grafana version : `kubectl describe pod prometheus-grafana-6444876565-84p67 -n monitoring | grep Image`
- check loki version: `kubectl -n monitoring get pod loki-0 -o jsonpath="{.spec.containers[*].image}"`

### Setup prometheus. (optional if not done)
- Fllow link - https://github.com/purvalpatel/kubernetes-tutorial/blob/d08d13ee86fd6766f6181e1659aa13bed3a9fa80/kube-prometheus-grafana-stack.md 

### Install Loki
```
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```
Create `values.yaml`
```
loki:
  auth_enabled: false

  commonConfig:
    replication_factor: 1

  storage:
    type: filesystem

  schemaConfig:
    configs:
      - from: 2024-01-01
        store: tsdb
        object_store: filesystem
        schema: v13
        index:
          prefix: loki_index_
          period: 24h

singleBinary:
  replicas: 1

deploymentMode: SingleBinary

read:
  replicas: 0
write:
  replicas: 0
backend:
  replicas: 0
```
Install
```
helm install loki grafana/loki \
  -n monitoring \
  -f loki-values.yaml
```
### Install promtail:
- This will collect the logs from the pods. this will run on each node as daemonset.
```
helm install promtail grafana/promtail -n monitoring
```
Verify it is being installed or not:
```
helm list -n monitoring

kubectl get pods -n monitoring
```

Check the connectivity now from Grafana: <br>
<img width="835" height="233" alt="image" src="https://github.com/user-attachments/assets/ee33b251-3f75-42a9-8e30-ae3267b25183" />

### Create Grafana dashboard:
<img width="730" height="144" alt="image" src="https://github.com/user-attachments/assets/a5428992-1fcf-4167-b69e-c241db242020" />

- Create Dashboard which have option to select namespace and all deployment under that namespace.

1. Create Dashboard first.
```
Add visualization -> Type (logs) -> Set "Panel name"
```
2. Add variables
- Dashboard -> Settings -> variables
- Add Variable
- Variable Type : `Query`
- Name : `namespace`
- Hide: `nothing`
- Data Source : `Loki`
- Query Type: `Label values`
- Label: `namespace`
- Selection Option: check `multi value`, `Include all option`
- Save.

**Add Another variable**
- Add variable
- Variable type: `Query`
- Name: `deployment`
- Hide: `nothing`
- Data source: `Loki`
- Query type: `Label values`
- Label: `app`     ( check label from Explore -> Loki )
- Stream selector : `{namespace=~"$namespace"}`
- Selection Option: check `multi value`, `Include all option`
- Save.

3. Add widget now
- Visualization - `logs`
- Query : `{namespace=~"$namespace", app=~"$deployment"}`
- Panel Name: `Logs - $namespace / $deployment`

<img width="1062" height="404" alt="image" src="https://github.com/user-attachments/assets/a044dba2-6a9b-4352-bb69-1469b31360d4" />

🎯 Logs Collection is completed here.

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
kubectl patch svc jaeger -n observability -p '{"spec": {"type": "NodePort"}}'}'
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
