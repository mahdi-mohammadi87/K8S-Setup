
# Kubernetes Practice Lab (kubeadm) — Ubuntu 22.04

- OS: Ubuntu Server 22.04
- Kubernetes: **v1.35 stable** (kubeadm/kubelet/kubectl)
- Runtime: containerd (systemd cgroups)
- CNI: Calico
- Pod CIDR: `10.10.0.0/16`

> Lab note: `/etc/hosts` is used for name resolution. In real environments, use DNS.

---

## Variables

Edit if needed:

```bash
export POD_CIDR="10.10.0.0/16"
export K8S_REPO="https://pkgs.k8s.io/core:/stable:/v1.35/deb/"
export K8S_KEYRING="/etc/apt/keyrings/kubernetes-apt-keyring.gpg"
export K8S_LIST="/etc/apt/sources.list.d/kubernetes.list"
````

---

## Step 1 — OS prep (ALL NODES)

### 1.1 Install base packages

```bash
sudo apt-get update -y
sudo apt-get install -y --no-install-recommends \
  curl ca-certificates gnupg lsb-release apt-transport-https
```

### 1.2 Set hostname (run the correct one per node)

```bash
# control-plane
sudo hostnamectl set-hostname k8s-master

# worker1
# sudo hostnamectl set-hostname k8s-worker1

# worker2
# sudo hostnamectl set-hostname k8s-worker2
```

### 1.3 `/etc/hosts` (lab-only)

```bash
sudo cp -a /etc/hosts /etc/hosts.bak.$(date +%F_%H%M%S)
sudo sed -i '/k8s-master/d;/k8s-worker1/d;/k8s-worker2/d' /etc/hosts

cat <<'EOF' | sudo tee -a /etc/hosts >/dev/null
192.168.100.20 k8s-master
192.168.100.21 k8s-worker1
192.168.100.22 k8s-worker2
EOF
```

### 1.4 Disable swap (required)

```bash
sudo swapoff -a || true
sudo sed -i.bak '/\sswap\s/s/^\(.*\)$/#\1/g' /etc/fstab
```

Verify:

```bash
free -h | sed -n '1,3p'
```

### 1.5 Kernel modules + sysctl

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

## Step 2 — containerd (ALL NODES)

### 2.1 Install

```bash
sudo apt-get update -y
sudo apt-get install -y containerd
```

### 2.2 Configure systemd cgroups

```bash
sudo mkdir -p /etc/containerd

if [ ! -f /etc/containerd/config.toml ]; then
  sudo containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
fi

sudo sed -i 's/^\(\s*SystemdCgroup\s*=\s*\)false/\1true/' /etc/containerd/config.toml

sudo systemctl enable --now containerd
sudo systemctl restart containerd
sudo systemctl is-active --quiet containerd && echo "containerd: OK"
```

---

## Step 3 — Kubernetes packages (ALL NODES)

### 3.1 Add pkgs.k8s.io repo + key

```bash
# Fail fast if variables are missing
[ -n "${K8S_REPO}" ] && [ -n "${K8S_KEYRING}" ] && [ -n "${K8S_LIST}" ] || { echo "ERROR: vars not set"; exit 1; }

# Ensure the key URL exists (avoid silent garbage)
curl -fsSLI "${K8S_REPO}Release.key" >/dev/null || { echo "ERROR: Cannot reach ${K8S_REPO}Release.key"; exit 1; }

sudo mkdir -p /etc/apt/keyrings
sudo rm -f "${K8S_LIST}" "${K8S_KEYRING}"

curl -fsSL "${K8S_REPO}Release.key" | sudo gpg --dearmor -o "${K8S_KEYRING}"

echo "deb [signed-by=${K8S_KEYRING}] ${K8S_REPO} /" \
| sudo tee "${K8S_LIST}" >/dev/null

sudo apt-get update -y
```

### 3.2 Install kubelet/kubeadm/kubectl and hold

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

## Step 4 — Initialize control-plane (CONTROL-PLANE ONLY)

Run on `k8s-master`.

### 4.1 kubeadm init (gated)

`kubeadm init` is not meant to be run repeatedly. We gate it by checking admin.conf.

```bash
if [ ! -f /etc/kubernetes/admin.conf ]; then
  sudo kubeadm init --pod-network-cidr="${POD_CIDR}"
else
  echo "Control-plane already initialized (/etc/kubernetes/admin.conf exists)."
fi
```

### 4.2 Configure kubectl for your user

```bash
mkdir -p $HOME/.kube
sudo cp -f /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Sanity:

```bash
kubectl cluster-info || true
kubectl get nodes -o wide || true
```

---

## Step 5 — Install CNI (Calico) (CONTROL-PLANE ONLY)

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```

Wait/verify:

```bash
kubectl -n kube-system get pods -l k8s-app=calico-node
kubectl get nodes
```

Nodes should become `Ready`.

---

## Step 6 — Join workers (WORKERS ONLY)

### 6.1 Generate join command (CONTROL-PLANE)

On `k8s-master`:

```bash
kubeadm token create --print-join-command
```

### 6.2 Run join on each worker

Run the printed command on each worker with sudo:

```bash
sudo kubeadm join k8s-master:6443 --token <...> --discovery-token-ca-cert-hash sha256:<...>
```

Verify from master:

```bash
kubectl get nodes -o wide
```

> Do **not** store join tokens in this repo.

---

## Validation Checklist (CONTROL-PLANE)

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl -n kube-system get pods
```

DNS test:

```bash
kubectl run dns-test --image=busybox:1.36 -it --rm --restart=Never -- \
  sh -c 'nslookup kubernetes.default.svc.cluster.local'
```

---

## Reset / Cleanup (ALL NODES)

### Workers

```bash
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d /var/lib/cni
sudo rm -rf $HOME/.kube
```

### Control-plane

```bash
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes /var/lib/etcd
sudo rm -rf /etc/cni/net.d /var/lib/cni
sudo rm -rf $HOME/.kube
```

> Do **NOT** use `--ignore-preflight-errors` to “make it work”.
> Fix the real cause. That’s the point of a serious lab.

---

## Troubleshooting

### Nodes stuck NotReady

Usually CNI not installed or failing:

```bash
kubectl -n kube-system get pods
kubectl -n kube-system describe pod <calico-pod>
```

### Kubelet logs

```bash
sudo journalctl -u kubelet -f
```

### containerd logs

```bash
sudo journalctl -u containerd -f
```
