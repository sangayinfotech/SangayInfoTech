# Step 1. Install containerd

**To install containerd, follow these steps on both VMs:**
1) Load the br_netfilter module required for networking.

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

2) To allow iptables to see bridged traffic, as required by Kubernetes, we need to set the values of certain fields to 1.
```bash
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

3) Apply the new settings without restarting.
```bash
sudo sysctl –-system
```
		
4) Install curl.
```bash
sudo apt install curl -y
```

5) Get the apt-key and then add the repository from which we will install containerd.
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

6) Update and then install the containerd package.
```bash
sudo apt update -y 
sudo apt install -y containerd.io
```

7) Set up the default configuration file.
```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

```bash
sudo vi /etc/containerd/config.toml
```
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    BinaryName = ""
    CriuImagePath = ""
    CriuPath = ""
    CriuWorkPath = ""
    IoGid = 0
    IoUid = 0
    NoNewKeyring = false
    NoPivotRoot = false
    Root = ""
    ShimCgroup = ""
    SystemdCgroup = true

```bash
sudo systemctl restart containerd
ps -ef | grep containerd
```

-----------------------------------------------------------------------------------------------------------------------------------------------
# Step 2. Install Kubernetes

With our container runtime installed and configured, we are ready to install Kubernetes.
1.Add the repository key and the repository.

```bash
 | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

2.Update your system and install the 3 Kubernetes modules.
```bash
sudo apt update -y
sudo apt install -y kubelet kubeadm kubectl
```

3.Set up the firewall by installing the following rules on the master node:
```bash
sudo ufw allow 6443/tcp
sudo ufw allow 2379/tcp
sudo ufw allow 2380/tcp
sudo ufw allow 10250/tcp
sudo ufw allow 10251/tcp
sudo ufw allow 10252/tcp
sudo ufw allow 10255/tcp
sudo ufw reload
```


4.And these rules on the worker node
```bash
sudo ufw allow 10251/tcp
sudo ufw allow 10255/tcp
sudo ufw reload
```

5.Finally, enable the kubelet service on both systems so we can start it
```bash
sudo systemctl enable kubelet
```

-----------------------------------------------------------------------------------------------------------------------------------------------

# Step 3. Setting up the cluster

1.Run the following command on the master node to allow Kubernetes to fetch the required images before cluster initialization:
```bash
sudo kubeadm config images pull
```

2.Initialize the First Master Node
```bash
sudo kubeadm init \ --control-plane-endpoint "LOAD_BALANCER_IP:6443" \ --upload-certs \ --pod-network-cidr=10.244.0.0/16 \ --cri-socket unix:///run/containerd/containerd.sock --v=5
```

3.Install Calico network plugin:
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.0/manifests/calico.yaml
```

4.Install Ingress-NGINX Controller:
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.49.0/deploy/static/provider/baremetal/deploy.yaml
```
	
5.Get the join command and certificate key from the first master node:
```bash
kubeadm token create --print-join-command --certificate-key $(kubeadm init phase upload-certs --upload-certs | tail -1)
```

Step 6: Join the other Master Node
```bash
sudo kubeadm init \
  --control-plane-endpoint "172.30.30.13:6443" \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16 \
  --cri-socket unix:///run/containerd/containerd.sock
```

6.All the master nodes
```bash	
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```	

7.Join the Worker Nodes
Get the join command from the first master node:
```bash
kubeadm token create --print-join-command
```

8.Run the join command as a root user on each worker node:
```bash
sudo kubeadm join LOAD_BALANCER_IP:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --cri-socket unix:///run/containerd/containerd.sock
```

9.Check the status of all nodes:
```bash
kubectl get nodes
```
