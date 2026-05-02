#### Description:
- Pod contains containers.
- One pod contains multiple containers in some situaltions. ( for e.g. sidecar or monitoring container with frontend application)

#### Create pod:
```
kubectl run mypod --image=nginx:latest --port=80
```
#### To get detailed information of pod:
```
kubectl describe pod <pod-name>
kubectl describe pod <pod-name> -n <namespace>
```

#### Retrive logs from specific pod:
```
kubectl logs <pod-name>
```

#### Get the list of pods:
```
kubectl get pods
kubectl get pods -A
```

#### Get pods from all namespaces:
```
kubectl get pods --all-namespaces
```

#### Get pods from specific namespace:
```
kubectl get pods --namespace <namespace>
```

#### Multicontainer pod:

Below is the example.
```YAML
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
    - name: app-container
      image: nginx:latest
      ports:
        - containerPort: 80
    - name: sidecar-container
      image: busybox
      command: ["sh", "-c", "while true; do echo Sidecar running; sleep 5; done"]
```

##### To check how many containers are running in single pod.
```
kubectl get pods -o wide

kubectl describe pod <pod-name>
```

### Health checks :

- When you start application in pod it automatically keep alive for you using process health checks.
- This healthchecks simply ensures that the main process of your application is always running, if it isn’t it simply restarts it.
- But in some cases process healthchecks is not enough, for example if application went to deadlock, then healtchecks still belives that application is running. But actually it is not serving requests.
- To address this, kubernetes introduced healthchecks for application liveness.
- This will checks the application functioning properly or not. Since this liveness health checks are application specific, you have to define them in pod manifest.

### Liveness probe

- Liveness defined per container, which means each container inside the pod is healthchecked seperately.

`pod-health.yaml`

```YAML
apiVersion: v1
kind: Pod
metadata:
  name: kuard
spec:
  containers:
    - image: gcr.io/kuar-demo/kuard-amd64:blue
      name: kuard
      livenessProbe:
        httpGet:
          path: /healthy
          port: 8080
        initialDelaySeconds: 5
        timeoutSeconds: 1
        periodSeconds: 10
        failureThreshold: 3
      ports:
        - containerPort: 8080
          name: http
          protocol: TCP
```
### Readiness probe:

- Liveness determines if an application is running properly.container that fails liveness checks are restarted.
- Readiness describes when a container is ready to serve user requests.

### TCP Probe:
- If your application doesnt have /health or any other endpoints then you can use this for simple monitoring.
- - This will provides autostart support to application and in simple monitoring.
```YAML
        # 🟢 TCP-based liveness probe (checks if app is reachable)
        livenessProbe:
          tcpSocket:
            port: 8000
          initialDelaySeconds: 5     # wait before first check
          periodSeconds: 10          # check every 10s
          failureThreshold: 3        # after 3 failures, restart

        # 🟢 TCP-based readiness probe (checks if app is ready for traffic)
        readinessProbe:
          tcpSocket:
            port: 8000
          initialDelaySeconds: 3
          periodSeconds: 10

```
Kubernetes tries to open a TCP connection on port 8000: <br>
- If it connects → pod is alive and ready
- If it fails 3 times → pod is restarted
- No HTTP endpoint required.

### Startup probe:

- Alternative way to manage slow starting containers.

### Resource management:
- we can assign resources to pod like CPU and Memory.
- Below is the sample code.

https://github.com/purvalpatel/kubernetes-sample/blob/1fb6d3149f3792373d261079a0dabfbd7a27ae86/introduction/resource-management-pod.yaml

### Capping resources to pod:

- We can provide minimum and maximum resource utilization. In our previous example we have used minimum resources.

https://github.com/purvalpatel/kubernetes-sample/blob/f67fc4d698e7e4ca4dd8fe0322d271c8b5d752d6/introduction/min-max-resources-pod.yaml

### Persist data with volumes:

https://github.com/purvalpatel/kubernetes-sample/blob/8d357405736b1a88cccdef9b96b69201d719e961/introduction/data-persist-volume.yaml


## Forcefully termninate the pod:
```
kubectl delete pod <pod-name> --grace-period=0 --force --namespace <namespace>
```

### Delete all resources from specific namespace:
```
kubectl delete all --all -n <namespace>
```
# PriorityClass

The Kubernetes scheduler maintains a queue of pending Pods. When resources are constrained:
1. Pods with Higher priority are scheduled before lower-priority pods.
2. If neccessary, kubernetes can evicat lower-priority pods to make room.

### How it works internally.
- Each pods get the priority value from PriorityClassName.
    - Higher integer → Higher priority.
    - default priority (0) if none assigned.
 
Scheduler flow:
1. Pod enters scheduling queue.
2. Sorted by priority ( decending )
3. Scheduler tries to Highest priority pod first.'

Example of PriorityClass:
```
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 100000
globalDefault: false
description: "High priority for critical workloads"
```
Description:
```
value: Integer priority (can be negative or positive)
globalDefault: If true → all Pods without a class get this
preemptionPolicy (optional):
PreemptLowerPriority (default)
Never (no preemption)
```
Use it into pod or deployment:
```
apiVersion: v1
kind: Pod
metadata:
  name: critical-pod
spec:
  priorityClassName: high-priority
  containers:
  - name: app
    image: nginx
```
## Preemption Behavior (Important)
| Pod           | Priority | CPU Required |
| ------------- | -------- | ------------ |
| A             | 100000   | 2 cores      |
| B             | 1000     | 2 cores      |
| Node capacity | 2 cores  |              |

If B is running and A comes: <br>
➡ Kubernetes **kills** Pod B <br>
➡ **Schedules Pod A** <br>
This is called **preemption**. <br>

### Use  `preemptionPolicy: Never`
- Pod will wait untill killing others.


## Use case:
Application is running on Pod-A with 2 cores. you have only 2 cores. if i schedule another same application then it went to pending.
- here i have to use priority class with more integer value then older one.
- here preemption kicks in.
- Old pod gets deleted and new one created.

> It can delete ANY lower-priority Pod.

