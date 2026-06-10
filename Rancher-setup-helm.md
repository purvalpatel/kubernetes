Rancher Setup with Helm:
------------------------
1. Add Helm chart Repository:
```
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable 
helm repo update 
```

2. Create Namespace.
```
kubectl create namespace cattle-system 
```

3. Install Rancher with helm.
Install Rancher with Helm and Your Chosen Certificate Option.
```
helm install rancher rancher-stable/rancher --namespace cattle-system --set hostname=10.10.110.25.sslip.io --set bootstrapPassword=YOUR_PASSWORD --set ingress.tls.source=secret --set replicas=1 --set ingress.nodePort.enabled=true --set ingress.nodePort.httpsPort=30443 
```

Here, <br>
**--set ingress.tls.source=secret** <br>
- Means You are telling rancher, dont generate TLS certificates automatically.
- I will provide my own SSL certificate and key inside a Kubernetes Secret.

For example if your server have domain and SSL then you can use that Certificates. <br>
Else Generate Certificates for 10.10.110.25.sslip.io <br>
<br>
<br>

Below steps are optional, <br>
<br>

a. Generate Self-Signed Certificates:
```
openssl req -x509 -newkey rsa:4096 -nodes -keyout tls.key -out tls.crt -days 365 \
  -subj "/CN=10.10.110.25.sslip.io"
```

b. Create secret.
```
kubectl -n cattle-system create secret tls tls-rancher-ingress \
  --cert=tls.crt --key=tls.key
```
Now `https://10.10.110.25.sslip.io` works on your machine.

4. Verify
```
kubectl get all -n cattle-system
kubectl get svc –n cattle-system 
```

5. Expose NodePort to access the service.
Option 1 - Patch service.
```
kubectl -n cattle-system patch svc rancher --type='json' -p='[ 
  {"op": "replace", "path": "/spec/type", "value": "NodePort"}, 
  {"op": "replace", "path": "/spec/ports/0/nodePort", "value": 30080}, 
  {"op": "replace", "path": "/spec/ports/1/nodePort", "value": 30569} 

]' 
```

Option 2 - Create YAML <br>
Create `Nodeport-service.yaml`. <br>
```
apiVersion: v1
kind: Service
metadata:
  name: rancher
  namespace: cattle-system
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      targetPort: 80
      nodePort: 30080
    - name: https
      port: 443
      targetPort: 443
      nodePort: 30569
  selector:
    app: rancher
```

Apply:
```
kubectl apply -f nodeport-service.yaml
```

Now check by accessing it in browser. <br>
https://server-ip:30569 <br>

If port 30569 port is globally not allowed then you can check by forwarding it with ssh. <br>
```
ssh -L 30569:localhost:30569 purval_admin@xx.xx.xx.xx -p2222
```
Now open local machine broswer and open below link, <br> 
https://localhost:30569 <br>

