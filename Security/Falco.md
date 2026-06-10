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
