---
layout: default
title: Kubernetes High-Availability On-Premise with Rook-Ceph
description: A comprehensive guide to deploying a high-availability Kubernetes cluster on-premise using Rook and Ceph for distributed, software-defined storage.
---

# Kubernetes High-Availability On-Premise with Rook-Ceph

<div class="alert info">
<p>This site provides a detailed, step-by-step guide for setting up a resilient, on-premise Kubernetes cluster with Rook-Ceph for persistent storage.</p>
</div>

## Table of Contents
- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Step 1: Prepare the Nodes](#step-1-prepare-the-nodes)
  - [Network Configuration](#network-configuration)
  - [Install Dependencies](#install-dependencies)
  - [Disable Swap](#disable-swap)
  - [Configure Kernel Modules &amp; Parameters](#configure-kernel-modules--parameters)
- [Step 2: Install Kubernetes Cluster](#step-2-install-kubernetes-cluster)
  - [Install `kubeadm`, `kubelet`, and `kubectl`](#install-kubeadm-kubelet-and-kubectl)
  - [Initialize the ControlPlane Node](#initialize-the-control-plane-node)
  - [Join Worker Nodes](#join-worker-nodes)
  - [Install a CNI Plugin (Calico)](#install-a-cni-plugin-calico)
- [Step 3: Deploy Rook-Ceph](#step-3-deploy-rook-ceph)
  - [Clone the Rook Repository](#clone-the-rook-repository)
  - [Deploy the Rook Operator](#deploy-the-rook-operator)
  - [Create the Ceph Cluster](#create-the-ceph-cluster)
  - [Verify the Ceph Cluster](#verify-the-ceph-cluster)
- [Step 4: Create and Use Storage](#step-4-create-and-use-storage)
  - [Create a CephBlockPool and StorageClass](#create-a-cephblockpool-and-storageclass)
  - [Create a PersistentVolumeClaim (PVC)](#create-a-persistentvolumeclaim-pvc)
  - [Deploy a Sample Application](#deploy-a-sample-application)
- [Troubleshooting](#troubleshooting)
- [JSON-LD (Structured Data)](#json-ld-structured-data)

---

## Architecture Overview
This guide details how to build a production-grade high-availability (HA) Kubernetes cluster on-premises with a robust, distributed storage backend using **Rook** and **Ceph**.

The Kubernetes cluster runs on a **private, isolated network**, and a dedicated **Edge Gateway VM** in the **DMZ** terminates the public traffic and forwards it to an **internal VIP**.

**Seven VMs with clear roles:**
- **1× Edge Gateway (DMZ)** → public IPs, **HAProxy** (TCP/80, TCP/443) → forwards to internal VIP.
- **3× Control‑Plane (Masters)** → HA **kube‑apiserver** via **HAProxy + Keepalived** with an **internal API VIP**.
- **3× Workers** → schedule application workloads and host **Rook‑Ceph OSDs**.

- **Kubernetes**: The container orchestrator. We will set up a multi-node HA cluster with three control plane nodes and three worker nodes.
- **Ceph**: A highly scalable, software-defined storage solution that provides block, object, and file storage. Rook will manage Ceph to provide persistent storage for our applications.

---

## Prerequisites
- **7 physical or virtual machines** running a compatible Linux distribution (e.g., Ubuntu 20.04/22.04, CentOS/RHEL 8+).
- **Static IPs** for all nodes.
- **Root or `sudo` access** on all nodes.

| Node Role       | Qty | vCPU | RAM   | OS Disk (Root) | Data Disks (Ceph OSDs)     | Rationale                                                                 |
|-----------------|-----:|-----:|------:|----------------|-----------------------------|----------------------------------------------------------------------------|
| **Edge Gateway**| 1   | 2+   | 4 GB  | 50 GB SSD      | N/A                         | Lightweight HAProxy acting as external L4 gateway.                         |
| **Master**      | 3   | 4+   | 16 GB | 100 GB SSD     | N/A                         | Handles etcd I/O and control‑plane workloads with headroom.               |
| **Worker**      | 3   | 4+   | 16 GB | 100 GB SSD     | 1× 500 GB SSD/NVMe (raw)    | Runs apps and high‑performance Ceph OSDs on raw devices.                   |

> **Note:** OSD devices must be **raw/unformatted**; Rook will prepare them.

---
## Step 1: Universal Preparation of Cluster Nodes (6 VMs)
Execute these commands on **all 6 Kubernetes nodes** (3 masters and 3 workers). The Edge Gateway is configured separately.

### Network Configuration
Ensure all nodes can communicate with each other over the network. A clear addressing plan is crucial.

| Node Role                 | Hostname        | IP Address(es)                                  | Network  |
|---------------------------|-----------------|--------------------------------------------------|----------|
| **Edge Gateway**          | `k8s-edge-gw`   | `200.x.x.10` (Public) / `10.10.1.99` (Priv) | DMZ/Priv |
| **Master 1**              | `k8s-master-1`  | `10.10.1.101`                                  | Private  |
| **Master 2**              | `k8s-master-2`  | `10.10.1.102`                                  | Private  |
| **Master 3**              | `k8s-master-3`  | `10.10.1.103`                                  | Private  |
| **Worker 1**              | `k8s-worker-1`  | `10.10.1.111`                                  | Private  |
| **Worker 2**              | `k8s-worker-2`  | `10.10.1.112`                                  | Private  |
| **Worker 3**              | `k8s-worker-3`  | `10.10.1.113`                                  | Private  |
| **Internal API VIP**      | —               | `10.10.1.120`                                  | Private  |
| **Services VIP (MetalLB)**| —               | `10.10.1.240` (example)                        | Private  |

Update `/etc/hosts` on all nodes for local DNS resolution:
```bash
cat <<EOF | sudo tee -a /etc/hosts
10.10.1.99   k8s-edge-gw
10.10.1.101  k8s-master-1
10.10.1.102  k8s-master-2
10.10.1.103  k8s-master-3
10.10.1.111  k8s-worker-1
10.10.1.112  k8s-worker-2
10.10.1.113  k8s-worker-3
EOF
```

### Install Containerd
We'll use `containerd` as the container runtime.
```bash
sudo apt-get update
sudo apt-get install -y containerd

sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
```

### Disable Swap
Kubernetes requires swap to be disabled.
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^(.*)$/#\1/g' /etc/fstab
```

### Configure Kernel Modules & Parameters
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

---

## Step 2: Install Kubernetes Cluster

### Install `kubeadm`, `kubelet`, and `kubectl`
On **all nodes**:
```bash
# Add Kubernetes APT repository
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Initialize the HA Control Plane
Before initializing the cluster, you need to set up **Internal API Load Balancing** with **HAProxy + Keepalived** on all three masters to provide the **internal API VIP `10.10.1.120`**.

**Install packages on all three masters:**
```bash
sudo apt-get update && sudo apt-get install -y haproxy keepalived
```

**Configure HAProxy on all masters** (`/etc/haproxy/haproxy.cfg`):
```haproxy
frontend kubernetes-api
    bind 10.10.1.120:6443
    mode tcp
    default_backend kubernetes-masters
backend kubernetes-masters
    mode tcp
    balance roundrobin
    server k8s-master-1 10.10.1.101:6443 check
    server k8s-master-2 10.10.1.102:6443 check
    server k8s-master-3 10.10.1.103:6443 check
```

**Configure Keepalived**: configure `k8s-master-1` as `MASTER` (priority **150**), the others as `BACKUP` (priority **100**).

**Enable services on all masters:**
```bash
sudo systemctl enable --now haproxy keepalived
```

**Initialize on `k8s-master-1`** with the internal API VIP:
```bash
sudo kubeadm init --control-plane-endpoint "10.10.1.120:6443" --upload-certs --pod-network-cidr=192.168.0.0/16
```
After initialization, run these commands to configure `kubectl`:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Save both the `kubeadm join` commands that are outputted - one for additional control planes and one for worker nodes.

### Join Additional Masters & Workers
**Join additional masters** (`k8s-master-2` and `k8s-master-3`) using the control-plane join command:
```bash
sudo kubeadm join 10.10.1.120:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash> --control-plane --certificate-key <certificate-key>
```

**Join worker nodes** using the worker join command:
```bash
sudo kubeadm join 10.10.1.120:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

### Install Flannel CNI
On any **control-plane node**, install the Flannel network plugin:
```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```
Wait a few moments, then verify that all 6 nodes (3 control planes, 3 workers) are in the `Ready` state:
```bash
kubectl get nodes -o wide
```

---

## Step 3: Deploy Rook-Ceph
With the Kubernetes cluster running, you can now deploy Rook to manage the Ceph storage cluster.

### Clone the Rook Repository
On the **control-plane node**:
```bash
git clone --single-branch --branch release-1.12 https://github.com/rook/rook.git
cd rook/deploy/examples
```

### Deploy the Rook Operator
```bash
kubectl apply -f crds.yaml -f common.yaml -f operator.yaml
```
Verify that the Rook operator is running:
```bash
kubectl -n rook-ceph get pod
```

### Create the Ceph Cluster
Now, define and create the Ceph cluster. Edit `cluster.yaml` to match your environment. Specifically, ensure the `storage` section correctly identifies the raw block devices on your worker nodes.

Example `storage` section in `cluster.yaml`:
```yaml
storage:
  useAllNodes: false
  useAllDevices: false
  nodes:
    - name: "k8s-worker-1"
      devices:
        - name: "sdb" # Replace with the name of your disk
    - name: "k8s-worker-2"
      devices:
        - name: "sdb" # Replace with the name of your disk
    - name: "k8s-worker-3"
      devices:
        - name: "sdb" # Replace with the name of your disk
```
Once configured, create the cluster:
```bash
kubectl apply -f cluster.yaml
```

### Verify the Ceph Cluster
It will take a few minutes for the Ceph cluster to be provisioned. Check the status of the pods in the `rook-ceph` namespace. You should see pods for `rook-ceph-mon`, `rook-ceph-mgr`, `rook-ceph-osd`, and others.
```bash
kubectl -n rook-ceph get pods --watch
```
Wait for the `rook-ceph-mon`, `rook-ceph-mgr`, and `rook-ceph-osd-prepare` pods to complete their work and see 3 `rook-ceph-osd-X` pods in the `Running` state.

**Deploy toolbox and verify storage backend:**
```bash
kubectl apply -f toolbox.yaml
kubectl -n rook-ceph exec -it $(kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}') -- bash
```

**Health checks inside the toolbox:**
- **General Health:** `ceph status` (should show `health: HEALTH_OK`)
- **OSDs:** `ceph osd tree` (should show 3 OSDs, all `up` and `in`, each associated with a different worker node)

---

## Step 4: Create and Use Storage

### Create a CephBlockPool and StorageClass
Create `CephBlockPool` and `StorageClass` for dynamic provisioning:

```yaml
# blockpool.yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
---
# storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  imageFormat: "2"
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
```

Apply and confirm:
```bash
kubectl apply -f blockpool.yaml
kubectl apply -f storageclass.yaml
kubectl get sc # Should show the new StorageClass 'rook-ceph-block'
```

### Create a PersistentVolumeClaim (PVC)
Now, you can request storage by creating a `PersistentVolumeClaim`.

Create a file named `pvc.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: rook-ceph-block
```
Apply it:
```bash
kubectl apply -f pvc.yaml
```
Check that the PVC is created and bound:
```bash
kubectl get pvc my-pvc
```

### Deploy a Sample Application
Finally, deploy a pod that uses this PVC.

Create a file named `pod.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-container
    image: busybox
    command: ["/bin/sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: my-storage
      mountPath: /data
  volumes:
  - name: my-storage
    persistentVolumeClaim:
      claimName: my-pvc
```
Apply it:
```bash
kubectl apply -f pod.yaml
```
Verify the pod is running and has mounted the volume.

---

## Troubleshooting
- **Pods stuck in `Pending`**: This often indicates a problem with the CNI or resource constraints. Check `kubectl describe pod <pod-name>`.
- **PVC not binding**: Check the `rook-ceph-operator` logs and the `StorageClass` definition.
- **Ceph health warnings**: Use `ceph status` inside the tools pod to diagnose issues with OSDs, MONs, or placement groups.

---

## JSON-LD (Structured Data)
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "TechArticle",
  "headline": "Kubernetes High-Availability On-Premise with Rook-Ceph",
  "description": "A comprehensive guide to deploying a high-availability Kubernetes cluster on-premise using Rook and Ceph for distributed, software-defined storage.",
  "about": ["Kubernetes", "Rook", "Ceph", "High-Availability", "On-Premise", "Storage"],
  "inLanguage": "en",
  "license": "https://www.apache.org/licenses/LICENSE-2.0",
  "creator": {
    "@type": "Person",
    "name": "Miguel Saca"
  }
}
</script>
