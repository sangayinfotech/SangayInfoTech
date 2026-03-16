**1. Install containerd on all nodes**

```bash
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```
```bash
cat <<EOF | sudo tee /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```
```bash
sudo apt update
sudo apt install -y curl ca-certificates gnupg lsb-release apt-transport-https software-properties-common
```

For containerd from Docker’s repo, use a keyring-based setup rather than apt-key:
```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
```bash
sudo apt update
sudo apt install -y containerd.io
```
Generate config:

```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```

Then set:

```bash
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```

You can do that quickly with:

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

Then restart and enable:

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd
```
Kubernetes recommends a compatible container runtime, and kubeadm-based installs commonly use containerd with the systemd cgroup driver.


**2. Install Kubernetes packages on all nodes**
<br>Use the current Kubernetes repo for your target minor version. Example for v1.35:

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list
```
```bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

```bash
sudo systemctl enable kubelet
```
The current kubeadm install docs use pkgs.k8s.io, with one repository per Kubernetes minor version.


**3. Firewall ports**
<br>Control-plane nodes
```bash
sudo ufw allow 6443/tcp
sudo ufw allow 2379:2380/tcp
sudo ufw allow 10250/tcp
sudo ufw allow 10257/tcp
sudo ufw allow 10259/tcp
sudo ufw reload
```

Worker nodes
```bash
sudo ufw allow 10250/tcp
sudo ufw allow 30000:32767/tcp
sudo ufw reload
```
These are the currently documented Kubernetes control-plane and worker ports.


**4. Initialize the first control-plane node**

Pull images first:

```bash
sudo kubeadm config images pull --cri-socket unix:///run/containerd/containerd.sock
```

Then initialize.

If you plan to use Calico with its usual default pool, use:

```bash
sudo kubeadm init \
  --control-plane-endpoint "LOAD_BALANCER_IP:6443" \
  --upload-certs \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket unix:///run/containerd/containerd.sock
```

Set up kubectl for your admin user:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown "$(id -u)":"$(id -g)" $HOME/.kube/config
```
The HA kubeadm workflow uses a stable control-plane endpoint such as a load balancer VIP/DNS name.

**5. Install Calico**
<br>Apply Calico after the first control plane is up.

<br>If you keep 192.168.0.0/16, use a Calico manifest/config that matches that Pod CIDR. Calico documentation notes its default IP pool is 192.168.0.0/16 unless changed.

**6. Join the second control-plane node**

From the first control-plane node, generate the join command:
```bash
kubeadm token create --print-join-command
```
Also get a certificate key:

``` bash
kubeadm init phase upload-certs --upload-certs
```

Then on the second control-plane node, use join, not init:

```bash
sudo kubeadm join LOAD_BALANCER_IP:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <certificate-key> \
  --cri-socket unix:///run/containerd/containerd.sock
```
That is the supported HA control-plane expansion flow.

**7. Join worker nodes**

On the first control-plane node:
```bash
kubeadm token create --print-join-command
```

On each worker:

```bash
sudo kubeadm join LOAD_BALANCER_IP:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --cri-socket unix:///run/containerd/containerd.sock
```

**8. Verify**

```bash
kubectl get nodes -o wide
```





