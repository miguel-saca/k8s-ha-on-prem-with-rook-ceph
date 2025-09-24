# Production-Grade On‑Prem Kubernetes HA Cluster with Rook‑Ceph and an Edge Gateway

> **Scope.** This guide describes how to build a **high‑availability Kubernetes cluster on‑premises** with a design that prioritizes security, resilience and with a **DMZ Edge Gateway**, **internal VIPs** via **MetalLB** handling all incoming and outgoing traffic, and **Rook‑Ceph** for distributed storage.

## Table of Contents
- [Production-Grade On‑Prem Kubernetes HA Cluster with Rook‑Ceph and an Edge Gateway](#production-grade-onprem-kubernetes-ha-cluster-with-rookceph-and-an-edge-gateway)
  - [Table of Contents](#table-of-contents)
  - [Architecture Overview](#architecture-overview)
    - [Service Exposure Options](#service-exposure-options)
    - [Cluster Topology](#cluster-topology)
  - [Hardware \& Network Specifications](#hardware--network-specifications)
    - [Network Addressing Plan](#network-addressing-plan)
  - [Universal Preparation of Cluster Nodes (6 VMs)](#universal-preparation-of-cluster-nodes-6-vms)
    - [System Configuration \& Kubernetes Prerequisites](#system-configuration--kubernetes-prerequisites)
    - [Install containerd \& Kubernetes Tools](#install-containerd--kubernetes-tools)
  - [Edge Gateway Configuration (1 VM)](#edge-gateway-configuration-1-vm)
  - [Build the HA Kubernetes Cluster](#build-the-ha-kubernetes-cluster)
    - [Internal API Load Balancing (HAProxy + Keepalived)](#internal-api-load-balancing-haproxy--keepalived)
    - [Cluster Initialization \& Node Join](#cluster-initialization--node-join)
  - [Distributed Storage with Rook‑Ceph](#distributed-storage-with-rookceph)
    - [Prepare Worker Nodes](#prepare-worker-nodes)
    - [Deploy Rook Operator](#deploy-rook-operator)
    - [Create the Ceph Cluster](#create-the-ceph-cluster)
    - [Verify Storage Backend](#verify-storage-backend)
    - [Dynamic Provisioning (StorageClass)](#dynamic-provisioning-storageclass)
  - [Expose Services with Gateway API](#expose-services-with-gateway-api)
    - [Install MetalLB](#install-metallb)
    - [Install Gateway Components (CRDs, Ingress Controller, TLS)](#install-gateway-components-crds-ingress-controller-tls)
    - [Sample Web App \& DNS](#sample-web-app--dns)
  - [Future Scalability](#future-scalability)
  - [References](#references)

---

## Architecture Overview

The Kubernetes cluster runs on a **private, isolated network**, and a dedicated **Edge Gateway VM** in the **DMZ** terminates the public traffic and forwards it to an **internal VIP**.

### Service Exposure Options

1) **Edge L4 → Internal VIP with MetalLB (Recommended)**: the Edge Gateway runs an **L4 TCP proxy** (HAProxy) that forwards to a **stable internal VIP** managed by **MetalLB**. This fully decouples the public DMZ from the cluster internals and gives you flexibility and security.

2) **Gateway API vs. Ingress**: **Gateway API** is the modern successor to classic Ingress, with stronger role separation and a richer, safer configuration model. Use Gateway API with an NGINX controller on bare metal.

> **Why this matters:** The internal VIP lets you move workloads or scale nodes without reconfiguring the DMZ. Gateway API improves multi‑team ops and security boundaries.

### Cluster Topology

Seven VMs with clear roles:

- **1× Edge Gateway (DMZ)** → public IPs, **HAProxy** (TCP/80, TCP/443) → forwards to internal VIP.
- **3× Control‑Plane (Masters)** → HA **kube‑apiserver** via **HAProxy + Keepalived** with an **internal API VIP**.
- **3× Workers** → schedule application workloads and host **Rook‑Ceph OSDs**.

---

## Hardware & Network Specifications

| Node Role       | Qty | vCPU | RAM   | OS Disk (Root) | Data Disks (Ceph OSDs)     | Rationale                                                                 |
|-----------------|-----:|-----:|------:|----------------|-----------------------------|----------------------------------------------------------------------------|
| **Edge Gateway**| 1   | 2+   | 4 GB  | 50 GB SSD      | N/A                         | Lightweight HAProxy acting as external L4 gateway.                         |
| **Master**      | 3   | 4+   | 16 GB | 100 GB SSD     | N/A                         | Handles etcd I/O and control‑plane workloads with headroom.               |
| **Worker**      | 3   | 4+   | 16 GB | 100 GB SSD     | 1× 500 GB SSD/NVMe (raw)    | Runs apps and high‑performance Ceph OSDs on raw devices.                   |

> **Note:** OSD devices must be **raw/unformatted** at first boot; Rook will prepare them.

### Network Addressing Plan

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

---

## Universal Preparation of Cluster Nodes (6 VMs)

Perform on the **three masters and three workers**. The Edge Gateway is handled later.

### System Configuration & Kubernetes Prerequisites

**Set hostnames and `/etc/hosts`** (all six nodes) and ensure mutual name resolution.

**Disable swap permanently**
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
- **Check:** `sudo swapon --show || true` or `free -h`. The output for swap must be empty or zero.

**Load kernel modules:** Enable `overlay` and `br_netfilter`.

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
```
- **Check:** `lsmod | grep overlay` and `lsmod | grep br_netfilter` should show the loaded modules.

**Enable sysctl settings**
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```
- **Check:** `sysctl net.ipv4.ip_forward` should return `net.ipv4.ip_forward = 1`.

### Install containerd & Kubernetes Tools

**containerd (systemd cgroups)**
Install `containerd`: Install the container runtime environment and configure it to use the `systemd` cgroup driver.
```bash
sudo apt-get update && sudo apt-get install -y containerd.io
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
```
- **Check:** `systemctl status containerd --no-pager` should show the service as `active (running)`.

**kubeadm, kubelet, kubectl (pin to Kubernetes v1.28 channel)**
Install `kubeadm`, `kubelet`, `kubectl`: Install the Kubernetes tools and set their versions.
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key   | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /'   | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
- **Check:** `kubelet --version` should show the installed version.

---

## Edge Gateway Configuration (1 VM)

> VM: `k8s-edge-gw` (DMZ). Purpose: **public TCP 80/443** → internal VIP `10.10.1.240` (MetalLB).

**Install HAProxy**
```bash
sudo apt-get update && sudo apt-get install -y haproxy
```

**Configure `/etc/haproxy/haproxy.cfg`**
This configuration accepts traffic on ports 80 and 443 of the public IP and forwards it to the internal VIP (`10.10.1.240`) which will be managed by MetalLB later.
```haproxy
frontend http_in
    bind 200.x.x.10:80
    mode tcp
    default_backend k8s_http_ingress

frontend https_in
    bind 200.x.x.10:443
    mode tcp
    default_backend k8s_https_ingress

backend k8s_http_ingress
    mode tcp
    server k8s_vip 10.10.1.240:80 check

backend k8s_https_ingress
    mode tcp
    server k8s_vip 10.10.1.240:443 check
```

**Enable & check**
```bash
sudo systemctl restart haproxy && sudo systemctl enable haproxy
systemctl status haproxy --no-pager
```
- **Check:** `systemctl status haproxy` should show the service as `active (running)`. `ss -tlpn | grep -E ':80|:443'` should show that HAProxy is listening on ports 80 and 443.

---

## Build the HA Kubernetes Cluster

### Internal API Load Balancing (HAProxy + Keepalived)

> Run on **all three masters** to provide the **internal API VIP `10.10.1.120`**.

**Install packages**
```bash
sudo apt-get update && sudo apt-get install -y haproxy keepalived
```

**HAProxy config (`/etc/haproxy/haproxy.cfg`)**
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

**Keepalived**: configure `k8s-master-1` as `MASTER` (priority **150**), the others as `BACKUP` (priority **100**).

**Enable & check**
```bash
sudo systemctl enable --now haproxy keepalived
```
- **Check:** On `k8s-master-1`, run `ip a`. You should see the VIP `10.10.1.120` assigned to the network interface.

### Cluster Initialization & Node Join

**Init on `k8s-master-1`**, create a `kubeadm-config.yaml` file pointing to the internal API VIP and run `kubeadm init`.
```bash
sudo kubeadm init --control-plane-endpoint "10.10.1.120:6443" --upload-certs
```

**Set up kubectl (on master‑1)**
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**Install Flannel CNI**
```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

**Join additional masters & workers** using the `kubeadm join` commands printed during init.


- **Check:** `kubectl get nodes -o wide`. You should see all 6 nodes (3 control planes, 3 workers) in `Ready` state.

---

## Distributed Storage with Rook‑Ceph

### Prepare Worker Nodes

**Identify the extra raw disk**
```bash
# On each worker: k8s-worker-{1..3}
lsblk -f
```
- **Check:** The output should show your data disk (e.g. `sdb`) without `FSTYPE`, `LABEL`, `UUID` or `MOUNTPOINT`.
  
**Install LVM**
```bash
sudo apt-get update && sudo apt-get install -y lvm2
```
- **Check:** `which lvm` should return a path, confirming the installation.

### Deploy Rook Operator

```bash
git clone --single-branch --branch release-1.12 https://github.com/rook/rook.git
cd rook/deploy/examples
kubectl apply -f crds.yaml -f common.yaml -f operator.yaml
```
- **Check:** `kubectl -n rook-ceph get pod -l app=rook-ceph-operator` Wait until the pod is in `Running` state.

### Create the Ceph Cluster

Create `cluster.yaml` and adjust device names if needed:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephVersion:
    image: quay.io/ceph/ceph:v17.2.6
  dataDirHostPath: /var/lib/rook
  mon:
    count: 3
    allowMultiplePerNode: false
  dashboard:
    enabled: true
    ssl: true
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

Apply and watch:
```bash
kubectl apply -f cluster.yaml
```
- **Check:** `kubectl -n rook-ceph get pod --watch`. Wait for the `rook-ceph-mon`, `rook-ceph-mgr`, and `rook-ceph-osd-prepare` pods to complete their work and see 3 `rook-ceph-osd-X` pods in the `Running` state.

### Verify Storage Backend

**Deploy toolbox and connect.** The toolbox is a pod containing Ceph client tools for debugging.
```bash
kubectl apply -f toolbox.yaml
kubectl -n rook-ceph exec -it $(kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}') -- bash
```

**Health checks inside the toolbox**
- **Check 1 (General Health):** `ceph status`. The output should show `health: HEALTH_OK`.
- **Check 2 (OSDs):** `ceph osd tree`. It should show 3 OSDs (`osd.0`, `osd.1`, `osd.2`), all `up` and `in`, each associated with a different worker node.

### Dynamic Provisioning (StorageClass)

Create `CephBlockPool` and `StorageClass`:

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
```
- **Check:** `kubectl get sc`. You should see the new `StorageClass` `rook-ceph-block`.

---

## Expose Services with Gateway API

### Install MetalLB
MetalLB will provide the **internal services VIP** (`10.10.1.240`) that the Edge Gateway points to.

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml
```

**Configure IP Pool (Layer 2 Mode):** Create `metallb-config.yaml` to define the IP range. Layer 2 mode is simpler for on-premise setups that don't have BGP routers.:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: private-vip-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.10.1.240-10.10.1.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
```

Apply & check:
```bash
kubectl apply -f metallb-config.yaml
```
- **Check:** `kubectl get pods -n metallb-system`. The `controller` and the `speakers` (one per node) must be `Running`.


### Install Gateway Components (CRDs, Ingress Controller, TLS)

**Gateway API CRDs**
```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml
```

**Ingress‑NGINX controller for bare metal**
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/baremetal/deploy.yaml
```
**Expose the Controller with MetalLB:**
```bash
kubectl edit svc ingress-nginx-controller -n ingress-nginx
# Copy spec.type from NodePort to LoadBalancer
```
- **Check:** `kubectl get svc -n ingress-nginx`. The `ingress-nginx-controller` service must have an `EXTERNAL-IP` from the MetalLB range (e.g. 10.10.1.240).

**cert‑manager + ClusterIssuer (Let’s Encrypt)**
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml
```
- **Check:** `kubectl get pods -n cert-manager`. Wait for all pods to be `Running`.

### Sample Web App & DNS

- Point your **public DNS** (e.g., `your-domain.com`) to the **Edge Gateway public IP** `200.x.x.10`.
- Deploy a simple app with **Gateway/HTTPRoute** (or classic Ingress) and a **Certificate** referencing `letsencrypt-prod`.

**Final checks**
```bash
kubectl get certificate
# Visit:
https://your-domain.com
```

---

## Future Scalability

**Add a new worker node**
```bash
# On a master:
sudo kubeadm token create --print-join-command
# Run the printed join command on the new worker VM.
kubectl get nodes
```

> **Tip:** For storage growth, add more (uniform) OSD devices per worker or add worker nodes and extend the Ceph cluster spec accordingly.

---

## References

- Kubernetes Services, MetalLB & bare‑metal exposure; pros/cons of NodePort vs LoadBalancer and VIPs.
- HA control plane using **Keepalived + HAProxy**.
- Rook‑Ceph deployment with **v17.2.6** Ceph image, Toolbox verification and RBD StorageClass.
- Gateway API (v1.0.0), **Ingress‑NGINX** controller for bare‑metal.
