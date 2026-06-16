RBAC  ( Role Based Access Control ) :
---------------------------------------
- Who can do what in yout kubernetes cluster.

**Who** - user, group and service account. <br>
**What** - resources like pod, services, deployments.<br>
**Actions** - get, list, create, update, delete. <br>

Think of it like giving permissions to **users, services accounts or groups**. <br>

### User:
- Human identity
- Kubernetes does not store users internally. it comes from kubeconfig client certificates, OIDC ( Gutlab, Okta etc.), Cloud IAM, Static Auth Roles)
- You can not list users from kubernetes itseld.
```
Developer -> Indentity provider (EKS/OICD/Client Certificate) -> Kubernetes API Server -> RBAC (RoleBinding) -> Allow/Deny
                                                                  (Validates Token)
```
**Certificate based authentication:**
```
Developer machine
|  (Client certificate + Private Key)
Kubernetes API server
| ( Verifies Certificate Against CA)
RBAC checks username from the certificate
|
Access granted/Denied
```
**OIDC (Gitlab) Authentication:**
```
Developers
|
Gtilab (OIDC Provider)
|  JWT
Kubernetes API Server
| OIDC Verification
RBAC ( RoleBinding/Groups)
|
Allow/Deny
```
### Group:
- Not like a K8s object
- Its Identity label attached to the user by auth system
- Kubernetes does not create or store groups
- Collection of users.
- K8s not creates a groups
- It only trust from the authentication providers

**Where it comes from**: <br>
Case 1: Certificate-based authentication <br>
When generating a certificate:
```
CN = purval
O  = ml-team
```
`CN` → becomes username <br>
`O` → becomes group <br>

That group name (`ml-team`) is what `RBAC` sees. <br>

Case 2: OIDC (GitLab login) <br>
When GitLab issues a `JWT token`, it may include:
```
{
  "email": "purval@test.ai",
  "groups": ["ml-team", "backend-team"]
}
```

**RBAC uses them.**
Example:
```
subjects:
  - kind: Group
    name: ml-team
```
This means: <br>

Anyone whose authentication token contains `ml-team` <br>
→ gets this role. <br>

You don’t need to bind every user individually. <br>
### ServiceAccount:
- is an identity used by Pods and controllers, not by humans.
- ServiceAccount → identity for applications running inside the cluster.
- If Kubernetes allowed all Pods full access, it would be chaos.
So each Pod gets an identity. <br>
That identity is a **ServiceAccount**. <br>

| Topic          | User                             | ServiceAccount                       |
| -------------- | -------------------------------- | ------------------------------------ |
| Used by        | Humans (developers, admins)      | Pods, controllers, applications      |
| Authentication | Certificates / OIDC              | Token automatically mounted          |
| Default?       | No user is created by default    | Yes, each namespace has `default` SA |
| Scope          | Cluster-wide                     | Namespace-scoped                     |
| RBAC           | Role/ClusterRole via RoleBinding | Same RBAC system                     |

For e.g. 
- only admins to create or delete resources. <br>
- Only developers to deploy apps. <br>
- Only monitoring tools can read logs. <br>

**RBAC is made of 3 parts:** <br>
1. **Role**:		can read pods
2. **ClusterRole**:		can read nodes 
3. **RoleBinding**:	dev can read pods in dev-ns

Mostly **ServiceAccount** is used from the **Gitlab CI**, **Jenkins**,**ArgoCD** kind of tools <br>
And **Users** are used for Developers to access Kubernetes cluster. <br>

Verify the RBAC is enabled or not. <br>
```
ps -ef | grep kube-apiserver | grep authorization-mode
```

### 1. Create namespace:
```
kubectl create namespace dev
```

### 2. Create role (developer access):
Create `dev-role.yaml` <br>

**Role** - Role always set permission with namespace. <br>
**ClusterRole** - Non namespaced resources. <br>

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: dev-role
  namespace: dev
rules:
- apiGroups: [""]  # Core resources
  resources: ["pods", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "delete"]
```

Apply changes:
```
kubectl apply -f dev-role.yaml
```
OR
```
kubectl apply -f https://raw.githubusercontent.com/purvalpatel/kubernetes-sample/main/dev-role.yaml
```

### 3. Create service account:
```
kubectl create serviceaccount dev -n dev
```

Delete service account: ( For Reference only. )
```
kubectl delete serviceaccount dev -n dev
```

### 4.Bind role to serviceaccount:
Create `dev-rolebinding.yaml`

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-binding
  namespace: dev
subjects:
- kind: ServiceAccount
  name: dev
  namespace: dev
roleRef:
  kind: Role
  name: dev-role
  apiGroup: rbac.authorization.k8s.io
```
Apply:
```
kubectl apply -f dev-rolebinding.yaml
```
OR
```
kubectl apply -f https://raw.githubusercontent.com/purvalpatel/kubernetes-sample/main/dev-rolebinding.yaml
```

### 5. Get the service account token:
```
kubectl -n <namespace> create token <service-account-name>
kubectl -n dev create token dev
```

Note: This token is not stored anywhere. So it is not retrievable. And this is token is generated for short time. It will expired. <br>
To create long lived token we have to provide duration parameter. <br>
```
kubectl -n dev create token dev --duration=14h
```

### 6. Create cluster and context:

```
kubectl config set-cluster dev-cluster \
  --server=https://10.7.10.23:6443 \
  --insecure-skip-tls-verify=true

kubectl config set-context dev-context \
  --cluster=dev-cluster \
  --user=dev-user \
  --namespace=dev-ns
```

Note: <br>
- Here --user=dev-user is not referring to any kubernetes user or service account. <br>
It referes to the entry in your kubeconfigs users section.	<br>
- You dont need to create kubernetes users manually when you are using kubeadm. Instead of this use serviceaccounts. <br>


### 7.provide this yaml to developers, this should be done in developers machine: <br>
#~./kube/config

```
apiVersion: v1
kind: Config
clusters:
- name: kubernetes
  cluster:
    server: https://kube.testai.io
    insecure-skip-tls-verify: true

contexts:
- name: dev-context
  context:
    cluster: kubernetes
    namespace: dev
    user: dev

current-context: dev-context

users:
- name: dev
  user:
token: <token>
```

**Note**: here user, **dev** is not the kubernetes user it is just reference in below section of token. <br>

Before that set reverse proxy of kubernetes cluster machine: <br>

/etc/nginx/conf.d/ssl.conf
```
server {
    listen 443 ssl;
    server_name kube.testai.io;

    ssl_certificate /etc/nginx/ssl/testai.io/crt.crt;
    ssl_certificate_key /etc/nginx/ssl/testai.io/crt.key;

    # Error pages configuration
    error_page 404 403 500 503 502 /error-page.html;

    location = /error-page.html {
        root /var/www/html;
        internal;
    }

    location / {
        proxy_pass https://10.7.10.23:6443;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

### 8. Now developers will run this from their machine:
```
kubectl get pods --namespace=dev
```

developers can create pods on cluster from their machine:
```
kubectl apply -f https://raw.githubusercontent.com/purvalpatel/kubernetes-sample/main/nginx-deployment.yaml
```

This will create pods on kubernetes cluster.
```
kubectl get pods --namespace=dev
```

### Delete the setup (For Reference Only):
1.Remove role binding:
```
kubectl delete rolebinding dev-binding -n dev
```
2.Remove role:
```
kubectl delete role dev -n dev
```

3.Remove service account:
```
kubectl delete serviceaccount dev -n dev
```

Provide Access of one Namespace to developer
--------------------------------------------
- We have one namespace (nUWol) and want to provide Edit access of that namespace to developer.

**Below are the steps that we need to follow.** <br>
Lets assume: <br>
**username** - nUWol_read <br>
**namespace** - nUWol <br>

## STEP 1 : Create namespace ( if dont have any ):
```
create namespace nUWol
```

## STEP 2 : Create Certificate for the user:
### Create Private Key.
```
openssl genrsa -out nUWol_read.key 2048
```

### Create CSR:
```
openssl req -new -key nUWol_read.key -out nUWol_read.csr -subj "/CN=nUWol_read/O=developers"
```

Here, <br>
CN = username (nUWol_read) <br>
O = group (developers) → optional <br>


### Approve CSR using kubernetes:

Create YAML. <br>
Generate `nUWol_read-csr.YAML` <br>

```BASH
echo "apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: nUWol_read
spec:
  groups:
  - system:authenticated
  request: $(cat nUWol_read.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth" > nUWol_read-csr.yaml
```
### Apply CSR: <br>
```
kubectl apply -f nUWol_read-csr.yaml
```

### Approve CSR:
```
kubectl certificate approve nUWol_read
```

### Download the Signed certificates:
```
kubectl get csr nUWol_read -o jsonpath='{.status.certificate}' | base64 -d > nUWol_read.crt
```
Now you have: <br>
**nUWol_read.key** <br>
**nUWol_read.crt** <br>


## STEP 3 : Create `RoleBinding.yaml`

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: nUWolread-access
  namespace: nUWol
subjects:
  - kind: User
    name: nUWol_read
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
```
Change,  <br>
- name of `rolebinding`, `namespace`, `username`. <br>
- If you want to make it Read only then use `view`, delete and re-create role-binding and it works.

Apply:
```
kubectl apply -f rolebinding.yaml
```

## STEP 4 : Create Kubeconfig for developer:
### Find your cluster details.
```
kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}'
```
output: https://x.x.110.25:6443 <br>

### Find cluster CA:
```
kubectl config view --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}'
```

## Create user kubeconfig ( `nUWolread.kubeconfig`)
nUWolread.kubeconfig
```
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority-data: <CA_DATA>
    server: <APISERVER_URL>
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: nUWol_read
    namespace: nUWol
  name: devuser-context
current-context: devuser-context
users:
- name: nUWol_read
  user:
    client-certificate: nUWol_read.crt
    client-key: nUWol_read.key

```
Note: <br>
Paste `<CA-DATA>` and `<APISERVER_URL>` which we have get in previous commands.
change **user**, **namespace** and **certificate names**.


## STEP 5: Give kubeconfig to the developer

Send: <br>

**nUWolread.kubeconfig** <br>
**nUWol_read.crt** <br>
**nUWol_read.key** <br>

They can run below command on their machine.
```
kubectl --kubeconfig=devuser.kubeconfig get pods
```


Allow Pod Exec Permissions:
-------------------------
- If we set View only ClusterRole permissions then user wont be able to exec the pod. for e.g. cassandra.
Rolebinding.yaml
```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: testread-access
  namespace: test
subjects:
  - kind: User
    name: test_read
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
```
For that need to create exec role: <br>
Create `exec-role.yaml`
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: exec-access
  namespace: test
rules:
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create", "get"]
```
Apply:
```
kubectl apply -f exec-role.yaml
```

Create Role-binding for this role: <br>
Create `rolebinding-exec.yaml`
```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: test-exec-access
  namespace: test
subjects:
  - kind: User
    name: test_read
roleRef:
  kind: Role
  name: exec-access
  apiGroup: rbac.authorization.k8s.io
```
Apply:
```
kubectl apply -f rolebinding-exec.yaml
```
