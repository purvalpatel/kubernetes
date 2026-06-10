### Purpose: 
ArgoCD is a declarative, GitOps continuous delivery tool for kubernetes. <br>

ArgoCD follows gitops patterns using Git repositories as a source. <br>

Kubernetes manifests defined in several ways: <br>
1. kustomize applications <br>
2. Helm <br>
3. Any custom configuration management tool <br>


ArgoCD Automates the deployment of the desired application state for the targetted environment. <br>
Any changes modificated done in a source Git will be automatically deployed on kubernetes environment. <br>

Features: <br>
1. Automated deployment of applications to specified target environments. <br>
2. Support multiple config management tools. (kustomize, helm) <br>
3. Ability to manage multiple clusters. <br>
4. SSO integration. <br>
5. Health status analysis of application resources. <br>
6. Automated configuration drift detection and visualization. <br>
7. WebUI which provides real-time view of application activity. <br>
8. webhook integration. <br>


Installation steps: 
-------------------
Create namespace:
```
kubectl create namespace argocd 
```
Create Resources:
```
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml 
```
Expose port with services: 
```
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}' 
```
 
List created resources: 
```
kubectl get all –n argocd 
```
 
### Login ArgoCD in browser: 

Use port of argocd-server service. <br>

http://<ip>:<port> 

 

#### How to get the default password of argocd login. 

Default username is : admin <br>

**Follow below steps to get password:**

1. Connect Argocd-server pod: 
```
kubectl -n argocd1 exec -it argocd-server-78484d9d9d-q7ghl – bash 
```
2. Run below command to get the password of admin user from argocd CLI: 
```
argocd admin initial-password 
```
3. Use this password to login in to browser. 

 Deployment Architecture:
 --------
 ### 1. If Project have 5 Manifests files.<br>
k8s/<br>
 ├── namespace.yaml<br>
 ├── configmap.yaml<br>
 ├── service.yaml<br>
 └── deployment.yaml<br>

Argo will apply in rough order:<br>

```
kubectl apply -f k8s/
```
it knows the dependencies order. like namesapces before deployments.<br>
It knows CRDs dependencies (CRDs firsts, CRs later) <br>

### 2. With Helm or kustomize

 If your repo uses Kutomize or helm, then Argo doesnt apply manifests manually. it will apply in the same order which you have provided in helm/kustomize.

### This process can be done by two ways:
#### 1. Manual way ( ArgoCD WebUI )
   
#### 2. Automated (Gitops)

Sample of nginx deployment with helm using ArgoCD:
-------------------------------------------------

##### 1. Generate helm chart for nginx.
```
helm create nginx
```

##### 2. Make changes in the configrations.
Edit chart.yaml <br>
```YAML
apiVersion: v2
name: myapp
version: 0.1.0
description: A simple Helm chart for my app
```

Edit values.yaml <br>
```YAML
replicaCount: 1

image:
  repository: nginx
  tag: "1.27"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
```
##### 3. Test the chart locally.
```
helm template nginx
```

##### 4. Push code to git repository.
```BASH
git add .
git commit -m "pushing code"
git branch -m master main
git fetch origin
git branch --set-upstream-to=origin/main main
git pull --rebase
git push origin main
```
Now your repo will looks like this <br>

https://github.com/your-org/myapp-helm <br>
└── nginx/ <br>
    ├── Chart.yaml <br>
    ├── values.yaml <br>
    └── templates/ <br>

##### 5.Now Create deployment in ArgoCD.
- Login ArgoCD. <br>
- Connect repository with ArgoCD with credentials.
     **Settings** -> **Repositories**.<br>
     **Connect Repo** -> Select "**VIA HTTP/HTTPS**" <br>
     Enter below details: <br>
     **Name**, **Project**, **Repository URL**, **Username** and **password**. <br>
     Or you can provide **Gitlab token** instead of username and password.<br>
     Now it is showing **Connected**. <br>

- Naviagate to **applications** ->**New App**. <br>
  **Application Name** <br>
  **Project Name** - (select project) <br>
  **Sync policy** (manual/Automatic) <br>
  **Source** - Git project Repository URL. <br>
  **Revision** - Select branch name <br>
  **Path** - Provide path where YAML files are located. <br>
  **Cluster URL** - Select Cluster URL ( mostly same as where argocd deployed ) <br>
  **Namespace** - Namespace where it should be deployed. <br>

##### 6. Create.
- Now it will start deploying application.


Note: <br>
You can delete the deployment directly from web ui. and pods will deleted. 

Sample of nginx deployment using ArgoCD:
-----------------------------------------
1. Process will be same just keep the deployment, namespaces, services YAML manifests files seperately.

Automated (GitOps)
------------------

- GitOps way.
- The real power of argoCD where everything is declarative inside Git, including ArgoCD Applicaiton definations.

### Steps:
1. Create Simple helm chart.
2. Push it into Git.
3. Create ArgoCD application YAML that Automatically deploy a chart.

First two steps are completed, so we are ignoring as of now. <br>

Step 3 : Create ArgoCD application YAML. <br> 
You can keep it in another Git repo or apply it directly. <br>
nginx-argocd.yaml <br>

```YAML
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd          # Argo CD’s namespace
spec:
  project: default
  source:
    repoURL: 'https://git.example.com/devops/myapp-helm.git'  # Your repo
    targetRevision: main                                      # Branch name
    path: '.'                                                 # Path of chart inside repo
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: myapp
  syncPolicy:
    automated:                      # Enables full GitOps automation
      prune: true                   # Delete resources removed from Git
      selfHeal: true                # Auto-fix drift
    syncOptions:
      - CreateNamespace=true        # Auto-create namespace if missing

```
Apply:
```
kubectl apply -f myapp-argo.yaml
```

Verify application status:
```
kubectl get applications -n argocd
```
Also you can check ArgoCD web UI and it is showing there as well.

Note:<br>
- If you delete manifest with `kubectl delete -f argocd-nginx.yml` then only application from web ui is deleted. but pods are running.
- You need to delete pods/deployment  manually.



Add new cluster in argoCD:
---------------------------

### Download argocd CLI in local machine:
```
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/
sudo mv argocd /usr/local/bin/
```
### Login ArgoCD:
Provide  admin user credentials of argocd.
```
argocd login xx.xx.xxx.xxx:30082 --username admin --password dev1@@123 --insecure

'admin:login' logged in successfully
Context 'xx.xx.xxx.xxx:30082' updated
```
### Verify it is logged in or not:
```
argocd account get
```

### List Existing clusters:
```
argocd cluster list

output:
SERVER                          NAME        VERSION  STATUS      MESSAGE  PROJECT
https://kubernetes.default.svc  in-cluster  1.31     Successful   
```

### Copy config file:
Now copy the context of `~/.kube/config` of kubernetes cluster server to local machine into `config` file. <br>

### Add cluster:
Here `config` is the config file name which is created recently.
```
argocd cluster add kubernetes-admin@kubernetes --kubeconfig config

output:
WARNING: This will create a service account `argocd-manager` on the cluster referenced by context `kubernetes-admin@kubernetes` with full cluster level privileges. Do you want to continue [y/N]? y  
{"level":"info","msg":"ServiceAccount \"argocd-manager\" created in namespace \"kube-system\"","time":"2025-12-02T15:25:27+05:30"}
{"level":"info","msg":"ClusterRole \"argocd-manager-role\" created","time":"2025-12-02T15:25:27+05:30"}
{"level":"info","msg":"ClusterRoleBinding \"argocd-manager-role-binding\" created","time":"2025-12-02T15:25:27+05:30"}
{"level":"info","msg":"Created bearer token secret \"argocd-manager-long-lived-token\" for ServiceAccount \"argocd-manager\"","time":"2025-12-02T15:25:27+05:30"}
Cluster 'https://xx.xx.xxx.xxx:6443' added
```

### List clusters:
```
argocd cluster list

output:
SERVER                          NAME                         VERSION  STATUS      MESSAGE  PROJECT
https://xx.xx.xx.xxx:6443       kubernetes-admin@kubernetes  1.31     Successful           
https://kubernetes.default.svc  in-cluster                   1.31     Successful       
```
Done. New cluster has been added. <br>
You can check with : Settings -> Cluster <br>
<img width="1686" height="292" alt="image" src="https://github.com/user-attachments/assets/fd3a20bd-ece8-4059-b0db-fca17ace4ae6" />
 <br>

### You can test this by:
```
argocd app create test-app \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://xx.xx.xxx.xx:6443 \
  --dest-namespace default \
  --project default
application 'test-app' created
```
### Create New application on new created cluster.
- Make sure `venv` folder is not there in your git Repository.

### Add new project:
- Settings -> Project -> **NEW PROJECT** <br>
- Add **Project Name**, **Description** and **Save**

Allow the Kubernetes cluster host now, <br>

- Click on the created **project name**. <br>
- Edit below things under **Destination**. <br>
<img width="1684" height="1010" alt="image" src="https://github.com/user-attachments/assets/932c4b8d-5bc4-47e7-a4fc-327e9692d44b" />




### Add Repository:
- Settings -> Repositories
- Connect Repo
<img width="1517" height="1003" alt="image" src="https://github.com/user-attachments/assets/eda33af1-e5d6-4e6b-9421-4f19c64e7d9a" />



### Now Create Application
- Login ArgoCD
- Click on **New App** <br>

**Application Name** : Any Name <br>
**Project Name** : Any Project Name <ve>
**Sync Policy** : Manual <br>
**Repository URL** : gitlab project URL <br>
**Revision**: branch-name <br>
**path**: path-where-k8s-deployment-files-are-there e.g. k8s/ <br>
**Cluster URL** : select cluster URL <br>
**Namespace**: Enter Namespace of application <br>

Click on **Create**. <br>

Now when you click on Sync it will sync all the resources.


### Create application in argoCD from backend ( Without Web UI)
Using Kubernetes Custom Resources is the most "Kubernetes-native" way to manage ArgoCD applications. <br>
Also it can be done with the argocd CLI.



Create ArgoCD User (Read-only Access):
-----------------------------------
1. Add user into configmap. <br>
Login ArgoCD server. <br>
```
kubectl -n argocd edit configmap argocd-cm
```
add below lines under `date`. <br>
```
data:
  accounts.dev1: login
  accounts.dev1.enabled: "true"
```

and save it. <br>

2. Change password with argocd CLI. <br>
Login first if not logged in. <br>
```
argocd login xx.xx.xx.xx:30082 --username admin --password dev1@123 --insecure
```
Change password now: <br>
```
argocd account update-password   --account dev1   --current-password ""   --new-password "dev1@123"
```

3. Set RBAC policy of readonly access: <br>
Login Kubernetes server and edit below configmap. <br>
```
kubectl -n argocd edit configmap argocd-rbac-cm
```
Add below lines, <br>
```
data:
  policy.csv: |
   ## Allow Admin access for admin user
    p, role:admin, applications, *, */*, allow
    p, role:admin, projects, *, *, allow
    p, role:admin, clusters, *, *, allow
    g, admin, role:admin

    # Read-only role
    p, role:readonly, applications, get, *, allow
    p, role:readonly, projects, get, *, allow
    p, role:readonly, clusters, get, *, allow


    # Assign readonly role to user 'dev1'
    g, dev1, role:readonly
  scopes: '[groups]'
```
Here, change username. <br>

4. Now, Login ArgoCD WebUI with this created user. <br>


Argo-Rollout:
---
### What is Argo Rollouts?

Argo Rollouts is a Kubernetes controller that provides advanced deployment strategies like:

- 🐦 Canary deployments
- 🔵🟢 Blue-Green deployments
- Progressive delivery with metrics-based promotion
- Automated rollback

👉 It replaces Deployment with a new CRD: Rollout

### Why not plain Kubernetes Deployment?
K8s Deployment supports:
- Rolling update only ❌

Argo Rollouts adds:
- Traffic shaping
- Pauses
- Manual approval
- Metric checks
- Gradual rollout

🧩 Architecture
```
Git / CI
  ↓
Argo CD
  ↓
Argo Rollouts Controller
  ↓
Kubernetes (Pods, Services, Ingress)
```
| Feature        | Deployment | Rollout |
| -------------- | ---------- | ------- |
| Rolling update | ✔          | ✔       |
| Canary         | ❌          | ✔       |
| Blue-Green     | ❌          | ✔       |
| Metric checks  | ❌          | ✔       |
| Auto rollback  | ❌          | ✔       |


**Note:** <br>
Canary and Blue-Green are deployment strategies <br>
They are not tied to a single tool. <br>
<br>
Argo Rollouts makes them easier, safer, and automated, but they absolutely can be done without it. <br>


### install Argo-rollout CRD's
```
kubectl create namespace argo-rollouts

kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

### Replace Deployment with Rollout

❌ OLD
```
apiVersion: apps/v1
kind: Deployment
```
✅ NEW
```
apiVersion: argoproj.io/v1alpha1
kind: Rollout
```

### Add blue green strategy:
```

  strategy:
    blueGreen:
      activeService: sbdd-frontend-active
      previewService: sbdd-frontend-preview
      autoPromotionEnabled: false
```
### Create TWo services ( Active and Preview )
Active service
```
apiVersion: v1
kind: Service
metadata:
  name: sbdd-frontend-active
  namespace: purvAI1
spec:
  type: NodePort
  selector:
    app: sbdd-frontend
  ports:
  - port: 3000
    targetPort: 3000
    nodePort: 30031

```
Create Preview service:
```
apiVersion: v1
kind: Service
metadata:
  name: sbdd-frontend-preview
  namespace: purvAI1
spec:
  type: ClusterIP
  selector:
    app: sbdd-frontend
  ports:
  - port: 3000
    targetPort: 3000

```
Rollout switches the service:
```
kubectl get rollouts -n purvAI1
kubectl get svc -n purvAI1
kubectl argo rollouts get rollout sbdd-frontend -n purvAI1
```

AnaylsisRuns:
----
How argoCD Rollouts decides whether a release is good or bad. <br>
Using real signals, not just a pod is running or not.

<br>
**An AnalysisRun executes checks (metrics, tests, webhooks) and tells Argo Rollouts whether to continue, pause, or rollback a deployment.**

### 🧩 Why AnalysisRuns exist

**Kubernetes Deployment checks only:**
- Pod started? ✅
- Container running? ✅

**But production needs:**
- Error rate?
- Latency?
- HTTP 5xx?
- Business KPIs?
- Smoke tests?

👉 That’s exactly what AnalysisRuns do.

### Analysis Template:
```
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate-check
spec:
  metrics:
  - name: http-success
    interval: 30s
    successCondition: result >= 0.99
    provider:
      prometheus:
        query: |
          sum(rate(http_requests_total{status=~"2.."}[1m])) /
          sum(rate(http_requests_total[1m]))

```

Rollout uses it:
```
strategy:
  blueGreen:
    prePromotionAnalysis:
      templates:
      - templateName: success-rate-check

```

### How to observ it live ?
```
kubectl argo rollouts get rollout my-app
```
Shows :
```
Step: 2/5
Analysis: Running
```
Watch Metrics:
```
kubectl get analysisrun
kubectl describe analysisrun <name>
```


###🔄 What happens during deployment?
- New version (Green) starts
- Traffic still on Blue
- AnalysisRun starts
- Prometheus metrics checked
- If success → promote
- If fail → rollback automatically

Only build services that changed:
----------------------------------
```
Any Chage -> Build All 10 services -> Push all images
```
Three clean approches:
1. Use `rules:chnages` per service
- Create per service job in gitlab CI

2. Git detect changes Dynamically
- `git --diff --name-only`

3. Split each microservice into seperate Gitlab project.
