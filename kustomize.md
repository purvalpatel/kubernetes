Kustomize:
-----------
Kustomize is a **Kubernetes-native configuration management tool** that lets you customize YAML manifests — without needing to modify the original files.<br>
it's like docker-compose for kubernetes.<br>

**Why kustomize exists:**
When you manage multiple environments (like dev, staging, prod), you often have almost the same Kubernetes manifests, differing only in a few fields — like:<br>
- image tag
- replica count
- config values
- resource limits

Instead of copying and editing YAMLs for each environment, Kustomize lets you define a base configuration and overlay environment-specific changes.

For example <br>

app/ <br>
├── base/ <br>
│   ├── deployment.yaml <br>
│   ├── service.yaml <br>
│   └── kustomization.yaml <br>
└── overlays/ <br>
    ├── dev/ <br> 
    │   └── kustomization.yaml <br>
    ├── staging/ <br> 
    │   └── kustomization.yaml  <br>
    └── prod/ <br>
        └── kustomization.yaml <br>

base kustomization.yaml. <br>
```
resources:
  - deployment.yaml
  - service.yaml
```

Here is the configuration of dev/prod/test environment.<br>
overlays/dev/kustomization.yaml
```
resources:
  - ../../base

namePrefix: dev-

commonLabels:
  env: dev

images:
  - name: nginx
    newTag: 1.27.1

```
apply:
```
kubectl apply -k .
```

Capabilities of kustomize:
------------------------
### 1️⃣ Combine Multiple Resources:
```
resources:
  - deployment.yaml
  - service.yaml
  - hpa.yaml
```
This tells Kustomize: <br>
Combine these into one final output.

### 2️⃣ Override Container Images
```
images:
  - name: docker.linux.app/purvAI/user-service
    newTag: abc123
```
Overrides image tag without editing deployment file. <br>

Very useful for CI/CD.

### 3️⃣ Add Labels to Everything
```
commonLabels:
  app: purvAI
  environment: dev
```
Adds labels to:
- Deployments
- Services
- Pods
- Everything

### 4️⃣ Add Annotations Globally
```
commonAnnotations:
  owner: devops
```

### 5️⃣ Change Namespace Automatically
```
namespace: purvAI-dev
```
Applies namespace to all resources.

### 6️⃣ Add Name Prefix or Suffix
```
namePrefix: dev-
nameSuffix: -v1
```
Transforms:
```
user-service → dev-user-service-v1
```
Useful for multiple environments.

### 7️⃣ Modify Specific Fields (Patching)

You can modify:
- replicas
- resources
- env vars
- probes
- any field

Using: <br>
patches: <br>
or
```
patchesStrategicMerge:
```
### 8️⃣ Generate ConfigMaps

Instead of writing YAML manually:
```
configMapGenerator:
  - name: app-config
    literals:
      - LOG_LEVEL=debug
      - FEATURE_FLAG=true
```
Kustomize generates full ConfigMap.

### 9️⃣ Generate Secrets
```
secretGenerator:
  - name: db-secret
    literals:
      - password=abc123
```
### 🔟 Manage Multiple Environments (Overlay Pattern)

You can create:
```
base/
overlays/dev/
overlays/prod/
```
