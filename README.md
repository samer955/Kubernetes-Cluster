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


