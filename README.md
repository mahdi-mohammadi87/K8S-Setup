# Kubernetes Cluster Setup on VMware Workstation

> **⚠️ DISCLAIMER: Educational Use Only**

> This guide is intended for learning, testing, and development purposes in a lab environment. The configurations provided (e.g., disabling firewalls, using simple passwords, local storage) are **not suitable for production environments**.

## 🏗 Architecture Overview

* **Hypervisor:** VMware Workstation (NAT Network: VMnet8)
* **OS:** Ubuntu Server 20.04/22.04 LTS
* **Container Runtime:** containerd
* **Orchestrator:** Kubernetes (Kubeadm)
* **CNI (Network):** Calico
* **Storage:** Rancher Local Path Provisioner
* **Nodes:**
  &#x20;   \* `master` - `192.168.100.20`
  &#x20;   \* `worker1` - `192.168.100.21`
  &#x20;   \* `worker2` - `192.168.100.22`

***

## 🚀 Part 1: Node Preparation (Run on All Nodes)

Perform the following steps on **Master**, **Worker1**, and **Worker2**.

### 1. Hostname & DNS Setup

Set unique hostnames and update the hosts file to ensure local DNS resolution.

```shellscript
# On Master
sudo hostnamectl set-hostname master
# On Worker1
sudo hostnamectl set-hostname worker1
# On Worker2
sudo hostnamectl set-hostname worker2

# On ALL nodes
cat <<EOF | sudo tee -a /etc/hosts
192.168.100.20 master
192.168.100.21 worker1
192.168.100.22 worker2
EOF
```



### 2. Disable Swap

Kubernetes requires swap to be disabled to function correctly.

```shellscript
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### 3. Kernel Modules & Sysctl Params

Load necessary modules and configure network bridging.

```shellscript
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

### 4. Install Container Runtime (containerd)

```shellscript
# Install containerd
sudo apt-get update
sudo apt-get install -y containerd

# Generate default config
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml

# Enable SystemdCgroup (Critical step for Kubeadm)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# Restart containerd
sudo systemctl restart containerd
```

### 5. Install Kubernetes Tools

Install `kubelet`, `kubeadm`, and `kubectl`.

```shellscript
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# Download Google's public key
curl -fsSL [https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key](https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key) | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] [https://pkgs.k8s.io/core:/stable:/v1.29/deb/](https://pkgs.k8s.io/core:/stable:/v1.29/deb/) /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install packages
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

***

## 🎮 Part 2: Initialize Control Plane (Master Only)

Run this only on the **Master** node (`192.168.100.20`).

```shellscript
# Initialize the cluster
sudo kubeadm init --apiserver-advertise-address=192.168.100.20 --pod-network-cidr=192.168.0.0/16

# Configure kubectl for the root user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install Calico Network Plugin
kubectl apply -f [https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml](https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml)
kubectl apply -f [https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml](https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml)
```

> **Important:** At the end of the init command, you will see a `kubeadm join` command. Copy it.

***

## 🔗 Part 3: Join Worker Nodes

Run the join command (copied from the previous step) on **Worker1** and **Worker2**.

```shellscript
sudo kubeadm join 192.168.100.20:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

***

## 💾 Part 4: Configure Storage Class

Since we are on bare-metal/VMware, we need a dynamic provisioner to handle PVCs. We will use **Rancher Local Path Provisioner**.

Run on **Master**:

```shellscript
# Install Local Path Provisioner
kubectl apply -f [https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml](https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml)

# Patch it to be the default StorageClass
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

## ✅ Verification

On Master, run:

```shellscript
kubectl get nodes
```

All nodes should be in `Ready` status.
