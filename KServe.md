Introduction
----
- Platform for bot h Generative and Predictive AI inference on kubernetes
- Deploy,Scale and Manage ML Models In production.

### In Simple words:
- **K8s** Manages Containers.
- **Kserve** Manages ML Models.

**Traditional Kubernetes Deployment** contains:
- Deployment,
- Services
- Autoscales
- Rollouts
- etc.

**Kserve** contains:
- Deploy Models and make it available as an API.

### Why Kserve?
- Expose it as an API
- Scale it based on traffic
- Update to new model versions
- Roll back if something breaks
- Monitor it
- Keep it stable

### What KServe does automatically:
- Creates Deployment
- Creates Service
- Creates Autoscaler
- Exposes endpoint
- Handles scaling up/down

```
Training -> Save Model -> Deployment with Kserve -> Application sends requests -> Models returns prediction.
```
## Real World Use Case 1 — API Serving
- Deploy All Models
- Scale based on traffic
- Monitor Health

## Real World Use Case 2 — LLM Deployment
If serving models using vLLM. <br>
and you dont want to manage Deployment, Services, Autoscaling etc. <br>

KServe allows canary deployment. <br>

## Real World Use Case 3 — AutoScaling
With Kserve:
- Scale based on traffic
- Scale to zero ( For small models )
- Scale up automatically with traffic increases.

| Tool       | What It Does            |
| ---------- | ----------------------- |
| Kubernetes | Manages containers      |
| Docker     | Runs containers         |
| vLLM       | Runs LLM inference      |
| Triton     | Runs ML inference       |
| **KServe** | Manages model lifecycle |

## When Should You Use KServe?
Use KServe if:
- You run models in Kubernetes
- You need autoscaling
- You need version control
- You serve multiple models
- You want production-grade ML infra

Installation:
----

1. Setup `cert-manager` first.
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

2. Setup `Kserve`
```
kubectl apply -f https://github.com/kserve/kserve/releases/download/v0.11.0/kserve.yaml
```
<img width="1210" height="590" alt="image" src="https://github.com/user-attachments/assets/7610a12d-cf75-4861-a59b-a03b86151db5" />

Verify:
```
kubectl get all -n kserve
```
<img width="1084" height="292" alt="image" src="https://github.com/user-attachments/assets/1f9aae7e-42c6-4edc-ab7f-6a834accc0cc" />

3. Install `Knative` [optional]
```
kubectl apply -f https://github.com/knative/serving/releases/latest/download/serving-crds.yaml
kubectl apply -f https://github.com/knative/serving/releases/latest/download/serving-core.yaml
```

Model vs. LLM
-------------

### Model
Any Machine learning program trained on data to make predictions. <br>

example, 
- spam detection model
- image recognition model
- Recommendataion model
- Language model

Trained AI system that perfoem a task. <br>


### LLM : Lanrge language model

- Specific type of model trained on huge text, datasets to understand & generate language. <br>
GPT4, Llama2, Mistral


vehicle = Model <br>
Car = LLM <Br>

| Feature    | Model         | LLM                  |
| ---------- | ------------- | -------------------- |
| Definition | any ML model  | large language model |
| Data used  | any data      | text data            |
| Size       | small → large | very large           |
| Use case   | many tasks    | language tasks       |


**In Kserve we can define vLLM runtime** <br>

### Architecture:
```
User
  │
  ▼
KServe (deployment + autoscaling)
  │
  ▼
vLLM (fast inference engine)
  │
  ▼
LLM (Llama / Mistral)
```

### Example YAML:
```YAML
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: llama2
spec:
  predictor:
    model:
      modelFormat:
        name: vllm
      storageUri: "hf://meta-llama/Llama-2-7b-chat-hf"
      runtime: kserve-vllm-runtime
```
