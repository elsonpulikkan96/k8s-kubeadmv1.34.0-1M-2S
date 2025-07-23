# kubernetes_cluster_kubeadm

**Introduction**

This guide explains how to set up a three-node Kubernetes v1.33 (latest) cluster using kubeadm. Kubernetes is a container orchestration platform, and Kubeadm is a tool used to create and manage Kubernetes clusters.

**What is kubeadm?**

kubeadm is a tool used to create Kubernetes clusters.

It automates the creation of Kubernetes clusters by bootstrapping the control plane, joining the nodes, etc. Follows the Kubernetes release cycle and Open-source tool maintained by the Kubernetes community.

**Prerequisites**

Before setting up the Kubernetes cluster, ensure the following prerequisites are met:

* Create three Ubuntu 22.04 LTS instances for the control plane, node-1, and node-2.

* Each instance must have a minimum specification of 2 CPUs and 2 GB RAM.

* Networking must be enabled between instances.

* Required ports must be allowed between instances.

* Swap must be disabled on instances.

**Steps**


```sh
# control-plane-node
sudo hostnamectl set-hostname control-plane
```

```sh
# node-1
sudo hostnamectl set-hostname node-1
```

```sh
# node-2
sudo hostnamectl set-hostname node-2
```
Update the hosts file on the control plane, node-1 and node-2 to enable communication via hostnames


```sh
# control-plane, node-1 and node-2
sudo vi /etc/hosts
172.31.81.34 control-plane
172.31.81.93 node-1
172.31.90.71 node-2
```
Disable swap on the control plane, node-1 and node-2 and if a swap entry is present in the fstab file then comment out the line

```sh
# control-plane, node-1 and node-2
sudo swapoff -a
sudo vi /etc/fstab
  # comment out swap entry
```
To set containerd as our container runtime on the control plane, node-1 and node-2, first, we need to load some Kernel modules and modify system settings

```sh
# control-plane, node-1 and node-2

cat << EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay

sudo modprobe br_netfilter
```

```sh
# control-plane, node-1 and node-2
cat << EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
```
**Package Installation steps**

```sh
# control-plane, node-1 and node-2

sudo apt-get update && sudo apt-get upgrade -y

sudo apt install -y containerd
```
Once the packages are installed, generate a default configuration file for containerd on the control plane, node-1 and node-2

```sh
# control-plane, node-1 and node-2
sudo mkdir -p /etc/containerd

sudo containerd config default | sudo tee /etc/containerd/config.toml
```
Change the SystemdCgroup value to true in the containerd configuration file and restart the service

```sh
# control-plane, node-1 and node-2

sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

sudo systemctl restart containerd
```
We need to install some prerequisite packages on the control plane, node-1 and node-2 for configuring the Kubernetes package repository

```sh
# control-plane, node-1 and node-2

sudo apt update

sudo apt install -y apt-transport-https ca-certificates curl gpg

```
Download the public signing key for Kubernetes package repository on the control plane, node-1 and node-2

```sh
# control-plane, node-1 and node-2
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
Add the appropriate Kubernetes apt repository on the control plane, node-1 and node-2
```sh
# control-plane, node-1 and node-2
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
Install kubeadm, kubelet and kubectl tools and hold their package version on the control plane, node-1 and node-2

```sh
# control-plane, node-1 and node-2

sudo apt-get update && sudo apt-get upgrade -y

sudo apt install -y kubeadm=1.33.0-1.1 kubelet=1.33.0-1.1 kubectl=1.33.0-1.1

sudo apt-mark hold kubeadm kubelet kubectl

```

```sh
# control-plane
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --kubernetes-version=1.33.0
```
Once the installation is completed, set up our access to the cluster on the control plane

```sh
# control-plane

mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Verify our cluster status by listing the nodes
But our nodes are in a NotReady state because we haven’t set up networking.
```sh
# control-plane

kubectl get nodes
NAME            STATUS     ROLES           AGE   VERSION
control-plane   NotReady   control-plane   45s   v1.33.0

kubectl get nodes -o wide
NAME            STATUS     ROLES           AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
control-plane   NotReady   control-plane   52s   v1.33.0   172.31.81.34   <none>        Ubuntu 22.04.3 LTS   6.2.0-1012-aws   containerd://1.7.2
```
Install the Calico network addon to the cluster and verify the status of the nodes

```sh
# control-plane

kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.2/manifests/calico.yaml
```

```sh
# control-plane

kubectl -n kube-system get pods
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-7c968b5878-x5trl   1/1     Running   0          46s
calico-node-grrf4                          1/1     Running   0          46s
coredns-76f75df574-cdcj2                   1/1     Running   0          4m19s
coredns-76f75df574-z4gxg                   1/1     Running   0          4m19s
etcd-control-plane                         1/1     Running   0          4m32s
kube-apiserver-control-plane               1/1     Running   0          4m34s
kube-controller-manager-control-plane      1/1     Running   0          4m32s
kube-proxy-78gqq                           1/1     Running   0          4m19s
kube-scheduler-control-plane               1/1     Running   0          4m32s

kubectl get nodes
NAME            STATUS   ROLES           AGE     VERSION
control-plane   Ready    control-plane   4m53s   v1.33.0
```
Once the networking is enabled, join our workload nodes to the cluster
Get the join command from the control plane using kubeadm

```sh
# control-plane

kubeadm token create --print-join-command

```
Once the join command is retrieved from the control plane, execute it in node-1 and node-2


```sh
# node-1 and node-2

sudo kubeadm join 172.31.81.34:6443 --token kvzidi.g65h3s8psp2h3dc6 --discovery-token-ca-cert-hash sha256:56c208595372c1073b47fa47e8de65922812a6ec322d938bd5ac64d8966c1f27

```
Verify our cluster and all the nodes will be in a Ready state on your control plane instance.

```sh
# control-plane

kubectl get nodes
NAME            STATUS   ROLES           AGE     VERSION
control-plane   Ready    control-plane   7m50s   v1.33.0
node-1          Ready    <none>          76s     v1.33.0
node-2          Ready    <none>          79s     v1.33.0

kubectl get nodes -o wide
NAME            STATUS   ROLES           AGE     VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
control-plane   Ready    control-plane   8m12s   v1.33.0   172.31.81.34   <none>        Ubuntu 22.04.3 LTS   6.2.0-1012-aws   containerd://1.7.2
node-1          Ready    <none>          98s     v1.33.0   172.31.81.93   <none>        Ubuntu 22.04.3 LTS   6.2.0-1012-aws   containerd://1.7.2
node-2          Ready    <none>          101s    v1.33.0   172.31.90.71   <none>        Ubuntu 22.04.3 LTS   6.2.0-1012-aws   containerd://1.7.2

```


For testing an Application Deployment, We can deploy a Wordpress pod, expose it as ClusterIP and verify its status.

```sh
# control-plane

kubectl run nginx --image=nginx --port=80 --expose
service/nginx created
pod/nginx created

kubectl get pods nginx -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP              NODE     NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          34s   192.168.247.1   node-2   <none>           <none>

kubectl get svc nginx
NAME    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
nginx   ClusterIP   10.102.86.253   <none>        80/TCP    56s

```
Access the Nginx default page using port forwarding in the control plane


```sh
# control-plane

kubectl port-forward svc/nginx 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80

curl -i http://localhost:8080
HTTP/1.1 200 OK
Server: nginx/1.25.3

```
```sh
```
```sh
```


* Reference
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/



