# introduction
- Multicluster management system built on Kubernetes.
- It Lets you manage multiple kubernetes clusters as they are one.

```
# without kubeAdmiral:
kubectl --context cluster-A apply -f app.yaml
kubectl --context cluster-B apply -f app.yaml
(Manage each cluster separately)

# With KubeAdmiral:
kubectl apply -f app.yaml
(KubeAdmiral distributes it A,B Automatically)
```

```
┌─────────────────────────────────────────┐
│           KubeAdmiral                   │
│         (Control Plane)                 │
│                                         │
│  Single API → Policy → Distribution     │
└──────────┬──────────┬───────────────────┘
           │          │          │
           ▼          ▼          ▼
       Cluster A   Cluster B   Cluster C
       ap-south-1  us-east-1   eu-west-1
```

> It handles all clusters.

## Core Concepts:
1. FederatedObject (Deploy to multiple clusters)
2. PropogationPolicy (Rules for Distribution)
3. ClusterSelector (Different config per cluster)

## Without kubeAdmiral:
OOM On Cluster A 
→ You manually delete job on A
→ You manually submit job on C
→ You manage credentials for each cluster separately
→ You track state acrosss clusters manually


## With kubeAdmiral:
OOM On Cluster A 
→ KubeAdmiral detects the pod unschedulable on A
→ Automatically migrates to cluster C
→ Single Control plane handles everything.

