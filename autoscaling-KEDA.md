
KEDA:
------

The Default kubernetes metrices for scaling is CPU, RAM. <br>
if we wanted to go above this then we have to use Advance kubernetes functionality use on top of this. i.e. **KEDA**  **(Kubernetes Event driven Autoscaling)** <br>

**Best For:** <br>
✔ Workloads triggered by queues (RabbitMQ, Kafka, SQS) <br>
✔ Cron jobs / timers <br>
✔ Scaling based on external events <br>
✔ Cloud-native microservices <br>
✔ Bursty traffic <br>

**Scales to zero (HPA cannot)** <br>

KEDA makes scaling very easy because it creates the HPA for you. <br>

**Verdict:** <br>

Best for queue/event-driven workloads and cost savings (scale to zero).

KEDA Work along side with the existing kubernetes component like HPA and can extend functionality without duplication or overwriding. <br>

**Deployment document**: https://keda.sh/docs/2.18/deploy/

**Install KEDA with Helm.<br>**
```
helm repo add kedacore https://kedacore.github.io/charts  
helm repo update
helm install keda kedacore/keda --namespace keda --create-namespace
```

Verify:
```
kubectl get pods -n keda

```
Now, we will understand **how we use it**. <br>

We, have grafana and prometheus setup. in Grafana there is metrics called, Latency is showing.<br>

**What is latency(95% percentile)?<br>**
95th percentile latency is a statistical metric used to measure the performance of a system, particulary in latency-sensitive applications like web service, APIs, or databases.<br>

Means, <br>
95% requests are faster<br>
5% Requests are slower.<br>

Now we will take this data from** prometheus for scaling**.<br>

For that KEDA gives us configuration called **scaledObject**.<br>
If you are directly dealing with kubernetes then you would use **hpa**.<br>
in this case we are using **scaledobject**, which is a wrapper on top of that.<br>

### Deployment Architecture:
```
Deployment -> ScaledObject ( Automatically install HPA )  -> Pods auto scaling

Means,
App emits metrics → Prometheus collects → KEDA reads → HPA decides → Kubernetes scales.

ServiceMonitor Requires if kube-prometheus-stack installed with helm.
```
## Deployment steps:
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
Create deployment:
```
kubectl apply -f deployment.yaml
```

### servicemonitor: ( Doesn't Required )

**Note: KEDA Does not require servicemonitor. if Prometheus is not installed with helm.** <br>

Prometheus ↔️ Application (requires ServiceMonitor if you want Prometheus to scrape your app) <br>
KEDA ↔️ Prometheus (no ServiceMonitor needed) <br>

Prometheus will scrape data from service in two ways: <br>

1. Annotations <br>
   - If service have below annotation `kubectl describe service <service-name>`  <br>

  ```
   prometheus.io/scrape: "true"
  ```
  Then prometheus will scrape the service. No need of servicemonitor. <br>
  Add this in service if you want to scrape with annotation. (Optional)
```
apiVersion: v1
kind: Service
metadata:
  name: fastapi-sentiment-service
  labels:
    app: fastapi-sentiment

  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8000"
    prometheus.io/path: "/metrics"
```

  Thats why in deployment this is mentioned. <br>
  <img width="372" height="94" alt="image" src="https://github.com/user-attachments/assets/28f4ecc8-3836-4bcd-9d47-c7c9e170fbe1" />

  
2. ServiceMonitor
   If you dont use annotations., you must use ServiceMonitor.
   if you have setup prometheus-stack with helm then you have to use servicemonitor.

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
Create:
```
kubectl apply -f servicemonitor.yaml
```

### ScaledObject:
Below is the example of http requests.
```YAML
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
   name: http-app-scaledobject
spec:
   scaleTargetRef:
      name: http-app
   minReplicaCount: 1
   maxReplicaCount: 10
   triggers:
      - type: prometheus
        metadata:
           serverAddress: http://prometheus-server.default.svc.cluster.local:9090
           metricName: http_requests_total
           threshold: '5'
           query: sum(rate(http_requests_total[1m])) 
```
Check created scaledobject.
```
kubectl get scaledobject
```

Note: ScaledObject will create HPA Automatically, unlinke  manual hpa creation. <br>

Check created hpa:<br>
KEDA will create linked Autoscaler automatically with scaledobject.
```
kubectl get hpa
```
Verify KEDA pods are running properly or not: `kubectget pods -n keda` <br>

You can add **multiple triggers in scaledobject.**<br>
Just add one more block.<br>


## Verify prometheus scraping metrics or not:
```
# forward port for temporary testing:
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9095:9090

curl http://localhost:9095/api/v1/targets | jq
curl -s http://localhost:9095/api/v1/query?query=requests_total | jq
```

### Now we will do Load Test:

Reduce the value for testing: <br>
```
while true; do   curl -s http://10.98.63.182:8000/predict -X POST -H "Content-Type: application/json"      -d '{"text":"hello"}' > /dev/null; done
```
<img width="1145" height="177" alt="image" src="https://github.com/user-attachments/assets/03b7ac49-d30c-47ec-94bb-450282ab0dea" />


## Troubleshooting:

1. If trying to delete sclaedobject and it is taking time to delete then,
List all the CRD's <br>
```
kubectl get crds | grep -i scale
```
Scaledobject may not delted due to finalizers, check finalizer.
```
kubectl get scaledobject fastapi-sentiment-keda -o yaml
```

Remove it by patching finalizer: <br>
```
kubectl patch scaledobject fastapi-sentiment-keda -p '{"metadata":{"finalizers":null}}' --type=merge
```

List CRD: <br>
```
kubectl get crds | grep -i scale
```
Remove scaled object:
```
kubectl patch crd scaledobjects.keda.sh --type=merge -p '{"metadata":{"finalizers":[]}}'
```

Forward Port from local:
```
ssh -L 9095:localhost:9090 merai@10.x.x.x -pxxxx

kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9095:9090

## Now open below link in local : http://localhost:9095

```

2. If You are using **kube-prometheus-stack** then annotations will not work. (unless you explicitly enable annotation-based scraping) < br>
  **Kube-prometheus-stack** : means prometheus + Grafana installed with Helm. <br>

  By default, **kube-prometheus-stack** requires a **ServiceMonitor** or **PodMonitor**. <br>
  
Scale to Zero:
--------------
KEDA polls Prometheus every 15 seconds <br>
Using your PromQL: <br>
```
sum(rate(requests_total[1m]))
```
KEDA receives a number = QPS. <br>
Example: <br>
0.00 → no traffic <br>
0.1 → very low traffic <br>
5.0 → heavy traffic <br>

Compare value to activationThreshold in scaledObject: <br>
```
activationThreshold: "0.1"
```
Set minReplicaCount = 0 <br>
```
minReplicaCount: 0
```

So, ScaledObject it will be like: <br>
```YAML
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: fastapi-sentiment-keda
  namespace: default
spec:
  scaleTargetRef:
    name: fastapi-sentiment   # your deployment name

  minReplicaCount: 0    ## here is for scale to zero
  maxReplicaCount: 10

  cooldownPeriod: 30
  pollingInterval: 15

  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus-kube-prometheus-prometheus.monitoring.svc:9090
        metricName: requests_qps
        query: |
          sum(rate(requests_total[1m]))
        threshold: "0.02"
        activationThreshold: "0.1"
```
But Scale to zero only works well with the Queue. ( kafka, Rabbitmq ) To doing the same for the web applications you need to configure **HTTP-Addon for KEDA**. <br>

KEDA HTTP Addon works well with Regular web apps: <br>
✔ Handles real **HTTP traffic** <br>
✔ Works even when **pods = 0** <br>
✔ Buffers requests during cold-start <br>
✔ Scales based on actual demand, not metrics <br>
✔ Safer and faster than Prometheus scaling <br>
✔ Designed specifically for web APIs (FastAPI, Flask, Node.js, Django, Go, etc.) <br>

### ✅ 3 Ways KEDA Can Scale to Zero: <br>

| Method                                                      | Scale to Zero?       | Best For                      | Works for Normal Web Apps?          |
| ----------------------------------------------------------- | -------------------- | ----------------------------- | ----------------------------------- |
| **Queue-based Scalers** (RabbitMQ, Kafka, Redis, SQS, etc.) | ✅ YES                | Background jobs               | ❌ Not for HTTP APIs                 |
| **Prometheus Scaler**                                       | ⚠️ NO (not for HTTP) | Microservices with metrics    | ❌ Cannot scale from 0 for HTTP apps |
| **KEDA HTTP Addon**                                         | ⭐ YES                | Normal web APIs/HTTP services | ✅ YES                               |


### Setup KEDA HTTP Addon: <br>
```BASH
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm upgrade --install keda kedacore/keda -n keda --create-namespace


helm upgrade --install keda-add-ons-http \
  kedacore/keda-add-ons-http \
  -n keda
```
Verify:
```BASH
kubectl get all -n keda
```

Create **http-scaledObject.yaml**
```YAML
apiVersion: http.keda.sh/v1alpha1
kind: HTTPScaledObject
metadata:
  name: fastapi-http-scaler
  namespace: default
spec:
  hosts:
    - "*"
  pathPrefixes:
    - "/"
  scaleTargetRef:
    service: fastapi-sentiment-service
    port: 8000
  replicas:
    min: 0
    max: 10
  scalingMetric:
    concurrency:
      targetValue: 100
```
**Apply:**
- Remove the old ScaledObject. <br>
- HTTP-Add on **does not create starndard HPA**. <br>
      - Uses its own controller + interceptor + queue proxy to scale your deployment. <br>
      - Applies scaling internally via the KEDA operator. <br>
      - so,  `kubectl get hpa` **won’t show anything.**
  
```
kubectl apply -f http-scaledobject.yaml
```

Verify scaling:
```
kubectl get deploy fastapi-sentiment -w
```

Scale to zero:
--------------

Install http-add-on:
```
helm repo add kedacore https://kedacore.github.io/charts
helm repo update

helm upgrade --install http-add-on kedacore/keda-add-ons-http \
  --namespace keda \
  --create-namespace \
  --version 0.11.1

```

check:
```
kubectl get all -n keda
```

Create `HTTPScaledObject.yaml`.

```
apiVersion: http.keda.sh/v1alpha1
kind: HTTPScaledObject
metadata:
  name: sbdd-frontend1-scale
  namespace: numol
spec:
  scaleTargetRef:
    name: sbdd-frontend1
    service: sbdd-frontend1    # ✅ moved here
    port: 3000

  hosts:
  - "10.10.110.25:30033"

  replicas:
    min: 0
    max: 2

  # 🔹 Request presence (yes/no)
  targetPendingRequests: 1

  # 🔹 Idle duration
  scaledownPeriod: 180

```
Create `Keda-HTTP-proxy.yaml`
```
apiVersion: v1
kind: Service
metadata:
  name: keda-http-interceptor-nodeport
  namespace: keda
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: http-add-on
  ports:
  - name: http
    port: 8080
    targetPort: 8080
    nodePort: 30033
```
Apply:
```
kubectl apply -f http-scaled-object.yaml
kubectl apply -f keda-HTTP-proxy.yaml
```

### test: 
Ine browser open link,
```
http://server-ip:30033
```

Now pod will spin up. and it will be there till 300 seconds.

Troubleshooting:
1. Request flow
```
Browser (Host: xx.xx.xxx.xx)
 ↓
NodeIP:30033
 ↓
keda-add-ons-http-interceptor
 ↓
HTTPScaledObject (host match ✅)
 ↓
xxxx-frontend1 Service
 ↓
xxxx-frontend1 Pod
```


Cron Scaling:
-----------
- Scaled up.down based on time
- Independent of CPU/memory
This is used for:
- Night time
- Weekend
- Office hours worjloads.

ScaledObject.yaml
```
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: sbdd-frontend-cron
  namespace: numol
spec:
  scaleTargetRef:
    name: sbdd-frontend
  minReplicaCount: 1
  maxReplicaCount: 5
  triggers:
  - type: cron
    metadata:
      timezone: Asia/Kolkata
      start: "0 9 * * *"     # 9 AM
      end:   "0 21 * * *"    # 9 PM
      desiredReplicas: "3"
```

## GPU Memory calculation for H100 GPU'S  Cluster only.
```
  - type: prometheus
    metadata:
      name: gpu_memory_utilization
      serverAddress: http://kube-prometheus-stack-prometheus.monitoring.svc:9090
      metricName: gpu_memory_utilization
      threshold: "96"
      query: |
        max(
          DCGM_FI_DEV_FB_USED{
            exported_namespace="vllm",
            exported_pod=~"vllm-llama.*"
          }
        ) / 81920 * 100    ## this is according to H100 - 80GB RAM

```
## GPU Memory Calculation for Mixed Cluster - H100 and H200.
```
  ## below is work according to any GPU h100 and H200
  - type: prometheus
    metadata:
      name: gpu_memory_utilization
      serverAddress: http://kube-prometheus-stack-prometheus.monitoring.svc:9090
      metricName: gpu_memory_utilization
      threshold: "95"
      query: |
        max(
          DCGM_FI_DEV_FB_USED{
            exported_namespace="vllm",
            exported_pod=~"vllm-llama.*"
          }
        )
        /
        (
          max(
            DCGM_FI_DEV_FB_USED{
              exported_namespace="vllm",
              exported_pod=~"vllm-llama.*"
            }
          )
          +
          max(
            DCGM_FI_DEV_FB_FREE{
              exported_namespace="vllm",
              exported_pod=~"vllm-llama.*"
            }
          )
        )
        * 100
```
## Set Stabilization window
> Default Cooldown stabilization for HPA is 300 Seconds.
```
  # 👉 default stabilizatinwindow for HPA is 300 seconds.
  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 60
```
## ScaledObject for LLM models:
```
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: llama3-keda-scaler
  namespace: vllm
spec:
  scaleTargetRef:
    name: vllm-llama

  minReplicaCount: 1
  maxReplicaCount: 3

  pollingInterval: 10  # Every 10 seconds KEDA checks Prometheus metrics
  cooldownPeriod: 60  # KEDA waits 60 seconds after the last high load and scale down.

  # 👉 default stabilizatinwindow for HPA is 300 seconds.
  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 60

  triggers:

  - type: cpu
    metricType: Utilization
    metadata:
      value: "70"

  - type: memory
    metricType: Utilization
    metadata:
      value: "80"

    # Request per second.
  - type: prometheus
    metadata:
      serverAddress: http://kube-prometheus-stack-prometheus.monitoring.svc:9090
      metricName: llama3_requests
      threshold: "700"
      query: |
        sum(rate(istio_requests_total{destination_workload="vllm-llama"}[1m]))

  # GPU utilization scaling
  - type: prometheus
    metadata:
      name: gpu_utilization
      serverAddress: http://kube-prometheus-stack-prometheus.monitoring.svc:9090
      metricName: gpu_utilization
      threshold: "95"
      query: |
        avg(
          DCGM_FI_DEV_GPU_UTIL{namespace="vllm", pod=~"vllm-llama.*"}
        )
# average GPU memory usage of the deployment
#  - type: prometheus
#    metadata:
#      name: gpu_memory_utilization
#      serverAddress: http://kube-prometheus-stack-prometheus.monitoring.svc:9090
#      metricName: gpu_memory_utilization
#      threshold: "96"
#      query: |
#        max(
#          DCGM_FI_DEV_FB_USED{
#            exported_namespace="vllm",
#            exported_pod=~"vllm-llama.*"
#          }
#        ) / 81920 * 100    ## this is according to H100 - 80GB RAM

  ## below is work according to any GPU h100 and H200
  - type: prometheus
    metadata:
      name: gpu_memory_utilization
      serverAddress: http://kube-prometheus-stack-prometheus.monitoring.svc:9090
      metricName: gpu_memory_utilization
      threshold: "95"
      query: |
        max(
          DCGM_FI_DEV_FB_USED{
            exported_namespace="vllm",
            exported_pod=~"vllm-llama.*"
          }
        )
        /
        (
          max(
            DCGM_FI_DEV_FB_USED{
              exported_namespace="vllm",
              exported_pod=~"vllm-llama.*"
            }
          )
          +
          max(
            DCGM_FI_DEV_FB_FREE{
              exported_namespace="vllm",
              exported_pod=~"vllm-llama.*"
            }
          )
        )
        * 100
```
