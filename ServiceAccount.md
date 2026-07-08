Pod is acting as this identity. what is it allowed to do? <br>

Suppose you have pod running inside kubernetes. it wants to:
- Create another pod
- Read secrets
- Create PVCs
- Delete deployments
- Watch nodes

Without and identity, kubernetes has no idea whether pod should be allowed to do those actions. <br>

Here, Service account Comes in.
```
Pod
 | Uses
ServiceAccount
 | Has Role/RoleBindings
Kubernetes API
```
### How it works?
```
Pod created
  |
Attach ServiceAccount 
  |
Mount JWT Token
  |
Application talks to K8s API.
```

here, JWT token automatically mounted inside the pod. i.e. ca.crt, namespace, token

### Example:
- Your application want to list pods

Every namespace has default ServiceAccount. <br>
ServiceAccount alone have no permissions. <br>
it comes from:
```
SA - Role Binding - Role
```
OR
```
SA - ClusterRoleBinding - ClusterRole
```
Role Can:
- Create Pods
- Read Secrets
- Create PVCs
- Delete Nodes

ServiceAccount is namespace scoped. <br>
Can Have two ServiceAccount for single namespace.
