## Phoenix Setup for LLM Model tracing:

### Setup

#### pheonix-deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: phoenix
  namespace: observability
spec:
  replicas: 1
  selector:
    matchLabels:
      app: phoenix
  template:
    metadata:
      labels:
        app: phoenix
    spec:
      containers:
      - name: phoenix
        image: arizephoenix/phoenix:latest
        ports:
        - containerPort: 6006
        - containerPort: 4317
        env:
        - name: PHOENIX_PORT
          value: "6006"
        - name: PHOENIX_ENABLE_OTLP
          value: "true"
        - name: PHOENIX_WORKING_DIR        # ← tell Phoenix where to persist data
          value: "/mnt/phoenix-data"
        volumeMounts:
        - name: phoenix-storage
          mountPath: /mnt/phoenix-data     # ← mount the volume
      volumes:
      - name: phoenix-storage
        persistentVolumeClaim:
          claimName: phoenix-pvc           # ← attach PVC
```
Apply : `kubectl apply -f pheonix-deployment.yaml` <br>

#### Create phoenix-pvc.yaml
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: phoenix-pvc
  namespace: observability
spec:
  storageClassName: local-path    # ← add this
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```
Apply: `kubectl apply -f phoenix-pvc.yaml` <br>

#### service.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: phoenix
  namespace: observability
spec:
  type: NodePort
  selector:
    app: phoenix
  ports:
  - name: ui
    port: 6006
    targetPort: 6006
  - name: otlp
    port: 4317
    targetPort: 4317
```

#### Create Project in Phoenix:
Name: vllm-observability

#### Setup OpenTelemetry Collector:
- Setup open-telemetry-collector which send traces to phoenix.

`otel-values.yaml`
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

    zipkin:
      endpoint: 0.0.0.0:9411

  processors:
    memory_limiter:
      check_interval: 5s
      limit_mib: 400
      spike_limit_mib: 100

    resource/phoenix_project:
      attributes:
        - key: openinference.project.name
          value: vllm-observability
          action: upsert

    batch: {}

  exporters:
    otlp/phoenix:
      endpoint: phoenix.observability.svc.cluster.local:4317
      tls:
        insecure: true

  service:
    pipelines:
      traces:
        receivers: [zipkin, otlp]
        processors: [memory_limiter, resource/phoenix_project, batch]
        exporters: [otlp/phoenix]

service:
  type: ClusterIP

```
#### install with helm:
```
helm install otel-collector open-telemetry/opentelemetry-collector   --namespace observability   -f otel-values.yaml

## for uninstall [ for reference only.]
helm uninstall otel-collector -n observability
```

#### create istio-operator:
```
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: tracing-config
  namespace: istio-system

spec:
  meshConfig:
    enableTracing: true
    defaultConfig:
      tracing:
        sampling: 100
        zipkin:
          address: otel-collector-opentelemetry-collector.observability.svc.cluster.local:9411
```
install
```
~/istio-1.29.0/bin/istioctl install -f istio-operator.yaml
```
verify:
```
kubectl get cm istio -n istio-system -o yaml | grep tracing -A5
```

Restart:
```
kubectl rollout restart deployment otel-collector-opentelemetry-collector -n observability
```


pipeline is:
```
User
 ↓
Istio ingressgateway
 ↓
Envoy proxy span
 ↓
OpenTelemetry Collector (Zipkin receiver)
 ↓
Phoenix (OTLP exporter)
```
