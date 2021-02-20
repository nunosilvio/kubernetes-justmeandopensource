# Install Kubernetes Cluster using kubeadm
Follow this documentation to set up a Kubernetes cluster on __Ubuntu 20.04 LTS__.

This documentation guides you in setting up a cluster with one master node and one worker node.

## Assumptions
|Role|FQDN|IP|OS|RAM|CPU|
|----|----|----|----|----|----|
|Master|kmaster10.nunosilvio.pt|10.10.10.10|Ubuntu 20.04|2G|2|
|Node|knode11.nunosilvio.pt|10.10.10.11|Ubuntu 20.04|2G|2|
|Master|kmaster20.nunosilvio.pt|10.10.10.20|Ubuntu 20.04|1G|1|
|Node|kmaster21.nunosilvio.pt|10.10.10.21|Ubuntu 20.04|1G|1|

## Define proxy
```
export http_proxy=http://10.10.10.1:3128/
export https_proxy=http://10.10.10.1:3128/
export no_proxy="10.96.0.0/12,192.168.0.0/16,10.10.10.0/24"
```
## On both Kmaster and Kworker
Perform all the commands as root user unless otherwise specified
##### Disable Firewall
```
ufw disable
```
##### Disable swap
```
swapoff -a; sed -i '/swap/d' /etc/fstab
```
##### Update sysctl settings for Kubernetes networking
```
cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```
##### Install docker engine
```
{
  apt install -y apt-transport-https ca-certificates curl gnupg-agent software-properties-common
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
  add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
  apt update
  apt install -y docker-ce=5:19.03.10~3-0~ubuntu-focal containerd.io
}
```
### Kubernetes Setup
##### Add Apt repository
```
{
  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
  echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
}
```
##### Install Kubernetes components
```
apt update && apt install -y kubeadm=1.18.5-00 kubelet=1.18.5-00 kubectl=1.18.5-00
```
##### In case you are using LXC containers for Kubernetes nodes
Hack required to provision K8s v1.15+ in LXC containers
```
{
  mknod /dev/kmsg c 1 11
  echo '#!/bin/sh -e' >> /etc/rc.local
  echo 'mknod /dev/kmsg c 1 11' >> /etc/rc.local
  chmod +x /etc/rc.local
}
```

## On kmaster
### Change docker proxy settings
```
mkdir -p /etc/systemd/system/docker.service.d

cat >>/etc/systemd/system/docker.service.d/http-proxy.conf<<EOF
[Service]
Environment="HTTP_PROXY=http://10.10.10.1:3128"
Environment="HTTPS_PROXY=http://10.10.10.1:3128"
Environment="NO_PROXY=localhost,127.0.0.1,10.96.0.0/12,192.168.0.0/16,10.10.10.0/24"
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```
### Download images using docker
```
kubeadm config images list
kubeadm config images pull

Example:
docker pull k8s.gcr.io/kube-apiserver:v1.18.16
docker pull k8s.gcr.io/kube-controller-manager:v1.18.16
docker pull k8s.gcr.io/kube-scheduler:v1.18.16
docker pull k8s.gcr.io/kube-proxy:v1.18.16
docker pull k8s.gcr.io/pause:3.2
docker pull k8s.gcr.io/etcd:3.4.3-0
docker pull k8s.gcr.io/coredns:1.6.7
```

##### Initialize Kubernetes Cluster
Update the below command with the ip address of kmaster
```
kubeadm init --apiserver-advertise-address=10.10.10.10 --pod-network-cidr=192.168.0.0/16  --ignore-preflight-errors=all
kubeadm init --apiserver-advertise-address=10.10.10.20 --pod-network-cidr=192.168.0.0/16  --ignore-preflight-errors=all
```
##### Deploy Calico network
```
kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://docs.projectcalico.org/v3.14/manifests/calico.yaml

or

curl -o ./calico.yaml -k https://docs.projectcalico.org/v3.14/manifests/calico.yaml
kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f ./calico.yaml
```

##### Cluster join command
```
kubeadm token create --print-join-command
```

##### To be able to run kubectl commands as non-root user
If you want to be able to run kubectl commands as non-root user, then as a non-root user perform these
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## On Kworker
##### Join the cluster
Use the output from __kubeadm token create__ command in previous step from the master server and run here.

## Verifying the cluster
##### Get Nodes status
```
kubectl get nodes
```
##### Get component status
```
kubectl get cs
```

Have Fun!!
