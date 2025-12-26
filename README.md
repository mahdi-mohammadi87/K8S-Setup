
# Kubernetes Practice Lab (kubeadm) — Ubuntu 22.04

* OS: Ubuntu Server 22.04
* Kubernetes: **v1.35 (stable)**
* Runtime: **containerd (Ubuntu package, systemd cgroups)**
* CNI: Calico
* Pod CIDR: `10.10.0.0/16`

> **Name resolution note:**
> `/etc/hosts` is intentionally used.
> In real environments, use proper DNS.

---

## Variables

```bash
export POD_CIDR="10.10.0.0/16"
export K8S_REPO="https://pkgs.k8s.io/core:/stable:/v1.35/deb/"
export K8S_KEYRING="/etc/apt/keyrings/kubernetes-apt-keyring.gpg"
export K8S_LIST="/etc/apt/sources.list.d/kubernetes.list"
```

---

## Step 1 — OS preparation (ALL NODES)

### 1.1 Base packages

```bash
sudo apt-get update -y
sudo apt-get install -y --no-install-recommends \
  curl ca-certificates gnupg lsb-release apt-transport-https
```

---

### 1.2 Hostnames (run **only** the correct one per node)

```bash
# control-plane
sudo hostnamectl set-hostname k8s-master

# worker1
# sudo hostnamectl set-hostname k8s-worker1

# worker2
# sudo hostnamectl set-hostname k8s-worker2
```

---

### 1.3 `/etc/hosts` (MANDATORY — ALL NODES)

```bash
sudo sed -i '/k8s-master.devops.local/d' /etc/hosts
sudo sed -i '/k8s-worker/d' /etc/hosts

cat <<'EOF' | sudo tee -a /etc/hosts
192.168.100.20 k8s-master.devops.local k8s-master
192.168.100.21 k8s-worker1.devops.local k8s-worker1
192.168.100.22 k8s-worker2.devops.local k8s-worker2
EOF
```

#### Hard validation (WORKERS ONLY — DO NOT SKIP)

```bash
getent hosts k8s-master.devops.local
curl -k https://k8s-master.devops.local:6443/healthz
nc -vz k8s-master.devops.local 6443
```

❌ If any of these fail → **STOP**
`kubeadm join` will otherwise hang silently.

---

### 1.4 Disable swap (required)

```bash
sudo swapoff -a || true
sudo sed -i.bak '/\sswap\s/s/^\(.*\)$/#\1/g' /etc/fstab
```

Verify:

```bash
free -h | sed -n '1,3p'
```

---

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

> Some kernels log harmless sysctl warnings.
> These are acceptable for this lab.

---

## Step 2 — containerd runtime (ALL NODES)

### 2.1 Enforce runtime consistency

This lab **does NOT support mixing runtimes**.

```bash
if dpkg -l | grep -q containerd.io; then
  echo "ERROR: containerd.io detected. Remove it and use Ubuntu containerd."
  exit 1
fi
```

---

### 2.2 Install containerd

```bash
sudo apt-get update -y
sudo apt-get install -y containerd
```

---

### 2.3 Configure systemd cgroups (idempotent)

```bash
sudo mkdir -p /etc/containerd

sudo containerd config default \
  | sudo tee /etc/containerd/config.toml >/dev/null

sudo sed -i \
  's/^\(\s*SystemdCgroup\s*=\s*\)false/\1true/' \
  /etc/containerd/config.toml

sudo systemctl daemon-reload
sudo systemctl enable --now containerd
sudo systemctl restart containerd
sudo systemctl is-active --quiet containerd && echo "containerd: OK"
```

---

## Step 3 — Kubernetes packages (ALL NODES)

### 3.1 Add pkgs.k8s.io repository (fail fast)

```bash
[ -n "${K8S_REPO}" ] && [ -n "${K8S_KEYRING}" ] && [ -n "${K8S_LIST}" ] \
  || { echo "ERROR: required variables not set"; exit 1; }

curl -fsSLI "${K8S_REPO}Release.key" >/dev/null \
  || { echo "ERROR: cannot reach Kubernetes repo"; exit 1; }

sudo mkdir -p /etc/apt/keyrings
sudo rm -f "${K8S_LIST}" "${K8S_KEYRING}"

curl -fsSL "${K8S_REPO}Release.key" \
  | sudo gpg --dearmor -o "${K8S_KEYRING}"

echo "deb [signed-by=${K8S_KEYRING}] ${K8S_REPO} /" \
  | sudo tee "${K8S_LIST}" >/dev/null

sudo apt-get update -y
```

---

### 3.2 Install kubelet / kubeadm / kubectl

```bash
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl >/dev/null
```

Verify:

```bash
kubeadm version
kubelet --version
kubectl version --client
```

---

## Step 4 — Initialize control-plane (CONTROL-PLANE ONLY)

### 4.1 kubeadm init (guarded)

```bash
if [ ! -f /etc/kubernetes/admin.conf ]; then
  sudo kubeadm init \
    --pod-network-cidr="${POD_CIDR}"
else
  echo "Control-plane already initialized."
fi
```

---

### 4.2 kubectl access

```bash
mkdir -p $HOME/.kube
sudo cp -f /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Sanity:

```bash
kubectl cluster-info
kubectl get nodes -o wide
```

---

## Step 5 — Calico CNI (CONTROL-PLANE)

### 5.1 Create CRDs (once)

```bash
if kubectl get crd bgpconfigurations.crd.projectcalico.org >/dev/null 2>&1; then
  echo "Calico CRDs already exist — skipping."
else
  curl -fsSL https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml \
  | awk 'BEGIN{RS="---"} /kind:[[:space:]]*CustomResourceDefinition/ {print "---" $0}' \
  | kubectl create -f -
fi
```

---

### 5.2 Apply Calico components (safe to re-run)

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```

---

### 5.3 Verify

```bash
kubectl -n kube-system get pods -l k8s-app=calico-node
kubectl get nodes
```

---

## Step 6 — Join workers (WORKERS ONLY)

### 6.1 Generate join command (CONTROL-PLANE)

```bash
kubeadm token create --print-join-command
```

---

### 6.2 Join each worker

```bash
sudo kubeadm join k8s-master.devops.local:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

> kubelet may fail **before** join — this is expected.
> It stabilizes automatically once join completes.

Verify from master:

```bash
kubectl get nodes -o wide
```

---

## Validation Checklist

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

DNS test:

```bash
kubectl run dns-test --image=busybox:1.36 -it --rm --restart=Never -- \
  sh -c 'nslookup kubernetes.default.svc.cluster.local'
```

---

## Reset / Cleanup

### Workers

```bash
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes /var/lib/kubelet
sudo rm -rf /etc/cni/net.d /var/lib/cni
```

### Control-plane

```bash
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes /var/lib/etcd
sudo rm -rf /etc/cni/net.d /var/lib/cni
```

> ❌ Never use `--ignore-preflight-errors` to “make it work”.
> Fix the root cause — that’s the point of this lab.

---

## Troubleshooting

### kubeadm join hangs

* Check DNS resolution
* Check TCP/6443 reachability
* Ensure containerd is running

### kubelet crash loop (`config.yaml missing`)

* Join did not complete
* Reset node and retry join

### containerd failures

```bash
sudo journalctl -u containerd -f
```
