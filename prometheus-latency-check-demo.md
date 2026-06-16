Purpose:
- Create Application which exposes the metrices of latency and it should showing into grafana.

1. Sample application - `app.py`
```
from flask import Flask, request
import time
import random
from prometheus_client import (
    Histogram,
    Counter,
    generate_latest,
    CONTENT_TYPE_LATEST,
)

app = Flask(__name__)

# 🔹 Request count
REQUEST_COUNT = Counter(
    "http_requests_total",
    "Total HTTP requests",
    ["method", "endpoint", "status"]
)

# 🔹 Latency histogram (THIS IS WHAT YOU NEED)
REQUEST_LATENCY = Histogram(
    "http_request_duration_seconds",
    "HTTP request latency",
    ["method", "endpoint"],
    buckets=(0.1, 0.2, 0.3, 0.5, 0.75, 1, 1.5, 2, 3, 5)
)

@app.route("/")
def hello():
    start = time.time()

    # Simulate variable latency
    delay = random.uniform(0.05, 1.5)
    time.sleep(delay)

    status = "200"
    REQUEST_COUNT.labels("GET", "/", status).inc()
    REQUEST_LATENCY.labels("GET", "/").observe(time.time() - start)

    return "Hello from latency demo\n"

@app.route("/metrics")
def metrics():
    return generate_latest(), 200, {"Content-Type": CONTENT_TYPE_LATEST}

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
```
Requirements.txt
```
flask
prometheus-client
```

Create `Dockerfile`
```
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 8080
CMD ["python", "app.py"]
```
Build and push image:
```
sudo docker build -t docker.xxx.xxx/purval/cicd:2.0 .
sudo docker push docker.xxx.xxx/purval/cicd:2.0
```

## Now create Kubernetes setup:
Create Namespace `namespace.yaml`:
```
apiVersion: v1
kind: Namespace
metadata:
  name: latency-demo
```
Create Deployment : `deployment.yaml`
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: latency-demo
  namespace: latency-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: latency-demo
  template:
    metadata:
      labels:
        app: latency-demo
    spec:
      containers:
      - name: app
        image: docker.linux.app/purval/cicd:2.0
        ports:
        - containerPort: 8080
```

Create service `service.yaml`:

```
apiVersion: v1
kind: Service
metadata:
  name: latency-demo-nodeport
  namespace: latency-demo
  labels:
    app: latency-demo
spec:
  type: NodePort
  selector:
    app: latency-demo
  ports:
  - name: metrics
    port: 8080          # Service port (inside cluster)
    targetPort: 8080    # Container port
    nodePort: 30950     # External port on node
```
### Create ServiceMonitor : `servicemonitor.yaml`
This is what prometheus scrape from your app.
```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: latency-demo
  namespace: monitoring
spec:
  namespaceSelector:
    matchNames:
    - latency-demo
  selector:
    matchLabels:
      app: latency-demo
  endpoints:
  - port: metrics
    path: /metrics
    interval: 15s
```

Apply:
```
kubectl apply namespace.yaml
kubectl apply deployment.yaml
kubectl apply sevrice.yaml
kubectl apply service-monitor.yaml
```
