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

If error then try latest version:
kubectl apply -f https://github.com/kserve/kserve/releases/latest/download/kserve.yaml

If still error of size, 262144 bytes
use kubectl create for CRS.
kubectl create -f https://github.com/kserve/kserve/releases/latest/download/kserve.yaml
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


---------------------------
### STEP 1 -- Sample program for generating model file [Optional]
Create, `sample_model_create.py`
```
# train_model.py
import joblib
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier

# Load the Iris dataset
iris = load_iris()
X_train, X_test, y_train, y_test = train_test_split(iris.data, iris.target, test_size=0.2)

# Train the model
model = RandomForestClassifier(n_estimators=100)
model.fit(X_train, y_train)

# Save the model to a file
joblib.dump(model, "iris_model.pkl")
```
Run:
```
python sample_model_create.py
```
Store model into S3 registry: `iris_model.pkl`
### STEP 2 -- install CRDS:
```
kubectl apply -f https://github.com/kserve/kserve/releases/latest/download/kserve.yaml
```

Use `create` instead of apply if getting error of Size,
```
kubectl create -f https://github.com/kserve/kserve/releases/latest/download/kserve.yaml
or
kubectl apply --server-side -f https://github.com/kserve/kserve/releases/latest/download/kserve.yaml
```

For example, Model is stored in `s3://numol/test-purval/iris_model.pkl` <br>

### STEP 3 -- Create secret in kubernetes for s3 :
```
kubectl create secret generic s3-secret \
--from-literal=AWS_ACCESS_KEY_ID=0B3AxN861NUCZ5xxG1T61WTx \
--from-literal=AWS_SECRET_ACCESS_KEY=_pUoc4IqCbxxYX2LC87w7KU8Kfjxg11gK83id50A408
```
OR by YAML, `s3-secret.yaml`    [Suggested]
```
apiVersion: v1
kind: Secret
metadata:
  name: netapp-s3-secret
  annotations:
    serving.kserve.io/s3-endpoint: mns3011.xxxx-ai.com
    serving.kserve.io/s3-usehttps: "0"
    serving.kserve.io/s3-verifyssl: "0"
type: Opaque
stringData:
  AWS_ACCESS_KEY_ID: 0B3AN86s1NUCZd5G1T61WTx
  AWS_SECRET_ACCESS_KEY: _pUoc4IqCbYXxx2LC87w7KU8Kfjg11xgK83id50A408
```

### STEP 4 -- Create serviceAccount for Kserve, `serviceaccount.yaml`:
```
apiVersion: v1
kind: ServiceAccount
metadata: 
  name: s3-sa
secrets:
- name: netapp-s3-secret
```

### STEP 5 -- Create inference service `model.yaml`:

```
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: sklearn-model
spec:
  predictor:
    serviceAccountName: s3-sa
    sklearn:
      storageUri: "s3://numol/test-purval/"
      image: kserve/sklearnserver:latest
```

Apply:
```
kubectl apply -f model.yaml
```

List: 
```
kubectl get inferenceservice
```

Output:
```
NAME            URL   READY   PREV   LATEST   PREVROLLEDOUTREVISION    LATESTREADYREVISION   AGE
sklearn-model
```
### Error 1:
***Error : It is not possible to use Knative deployment mode when Knative Services are not available***

if showing like this then there is  issue:
```
kubectl describe inferenceservice sklearn-model
```

####  Now install Knative: [optional]
CRDS:
```
kubectl apply -f https://github.com/knative/serving/releases/latest/download/serving-crds.yaml
```

Core Components:
```
kubectl apply -f https://github.com/knative/serving/releases/latest/download/serving-core.yaml
```

Networking : 
```
kubectl apply -f https://github.com/knative/net-istio/releases/latest/download/net-istio.yaml
```

verify:
```
kubectl get pods -n knative-serving
```

Note: ***Disable Knative*** If you don't need serverless scaling, you can run KServe in Raw Deployment Mode.

### Edit configmap:
```
kubectl edit configmap inferenceservice-config -n kserve
```
change:
```
deploy:
  defaultDeploymentMode: serverless
```
To 
```
deploy:
  defaultDeploymentMode: RawDeployment
```
- **Restart** all Kserve Pods.

- ***RawDeployment*** is best for OnPremises clusters.
- ***Knative*** is for Serveless and scaling.

### Error 2:
After edit got this another error: <br>
***no runtime found to support predictor with model type***

```
  Type     Reason         Age                   From                Message
  ----     ------         ----                  ----                -------
  Warning  InternalError  28s (x16 over 3m12s)  v1beta1Controllers  no runtime found to support predictor with model type: {sklearn <nil>}
```

***In older setups with Knative, KServe automatically installs runtimes.***

But in ***RawDeployment*** mode, you must install runtimes manually like:
```
sklearn runtime
pytorch runtime
tensorflow runtime
triton runtime
vllm runtime
```

### STEP 6 -- Install runtime:
```
kubectl apply -n kserve -f https://raw.githubusercontent.com/kserve/kserve/release-0.12/config/runtimes/kserve-sklearnserver.yaml
```

verify:
```
kubectl get clusterservingruntime
```

Okay, now pod is running:
```
kubectl get inferenceservice
```
Output:
```
NAME            URL                                        READY   PREV   LATEST   PREVROLLEDOUTREVISION   LATESTREADYREVISION   AGE
sklearn-model   http://sklearn-model-default.example.com   True                                                                  3h10m
```

Kserve will create automatic k8s resources:
```
kubectl get svc 
```
### STEP 7 --Test Mode Inferencing:

Forward port for testing:

```
kubectl port-forward svc/sklearn-model-predictor 8082:80
```
Test  with curl:
```
curl -X POST http://localhost:8082/v1/models/sklearn-model:predict \
     -H "Content-Type: application/json" \
     -d '{
           "instances": [
             [5.1, 3.5, 1.4, 0.2],
             [6.2, 3.4, 5.4, 2.3]
           ]
         }'
{"predictions":[0,2]}
```

output:
```
{"predictions":[0,2]}
```
