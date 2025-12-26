
<div align="center">
  <h1>Kubernetes Setup</h1>
  <p><strong>Ubuntu 22.04 LTS • kubeadm • containerd • Calico CNI</strong></p>
</div>

<br/>

## 📋 Table of Contents
* [Prerequisites](#prerequisites)
* [Node Preparation](#node-preparation)
* [Containerd Setup](#containerd-setup)
* [Kubernetes Components](#kubernetes-components)
* [Master Node](#master-node)
* [Calico CNI](#calico-cni)
* [Worker Nodes](#worker-nodes)
* [Verification](#verification)
* [Reset Script](#reset-script)
* [Production Hardening](#hardening)

---

## 🧩 Prerequisites {#prerequisites}

| Component | Master Node | Worker Node |
|-----------|-------------|-------------|
| **CPU** | 2+ cores | 2+ cores |
| **RAM** | 4GB+ | 2GB+ |
| **Disk** | 20GB+ | 20GB+ |
| **OS** | Ubuntu 22.04 LTS | Ubuntu 22.04 LTS |

**Network Requirements:**
- Static IPs on all nodes
- DNS resolution between nodes
- Open ports: `6443`, `10250`, `2379-2380`, `10251-10252`

---

## ⚙️ Node Preparation (All Nodes) {#node-preparation}

```
# ========================================
# 1. DISABLE SWAP (CRITICAL)
# ========================================
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# ========================================
# 2. KERNEL & NETWORKING
# ========================================
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

<details>
<summary>✅ Verify Preparation</summary>

```
# Check swap is OFF
swapon --show=false

# Check modules loaded
lsmod | grep -e overlay -e br_netfilter

# Check sysctl
sysctl net.bridge.bridge-nf-call-iptables net.ipv4.ip_forward
```
</details>

---

## 📦 Containerd Setup {#containerd-setup}

```
# Docker repository & GPG
sudo apt-get update && sudo apt-get install -y ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu \
 $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install containerd
sudo apt-get update && sudo apt-get install -y containerd.io

# ========================================
# CONFIGURE SYSTEMD CGROUP (PRODUCTION)
# ========================================
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml

# Enable systemd cgroup driver
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```

<details>
<summary>🔍 Verify Containerd</summary>

```
sudo systemctl status containerd
sudo ctr version
```
</details>

---

## 🧰 Kubernetes Components {#kubernetes-components}

```
# NEW Kubernetes repo (pkgs.k8s.io)
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | \
sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | \
sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

---

## 🌐 Master Node {#master-node}

```
# 🚀 Initialize Control Plane
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --control-plane-endpoint=<MASTER_IP> \
  --upload-certs

# Setup kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

echo "✅ **Copy the JOIN COMMAND above for worker nodes!**"
```

---

## 🌉 Calico CNI {#calico-cni}

```
# Install Tigera Operator
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml

# Apply Custom Resources
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml
kubectl create -f custom-resources.yaml

# Wait for readiness
watch kubectl get pods -n calico-system
```

---

## 👥 Worker Nodes {#worker-nodes}

```
# Use join command from master output
sudo kubeadm join <MASTER_IP>:6443 --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

### 🔄 Regenerate Join Command
```
sudo kubeadm token create --print-join-command
```

---

## ✅ Verification {#verification}

```
kubectl get nodes -o wide
kubectl get pods -A

# Test deployment
kubectl run nginx --image=nginx --restart=Never
kubectl expose pod nginx --port=80 --type=NodePort

# Check NodePort
kubectl get svc nginx
```

---


## 🧯 Troubleshooting {#troubleshooting}

If you encounter any issues during the setup process (such as network failures, join errors, or kubelet startup problems),  
you can safely reset the node to a clean state by running the following commands:

🔄 Reset all Kubernetes configurations on this node
```
sudo kubeadm reset -f
```

🧹 Remove leftover configuration and network files
```
sudo rm -rf /etc/cni/net.d /var/lib/cni/ /etc/kubernetes/ $HOME/.kube
```

🌐 Clean up network rules to avoid routing conflicts
```
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X
sudo ipvsadm -C || true
```

🔁 Restart containerd to ensure a healthy runtime
```
sudo systemctl restart containerd
```

> ✅ After running these commands, the node will be completely clean and ready to rejoin the cluster using  
> `kubeadm init` (for Control Plane) or `kubeadm join` (for Worker Nodes).
>
> ⚠️ **Note:** If you are using a custom CNI or network overlay (like Calico or Cilium),  
> make sure to remove its old pods or network policies before reapplying new configurations.

---

## 🛡️ Hardening {#hardening}

### 🔒 Firewall Rules
```
sudo ufw allow 6443/tcp    # API Server
sudo ufw allow 10250/tcp   # Kubelet  
sudo ufw allow 2379:2380/tcp # etcd
sudo ufw allow 10251/tcp   # Scheduler
sudo ufw allow 10252/tcp   # Controller
```

### 📊 Metrics Server
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### 💾 Backup Script
```
tar czf k8s-backup-$(date +%Y%m%d).tar.gz \
  /etc/kubernetes/pki \
  /etc/kubernetes/admin.conf \
  /var/lib/etcd
```
