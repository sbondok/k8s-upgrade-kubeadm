````markdown
# Upgrade Procedure for a kubeadm Cluster from v1.28 to v1.29

This document details the step-by-step process for upgrading a kubeadm-managed Kubernetes cluster from version 1.28 to 1.29.
The same steps can be used to perform upgrading k8s to the latest version 1.33, which I've done already in my home lab.

Engaging in this structured learning exercise—by performing the complete upgrade cycle for each version increment from 1.29 to 1.33—served as a catalyst for deeper technical inquiry. Each iteration demystified another layer of the system, reinforcing my understanding of cluster operations and providing a lucid perspective on the architectural principles of Kubernetes

## Cluster Topology:

- **Control Plane Node**: `k8s-control`
- **Worker Nodes**: `k8s-node1`, `k8s-node2`

---

## Phase 1: Control Plane Node Upgrade (`k8s-control`)

The upgrade process begins on the single control plane node.

### 1. Update Package Repository Source

First, update the APT package repository to point to the desired Kubernetes version (v1.29).

```bash
# Gain root privileges
sudo -i

# Edit the Kubernetes repository source list
vim /etc/apt/sources.list.d/kubernetes.list
```
````

In the editor, modify the line to change the version from `/v1.28/` to `/v1.29/`:

```text
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /
```

Save the file and exit.  
Update the package index with the new repository:

```bash
apt update
```

### 2. Upgrade `kubeadm`

Identify the available `kubeadm` versions and install the target version.

```bash
# List available versions of kubeadm
apt-cache madison kubeadm

# Unhold, install the specific version, and re-hold the package
apt-mark unhold kubeadm && \
apt-get update && \
apt-get install -y kubeadm='1.29.0-1.1' && \
apt-mark hold kubeadm
```

Pinning to a specific patch version like `1.29.0-1.1` provides deterministic upgrades. Alternatively, you could install the latest available patch for the minor version by using a wildcard. The official documentation often recommends this approach to ensure you get the latest security fixes for that minor release.

```bash
# Verify the kubeadm version after installation
kubeadm version
```

### 3. Plan and Apply the Upgrade

Check the cluster state and apply the control plane upgrade.

```bash
# Verify the cluster is ready for upgrade and review the proposed changes
kubeadm upgrade plan

# Apply the upgrade to the control plane components
kubeadm upgrade apply v1.29.0
```

### 4. Upgrade kubelet and kubectl on the Control Plane Node

As noted, the kubelet is not automatically upgraded. Before upgrading it, the node must be drained to safely move its workloads.

```bash
# Exit the root session to run kubectl as a regular user
exit

# Drain the control plane node to evict pods
kubectl drain k8s-control --ignore-daemonsets
```

After draining, check the node's status. It should show `Ready,SchedulingDisabled`:

```bash
kubectl get nodes
```

Now, proceed with upgrading the `kubelet` and `kubectl` packages.

```bash
# Regain root privileges
sudo -i

# Unhold, install, and re-hold the kubelet and kubectl packages
apt-mark unhold kubelet kubectl && \
apt-get update && \
apt-get install -y kubelet='1.29.0-1.1' kubectl='1.29.0-1.1' && \
apt-mark hold kubelet kubectl

# Reload the systemd manager configuration and restart the kubelet
systemctl daemon-reload
systemctl restart kubelet

# Verify the kubelet service is active and running
systemctl status kubelet
```

### 5. Uncordon the Control Plane Node

Bring the node back into service by making it schedulable again.

```bash
# Exit root session
exit

# Mark the node as schedulable
kubectl uncordon k8s-control
```

Verify the final status and version of the control plane node:

```bash
kubectl get nodes
```

The output should now show `k8s-control` as `Ready` with `v1.29.0`.

---

## Phase 2: Worker Node Upgrade (`k8s-node1`)

The following steps must be performed on each worker node, one at a time.

### 1. Upgrade kubeadm on the Worker Node

Log into `k8s-node1` and perform the same repository and kubeadm package upgrade as on the control plane.

```bash
# On k8s-node1, gain root privileges
sudo -i

# Edit the repository source to point to v1.29
vim /etc/apt/sources.list.d/kubernetes.list
# ... change v1.28 to v1.29 ...
# Update package list
apt update

# Upgrade kubeadm
apt-mark unhold kubeadm && \
apt-get update && \
apt-get install -y kubeadm='1.29.0-1.1' && \
apt-mark hold kubeadm
```

### 2. Upgrade the Worker Node Configuration

Run `kubeadm upgrade node` to update the local kubelet configuration on the worker.

```bash
# On k8s-node1
kubeadm upgrade node
```

### 3. Drain the Worker Node

From the control plane node (`k8s-control`), drain the worker node.

```bash
# On k8s-control
kubectl drain k8s-node1 --ignore-daemonsets --force --delete-emptydir-data
```

> **Note**: Using `--force` is necessary when pods are not managed by a ReplicaSet, ReplicationController, Job, DaemonSet, or StatefulSet. The `--delete-emptydir-data` flag is required if pods use `emptyDir` volumes, as their data will be lost upon eviction. Use these flags with a clear understanding of their impact on your specific workloads.

### 4. Upgrade kubelet and kubectl on the Worker Node

Back on the worker node (`k8s-node1`), upgrade the remaining packages and restart the kubelet.

```bash
# On k8s-node1
apt-mark unhold kubelet kubectl && \
apt-get update && \
apt-get install -y kubelet='1.29.0-1.1' kubectl='1.29.0-1.1' && \
apt-mark hold kubelet kubectl

systemctl daemon-reload
systemctl restart kubelet
systemctl status kubelet
```

### 5. Uncordon the Worker Node

From the control plane, bring the worker node back online.

```bash
# On k8s-control
kubectl uncordon k8s-node1
```

---

## Final Steps

Repeat all steps from **Phase 2** for the remaining worker node (`k8s-node2`).  
After all nodes are upgraded, run:

```bash
kubectl get nodes
```

This verifies that the entire cluster is `Ready` and running `v1.29.0`.

---

By following this structured upgrade process, cluster availability is maintained. This sequential approach is the recommended practice for kubeadm cluster upgrades.

```

```

