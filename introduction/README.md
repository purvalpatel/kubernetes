Kubernetes cluster contains two types of resources:
--------------------------------------------------
#### 1.Control plane		
cordinates the cluster ( responsible for managing cluster)

- Cloud-Controller-manager & Cloud provider API:<br>
  &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;The Cloud Provider API provides hooks for Kubernetes to request resources from the cloud, such as: Nodes, Loadbalanceres, Volumes, Route & networking.
- etcd:<br>
  &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Key-value database
- Kube-api-server:<br>
  &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Frontend for kubernetes control plane
- Kube-Scheduler:<br>
  &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Schedule pod on node.
- Kube-controller-manager  :<br>
  &nbsp;&nbsp;Runs controller process. there are three different types of controllers. </br>
          - Node controller: Responsible for noticing and responding when nodes go down. </br>
          - Job controller: Watches for Job objects that represent one-off tasks, then creates Pods to run those tasks to completion. </br>
          - EndpointSlice controller: Populates EndpointSlice objects (to provide a link between Services and Pods). </br>
          - ServiceAccount controller: Create default ServiceAccounts for new namespaces. </br>

#### 2.Nodes				
workers that run the application	(VM or physical machine )

- Kubelet - each node as agent to managing the node.
- Containerd - tool for managing the containers on worker node.

##### Key concepts:

- Pod					-  Smallest deployment unit, 1 or more container
- Node				-  worker machine VM or physical machine
- Cluster				-  A set of nodes managed by kubernetes
- Deployment			-  Managing pods, handling updates and scaling
- Service				- Exposing pods inside/outside the container
- Namespace			- Virtual cluster inside the kubernetes cluster

<img width="1402" height="882" alt="image" src="https://github.com/user-attachments/assets/27fc321d-490a-4ea1-93df-3c9e227beeff" />


### List kuberbetes clusters/context:
```
kubectl config get-contexts
```
### Current  selected Cluster:
```
kubectl config current-context
```
### List Actual Cluster Endpoints:
```
kubectl config view
```
### List nodes
```
kubectl get nodes
```
