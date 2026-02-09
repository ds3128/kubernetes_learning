# Kubernetes Cluster Installation --- Step-by-Step Guide

This guide provides a structured and simplified process to install a
Kubernetes cluster (control plane + worker nodes) using `kubeadm`.
Versions are managed automatically via variables.

------------------------------------------------------------------------

## 📦 Version Management (Edit once here)
``` bash
export K8S_VERSION="1.35.0-1.1"
export K8S_MAJOR="v1.35.0"
export CONTAINERD_VERSION="2.2.1"
export RUNC_VERSION="1.4.0"
export CNI_VERSION="1.9.0"
export CALICO_VERSION="3.31.3"
export POD_CIDR="10.10.0.0/16"
```

``` bash
# Kubernetes version (dernière stable GA)
K8S_VERSION="1.35.0-1.1"
K8S_MAJOR="v1.35.0"

# Runtime / dependencies
CONTAINERD_VERSION="2.2.1"
RUNC_VERSION="1.4.0"
CNI_VERSION="1.9.0"
CALICO_VERSION="3.31.3"

# Network
POD_CIDR="10.10.0.0/16" (tunnel that we will use to create communication between pod network
and host network to have external access)
MASTER_NAME="master"
WORKER1_NAME="worker1"
WORKER2_NAME="worker2"
```

------------------------------------------------------------------------

## 🖥️ Step 1 --- Prepare Hosts (All Nodes)

Define hostnames/IPs so machines resolve each other.

``` bash
printf "\n192.168.1.94 master\n192.168.1.96 worker1\n192.168.1.97 worker2\n" >> /etc/hosts
```

**Explanation:**\
Adds static name resolution (no DNS required).
``` bash
IP_NAT_MASTER = "192.168.174.128"
IP_NAT_WORKER_1 = "192.168.174.131"
IP_NAT_WORKER_2 = "192.168.174.132"
```

------------------------------------------------------------------------

## ⚙️ Step 2 --- Kernel Modules & Networking (All Nodes)

Enable required modules for container networking.

``` bash
printf "overlay\nbr_netfilter\n" >> /etc/modules-load.d/containerd.conf
modprobe overlay
modprobe br_netfilter
```

**Explanation:**\
- `overlay` → filesystem driver for containers\
- `br_netfilter` → allows iptables filtering on bridges

Configure sysctl:

``` bash
printf "net.bridge.bridge-nf-call-iptables = 1\nnet.ipv4.ip_forward = 1\nnet.bridge.bridge-nf-call-ip6tables = 1\n" >> /etc/sysctl.d/99-kubernetes-cri.conf
sysctl --system
```

**Explanation:**\
Applies networking parameters required by Kubernetes routing.

------------------------------------------------------------------------

## 📦 Step 3 --- Install Container Runtime (containerd)

Download and install:

``` bash
wget https://github.com/containerd/containerd/releases/download/v${CONTAINERD_VERSION}/containerd-${CONTAINERD_VERSION}-linux-amd64.tar.gz -P /tmp/

tar Cxzvf /usr/local /tmp/containerd-${CONTAINERD_VERSION}-linux-amd64.tar.gz

wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service -P /etc/systemd/system/

systemctl daemon-reload

systemctl enable --now containerd
```

Install `runc`:

``` bash
wget https://github.com/opencontainers/runc/releases/download/v${RUNC_VERSION}/runc.amd64 -P /tmp/

install -m 755 /tmp/runc.amd64 /usr/local/sbin/runc
```

Install CNI plugins:

``` bash
wget https://github.com/containernetworking/plugins/releases/download/v${CNI_VERSION}/cni-plugins-linux-amd64-v${CNI_VERSION}.tgz -P /tmp/

mkdir -p /opt/cni/bin

tar Cxzvf /opt/cni/bin /tmp/cni-plugins-linux-amd64-v${CNI_VERSION}.tgz
```

Configure containerd:

``` bash
mkdir -p /etc/containerd

containerd config default | tee /etc/containerd/config.toml
```

Edit config:

    SystemdCgroup = true

Restart:

``` bash
systemctl restart containerd
```

------------------------------------------------------------------------

## 📴 Step 4 --- Disable Swap (All Nodes)

``` bash
swapoff -a
```

Also remove swap entry from `nano /etc/fstab`.

**Explanation:** Kubernetes scheduler requires swap disabled.

------------------------------------------------------------------------

## ☸️ Step 5 --- Install Kubernetes Tools

``` bash
apt-get update && apt-get install -y apt-transport-https ca-certificates curl gpg
mkdir -p -m 755 /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/${K8S_MAJOR}/deb/Release.key  | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/${K8S_MAJOR}/deb/ /"  | tee /etc/apt/sources.list.d/kubernetes.list

apt-get update
apt-get install -y kubelet=${K8S_VERSION} kubeadm=${K8S_VERSION} kubectl=${K8S_VERSION}
apt-mark hold kubelet kubeadm kubectl
```

Check swap is off:

``` bash
free -m
```

------------------------------------------------------------------------

## 🎛️ Step 6 --- Initialize Control Plane (Master Only)

``` bash
kubeadm init --pod-network-cidr ${POD_CIDR} --kubernetes-version ${K8S_VERSION} --node-name ${MASTER_NAME}
export KUBECONFIG=/etc/kubernetes/admin.conf
```

**Explanation:**\
Bootstraps cluster and generates credentials.

------------------------------------------------------------------------

## 🌐 Step 7 --- Install Calico Network Plugin

``` bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v${CALICO_VERSION}/manifests/tigera-operator.yaml
wget https://raw.githubusercontent.com/projectcalico/calico/v${CALICO_VERSION}/manifests/custom-resources.yaml
```

Edit CIDR inside `custom-resources.yaml` to match `${POD_CIDR}`:

``` bash
kubectl apply -f custom-resources.yaml
```

Verify:

``` bash
kubectl get pods -n calico-system
kubectl get nodes
```

------------------------------------------------------------------------

## 🔗 Step 8 --- Join Worker Nodes

Run on master:

``` bash
kubeadm token create --print-join-command
```

Run output command on each worker.

------------------------------------------------------------------------

## 🏷️ Step 9 --- Label Workers

``` bash
kubectl label node worker1 node-role.kubernetes.io/worker=worker
kubectl label node worker2 node-role.kubernetes.io/worker=worker
```

------------------------------------------------------------------------

# ✅ Cluster Ready

You now have: - Container runtime configured - Kubernetes installed -
Network plugin running - Workers joined

------------------------------------------------------------------------

# ☸️ Check if all pods are done 
``` bash
kubectl get pods -o wide --all-namespaces
```

# ☸️ If cali-node for each node are 0/1, use those comand

``` bash
sudo ufw allow from <worker-ip> to any port 6443 comment 'K8s API from worker'
sudo ufw allow 6443/tcp from x.x.x.x 
sudo ufw allow 6443/tcp
sudo ufw allow 179/tcp
sudo ufw allow 5473/tcp
sudo ufw allow 2379:2380/tcp  
sudo ufw allow 10250/tcp  
sudo ufw allow 10259/tcp  
sudo ufw allow 10257/tcp  
sudo ufw reload
ufw status

kubectl -n calico-system set env daemonset/calico-node IP_AUTODETECTION_METHOD=interface=en.*|eth.*

kubectl -n calico-system delete pod -l k8s-app=calico-node

kubectl get pods -n calico-system -o wide | grep calico-node
```

``` bash
namespaces will be use to regroup resources together
kubectl get namespace
kubectl get all --namespace kube-system
```

``` bash
kubectl api-resources | grep pod
```

``` bash
kubectl explain pod | more
kubectl explain pod.spec | more
kubectl explain pod.spec.containers | more
kubectl explain --recursive | more
```

# Describe an resources

``` bash
kubectl describe nodes worker1 | more
kubectl describe nodes master | more
kubectl --help
```

# How to active an autocompletion

``` bash
sudo apt-get install -y bash-completion
echo "source <(kubectl completion bash)" >> ~/.bashrc
source ~/.bashrc
kubectl g[tab][tab] po[tab][tab] --all[tab][tab]
```

# How to deploy app in kubernetes 

``` bash
Imperative declaration :

kubectl create deployment nginx --image=nginx

kubectl run nginx --image=nginx

kubectl get/describe/delete deployment nginx

Declarative approach (using manifest file to define the wish state): 

kubectl apply -f test_manifest.yaml
```

# Appliquer le manifeste
kubectl apply -f deployment.yaml

# Voir les pods créés
kubectl get pods

# Vérifier le déploiement
kubectl get deployment test-deployment

# Supprimer
kubectl delete -f deployment.yaml

# General command 
kubectl create deployment NAME --image=image -- [COMMAND] [args...] [options]

# use this command to get the app open on worker node
sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps


kubectl logs nginx-app-574b8c4d9c-tncc8

kubectl exec -it nginx-app-574b8c4d9c-tncc8 -- /bin/sh

kubectl describe replicaset nginx-app-574b8c4d9c-tncc8