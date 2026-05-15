If you are using kubeadm, then certificate management is mostly your responsibility. <br>

**Kubeadm does NOT fully automate** certificate lifecycle like **RKE2/K3s**. <br>

So you should implement: <br>
- expiry monitoring
- renewal automation
- alerts

especially for production/self-hosted clusters. <br> <br>

### Check Certificate Expiration

Run on control plane node:
```
kubeadm certs check-expiration
```

Output:
```
CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                 Apr 06, 2027 05:10 UTC   325d            ca                      no      
apiserver                  Apr 06, 2027 05:10 UTC   325d            ca                      no      
apiserver-etcd-client      Apr 06, 2027 05:10 UTC   325d            etcd-ca                 no      
apiserver-kubelet-client   Apr 06, 2027 05:10 UTC   325d            ca                      no      
controller-manager.conf    Apr 06, 2027 05:10 UTC   325d            ca                      no      
etcd-healthcheck-client    Apr 06, 2027 05:10 UTC   325d            etcd-ca                 no      
etcd-peer                  Apr 06, 2027 05:10 UTC   325d            etcd-ca                 no      
etcd-server                Apr 06, 2027 05:10 UTC   325d            etcd-ca                 no      
front-proxy-client         Apr 06, 2027 05:10 UTC   325d            front-proxy-ca          no      
scheduler.conf             Apr 06, 2027 05:10 UTC   325d            ca                      no      
super-admin.conf           Apr 06, 2027 05:10 UTC   325d            ca                      no      

CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
ca                      Mar 30, 2035 06:20 UTC   8y              no      
etcd-ca                 Mar 30, 2035 06:20 UTC   8y              no      
front-proxy-ca          Mar 30, 2035 06:20 UTC   8y              no      
```

Critical ones:
```
apiserver
apiserver-kubelet-client
front-proxy-client
etcd-server
etcd-peer
admin.conf
scheduler.conf
controller-manager.conf
```

### Renew Certificates
Manual renewal:
```
kubeadm certs renew all
```
Then restart kubelet:
```
systemctl restart kubelet
Restart Static Pods
```
Sometimes needed:
```
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
sleep 10
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
```
Repeat for:
```
controller-manager
scheduler
etcd
```
Kubelet recreates pods automatically.

## Best Practice
For kubeadm clusters: <br>

Always Have: <br>
- Prometheus
- Alertmanager
- cert-exporter
- renewal SOP/documentation
- etcd backups
