Check kubernetes components containers are running or not.
```
crictl --runtime-endpoint=unix:///run/containerd/containerd.sock ps -a | egrep 'etcd|apiserver|controller|scheduler'
OR
sudo crictl ps -a | grep -E "(etcd|apiserver|controller-manager|scheduler)"
```

check logs:
```
sudo crictl --runtime-endpoint=unix:///run/containerd/containerd.sock logs 6dbc317cd4eb7 | tail -100
OR
sudo crictl logs dfea48b8b1f82
```
Static pods manifests file:
```
ls /etc/kubernetes/manifests/
```

Force stop all containers:
if error:
```
rm: cannot remove '/var/lib/kubelet/pods/417fa5dc-f33f-4532-903a-f8a5d027d425/volumes/kubernetes.io~projected/kube-api-access-dmmpd': Device or resource busy

```
Then run,
```
# Stop all running containers
crictl --runtime-endpoint=unix:///run/containerd/containerd.sock stop $(crictl --runtime-endpoint=unix:///run/containerd/containerd.sock ps -q) 2>/dev/null

# Remove all containers
crictl --runtime-endpoint=unix:///run/containerd/containerd.sock rm $(crictl --runtime-endpoint=unix:///run/containerd/containerd.sock ps -aq) 2>/dev/null

# Stop all pod sandboxes
crictl --runtime-endpoint=unix:///run/containerd/containerd.sock stopp $(crictl --runtime-endpoint=unix:///run/containerd/containerd.sock pods -q) 2>/dev/null

# Remove all pod sandboxes
crictl --runtime-endpoint=unix:///run/containerd/containerd.sock rmp $(crictl --runtime-endpoint=unix:///run/containerd/containerd.sock pods -q) 2>/dev/null
```
```
# Find all mounted kubelet volumes
mount | grep kubelet | awk '{print $3}' | sort -r > /tmp/kubelet_mounts.txt

# Unmount them one by one
while read mount_point; do
  umount "$mount_point" 2>/dev/null || umount -l "$mount_point" 2>/dev/null
done < /tmp/kubelet_mounts.txt

# Force unmount any remaining
umount -l /var/lib/kubelet/pods/*/volumes/* 2>/dev/null

# Remove Kubernetes directories
rm -rf /etc/kubernetes/
rm -rf /var/lib/etcd/
rm -rf ~/.kube/
rm -rf /etc/cni/net.d/

# Force remove kubelet directory (if still fails, reboot)
rm -rf /var/lib/kubelet/ 2>/dev/null || echo "Some files still busy, will clean after reboot"

# Clean up container runtime
systemctl restart containerd
```

Verify permissions:
```
kubectl auth can-i create deployments --namespace=ml-dev
```
