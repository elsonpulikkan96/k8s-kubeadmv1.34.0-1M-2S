# kubernetes_cluster_kubeadm

**Introduction**

This guide explains how to set up a three-node Kubernetes v1.33 (latest) cluster using kubeadm. Kubernetes is a container orchestration platform, and Kubeadm is a tool used to create and manage Kubernetes clusters.

**What is kubeadm?**

Kubeadm is a tool used to create Kubernetes clusters.

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
sudo hostnamectl set-hostname k8s-control-plane
```

```sh
# node-1
sudo hostnamectl set-hostname k8s-worker-node-1
```

```sh
# node-2
sudo hostnamectl set-hostname k8s-worker-node-2
```
Update the hosts file on the control plane, node-1 and node-2 to enable communication via hostnames


```sh
# control-plane, node-1 and node-2
sudo tee /etc/hosts <<'EOF'
10.0.175.226 k8s-control-plane
10.0.245.172 k8s-worker-node-1
10.0.84.109  k8s-worker-node-2
EOF
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
Download the public signing key for the Kubernetes V1.34 package repository on the control plane, node-1 and node-2

```sh
# control-plane, node-1 and node-2
****
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key \
 | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
Add the appropriate Kubernetes apt repository on the control plane, node-1 and node-2
```sh
# control-plane, node-1 and node-2
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' \
 | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
Install kubeadm, kubelet and kubectl tools and hold their package version on the control plane, node-1 and node-2

```sh
# control-plane, node-1 and node-2

sudo apt update
sudo apt install -y kubeadm=1.34.0-1.1 kubelet=1.34.0-1.1 kubectl=1.34.0-1.1
sudo apt-mark hold kubeadm kubelet kubectl
kubeadm version
kubectl version --client
kubelet --version
```
Initialise Control Plane
```sh
# control-plane
Initialize Control Plane (on k8s-control-plane)
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
root@k8s-control-plane:~# kubectl get nodes -o wide
NAME                STATUS   ROLES           AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION    CONTAINER-RUNTIME
k8s-control-plane   Ready    control-plane   41m   v1.34.1   10.0.175.226   <none>        Ubuntu 24.04.3 LTS   6.14.0-1011-aws   containerd://1.7.28
worker-node-1       Ready    <none>          33m   v1.34.0   10.0.245.172   <none>        Ubuntu 24.04.3 LTS   6.14.0-1011-aws   containerd://1.7.28
worker-node-2       Ready    <none>          29m   v1.34.0   10.0.84.109    <none>        Ubuntu 24.04.3 LTS   6.14.0-1011-aws   containerd://1.7.28



```
Install the Calico network addon to the cluster and verify the status of the nodes

```sh
# control-plane
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.0/manifests/calico.yaml
kubectl -n kube-system get pods -o wide | grep calico

```

```sh

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
Verify our cluster and all the nodes and pods will be in a Ready and Running state on your control plane instance.

```sh
root@k8s-control-plane:~# kubectl -n kube-system get pods
NAME                                        READY   STATUS    RESTARTS   AGE
calico-kube-controllers-5766bdd7c-7l22g     1/1     Running   0          42m
calico-node-7w99x                           1/1     Running   0          42m
calico-node-nvlcd                           1/1     Running   0          32m
calico-node-qng68                           1/1     Running   0          35m
coredns-66bc5c9577-cfztq                    1/1     Running   0          43m
coredns-66bc5c9577-q7b8c                    1/1     Running   0          43m
etcd-k8s-control-plane                      1/1     Running   0          44m
kube-apiserver-k8s-control-plane            1/1     Running   0          44m
kube-controller-manager-k8s-control-plane   1/1     Running   0          44m
kube-proxy-jpcvp                            1/1     Running   0          32m
kube-proxy-rz7pm                            1/1     Running   0          35m
kube-proxy-s44zn                            1/1     Running   0          44m
kube-scheduler-k8s-control-plane            1/1     Running   0          44m
```

For testing an Application Deployment, We can deploy a Wordpress pod, expose it as ClusterIP and verify its status.

```sh
# control-plane

root@k8s-control-plane:~# kubectl run nginx --image=nginx --port=80 --expose
service/nginx created
pod/nginx created


root@k8s-control-plane:~# kubectl get pods nginx -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP              NODE            NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          6s    10.244.212.65   worker-node-1   <none>           <none>

root@k8s-control-plane:~# kubectl get svc nginx
NAME    TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
nginx   ClusterIP   10.109.8.74   <none>        80/TCP    57

```
Access the Nginx default page using port forwarding in the control plane


```sh
# control-plane

root@k8s-control-plane:~# kubectl port-forward svc/nginx 8080:80 >/dev/null 2>&1 & sleep 1
[1] 33490

root@k8s-control-plane:~# curl -I http://localhost:8080
HTTP/1.1 200 OK
Server: nginx/1.29.2
Date: Tue, 28 Oct 2025 00:05:57 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 07 Oct 2025 17:04:07 GMT
Connection: keep-alive
ETag: "68e54807-267"
Accept-Ranges: bytes

* Reference
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

