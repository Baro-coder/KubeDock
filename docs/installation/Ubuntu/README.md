# Ubuntu Kubernetes Cluster

## Ubuntu-based Kubernetes cluster developed in virtual lab (VirtualBox)

---

## **Lab Setup**

Four **Ubuntu Server 22 LTS** systems with following details:

- **Master Node** : 0 : *ubuntu-kube-master* : 192.168.60.40
- **Worker Node** : 1 : *ubuntu-kube-worker-1* : 192.168.60.51
- **Worker Node** : 2 : *ubuntu-kube-worker-2* : 192.168.60.52
- **Worker Node** : 3 : *ubuntu-kube-worker-3* : 192.168.60.53

There is set up administrative account at every system: `kadmin`.

---

## **System preparation**

All the preparation operation must be run as root at all nodes.

### **Networking**

Run on all nodes:

- NAT
- Host-Only (vboxnet)

#### **NAT and internal static IP**

*Interfaces:*

|   interface   |  adapter  |     network     |   address    |
|---------------|-----------|-----------------|--------------|
|enp0s3         |    Bridged    |      192.168.0.0/24       |     192.168.0.`x`     |

*Configuration:*

Edit file `/etc/netplan/00-installer-config.yaml`:

``` console
sudo vim /etc/netplan/00-installer-config.yaml
```

Example for master node:

``` yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: no
      addresses: [192.168.0.40/24]
      routes:
        - to: default
          via: 192.168.0.1
      nameservers:
        addresses: [192.168.0.1]
```

Apply changes:

``` console
sudo netplan apply
```

*Hosts table:*

Edit file `/etc/hosts`:

``` console
sudo vim /etc/hosts
```

Lines to add:

``` text
# Cluster Nodes
192.168.0.40  ubuntu-kube-master
192.168.0.51  ubuntu-kube-worker-1
192.168.0.52  ubuntu-kube-worker-2
192.168.0.53  ubuntu-kube-worker-3
```

### **SWAP**

Disable swap and prevent system from mounting swap:

``` console
sudo swapoff --all && sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

---

## **Install Docker Engine**

Run on all nodes:

Remove all the conflicting packages:

``` console
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg -y; done
```

Add Docker's GPG key:

``` console
sudo install -m 0755 -d /etc/apt/keyrings &&
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg &&
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

Set up the repository:

``` console
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Update indexes:

``` console
sudo apt update
```

Install Docker Engine, containerd and Docker Compose

``` console
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

Verify if the Docker Engine was installed properly by runnning the `hello-world` image:

``` console
sudo docker run hello-world
```

---

## **Install CRI Docker**

### [GUIDE](https://github.com/Mirantis/cri-dockerd)

Install GO:

``` console
sudo apt install golang -y
```

Get `cri-docker` source code:

``` console
git clone https://github.com/Mirantis/cri-dockerd.git
```

Build:

``` console
cd cri-dockerd &&
mkdir bin &&
sudo go build -o bin/cri-dockerd
```

Install and enable `cri-docker` systemd service unit:

``` console
sudo mkdir -p /usr/local/bin &&
sudo install -o root -g root -m 0755 bin/cri-dockerd /usr/local/bin/cri-dockerd &&
sudo cp -a packaging/systemd/* /etc/systemd/system &&
sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service &&
sudo systemctl daemon-reload &&
sudo systemctl enable cri-docker.service &&
sudo systemctl enable --now cri-docker.socket &&
cd .. &&
rm -fr cri-dockerd
```

---

## **Kubernetes Tools**

### **Add Repository**

Install required tools (on all nodes):

``` console
sudo apt-get install -y ca-certificates curl
```

Add K8S key (on all nodes):

``` console
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
```

Add K8S repository (on all nodes):

``` console
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Update node (on all nodes):

``` console
sudo apt update
```

---

### **Install Kubernetes tools**

Install K8S components (on all nodes):

``` console
sudo apt-get install -y kubelet kubeadm  kubectl &&
sudo apt-mark hold kubelet kubeadm kubectl
```

---

## **Kubernetes Cluster**

Execute on master node:

``` console
sudo kubeadm init --control-plane-endpoint=192.168.0.40 --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.0.40 --cri-socket unix:///var/run/cri-dockerd.sock
```

*Output:*

``` text
our Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join 192.168.0.40:6443 --token ht9fqt.am7jhy11q24tsod2 \
 --discovery-token-ca-cert-hash sha256:8698a3a02659b95a44525e2fee762b8a15e3710b837b6cc276877fd6325a33b4 \
 --control-plane 

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.0.40:6443 --token ht9fqt.am7jhy11q24tsod2 \
 --discovery-token-ca-cert-hash sha256:8698a3a02659b95a44525e2fee762b8a15e3710b837b6cc276877fd6325a33b4
```

According to command output run these commands to enable cluster control from regular user level:

``` console
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Or run this command if you are root user:

``` console
export KUBECONFIG=/etc/kubernetes/admin.conf
```

To add worker nodes to cluster run command this command on worker node (parameters' values got from master `kubeadm init` output):

``` console
sudo kubeadm join 192.168.0.40:6443 --token ht9fqt.am7jhy11q24tsod2  --discovery-token-ca-cert-hash sha256:8698a3a02659b95a44525e2fee762b8a15e3710b837b6cc276877fd6325a33b4 --cri-socket unix:///var/run/cri-dockerd.sock
```

If you want to generate join command to add other nodes, at first generate new token:

``` console
sudo kubeadm token generate
```

Then produce join command:

``` console
sudo kubeadm token create <TOKEN_GENERATE_OUTPUT> --print-join-command
```

---

## **Network Plugin**

[Install Calico guide](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises)

**Manifest:**

Download the Calico networking manifest for the Kubernetes API datastore:

``` console
curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml -O
```

*Note:*

  If you are using pod CIDR 192.168.0.0/16, skip to the next step. If you are using a different pod CIDR with kubeadm, no changes are required - Calico will automatically detect the CIDR based on the running configuration. For other platforms, make sure you uncomment the CALICO_IPV4POOL_CIDR variable in the manifest and set it to the same value as your chosen pod CIDR.

[Customize](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/config-options) the manifest as necessary.

Apply the manifest:

``` console
kubectl apply -f calico.yaml
```

---

## **Debug Command Pallete**

Get cluster info:

``` console
kubectl cluster-info
```

Get nodes info:

``` console
kubectl get nodes -A
```

Get pods info:

``` console
kubectl get pods -A
```

Check components status:

``` console
kubectl get cs
```

Check all:

``` console
clear && kubectl cluster-info && echo; echo; kubectl get nodes; echo && kubectl get pods -A && echo; echo; kubectl get cs
```

Check listen IPv4 listen ports list:

``` console
sudo netstat -lpnt4 | tr -s ' ' | cut -d ' ' -f4 | cut -d ':' -f2 | sort -h | tail -n +3
```

---

## **Example**

> To develop...
