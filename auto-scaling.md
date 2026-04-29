# kubernetes Autoscaling has three layers:

| Layer         | Tool          | What it scales                      |
| ------------- | ------------- | ----------------------------------- |
| Pod replicas  | HPA / KEDA    | Number of pods                      |
| Pod resources | VPA           | CPU/memory requests                 |
| Worker nodes  | **Karpenter** | EC2/VM nodes for unschedulable pods |


Auto-scaling with prometheus
-----------------------------

1. Application -> Exposing prometheus metrics from application. app.py, /metrics
2. Prometheus scraps it --> ServiceMonitor  tell Prometheus how to scrape metrics from your application.
3. Prometheus adapter --> HPA Cant reads metrices from Prometheus, You must **install Prometheus adapter** into  prometheus and configure it. it expose metrics under **custom metrics** of kubernetes. its a bridge between **promethes** and **k8s custom metrics** api. 
4. HPA reads metrics --> HPA reads metrics from Prometheus adapter.
5. Scale pods

**So the flow will be**: <br>
```
[Your app] -> [Prometheus] -> [Prometheus Adapter] -> [Kubernetes HPA] -> [Scaling] 
```

In below example we are autoscaling according to **request_total** metrics.

## 1. Building application FastAPI.
app.py
```
from fastapi import FastAPI
from pydantic import BaseModel
from transformers import pipeline
from prometheus_client import Counter, make_asgi_app

app = FastAPI()
requests_total = Counter("requests_total", "Total API Requests")

# Load HF model
classifier = pipeline("sentiment-analysis", model="distilbert-base-uncased")

class Request(BaseModel):
    text: str

@app.post("/predict")
def predict(req: Request):
    requests_total.inc()
    out = classifier(req.text)
    return {"result": out}

metrics_app = make_asgi_app()
app.mount("/metrics", metrics_app)
```
requirements.txt
```
fastapi
uvicorn
transformers
torch
prometheus-client
```

Build docker image: <br>
Dockerfile:
```
FROM python:3.10-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy app
COPY app.py .

# Download the model during build (optional)
RUN python -c "from transformers import pipeline; pipeline('sentiment-analysis', model='distilbert-base-uncased')"

# Expose port
EXPOSE 8000

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```
Build and push dockerimage:
```
docker build -t <your-dockerhub-user>/hf-demo:latest .
docker push <your-dockerhub-user>/hf-demo:latest
```
## 2. Now create deployment: <br>
deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-sentiment
  labels:
    app: fastapi-sentiment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fastapi-sentiment
  template:
    metadata:
      labels:
        app: fastapi-sentiment
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8000"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      - name: fastapi-app
        image: docker.merai.app/harshal/hf-model:0.2
        ports:
        - containerPort: 8000
        env:
        - name: PYTHONUNBUFFERED
          value: "1"
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /docs
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /docs
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: fastapi-sentiment-service
  labels:
    app: fastapi-sentiment
spec:
  selector:
    app: fastapi-sentiment
  ports:
  - name: http
    port: 8000
    targetPort: 8000
    protocol: TCP
  type: ClusterIP
```

Apply changes:
```
kubectl apply -f deployment.yaml
```

## 3. Create servicemonitor to tell prometheus to scrap metrics from application:
```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: fastapi-sentiment-monitor
  namespace: monitoring
  labels:
    release: prometheus
spec:
  namespaceSelector:
    matchNames:
    - default
  selector:
    matchLabels:
      app: fastapi-sentiment
  endpoints:
    - port: http        # MUST match the service port name
      path: /metrics
      interval: 15s
```
apply:
```
kubectl apply -f servicemonitor.yaml
```

Verify the metrics are exposing in metrics or not:
```
curl http://<clusterip>:8000/metrics/
# / is mandadory at the end.
```
output:<br>
<img width="663" height="223" alt="image" src="https://github.com/user-attachments/assets/6b6864eb-a400-4d34-b667-30ba0e8e10cb" />


## 4. Create prometheus adapter to get metrics from prometheus, and expose it into custom metrics.
Install prometheus-adapter if not available:
```
# Check available or not?
kubectl get all -n monitoring

# Install now
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

prometheus-adapter-values.yaml
```
prometheus:
  url: http://prometheus-kube-prometheus-prometheus.monitoring.svc
rules:
  custom:
    - seriesQuery: 'requests_total{namespace!="",pod!=""}'
      resources:
        overrides:
          namespace:
            resource: namespace
          pod:
            resource: pod
      name:
        matches: "^requests_total$"
        as: "requests"
      metricsQuery: 'sum(rate(requests_total{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'
```
upgrade helm for prometheus-adapter:
```
helm upgrade --install prometheus-adapter prometheus-community/prometheus-adapter   -f prometheus-adapter-values.yaml -n monitoring
```

Verify metrics are showing in custom metrics or not:
```
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/requests"
```
## 5. Create Horizontal pod autoscaler:
hpa-request-total.yaml
```
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: fastapi-sentiment-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: fastapi-sentiment   # your deployment name
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: requests
      target:
        type: AverageValue
        averageValue: "10"  ## 10 requests/second per pod.

```
Apply:
```
kubectl apply -f hpa-request-count.yaml
```

Verify hpa:
```
kubectl get hpa
# If not showing then delete prometheus-adapter:
kubectl -n monitoring delete pod -l app.kubernetes.io/name=prometheus-adapter
```

## 6. Test:
```
while true; do   curl -s http://10.99.219.66:8000/predict -X POST -H "Content-Type: application/json"      -d '{"text":"hello"}' > /dev/null; done
```

Now check the metrics, in that you can see the request_total is increasing:
```
curl http://clusterip:8000/metrics/
```
Now check the HPA:
```
kubectl get hpa
```

<img width="1020" height="70" alt="image" src="https://github.com/user-attachments/assets/d8086268-6c38-4863-b6f6-04219173e76c" />

Here, `9m/10` means: <br>
**CURRENT_VALUE/TARGET_VALUE** <br>
This is your **HPA** Settings.

`m` means **milli-units** **(1/1000)**
10 = 1 unit of the custom metric, **10 requests per second** <br>
1m = 1/1000 of a unit, means **0.001 requests per seconds**. <br>

“Scale up when each pod has more than 10 requests per second.” <br>

<img width="722" height="133" alt="image" src="https://github.com/user-attachments/assets/2c21a594-09be-4ee3-a81f-458b40c2eace" />

### Troubleshooting:

1. Check metrices are showing in **prometheus** or not: <br>

To check the metrics are showing in prometheues you first need to **forward a port** of prometheus so you can check. <br>
This is only for testing purpose. <br>
```
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9091:9090
```
then check,
```
curl -s http://localhost:9091/api/v1/query?query=requests_total | jq
curl http://localhost:9091/api/v1/label/__name__/values
```

2. Check the custom metrics which is exposed by prometheus-adapter is showing or not.
```
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 
```
If it is showing `[]` empty like below then,
```
{"kind":"APIResourceList","apiVersion":"v1","groupVersion":"custom.metrics.k8s.io/v1beta1","resources":[]}
```

restart prometheus-adapter:
```
kubectl -n monitoring delete pod -l app.kubernetes.io/name=prometheus-adapter
# OR
kubectl -n monitoring delete pod -l app=prometheus-adapter
```

Check which prometheus data is getting fetched by prometheus-adapter:
```
kubectl -n monitoring get deploy prometheus-adapter -o yaml | grep -i prometheus -A3 -B3
```
it should be from `monitoring` namespace. <br>
`http://prometheus-kube-prometheus-prometheus.monitoring.svc:9090` that we have mentioned in `prometheus-adapter-values.yaml`.

Now prometheus adapter is getting the data from custom metrics :
```
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 | jq
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/requests"
```
   
Now check hpa,
```
kubectl get hpa
```

**Note:**  <br>
- Metric name from hpa.yaml `metrics.type.metric.name` should be match the **configmap of prometheus-adapter**. <br>
- You can verify: `kubectl -n monitoring get configmap prometheus-adapter -o yaml` <br>
- If you want to add another metrics e.g. `active_sessions` then you have to add that metrics into fastAPI. <br>

### Other commands:
Remove prometheus-adapter:
```
helm list -n monitoring
helm uninstall prometheus-adapter -n monitoring
```
If want to expose service port then,
```
kubectl port-forward svc/fastapi-sentiment-service 8086:8003
```
![IMG_1234](https://github.com/user-attachments/assets/2dd72eb0-f5fc-4ad0-8d5c-f826b186c290)
