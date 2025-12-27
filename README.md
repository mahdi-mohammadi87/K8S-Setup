
# Kubernetes Cluster Setup using kubeadm & containerd (lab)

---

## Cluster Topology

| Role          | Hostname      | FQDN                       | IP Address       |
| ------------- | ------------- | -------------------------- | ---------------- |
| Control Plane | `k8s-master`  | `k8s-master.devops.local`  | `192.168.100.20` |
| Worker Node 1 | `k8s-worker1` | `k8s-worker1.devops.local` | `192.168.100.21` |
| Worker Node 2 | `k8s-worker2` | `k8s-worker2.devops.local` | `192.168.100.22` |

---

## Versions

* Kubernetes: **v1.33.2**
* kubeadm / kubelet / kubectl: `1.33.2-1.1`
* Container Runtime: **containerd**
* Pause image: `registry.k8s.io/pause:3.10`
* CNI: **Calico v3.27.0**

---

## 1. Base OS Preparation (ALL NODES)

### 1.1 Install required base packages

```bash
sudo apt-get update -y
sudo apt-get install -y --no-install-recommends \
  curl ca-certificates gnupg lsb-release apt-transport-https
```

---

### 1.2 Configure hostname (run only the correct command per node)

**Control Plane**

```bash
sudo hostnamectl set-hostname "k8s-master.devops.local"
```

**Worker Node 1**

```bash
sudo hostnamectl set-hostname "k8s-worker1.devops.local"
```

**Worker Node 2**

```bash
sudo hostnamectl set-hostname "k8s-worker2.devops.local"
```

Re-login or restart your shell after changing hostname.

---

### 1.3 Configure `/etc/hosts` (MANDATORY — ALL NODES)

Improper hostname resolution will break:

* kubeadm init
* kubeadm join
* API server TLS
* CNI networking

Clean old entries and append correct mappings:

```bash
sudo sed -i '/k8s-master.devops.local/d' /etc/hosts
sudo sed -i '/k8s-worker/d' /etc/hosts

cat <<'EOF' | sudo tee -a /etc/hosts
192.168.100.20 k8s-master.devops.local k8s-master
192.168.100.21 k8s-worker1.devops.local k8s-worker1
192.168.100.22 k8s-worker2.devops.local k8s-worker2
EOF
```

---

### 1.4 Hard validation (WORKERS ONLY)

If these checks fail, **STOP** and fix DNS / routing.

```bash
getent hosts k8s-master.devops.local
curl -k https://k8s-master.devops.local:6443/healthz
nc -vz k8s-master.devops.local 6443
```

Expected result:

* Hostname resolves
* API server responds with `ok`
* Port `6443` is reachable

---

## 2. Install and Configure containerd (ALL NODES)

### 2.1 Install containerd

```bash
sudo apt update
sudo apt-get install -y containerd
```

---

### 2.2 Configure containerd for Kubernetes

* Enable `SystemdCgroup`
* Set pause image to Kubernetes-compatible version

```bash
sudo mkdir -p /etc/containerd

containerd config default \
| sed 's/SystemdCgroup = false/SystemdCgroup = true/' \
| sed 's|sandbox_image = ".*"|sandbox_image = "registry.k8s.io/pause:3.10"|' \
| sudo tee /etc/containerd/config.toml > /dev/null

sudo systemctl restart containerd
```

---

### 2.3 Disable swap (required by Kubernetes)

```bash
sudo swapoff -a
```

> Persistent swap disabling via `/etc/fstab` is environment-specific and intentionally not enforced here.

---

## 3. Install Kubernetes Components (ALL NODES)

### 3.1 Prepare APT and GPG keyring

```bash
sudo apt update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p -m 755 /etc/apt/keyrings
```

---

### 3.2 Add Kubernetes APT repository (v1.33)

```bash
sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

---

### 3.3 Install pinned Kubernetes versions

```bash
sudo apt-get update

KUBE_VERSION="1.33.2-1.1"

sudo apt-get install -y \
  kubelet=$KUBE_VERSION \
  kubeadm=$KUBE_VERSION \
  kubectl=$KUBE_VERSION

sudo apt-mark hold kubelet kubeadm kubectl
```

---

## 4. Enable IP Forwarding (ALL NODES)

Required for pod networking and CNI routing.

```bash
# Required kernel module for Kubernetes networking (Calico/iptables)
sudo modprobe br_netfilter

# Ensure it loads on reboot
echo br_netfilter | sudo tee /etc/modules-load.d/br_netfilter.conf
```

```bash
# Kubernetes networking prerequisites (ALL NODES)
sudo modprobe br_netfilter
echo br_netfilter | sudo tee /etc/modules-load.d/br_netfilter.conf

cat <<'EOF' | sudo tee /etc/sysctl.d/99-kubernetes-network.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

```bash
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.ipv4.ip_forward
lsmod | grep br_netfilter
```

---

## 5. Initialize Kubernetes Control Plane (CONTROL PLANE ONLY)

### 5.1 Initialize cluster

```bash
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket=unix:///run/containerd/containerd.sock
```

---

### 5.2 Configure kubectl access

```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

### 5.3 Reset instructions (if reinitialization is required)

```bash
sudo kubeadm reset --cri-socket=unix:///run/containerd/containerd.sock
sudo rm -rf /etc/kubernetes /var/lib/etcd
```

---

## 6. Install CNI Plugin (CONTROL PLANE ONLY)

### Calico (Recommended)

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```

Wait until all `kube-system` pods are running.

---

## 7. Join Worker Nodes to the Cluster (WORKERS ONLY)

After `kubeadm init`, a **join command** is printed.

Run **that exact command** on each worker node.

⚠️ **DO NOT reuse example tokens**

```bash
sudo kubeadm join <CONTROL_PLANE_IP>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

If lost, regenerate on the control plane:

```bash
kubeadm token create --print-join-command
```

---

## 8. Post-Installation Validation (CONTROL PLANE)

```bash
kubectl get nodes -o wide
kubectl get pods -n kube-system -o wide
```

