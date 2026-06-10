Introduction:
-------------
Slack is used for sending alerts like ntfy.

### How to setup:
1. Create Slack Account:
2. Create Channel
- Left panel -> Channels
3. Create webhook.
- Admin Settings (Left pane) -> Apps and workflows -> search "Incoming webhooks"
- Copy Webhook URL

Test Payload:
```
# Simple alert
curl -X POST --data-urlencode "payload={\"channel\": \"##purval-alerts\", \"text\": \"this is for testing.\"}" https://hooks.slack.com/services/xxxx/xxxx/xxxxx

# Alert with specific Emoji
curl -X POST --data-urlencode "payload={\"channel\": \"##purval-alerts\", \"username\": \"webhookbot\", \"text\": \"This is for testing.\"}" https://hooks.slack.com/services/xxxx/xxxx/xxxxx
```

Use Slack for sending alerts from kubernetes:
--------------------------------------------
- By default Alerts are not set anywhere. it is **null**. No email, No Slack.
- They are only visible in the Alertmanager UI.

### How Slack Alerts Work (Quick Overview)
```
Prometheus → Alertmanager → Slack
```
- Prometheus evaluates alert rules
- Alertmanager sends notifications
- Slack receives alerts in a channel

### Step 1: Create Slack Incoming Webhook

- Go to: `Slack → Settings → Apps`
- Search and install Incoming Webhooks
- Choose your workspace
- Select a channel (e.g. #k8s-alerts)
- Copy the Webhook URL

Example: <br>
```
https://hooks.slack.com/services/T000/B000/XXXX
```
📌 Keep this URL safe. <br>

### Step 2: Add Slack config to Alertmanager

Check Alertmanager secret name <br>
```
kubectl get secret -n monitoring | grep alertmanager 
```

Usually:
```
alertmanager-prometheus-kube-prometheus-alertmanager
```

Edit Alertmanager config
```
kubectl edit secret alertmanager-prometheus-kube-prometheus-alertmanager -n monitoring
```

You will see:
```
data:
  alertmanager.yaml: <BASE64>
```
⚠️ Don’t edit base64 directly. <br>

✅ Correct way: use kubectl edit with decoded YAML <br>

Run:
```
kubectl get secret alertmanager-prometheus-kube-prometheus-alertmanager \
-n monitoring -o jsonpath='{.data.alertmanager\.yaml}' | base64 -d > alertmanager.yaml
```

Edit file: <br>

`nano alertmanager.yaml`

✍️ Example Slack Alertmanager Config (WORKING) <br>

Replace <SLACK_WEBHOOK_URL> and <CHANNEL>: <br>
```
global:
  resolve_timeout: 5m

route:
  receiver: slack-notifications
  group_by: ['alertname', 'namespace']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

receivers:
- name: slack-notifications
  slack_configs:
  - api_url: '<SLACK_WEBHOOK_URL>'  ## change webhook URL
    channel: '#k8s-alerts'        ## change channel name
    send_resolved: true
    title: '[{{ .Status | toUpper }}] {{ .CommonAnnotations.summary }}'
    text: >-
      {{ range .Alerts }}
      *Alert:* {{ .Annotations.summary }}
      *Description:* {{ .Annotations.description }}
      *Namespace:* {{ .Labels.namespace }}
      *Severity:* {{ .Labels.severity }}
      {{ end }}
```

Save file. <br>

### Step 3: Update secret with new config <br>
```
kubectl create secret generic alertmanager-prometheus-kube-prometheus-alertmanager \
-n monitoring \
--from-file=alertmanager.yaml \
--dry-run=client -o yaml | kubectl apply -f -
```

Restart Alertmanager:
```
kubectl rollout restart statefulset alertmanager-prometheus-kube-prometheus-alertmanager -n monitoring
```

### Step 4: Verify Alertmanager UI

Port-forward: <br>
```
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-alertmanager 9093
```

Open browser: <br>
```
http://localhost:9093
```

Check:
```
Status → Config (Slack receiver visible)
```
Alerts tab

### Step 5: Add an Example Alert Rule (Namespace CPU)

Create file `namespace-alert.yaml`:
```
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: namespace-cpu-alert
  namespace: monitoring
spec:
  groups:
  - name: namespace.rules
    rules:
    - alert: NamespaceHighCPUUsage
      expr: |
        sum(rate(container_cpu_usage_seconds_total{namespace="default"}[5m])) > 1
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "High CPU usage in namespace default"
        description: "CPU usage in namespace default is above threshold"
```

Apply:
```
kubectl apply -f namespace-alert.yaml
```

### Step 6: Test Alert (Quick Test)

Create a fake alert:
```
- alert: TestSlackAlert
  expr: vector(1)
  for: 1m
  labels:
    severity: info
  annotations:
    summary: "Test alert"
    description: "This is a test Slack alert"
```

You should receive a Slack message in ~1 minute.
