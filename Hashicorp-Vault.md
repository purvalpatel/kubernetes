# introduction

| kubernetes secret | Vault |
| ------------------ | ----- |
| **Built into kubernetes** <br> stores secret inside etcd. | **Seperate dedicated tool** <br> its own server, its own storage. |
| **NOT Encrypted by default** <br> stored as base64 in etcd (not safe) | Always encypted. <br> AES-256, built for secret storage. |
| **Basic - k8s RBAC only** <br> anyone with kubectk access can read. | **Fine-grained policies** <br> peer servicesm per env, per path |
| **Manual** <br> you update and redeploy path | **Automatic** <br> rotate DB passwords, certs, keys |
| **Limited** <br> hard to track who read what secret | **Full audit trail** <br> every read/write logged with identity |
| **Simple** <br> already in k8s | **Extra setup** <br> Seperate service |

### Anyone can read secrets:
```
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 -d
kubectl -n numol get secret mongo-secret -o jsonpath='{.data.MONGODB_URI}' | base64 -d
```

##  Two main approches to use Valult secrets in Kubernetes deployment:
1. Vault Agent Injector (sidecar injection)  - Common approch.
2. Vault Secret Operator (VSO)  - newer ,k8s-native

### When to use VSO:
- Pre-built images, can't change entrypoint
- Need secrets as env vars
- Already using envFrom / secretRef
- Migrating existing K8s secrets to Vault

### When to use Vault Agent Injector:
- Need live secret rotation without restart
- Need secrets as files on disk

## installation 

### Step 1 — Install Vault in Kubernetes
```
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

kubectl create namespace vault

helm install vault hashicorp/vault \
  --namespace vault \
  --set "server.dev.enabled=true"
```
> 👉 This starts Vault in dev mode (easy testing)


### Step 2 — Port forward Vault UI
```
kubectl port-forward svc/vault -n vault 8200:8200
```
OR <br>

Edit ClusterIP with NodePort:
```
kubectl patch svc/vault -n vault -p '{"spec":{"type":"NodePort"}}'
```
Open:
```
http://localhost:8200
```

Dev token:
```
root
```
![Vault](image-20.png)

## Approach 1: Vault Agent Injector (Annotations-based)

 
### Step 1 — Enable Kubernetes Auth

Exec into Vault pod:
``` 
kubectl exec -it -n vault vault-0 -- sh
```
Inside pod:
```
vault login root

vault auth enable kubernetes
```
- This command tells hashicorp valut to start trusting kubernetes as identity provider. before this valut dont know who your pods are.
- This can be check from the web ui -> Access Tab. enables authentication backend: auth/kubernetes/

### Step 2 — Configure Kubernetes Auth
```
vault write auth/kubernetes/config \
  token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

### Step 3 — Create Secret in Vault
```
vault kv put secret/numol/db \
  username="numol_user" \
  password="supersecret123"

## for s3 numol bucket credentials: ( reference only)
vault kv put secret/numol/s3_numol \
  S3_ACCESS_KEY="0B3AN861NUCZ5G1T61WT" \
  S3_SECRET_KEY="_pUoc4IqCbYX2LC87w7KU8Kfjg11gK83id50A408" \
  S3_ENDPOINT="http://mns3006.nuvo-ai.com"

## for mongo URL: ( reference only)
vault kv put secret/numol/mongo \
  MONGO_URI="mongodb://numol:Nuvo%402026@10.10.110.159:32017/numoldb?authMechanism=SCRAM-SHA-256&directConnection=true" \
  JWT_SECRET="your-secret"
```

### Step 4 — Create Policy
- Create policy which contains secret.
```
## add secret s3_num into policy numol-policy 
vault policy write numol-policy - <<EOF
path "secret/data/numol/s3_numol" {
  capabilities = ["read"]
}
path "secret/metadata/numol/s3_numol" {
  capabilities = ["read"]
}
EOF

## Wildcard Policy for all secrets under numol path:
------------------------------------------------
vault policy write numol-policy - <<EOF
path "secret/data/numol/*" {
  capabilities = ["read"]
}
path "secret/metadata/numol/*" {
  capabilities = ["read"]
}
EOF

## for mongo URL: [Reference only]
## add mongo secret into numol-mongo-policy

vault policy write numol-mongo-policy - <<EOF
path "secret/data/numol/mongo" {
  capabilities = ["read"]
}
EOF
```

### Step 5 — Create Role (bind to K8s)
```
vault write auth/kubernetes/role/numol-role \
  bound_service_account_names=numol-sa \        ## service account name
  bound_service_account_namespaces=numol \      ## namespace
  policies=numol-policy \                       ## policy added here.
  ttl=1h

## If already exists [Update with new Policy]:
vault write auth/kubernetes/role/numol-role \
  bound_service_account_names=numol-sa \
  bound_service_account_namespaces=numol \
  policies="numol-policy,numol-s3-policy,numol-mongo-policy" \
  ttl=1h

```

### Step 6 — Enable Vault Injector

Outside pod:
```
kubectl apply -f https://raw.githubusercontent.com/hashicorp/vault-k8s/main/deploy/injector.yaml
```
### Step 7 — Create ServiceAccount
```YAML
apiVersion: v1
kind: ServiceAccount
metadata:
  name: numol-sa
  namespace: numol
```
Apply:
```
kubectl apply -f sa.yaml
```
### Step 8 — Deploy App with Vault Injection

For S3 bucket List:
```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: s3-test-deployment
  namespace: numol
spec:
  replicas: 1
  selector:
    matchLabels:
      app: s3-test
  template:
    metadata:
      labels:
        app: s3-test
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "numol-role"

        # Inject S3 secret
        vault.hashicorp.com/agent-inject-secret-s3: "secret/data/numol/s3_numol"

        # Convert secret to env variables
        vault.hashicorp.com/agent-inject-template-s3: |
          {{- with secret "secret/data/numol/s3_numol" -}}
          export AWS_ACCESS_KEY_ID="{{ .Data.data.S3_ACCESS_KEY }}"
          export AWS_SECRET_ACCESS_KEY="{{ .Data.data.S3_SECRET_KEY }}"
          {{- end }}
        sidecar.istio.io/inject: "false"        ## Do not inject istio for this pod. because vault pods are running without istio.

    spec:
      serviceAccountName: numol-sa

      containers:
      - name: aws-cli
        image: amazon/aws-cli
        command:    ## this will override the default startup command of container.
          - /bin/sh
          - -c
          - |
            echo "Loading Vault secrets..."
            source /vault/secrets/s3

            echo "Setting S3 config..."
            aws configure set default.s3.addressing_style path

            echo "Listing buckets from NetApp S3..."
            aws s3 ls \
              --endpoint-url http://mns3006.nuvo-ai.com \
              --no-verify-ssl

            sleep 3600

```

### Step 11 — Apply & Verify
```
kubectl apply -f deployment.yaml
```

Check logs:
```
kubectl logs -n numol deploy/test-app
```
You should see:
```
username=xxx_user
password=supersecret123
```

🧠 What just happened
```
Pod → ServiceAccount → Vault auth → policy check → secret injected → file mounted
```

Secrets appear at:
```
/vault/secrets/db
```
![alt text](image-21.png)
- You can Edit Username and password from web ui of Vault too.


## Approach 2: Vault Secret Operator (VSO)
- Newer, Kubernetes-native approach.
- VSO syncs vault secrets directly into native kubernetes secrets, so your existing deployment need zero change.

- Reconciles - Continuously checking the k8s secret matches what is in the vault. if vault changes, k8s secret updates automatically.
- Three main CRDs:
1. VaultConnection - where is vault ?
2. VaultAuth - How to authenticate with vault ?
3. VaultStaticSecret -  What secret to fetch ?

> Note: This requires pod restart. <br>
For zero downtime -> "Agent injector" approach is better. <br>


### Setup VSO:
1. Install VSO:
```
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

# Verify repo added
helm search repo hashicorp/vault-secrets-operator

helm install vault-secrets-operator hashicorp/vault-secrets-operator \
  -n vault-secrets-operator-system \
  --create-namespace \
  --set defaultVaultConnection.enabled=true \
  --set defaultVaultConnection.address="http://vault.vault.svc.cluster.local:8200"
  # 👆 change vault address if your vault is in different namespace/address
```
Verify:
```
kubectl get pods -n vault-secrets-operator-system

kubectl get crd | grep vault
```

2. verify kubernetes auth is enabled or not in vault:
```
vault auth list
```
If not enabled, enable it.
```
kubectl exec -it -n vault vault-0 -- sh
vault login root
vault auth enable kubernetes
```

3. Confirm role exists or not. if not then create it.

- Create Secret:
```BASH
vault kv put secret/numol/mongo \
  MONGO_URI="mongodb://numol:Nuvo%402026@10.10.110.159:32017/numoldb?authMechanism=SCRAM-SHA-256&directConnection=true" \
  JWT_SECRET="your-secret"
```

Create Role:
```BASH
vault write auth/kubernetes/role/numol-role \
  bound_service_account_names=numol-sa \
  bound_service_account_namespaces=numol \
  policies=numol-policy \
  ttl=1h
```

4. Confirm policy is created or not. if not then create it.

Create Policy if not exists:
```BASH
vault policy write numol-policy - <<EOF
path "secret/data/numol/*" {
  capabilities = ["read"]
}
path "secret/metadata/numol/*" {
  capabilities = ["read"]
}
EOF
```
> Note:<br>
Every time you add a new secret path you must update the policy to include it. Otherwise VSO gets permission denied.

Below is the sample: [ For reference only.]
```
kubectl exec -n vault vault-0 -- vault policy write numol-policy - <<EOF
path "secret/data/numol/mongo" {
  capabilities = ["read"]
}
path "secret/metadata/numol/mongo" {
  capabilities = ["read"]
}
path "secret/data/numol/s3_numol" {
  capabilities = ["read"]
}
path "secret/metadata/numol/s3_numol" {
  capabilities = ["read"]
}
EOF
```

5. Create ServiceAccount if not created.
```YAML
apiVersion: v1
kind: ServiceAccount
metadata:
  name: numol-sa
  namespace: numol
```

6. Create VaultConnection.
```YAML
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultConnection
metadata:
  name: vault-connection
  namespace: numol
spec:
  address: http://vault.vault.svc.cluster.local:8200
  skipTLSVerify: true

```

7. Create VaultAuth
```YAML
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultAuth
metadata:
  name: numol-vault-auth
  namespace: numol
spec:
  vaultConnectionRef: vault-connection
  method: kubernetes
  mount: kubernetes
  kubernetes:
    role: numol-role        # ← role you already created in vault
    serviceAccount: numol-sa

```
8. Create VaultStaticSecret
```YAML
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: mongo-numol-secret
  namespace: numol
spec:
  type: kv-v2
  mount: secret
  path: numol/mongo          # ← vault path without secret/data prefix
  destination:
    name: mongo-numol-k8s-secret   # ← K8s secret name VSO will create
    create: true
  refreshAfter: 30s
  vaultAuthRef: numol-vault-auth

```

Verify staticsecret is created or not:
- No need to create Kubernetes secret manually, it will get created automatically by VSO.
```BASH
kubectl get vaultstaticsecret -n numol
kubectl get secret -n numol
```

9. Sample deployment which uses the secret:
Configmap.yaml
```YAML
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongo-test-script
  namespace: numol
data:
  test.py: |
    import os
    from pymongo import MongoClient
    import time

    print("=== START ===", flush=True)

    uri = os.getenv("MONGO_URI")
    print("Mongo URI:", uri, flush=True)

    try:
        client = MongoClient(uri, serverSelectionTimeoutMS=3000)
        client.server_info()
        print("=== SUCCESS: Mongo Connected ===", flush=True)
    except Exception as e:
        print("=== ERROR ===", e, flush=True)

    while True:
        time.sleep(60)
```
Deployment.yaml
```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo-test
  namespace: numol
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo-test
  template:
    metadata:
      labels:
        app: mongo-test
    spec:
      containers:
      - name: mongo-test
        image: python:3.10-slim

        command: ["sh", "-c"]
        args:
        - pip install pymongo && python /app/test.py

        volumeMounts:
        - name: script-volume
          mountPath: /app

        env:
        - name: MONGO_URI
          valueFrom:
            secretKeyRef:
              name: mongo-numol-k8s-secret
              key: MONGO_URI

      volumes:
      - name: script-volume
        configMap:
          name: mongo-test-script
```

Verify VSO create the K8s secret:
```
kubectl get secret s3-numol-k8s-secret -n numol
kubectl describe secret s3-numol-k8s-secret -n numol
```

## Add New Secret Steps:
Step 1: Put Secret in Vault
```
kubectl exec -n vault vault-0 -- vault kv put secret/numol/s3_numol \
  S3_ACCESS_KEY="your-access-key" \
  S3_SECRET_KEY="your-secret-key" \
  S3_ENDPOINT="http://mns3006.nuvo-ai.com"
```
Step 2: Update Policy to include new secret path
```
kubectl exec -n vault vault-0 -- vault policy write numol-policy - <<EOF
path "secret/data/numol/mongo" {
  capabilities = ["read"]
}
path "secret/metadata/numol/mongo" {
  capabilities = ["read"]
}
path "secret/data/numol/s3_numol" {
  capabilities = ["read"]
}
path "secret/metadata/numol/s3_numol" {
  capabilities = ["read"]
}
EOF
```
> ⚠️ Every time you add a new secret path you must update the policy to include it. Otherwise VSO gets permission denied.

Step 3: Create new VaultStaticSecret for the new secret
```YAML
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: s3-numol-secret          # ← new name for this VSS
  namespace: numol
spec:
  type: kv-v2
  mount: secret
  path: numol/s3_numol           # ← new vault path
  destination:
    name: s3-numol-k8s-secret    # ← new K8s secret name VSO will create
    create: true
  refreshAfter: 30s
  vaultAuthRef: numol-vault-auth  # ← same, already exists
```
Add to your deployment.yaml:
```YAML
envFrom:
  - configMapRef:
      name: sbdd-service-config
  - secretRef:
      name: mongo-numol-k8s-secret   # existing
  - secretRef:
      name: s3-numol-k8s-secret   
```

### For EVERY new secret:
```
┌─────────────────────────────────────────────────────┐
│  ✅ VaultConnection    → already exists, reuse      │
│  ✅ VaultAuth          → already exists, reuse      │
│  ✅ ServiceAccount     → already exists, reuse      │
│                                                     │
│  ⬜ vault kv put       → new secret in Vault        │
│  ⬜ vault policy write → update policy              │
│  ⬜ VaultStaticSecret  → new YAML per secret        │
│  ⬜ deployment envFrom → add new secretRef          │
└─────────────────────────────────────────────────────┘
```
------------------


## Production Improvements (important for you)
1. Disable dev mode
- Use proper Helm config:
```
server:
  ha:
    enabled: true
```

2. Use external storage
- Consul / Raft backend
3. Auto TLS
- Use cert-manager

4. Rotate secrets
- Use dynamic engines (DB, AWS)


## Other commands:
1. View policy list:
```
vault policy list
```
2.  List attached policy with this role:
```
vault read auth/kubernetes/role/numol-role
```
3. Delete secret:
```
vault kv delete secret/numol/s3_numol
## delete permenantly with all versions.
vault kv metadata delete secret/numol/s3_numol
```
4. delete policy
```
vault policy delete numol-policy
```
5. delete role
```
vault delete auth/kubernetes/role/numol-role
```
6. List roles:
```
vault list auth/kubernetes/role
```
7. List secrets:
```
vault kv get secret/numol/s3_numol
```
8. Verify kubernetes /auth is configured or not:
```
vault auth list
```
9. Enable Audit logs:
```
# Exec into vault pod
kubectl exec -it -n vault vault-0 -- sh

# Enable audit log to stdout (easiest in Kubernetes)
vault audit enable file file_path=stdout

# OR enable to a file
vault audit enable file file_path=/vault/logs/audit.log

# Verify audit is enabled
vault audit list
```
> Note - We can persist logs using pvc too.

### Troubleshooting:
- Get logs of specific container from the pod:
```
kubectl logs -n numol s3-test-deployment-57f5974df8-jwlb6 -c vault-agent-init
```
- If pods running in namespace is with the istio envoy and pod running inside vault namespace are without istio envoy then it will not connect.

