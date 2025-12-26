# Kubernetes Practice Lab (kubeadm) — Ubuntu 22.04


- OS: Ubuntu Server 22.04
- Container runtime: containerd
- Kubernetes install method: kubeadm
- CNI: Calico
- Pod CIDR: `10.10.0.0/16`

---

## Prerequisites

- Root or passwordless sudo on all nodes
- Nodes can reach each other over the network
- (Recommended) DNS records for hostnames  
  For lab simplicity, we use `/etc/hosts`.

---

## Variables (Edit if needed)

Set these at the top of your shell session on **each node**:

```bash
export K8S_MINOR="1.26"            # choose a tested minor version for your environment
export POD_CIDR="10.10.0.0/16"      # must match your CNI config expectation
export K8S_REPO="https://pkgs.k8s.io/core:/stable:/v${K8S_MINOR}/deb/"
export K8S_KEYRING="/etc/apt/keyrings/kubernetes-apt-keyring.gpg"
export K8S_LIST="/etc/apt/sources.list.d/kubernetes.list"
````

> If you change `K8S_MINOR`, keep kubeadm/kubelet/kubectl **consistent** across all nodes.

---

## Step 1 — Base OS preparation (ALL NODES)

### 1.1 Install required packages

```bash
sudo apt-get update -y
sudo apt-get install -y --no-install-recommends \
  curl ca-certificates gnupg lsb-release apt-transport-https
```

### 1.2 Set hostname (run the correct one per node)

```bash
# Control-plane
sudo hostnamectl set-hostname k8s-master

# Worker 1
# sudo hostnamectl set-hostname k8s-worker1

# Worker 2
# sudo hostnamectl set-hostname k8s-worker2
```

### 1.3 Add `/etc/hosts` entries (lab-only)

This is idempotent: it removes old matching lines and re-adds the desired mapping.

```bash
sudo cp -a /etc/hosts /etc/hosts.bak.$(date +%F_%H%M%S)

sudo sed -i '/k8s-master/d;/k8s-worker1/d;/k8s-worker2/d' /etc/hosts
cat <<'EOF' | sudo tee -a /etc/hosts >/dev/null
192.168.100.20 k8s-master
192.168.100.21 k8s-worker1
192.168.100.22 k8s-worker2
EOF
```

### 1.4 Disable swap (required by kubelet)

```bash
sudo swapoff -a || true
# Comment out any active swap entries in /etc/fstab (idempotent)
sudo sed -i.bak '/\sswap\s/s/^\(.*\)$/#\1/g' /etc/fstab
```

Verify:

```bash
free -h | sed -n '1,3p'
```

### 1.5 Kernel modules and sysctl for Kubernetes

```bash
cat <<'EOF' | sudo tee /etc/modules-load.d/k8s.conf >/dev/null
overlay
br_netfilter
EOF

sudo modprobe overlay || true
sudo modprobe br_netfilter || true

cat <<'EOF' | sudo tee /etc/sysctl.d/99-kubernetes.conf >/dev/null
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system >/dev/null
```

---

## Step 2 — Install and configure containerd (ALL NODES)

### 2.1 Install containerd

```bash
sudo apt-get update -y
sudo apt-get install -y containerd
```

### 2.2 Generate containerd config and enable systemd cgroups

```bash
sudo mkdir -p /etc/containerd

# Only generate default config if missing
if [ ! -f /etc/containerd/config.toml ]; then
  sudo containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
fi

# Ensure SystemdCgroup = true (idempotent)
sudo sed -i 's/^\(\s*SystemdCgroup\s*=\s*\)false/\1true/' /etc/containerd/config.toml

sudo systemctl enable --now containerd
sudo systemctl restart containerd
sudo systemctl is-active --quiet containerd && echo "containerd: OK"
```

---

## Step 3 — Install Kubernetes packages (ALL NODES)

### 3.1 Add Kubernetes apt repo key + list

```bash
sudo mkdir -p /etc/apt/keyrings

# Fetch key if missing
if [ ! -f "${K8S_KEYRING}" ]; then
  curl -fsSL "${K8S_REPO}Release.key" | sudo gpg --dearmor -o "${K8S_KEYRING}"
fi

# Write the repo list (overwrite each time to keep it correct)
echo "deb [signed-by=${K8S_KEYRING}] ${K8S_REPO} /" | sudo tee "${K8S_LIST}" >/dev/null

sudo apt-get update -y
```

### 3.2 Install kubelet/kubeadm/kubectl + hold versions

```bash
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl >/dev/null
```

Verify:

```bash
kubeadm version || true
kubelet --version || true
kubectl version --client || true
```

---

## Step 4 — Initialize the control-plane (CONTROL-PLANE ONLY)

Run on `k8s-master`:

### 4.1 Initialize cluster

`kubeadm init` is not meant to be “re-run forever”. We gate it by checking if admin.conf exists.

```bash
if [ ! -f /etc/kubernetes/admin.conf ]; then
  sudo kubeadm init --pod-network-cidr="${POD_CIDR}"
else
  echo "/etc/kubernetes/admin.conf exists — control-plane already initialized."
fi
```

### 4.2 Configure kubectl for your user

```bash
mkdir -p $HOME/.kube
sudo cp -f /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Sanity check:

```bash
kubectl get nodes -o wide || true
kubectl cluster-info || true
```

---

## Step 5 — Install CNI (Calico) (CONTROL-PLANE ONLY)

Apply Calico manifest:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```

Wait and verify:

```bash
kubectl -n kube-system get pods -l k8s-app=calico-node
kubectl get nodes
```

Nodes should eventually become `Ready`.

---

## Step 6 — Join worker nodes (WORKERS ONLY)

### 6.1 Generate join command (CONTROL-PLANE)

Run on `k8s-master`:

```bash
kubeadm token create --print-join-command
```

Copy the output and run it on each worker with sudo, for example:

```bash
sudo kubeadm join k8s-master:6443 --token <...> --discovery-token-ca-cert-hash sha256:<...>
```

> ✅ Do **not** store tokens in this repo. Generate them when needed.

Verify from master:

```bash
kubectl get nodes -o wide
```

---

## Validation Checklist

Run on `k8s-master`:

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl -n kube-system get pods
```

Quick DNS test:

```bash
kubectl run dns-test --image=busybox:1.36 -it --rm --restart=Never -- sh -c 'nslookup kubernetes.default.svc.cluster.local'
```

---

## Cleanup / Reset (ALL NODES)

### Worker cleanup

```bash
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d
sudo rm -rf /var/lib/cni
sudo rm -rf $HOME/.kube
```

### Control-plane cleanup

```bash
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes /var/lib/etcd
sudo rm -rf /etc/cni/net.d /var/lib/cni
sudo rm -rf $HOME/.kube
```

> ❌ Do not use `--ignore-preflight-errors` to “make it work”.
> Fix the cause. That’s the whole point of a serious lab.

---

## Notes / Common Issues

* Nodes stuck `NotReady` right after init/join:

  * You probably haven’t installed the CNI yet, or CNI pods are failing.
  * Check:

    ```bash
    kubectl -n kube-system get pods
    kubectl -n kube-system describe pod <calico-pod>
    ```

* Kubelet issues:

  ```bash
  sudo journalctl -u kubelet -f
  ```

* containerd issues:

  ```bash
  sudo journalctl -u containerd -f
  ```

