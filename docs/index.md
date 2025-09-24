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
This guide details how to build a high-availability (HA) Kubernetes cluster with a robust, distributed storage backend using **Rook** and **Ceph**.

- **Kubernetes**: The container orchestrator. We will set up a multi-node cluster with one control plane and multiple worker nodes.
- **Rook**: An open-source cloud-native storage orchestrator for Kubernetes. Rook automates the deployment, configuration, scaling, and management of storage services.
- **Ceph**: A highly scalable, software-defined storage solution that provides block, object, and file storage. Rook will manage Ceph to provide persistent storage for our applications.

---

## Prerequisites
- **3 or more physical or virtual machines** running a compatible Linux distribution (e.g., Ubuntu 20.04/22.04, CentOS/RHEL 8+).
- **Static IPs** for all nodes.
- **Root or `sudo` access** on all nodes.

| Node Role       | Qty | vCPU | RAM   | OS Disk (Root) | Data Disks (Ceph OSDs)     | Rationale                                                                 |
|-----------------|-----:|-----:|------:|----------------|-----------------------------|----------------------------------------------------------------------------|
| **Control-Plane**| 1+  | 2+   | 4 GB+ | 100 GB SSD     | N/A                         | Handles etcd I/O and control-plane workloads with headroom.               |
| **Worker**      | 2+  | 4+   | 16 GB+| 100 GB SSD     | 1× 500 GB SSD/NVMe (raw)    | Runs apps and high-performance Ceph OSDs on raw devices.                   |

> **Note:** OSD devices must be **raw/unformatted**; Rook will prepare them.

---

## Step 1: Prepare the Nodes
Execute these commands on **all nodes** (control-plane and workers).

### Network Configuration
Ensure all nodes can communicate with each other over the network. A clear addressing plan is crucial.

| Node Role                 | Hostname        | IP Address(es)                                  | Network  |
|---------------------------|-----------------|--------------------------------------------------|----------|
| **Control-Plane**              | `k8s-master`  | `10.10.1.101`                                  | Private  |
| **Worker 1**              | `k8s-worker-1`  | `10.10.1.111`                                  | Private  |
| **Worker 2**              | `k8s-worker-2`  | `10.10.1.112`                                  | Private  |
| **Internal API VIP**      | —               | `10.10.1.120`                                  | Private  |
| **Services VIP (MetalLB)**| —               | `10.10.1.240` (example)                        | Private  |

Update `/etc/hosts` on all nodes for local DNS resolution:
```bash
cat <<EOF | sudo tee -a /etc/hosts
10.10.1.101  k8s-master
10.10.1.111  k8s-worker-1
10.10.1.112  k8s-worker-2
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

### Initialize the Control Plane Node
On the **control-plane node only**:
```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --control-plane-endpoint=k8s-master
```
After initialization, run these commands to configure `kubectl`:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Save the `kubeadm join` command that is outputted. You will need it to join the worker nodes.

### Join Worker Nodes
On **each worker node**, run the `kubeadm join` command you saved from the previous step. It will look something like this:
```bash
sudo kubeadm join k8s-master:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

### Install a CNI Plugin (Calico)
On the **control-plane node**, install the Calico network plugin:
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
```
Wait a few moments, then verify that all nodes are in the `Ready` state:
```bash
kubectl get nodes
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
  useAllNodes: true
  useAllDevices: false
  devices:
    - name: "sdb"
    - name: "nvme0n1"
```
Once configured, create the cluster:
```bash
kubectl apply -f cluster.yaml
```

### Verify the Ceph Cluster
It will take a few minutes for the Ceph cluster to be provisioned. Check the status of the pods in the `rook-ceph` namespace. You should see pods for `rook-ceph-mon`, `rook-ceph-mgr`, `rook-ceph-osd`, and others.
```bash
kubectl -n rook-ceph get pods
```
Once all pods are running, you can check the Ceph status by exec-ing into the `rook-ceph-tools` pod:
```bash
kubectl -n rook-ceph exec -it $(kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}') -- ceph status
```

---

## Step 4: Create and Use Storage

### Create a CephBlockPool and StorageClass
First, create a `CephBlockPool`, which defines a logical pool for block storage. Then, create a `StorageClass` that uses this pool.

Navigate to `rook/deploy/examples/csi/rbd` and apply the `storageclass.yaml`:
```bash
cd ../csi/rbd # From the examples directory
kubectl apply -f storageclass.yaml
```
This creates a `StorageClass` named `rook-ceph-block`.

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
