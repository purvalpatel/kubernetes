Observability with servicemesh:
------------------------------
```
metrics -> Prometheus -> Grafana
Logs -> Loki -> Grafana
Traces -> Jaeger/Tempo -> UI
```
**Here we are testing for the 1st one. metrics collection with servicemesh.**

### Step 1:  Istio Sidecar injected with the namespace.

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
    targetPort: 15090

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



## Logs Monitoring with Loki in Grafana (Kubernetes)
### Setup prometheus. (optional if not done)
- Fllow link - https://github.com/purvalpatel/kubernetes-tutorial/blob/d08d13ee86fd6766f6181e1659aa13bed3a9fa80/kube-prometheus-grafana-stack.md 

- Note Loki `2.x.x` version ins not compatible with Grafana `12.x.x` version. the issue is loki is not connecting grafana from web ui. either connectivity works from backend. so for that need to use new version i.e. `3.x.x`.
- check Grafana version : `kubectl describe pod prometheus-grafana-6444876565-84p67 -n monitoring | grep Image`
- check loki version: `kubectl -n monitoring get pod loki-0 -o jsonpath="{.spec.containers[*].image}"`
Install Loki
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

Verify it is being installed or not:
```
helm list -n monitoring

kubectl get pods -n monitoring
```

Check the connectivity now from Grafana: <br>
<img width="835" height="233" alt="image" src="https://github.com/user-attachments/assets/ee33b251-3f75-42a9-8e30-ae3267b25183" />
