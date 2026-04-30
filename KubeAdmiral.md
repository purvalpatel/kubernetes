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
OOM On Cluster A  <br>
→ You manually delete job on A <br>
→ You manually submit job on C <br>
→ You manage credentials for each cluster separately <br>
→ You track state acrosss clusters manually <br>


## With kubeAdmiral:
OOM On Cluster A <br>
→ KubeAdmiral detects the pod unschedulable on A <br>
→ Automatically migrates to cluster C <br>
→ Single Control plane handles everything <br>

