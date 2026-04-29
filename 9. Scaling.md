Autoscaling:
-------------
Up or Down the resources according to usage.

Think of it like:<br><br>

When your website gets more visitors → Kubernetes automatically adds more servers (Pods).<br>
When traffic goes down → It removes extra ones to save cost.<br>

Methods: <br>
| Type                                | What it Scales          | Example Use                         |
| ----------------------------------- | ----------------------- | ----------------------------------- |
| **HPA** – Horizontal Pod Autoscaler | Number of Pods          | More traffic → more Pods            |
| **VPA** – Vertical Pod Autoscaler   | CPU/memory for each Pod | Same Pods → give them more power    |
| **CA** – Cluster Autoscaler         | Number of Nodes         | Adds/removes Nodes from the cluster |


Two scale commands are there,
1. Scale
2. autoscale ( it uses HPA by default )
   
### 🧩 1. Horizontal Pod Autoscaler (HPA)
- **Increases or decreases Pod replicas automatically based on CPU or memory usage.**
HPA by default comes with the kubernetes.

You already have a Deployment:<br>
```
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80
```

Now enable autoscaling<br>
```
kubectl autoscale deployment nginx --min=2 --max=10 --cpu-percent=70
```


Below is the YAML of horizontal pod autoscaler.

```YAML
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

- Minimum 2 Pods
- Maximum 10 Pods
- If CPU usage goes above 70%, add more Pods
- If CPU usage drops below, remove Pods

Check status:
```
kubectl get hpa
```

<img width="979" height="71" alt="image" src="https://github.com/user-attachments/assets/4ccd68bd-0aaf-494f-b421-c2eefb4891fa" />

HPA autoscaler will not show the metrices without metrics-server installed. <br>
⚠️ HPA cannot read CPU metrics <br>
⚠️ Your cluster does NOT have metrics-server installed <br>
⚠️ So HPA will NEVER autoscale <br>
```
kubectl top pod
```
If this is not showing then, <br>
install metrics server:
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
Edit:
```
kubectl -n kube-system edit deployment metrics-server
```
Add this under `spec.template.spec.containers[0].args`:
```
        - --kubelet-insecure-tls
        - --kubelet-preferred-address-types=InternalIP
```
Then restart,
```
kubectl rollout restart deployment metrics-server -n kube-system
```
Now check,
```
kubectl top pod
```
<img width="656" height="73" alt="image" src="https://github.com/user-attachments/assets/cbc298b6-8d70-4616-8273-b2190f9b49ec" />

```
kubectl get hpa
```
<img width="890" height="68" alt="image" src="https://github.com/user-attachments/assets/ebe4075b-6d27-40dc-a30b-844bfbb0bfe7" />

Note: <br>
HPA does NOT generate metrics. It only reads metrics from Kubernetes APIs. <br>

HPA can read three types of metrics:  <br>
1️⃣ Resource metrics → CPU / Memory <br>
2️⃣ Custom metrics → App-specific metrics (QPS, latency, queue size, Kafka lag…) <br>
3️⃣ External metrics → Metrics outside the cluster (CloudWatch, Pub/Sub, RabbitMQ….) <br>

| API                     | Provided By               | Required For                       |
| ----------------------- | ------------------------- | ---------------------------------- |
| metrics.k8s.io          | metrics-server            | CPU/Memory autoscaling             |
| custom.metrics.k8s.io   | Prometheus Adapter        | app custom metric autoscaling      |
| external.metrics.k8s.io | Prometheus Adapter / KEDA | cloud/external metrics autoscaling |

Autoscaling example described here: <br>
https://github.com/purvalpatel/simple-model-train-deployment/blob/main/Model-Deployment-k8s.md

### 🧩 2. Vertical Pod Autoscaler (VPA)
While HPA scales the number of pods <br>
VPA Adjusts CPU and memory requests/limits for Pods automatically.<br>
VPA doesn't come with kubernetes by default, it is seperate project you can get from github.<br>

#### Install VerticalPodAutoscaler:<br>
```
git clone https://github.com/kubernetes/autoscaler.git
cd vertical-pod-autoscaler
./hack/vpa-up.sh
```
Now it is installed, you can verify with:
```
kubectl describe vpa
kubectl --namespace=kube-system get pods|grep vpa
kubectl get customresourcedefinition | grep verticalpodautoscalers
```

Beelow is the example of VPA configuration:

Below is the Auto Assignment of resources.<br>
Used for databases, stateful applications, frontend <br>

```YAML
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: nginx-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: nginx
  updatePolicy:
    updateMode: "Auto"  # Can be "Off", "Initial", or "Auto"
```
Apply it:
```
kubectl apply -f vpa.yaml
```
VPA needs time to collect metrices(After some time):
```
kubectl describe vpa nginx-vpa
```
It will show lower bound and upper bound.

### cleanup VPA:
```
kubectl delete vpa nginx-vpa
kubectl delete deployment nginx-deployment
```
### We can define resourcePolicy with MinAllowed and MaxAllowed:

This is manual restriction of resources <br>
which is used for predictable workloads.<br>
```
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: nginx-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  updatePolicy:
    updateMode: "Auto"   # Off | Initial | Auto
  resourcePolicy:
    containerPolicies:
    - containerName: nginx
      minAllowed:
        cpu: "100m"
        memory: "128Mi"
      maxAllowed:
        cpu: "500m"
        memory: "512Mi"
      controlledResources: ["cpu", "memory"]

```
#### Limitations:
Whenever VPA updates the pod resources, the pod is recreated, which causes all running containers to be recreated. The pod may be recreated on a different node.

### 🧩 3. Cluster Autoscaler (CA)
if Pods can’t start because there aren’t enough Nodes, Cluster Autoscaler adds new Nodes (VMs). <br>
When load decreases, it removes unused Nodes. <br>

This usually works in cloud environments like AWS, GCP, Azure, etc.<br>


Other Auto scaling Techniques:
----------------------
✅ **GPU-based** autoscaling  <br>
✅ **KEDA-based** autoscaling (queue / event driven)  <br>
✅ **Prometheus-based** autoscaling  <br>
✅ **Istio autoscaling** using metrics <br>

### If You have: <br>
GPU workloads (H100, 4050, MIG, JupyterHub) <br>
Prometheus <br>
Mixed microservices <br>
Heavy observability <br>

### Event-driven systems (potentially) <br>
**✔ Best combo for you:** <br>

🔹 1. **GPU workloads** → GPU-based autoscaling (DCGM + Prometheus) <br>
      No alternatives. BEST for H100/MIG. <br>

🔹 2. **Async/Queue workloads** → KEDA <br>
      Mandatory if using RabbitMQ/Kafka/SQS. <br>

🔹 3. **Microservices API workloads** → Prometheus Adapter <br>
      Most stable and production proven. <br>

🔹 4. **Istio-based traffic** → Istio autoscaling <br>
      Only if you run Istio mesh. <br>

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

### Now we will do Load Test:

Install some tool like, HTTP tool generator, **hey**<br>
```
hey -z 1m -c 10 http://<EXTERNAL-IP>
```

# Karpenter

## Kubernetes autoscaling has three layers.
| Layer         | Tool          | What it scales                      |
| ------------- | ------------- | ----------------------------------- |
| Pod replicas  | HPA / KEDA    | Number of pods                      |
| Pod resources | VPA           | CPU/memory requests                 |
| Worker nodes  | **Karpenter** | EC2/VM nodes for unschedulable pods |


### So usual flow is :
-> HPA creates more pods
-> Some pods become pending
-> Karpenter sees unschedulable pods
-> Karpenter provisions new-sized nodes
-> Karpenter scheduler places pods.

#### Main concepts of karpenter:
1. NodePool: what kind of nodes karpenter allowed to create. i.e. Zones, architectures, instance types, etc.

2. NodeClass : Provider specific infrastructure configuration. For AWS, it is called AWSNodeClass. It defines the instance type, AMI, etc.

3. NodeClaim : is the actual node capacity request created by karpenter.

> AWS Specifically recommends karpenter for clusters with changing capacity needs or diverse compute requirements.


## Cluster Autoscaler vs Karpenter:
- Cluster Autoscaler: Usually scales predefined node groups.  
- Karpenter: Provisions nodes directly based on pending pod requirements.

| Feature       | Cluster Autoscaler                     | Karpenter                                                |
| ------------- | -------------------------------------- | -------------------------------------------------------- |
| Scaling model | Scales existing node groups            | Provisions nodes more directly from flexible constraints |
| Node groups   | Required/pre-configured                | Not the central model                                    |
| Flexibility   | More limited by node group definitions | More dynamic instance selection                          |
| Scope         | Node autoscaling                       | Node autoscaling plus broader node lifecycle behavior    |


> HPA creates pods; Karpenter creates the nodes those pods need.
