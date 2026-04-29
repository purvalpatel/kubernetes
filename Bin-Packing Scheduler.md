# Bin packing scheduler
- It is Scheduling strategy that tries to pack workloads tightly on to the fewest number of nodes leaving the other nodes free.

## Core Idea:
```
Node = bin
Pods = items
```

## Goal:
- Fit items into bins such as minimum number of bins are used.
- minimum number of bins are used.
- Leftover space is minimized.

## Without bin packing:
- Node 1: 30%
- Node 2: 30%
- Node 3: 10%

## with bin packing:
- Node1: 90 %
- Node2: 90%
- Node3: 0% (could be removed)

> Kubernetes default scheduler is balanced, not bin packing. <br>
> But you can biased it towrds bin packing.

## How Bin Packing Achieved:
1. Resource requests
Scheduler uses:
```
resources:
  requests:
    cpu: "2"
    memory: "4Gi"
```

If requests are accurate:

👉 Scheduler can tightly pack pods <br>
👉 If not → bin packing breaks <br>

2. Scoring strategy
Kubernetes uses scoring plugins like:
```
NodeResourcesFit
NodeResourcesLeastAllocated
NodeResourcesMostAllocated
```
For bin packing, you want:
```
NodeResourcesMostAllocated
```
Meaning: <br>

👉 Prefer nodes that are already heavily used <br>

3. Pod Affinity/anti-affinity <br>
👉 Try placing pods on same node.


## Where Bin packing is critical.
1. Auto sclaing clusters
2. GPU Workloads.

## Better solution for you

1. **Volcano scheduler** (strong recommendation)

It provides:
- Bin packing
- Gang scheduling (important for multi-GPU jobs)
- Queue-based scheduling
- Fair share across users

2. **Kueue** (modern Kubernetes-native queue)

- Works with Jobs
- Supports quotas
- Integrates with default scheduler
