How to secure kubernetes cluster?
----------------------------
### 1. cluster and infrastructure security
- Restrict access to:
  - Kube-apiserver
  - etcd
- Never expose API server port to public internet.
<br>

**Restrict global access port of API server.** <br>
**Disable Anonymous auth:** <br>
```
--anonymous-auth=false
```

### 2. Authorization and Authentication
- RBAC

### 3. Pod and Container security
- Use pod security standards (PSS)
```
kubectl label ns prod pod-security.kubernetes.io/enforce=restricted
```
Restricted means: <br>
- No privileged pods
- No hostPath
- No root user

- Run Containers as non-root user.
```
securityContext:
  runAsNonRoot: true
  readOnlyRootFilesystem: true
```

🚫 Disable dangerous settings:
`privileged: true` <br>
`hostNetwork: true` <br>
`hostPID: true` <br>

### 4. Image security
- Scan images ( Trivy )
- Example of image scan with `trivy`:
```
docker run -v /var/run/docker.sock:/var/run/docker.sock -v $HOME/Library/Caches:/root/.cache/ aquasec/trivy:0.68.2 image python:3.4-alpine
```
- This will show all the vulnerabilities of the image.

- Use Trusted registry with `cosign`
```
cosign verify myimage:1.0
```

### 5. Network Security
- Network policy
  - Deny Traffic and then explicitly allow traffic.
- Service Mesh `istio`

### 6. Secret Management
- Do not store secret in Plane YAML. instead of use kubernetes secrets.
```
kubectl create secret generic db-secret --from-literal=password=xyz
```

### 7. Runtime Security
- Use Runtime security tools ( `Falco`, `Tetragon` )
- Alert if shell executed inside container

### 8. Logging, Auditing & Monitoring
📜 Enable Audit Logs
```
--audit-log-path=/var/log/kube-apiserver.log
```
Track:<br>
- Who deleted pod
- Who accessed secrets

Monitoring <br>
- Prometheus
- Grafana
- Alertmanager

### 9. Kubernetes API Hardening
- Disable unused APIs
- Disable legacy auth

Use admission controllers: <br>
- PodSecurity
- OPA Gatekeeper
- Kyverno

Example Kyverno policy:
```
require runAsNonRoot=true
```

### 10. Node Security
🔐 Harden nodes <br>
- Minimal OS (COS, Bottlerocket)
- Regular patching
- Disable SSH where possible

Falco:
-----
Falco monitors system calls and detects abnormal behaviour at runtime using rules and raises alerts. <br>

**Examples of things Falco can detect:** <br>
- Shell opened inside a container (/bin/bash)
- Container running as root
- Writing to /etc/passwd
- Crypto miner execution
- Privilege escalation
- Unexpected network connections
<br>

⚠️ Falco does not prevent attacks — it detects and alerts. 

### Setup Falco:
```
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update
helm install falco falcosecurity/falco   --namespace falco   --create-namespace   --set driver.kind=ebpf
```
Check all the pods:
```
kubectl get all -n falco
```
Verify the Alerts are generating or not:
Check logs of Falco:
```
kubectl logs falco-l559s -n falco -f
```

Login some pod and check the logs in falco are generating or not.
```
kubectl -n milvus exec -it milvus-etcd-0 -- /bin/sh
```
It should look like <br>
<img width="1916" height="103" alt="image" src="https://github.com/user-attachments/assets/54ac6cfe-1d9c-42d9-a7d5-15fab134f5a0" />

#### Troubleshooting
If Falco pod is not running. <br>
Prerequisites:
```
sudo sysctl fs.inotify.max_user_instances=8192
sudo sysctl fs.inotify.max_user_watches=524288
```
Make persistent:
```
echo "fs.inotify.max_user_instances=8192" | sudo tee -a /etc/sysctl.conf
echo "fs.inotify.max_user_watches=524288" | sudo tee -a /etc/sysctl.conf
```

### Send Alerts on Slack now:
Below is the Falco Stack.
```
Falco
  → Falcosidekick (router)
      → Slack / Teams / Webhook / SIEM
```
Falco detects and Falcosidekick delivers. <br>

### Flacosidekick setup: <br>
Add repo,
```
helm upgrade falco falcosecurity/falco -n falco \
  --set falcosidekick.enabled=true \
  --set falco.jsonOutput=true \
  --set falco.httpOutput.enabled=true \
  --set falco.httpOutput.url=http://falcosidekick:2801/ \
  --set falcosidekick.config.minimumpriority=debug \
  --set falcosidekick.config.slack.webhookurl="https://hooks.slack.com/services/T0XXX/B0XXX/XXXX" \
  --set falcosidekick.config.slack.channel="#purval-alerts" \
  --set falcosidekick.config.slack.username="Falco"
```

And done. you will start receiving alerts on slack: <br>

<img width="1913" height="985" alt="image" src="https://github.com/user-attachments/assets/9aa29354-db3a-4c7b-88fa-b2f664dc092b" />

#### Falco Priority lavels:
```
Debug
Info
Notice
Warning
Error
Critical
Alert
Emergency
```
