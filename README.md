# Install Kubernetes Cluster with Kubeadm on Ubuntu 22.04

Set up the system hostname and the /etc/hosts file on all the servers:

`sudo hostnamectl set-hostname cplane1`, same for worker nodes but with different name such as `worker1` and `worker2`

Modify the /etc/hosts with `sudo nano /etc/hosts` so that each hostname is pointed to the corrected IP address:
```
192.168.5.10 cplane1
192.168.5.25 worker1
192.168.5.26 worker2`
````

## Enable Kernel Modules and Disable SWAP

Enable the kernel modules "overlay" and "br_netfilter".
```
sudo modprobe overlay
sudo modprobe br_netfilter
```
To make it permanent:
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```
Create the systemctl params required:
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```
To apply the changes without reboot:
`sudo sysctl --system`

To disable SWAP:
`sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab`

Turn off the SWAP on the current session:
```
sudo swapoff -a
free -m
```
SWAP should now have the value 0.

## Install Container runtime (Master and Worker nodes). (Docker will be used in this tutorial)

Add repo and Install packages:
```
sudo apt update
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update
sudo apt install -y containerd.io docker-ce docker-ce-cli
```

Create required directories:

`sudo mkdir -p /etc/systemd/system/docker.service.d`

Create daemon json config file:

```
sudo tee /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

Start and enable Services:

```
sudo systemctl daemon-reload 
sudo systemctl restart docker
sudo systemctl enable docker
```
Install Mirantis cri-dockerd as Docker Engine shim for Kubernetes:

Get latest version:
```
VER=$(curl -s https://api.github.com/repos/Mirantis/cri-dockerd/releases/latest|grep tag_name | cut -d '"' -f 4|sed 's/v//g')
echo $VER
```
For Intel 64-bit CPU:
```
wget https://github.com/Mirantis/cri-dockerd/releases/download/v${VER}/cri-dockerd-${VER}.amd64.tgz
tar xvf cri-dockerd-${VER}.amd64.tgz
```

Move cri-dockerd binary package to /usr/local/bin directory:

`sudo mv cri-dockerd/cri-dockerd /usr/local/bin/`

Validate installation:

`cri-dockerd --version`

Configure systemd units for cri-dockerd:
```
wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service
wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket
sudo mv cri-docker.socket cri-docker.service /etc/systemd/system/
sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
```

Start and enable the services:
```
sudo systemctl daemon-reload
sudo systemctl enable cri-docker.service
sudo systemctl enable --now cri-docker.socket
```

## Installing Kubernetes Package

`sudo apt install apt-transport-https ca-certificates curl -y`

Add the Kubernetes repository and GPG key:
```
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
Update and refresh Ubuntu repository and package index:

`sudo apt update`

Install kubelet, kubeadm, kubectl:

`sudo apt install kubelet kubeadm kubectl`

`sudo apt-mark hold kubelet kubeadm kubectl`

## Initializing Kubernetes Control Plane

`sudo kubeadm init --apiserver-advertise-address=192.168.56.15 --pod-network-cidr=192.168.0.0/16 --cri-socket unix:///run/cri-dockerd.sock`

Setup Kubernetes credentials:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Install Calico:

`curl https://raw.githubusercontent.com/projectcalico/calico/v3.24.5/manifests/calico.yaml -O`

`kubectl apply -f calico.yaml`


## Adding Worker Nodes to Kubernetes
After initializing the Kubernetes Control Plane on the "cplane1" server, you will add worker nodes "worker1" and "worker2" to the Kubernetes Cluster.
Copy the join command in the worker node:

```
sudo kubeadm join 192.168.56.15:6443 --token po3sb1.oux4z76nwb0veuna \
--discovery-token-ca-cert-hash sha256:f5068150fabaf85f3d04e19a395c60d19298ba441e2d9391e20df3267ea6cd28 --cri-socket unix:///run/cri-dockerd.sock 
```

If you forget the join command, run this command in the master node and paste the output in the worker nodes:

`kubeadm token create --print-join-command`

Run `kubectl get nodes` in the master node to visualize the nodes.










