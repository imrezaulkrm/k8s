
# Kubernetes Cluster Setup Guide (Kubeadm, Containerd, Flannel)

This guide walks you through setting up a Kubernetes cluster (v1.32) using `kubeadm`, `containerd`, and the **Flannel** CNI plugin.

---

## Common Steps (Run on **Both Master & Worker Nodes**)

### 1. Disable Swap

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```
### 2. Load Required Kernel Modules
```bash
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```
### 3. Set Required sysctl Params
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
```
### 4. Install and Configure Containerd
```bash
sudo apt update
sudo apt install -y containerd

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```
### 5. Install Kubernetes Tools (kubelet, kubeadm, kubectl)
```bash
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl gpg

sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/kubernetes-archive-keyring.gpg

echo "deb [signed-by=/etc/apt/trusted.gpg.d/kubernetes-archive-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
## Master Node Only
### 6. Initialize the Cluster
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
>##### Copy and save the `kubeadm join` command printed at the end!
### 7. Set up `kubeconfig` for kubectl
```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
### 8. Install Flannel CNI

    kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

### 9. (Optional) Allow Master to Schedule Pods

    kubectl taint nodes --all node-role.kubernetes.io/control-plane-

## Worker Node Only
**After completing steps 1–5, join the cluster using the join command you copied earlier:**

    sudo kubeadm join <MASTER-IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>

**Example**

    sudo kubeadm join 192.168.0.10:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash
    sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
## *Verify the Cluster (From Master Node)*

    kubectl get nodes

**Expected Output:**

    NAME     STATUS   ROLES           AGE     VERSION
    master   Ready    control-plane   XXm     v1.32.x
	worker   Ready    <none>          XXm     v1.32.x

---

Just copy the content above and paste it into a new file. Save it as `README.md`, and you'll be good to go!
