Setup Local Kubernetes Cluster (KinD)
---------------
- This is the kubernetes cluster running on single node and inside physical node multiple containers are running as virtual node. this is for developement testing cluster.

Installation:
```
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-$(uname)-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind
kind version
```
Create Cluster:
```
kind create cluster
```
Cluster info:
```
kubectl cluster-info --context kind-kind
```
Delete cluster (Optional): 
```
kind delete cluster
```

Create KinD Cluster with multiple nodes:
- Remove the existing cluster
- Create kind-cluster.yml
```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

nodes:
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker   # new node

```
Apply command:
```
kind create cluster --name dev-cluster --config kind-cluster.yml
```
You can check the nodes:
```
kubectl get nodes
```
<img width="671" height="114" alt="image" src="https://github.com/user-attachments/assets/53e9269a-9ef5-4caf-9177-c2ff048ffbe6" />


Basics of kubernetes cluster:
----------------------------
1. Create kubernetes cluster
2. Deploy an app
3. Explore your app
4. Expose your app publicly
5. Scale up your app
6. Update your app

Project deployment links - https://github.com/purvalpatel/Sample-mlops-project

Kubernetes cluster components:
------------------------------
https://github.com/purvalpatel/kubernetes-tutorial/edit/main/README.md
Cluster contains two type of resources.
1. Control plane		- cordinates the cluster ( responsible for managing cluster)
   - Cloud-Controller-manager
   - Etcd
   - Api-server
   - Scheduler
   - Controller-manager

2. Nodes				- workers that run the application	(VM or physical machine )
    - Kubelet - each node as agent to managing the node.
    - Containerd - tool for managing the containers on worker node.
  
Installation:
--------------

1.Swap disable

`swapoff -a`

2.Install tools: (on both master and worker)
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg


echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl conntrack
sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl enable --now kubelet
```

3.Enable IPv4 packet forwarding:
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.confnet.ipv4.ip_forward = 1
EOF

sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/99-kubernetes-ip-forward.conf

sudo sysctl --system

sysctl net.ipv4.ip_forward
```

Set container runtime for kubernetes:

Ref: https://v1-31.docs.kubernetes.io/docs/setup/production-environment/container-runtimes/

Install containerd:(on both master and worker)
Note: if containerd.io package is already there then it will fine. do not delete docker-ce if it is already installed.
```bash
sudo apt install containerd
```

Create default config ( on master and worker )
```
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```
Change default storage of containerd:
```
systemctl stop containerd
systemctl stop kubelet
Open.
nano /etc/containerd/config.toml
Change,
root = "/data/containerd"      ## provide external storage location here.

## if there is already data in /var/lib/containerd, then sync it.
sudo rsync -avx /var/lib/containerd/ /data/containerd/

## set permissions.
sudo chown -R root:root /data/containerd

## restart services
systemctl restart containerd
systemctl restart kubelet
```

Use systemd as cgroup driver
```
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

Restart service
```
sudo systemctl restart containerd
```

4.Initialize master node:
```BASH
sudo kubeadm init --pod-network-cidr=192.168.0.0/16

mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

5.Install CNI plugin:
```BASH
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

### Join worker nodes:
```
kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

Testing cluster:
```
kubectl get nodes
kubectl get pods -A
```

Now Create Local-Path-Provisioner:
------------------------------
Create Storage class:
```
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
```
Verify:
```
kubectl get storageclass
```
Then create the PVC storage local path,
Note: Make sure to use this in external storage.
```
sudo mkdir -p /opt/local-path-provisioner
sudo chmod 0777 /opt/local-path-provisioner
```
If want to change the local storage path then, change the path:
```
kubectl edit configmap local-path-config -n local-path-storage
```
Make it default storageclass:
```
# Check first
kubectl get storageclass

kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

Cluster management:
------------------

**For Single node cluster:**
```BASH
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

**Verify cluster is healthy or not:**
```
kubectl get componentstatuses
```
**Check which CIDR used at the time of kubeadm initialization:**
```
kubectl get configmap -n kube-system -o yaml
```

**Reset cluster or worker:**
```
kubeadm reset
```

**How to check which runtime kubernetes is using currently:**
```
kubectl get nodes -o wide
```
Kuberenetes can use below runtimes:

1. Containerd
2. CRI-O

#### How to get the Kubernetes token for joining nodes (if lost):

Command for joining nodes:
```
kubeadm join 10.10.110.22:6443 --token 8k5elt.852433dupe8bp53o \ --discovery-token-ca-cert-hash sha256:2ca59e3405dec39af108c05015630e27d478c454fd1b7d7c8d38573f76b4b356 
```

#### List existing token:
```
kubeadm token list
```

If no token found then create new,
```
kubeadm token create
```

To get with full join command,
```
kubeadm token create --print-join-command
```

Get the ca-cert-hash:
```
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | \
openssl rsa -pubin -outform DER 2>/dev/null | \
sha256sum | awk '{print $1}'
```
This will show the token.


Get the kubeadm server URL:
```
kubectl config view -o yaml | grep -i server
```

So the final command for join cluster is:
```
kubeadm join 10.10.20.37:6443 --token estw9q.qryflwiss25nnxxm \ --discovery-token-ca-cert-hash sha256:2ca59e3405dec39af108c05015630e27d478c454fd1b7d7c8d38573f76b4b356 
```

List all resources:
```
kubectl api-resources
```

Contexts:
---------

Contexts are connection profiles of your kubectl.

Context tells kubectl:
- which cluster to connect.
- which user credentials to use.
- which namespace to use.

1. Set default namespace.
```
kubectl config set-context my-context --namespace=mystuff
```

2. List the contexts.
```
kubectl config get-contexts
```

### Users in kubernetes:

Kubernetes does not store users inside the cluster. Like, there is no users like other service.

There is two main types of users.
1. Human users  - Admins, developers which is created on server. And they can use kubectl.
2. Service accounts - this are the actual objects inside kubernetes.
```
kubectl create serviceaccount my-app
```

If you want to allow Nvidia GPU to kubernetes then follow below link,
https://github.com/purvalpatel/kubernetes-tutorial/blob/main/NVIDIA-GPU-Pass-Kubernetes.md <br>



Kubernetes Object Categories:
----------------------------
| **Category**                   | **Objects**                                                     |
| ------------------------------ | --------------------------------------------------------------- |
| **Core**                       | Pods, Services, ConfigMap, Secret, PVC, PV, Node, Namespace     |
| **Workloads**                  | Deployment, ReplicaSet, StatefulSet, DaemonSet, Job, CronJob    |
| **Network**                    | Ingress, NetworkPolicy, EndpointSlice                           |
| **Storage**                    | StorageClass, VolumeAttachment, CSIDriver                       |
| **RBAC**                       | Role, ClusterRole, RoleBinding, ClusterRoleBinding              |
| **Autoscaling**                | HPA, VPA                                                        |
| **API Extensions**             | CRD, APIService, Webhooks                                       |
| **Full Routing (Gateway API)** | GatewayClass, Gateway, HTTPRoute, TCPRoute, UDPRoute, GRPCRoute |


## Production ready Architecture:
<img width="1421" height="851" alt="Untitled Diagram drawio(3)" src="https://github.com/user-attachments/assets/df669989-d929-493a-93a5-5583b8cb78da" />


## Install Kubernetes 1.33.9 version


Add repo:
```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Install:
```
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Initialize Control Plane:
```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

## Certificate Expiration Alerts.
- Certificate management is mostly yout responsibility if using kubeadm.
- Kubeadm mostly does not fully automate certificate like RKE2/K3s.

Check Expiration:
```
kubeadm certs check-expiration
```

Manual renew:
```
kubeadm certs renew all
```

Code for sending alerts on certificate expiry, `cert-expiry.sh`
```
#!/bin/bash

PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

THRESHOLD=30

OUTPUT=$(kubeadm certs check-expiration 2>/dev/null)

EXPIRING=$(echo "$OUTPUT" | awk -v threshold="$THRESHOLD" '

/CERTIFICATE/ {next}
/CERTIFICATE AUTHORITY/ {next}
/^$/ {next}
/Reading configuration/ {next}
/FYI:/ {next}

{
    for(i=1;i<=NF;i++) {

        if($i ~ /^[0-9]+d$/) {

            days=$i
            gsub("d","",days)

            if(days < threshold) {
                print $0
            }
        }
    }
}
')

echo "$EXPIRING -------------"
if [ -n "$EXPIRING" ]; then

    echo "Kubernetes certificates expiring soon!"
    echo "$EXPIRING"

    BOT_TOKEN="YOUR_BOT_TOKEN"
    CHAT_ID="YOUR_CHAT_ID"

    MESSAGE="⚠️ Kubernetes certificates expiring soon:

$EXPIRING"

  echo "Send alert from here."

else
    echo "All certificates are healthy."
fi
```
