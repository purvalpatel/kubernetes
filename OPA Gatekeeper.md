OPA Gatekeeper is Kubernetes policy engine.

It allows you to defines rules for waht is allowed or forbidden in your cluster.

Think of it as:
```
Kubernetes admission security + Governance layer
```

### High level architecture.
```
Developer
   ↓
kubectl apply
   ↓
Kubernetes API Server
   ↓
OPA Gatekeeper Admission Controller
   ↓
Policy Validation
   ↓
ALLOW or DENY
   ↓
Resource Created
```

### Why kubernetes needs this?
```
privileged containers
containers running as root
unlimited CPU/memory usage
public LoadBalancers
untrusted Docker images
host filesystem mounts
insecure capabilities
```

## installation:
```
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts

helm install gatekeeper/gatekeeper \
  --name-template=gatekeeper \
  --namespace gatekeeper-system \
  --create-namespace

## Verify
kubectl get pods -n gatekeeper-system
```
You should see:
- controller-manager
- audit pod

## Audit Mode

Gatekeeper can also scan existing resources.

Example:
```
These 12 deployments violate policy
```
without blocking them yet.

Very useful for gradual rollout.

## Gatekeeper vs Kyverno

| Feature                | Gatekeeper  | Kyverno      |
| ---------------------- | ----------- | ------------ |
| Policy language        | Rego        | YAML         |
| Flexibility            | Very high   | Medium       |
| Ease of use            | Harder      | Easier       |
| Enterprise usage       | Very common | Growing fast |
| Kubernetes-native feel | Lower       | Higher       |

## Kubernetes Native Way → LimitRange
Gatekeeper
```
validate + deny
```
LimitRange
```
auto-default resources
```

Flow:
```
User deploys pod
        ↓
LimitRange adds defaults
        ↓
Gatekeeper validates rules
        ↓
Pod accepted
```

## Real world patterns
| Task                  | Best Tool     |
| --------------------- | ------------- |
| Set default resources | LimitRange    |
| Enforce policies      | Gatekeeper    |
| Mutate manifests      | Kyverno       |
| Namespace quotas      | ResourceQuota |
