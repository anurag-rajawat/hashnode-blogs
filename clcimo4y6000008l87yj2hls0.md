---
title: "Kubernetes Setup (Part - 2)"
seoTitle: "Kubernetes local setup"
seoDescription: "Setup a Kubernetes cluster using Minikube, Kind or kubeadm"
datePublished: Thu Jan 05 2023 05:08:22 GMT+0000 (Coordinated Universal Time)
cuid: clcimo4y6000008l87yj2hls0
slug: kubernetes-setup-part-2
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1672304982542/67aa9576-0aa5-44e0-871e-ed3baf17aa81.png
tags: kubernetes, kubernetes-setup

---

### Introduction

After learning about Kubernetes in the [previous article](https://anuragr.hashnode.dev/introduction-to-kubernetes), in this tutorial, we'll set up a Kubernetes cluster on our workstations to learn and play with it.

Fortunately, there are many ways that you can try out to run Kubernetes locally, and they are all open-source.

* [Minikube](https://minikube.sigs.k8s.io/docs/): It is the well-known and easiest way to set up Kubernetes locally. Single node cluster with all control plane components and worker node on a single node.
    
* [Kind](https://kind.sigs.k8s.io): Kind stands for *Kubernetes in docker.* It was originally designed for testing Kubernetes but may also be used for local development or continuous integration. It uses docker containers to create cluster(s). It also supports ingress/LB and we can use it to set up a highly available multi node-cluster for our development and learning purposes.
    
* [Kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/): Kubeadm is a tool to set up a production-like cluster locally on a workstation for development and testing purposes with best practices for creating Kubernetes clusters.
    
* [MicroK8s](https://microk8s.io): MicroK8s is a CNCF-certified upstream Kubernetes deployment that runs entirely on your workstation or edge device, from Canonical.
    

I'll not cover all of the above but some of them.

## Installation and Setup

*These instructions are from the official websites.*

### Prerequisites

* 2 CPUs or more
    
* 2GB of free memory
    
* 20GB of free disk space
    
* Container or a virtual machine manager, such as [Docker](https://minikube.sigs.k8s.io/docs/drivers/docker/), [QEMU](https://minikube.sigs.k8s.io/docs/drivers/qemu/), [Hyperkit](https://minikube.sigs.k8s.io/docs/drivers/hyperkit/), [Hyper-V](https://minikube.sigs.k8s.io/docs/drivers/hyperv/), [KVM](https://minikube.sigs.k8s.io/docs/drivers/kvm2/), [Parallels](https://minikube.sigs.k8s.io/docs/drivers/parallels/), [Podman](https://minikube.sigs.k8s.io/docs/drivers/podman/), [VirtualBox](https://minikube.sigs.k8s.io/docs/drivers/virtualbox/), or [VMware Fusion/Workstation](https://minikube.sigs.k8s.io/docs/drivers/vmware/)
    
* Install Kubectl from [here](https://kubernetes.io/docs/tasks/tools/#kubectl) to interact with your cluster
    

## 1\. Minikube

1. Download the minikube binary for your OS from [here](https://minikube.sigs.k8s.io/docs/start/)
    
2. Verify that you've installed minikube by opening a command prompt and typing the following command:
    
    ```bash
    $ minikube version
    minikube version: v1.28.0
    commit: 986b1ebd987211ed16f8cc10aed7d2c42fc8392f
    ```
    
3. Start your cluster, it will bootstrap a single-node cluster.
    
    ```bash
    $ minikube start
    😄  minikube v1.28.0 on Darwin 13.0.1 (arm64)
    ✨  Using the docker driver based on existing profile
    👍  Starting control plane node minikube in cluster minikube
    🚜  Pulling base image ...
    🔄  Restarting existing docker container for "minikube" ...
    🐳  Preparing Kubernetes v1.25.3 on Docker 20.10.20 ...
    🔎  Verifying Kubernetes components...
        ▪ Using image docker.io/kubernetesui/dashboard:v2.7.0
        ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
        ▪ Using image k8s.gcr.io/metrics-server/metrics-server:v0.6.1
        ▪ Using image docker.io/kubernetesui/metrics-scraper:v1.0.8
    💡  Some dashboard features require the metrics-server addon. To enable all features please run:
    
    	minikube addons enable metrics-server
    
    🌟  Enabled addons: storage-provisioner, default-storageclass, metrics-server, dashboard
    🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
    ```
    
4. Verify the status of your cluster and node
    
    ```bash
    $ minikube status
    minikube
    type: Control Plane
    host: Running
    kubelet: Running
    apiserver: Running
    kubeconfig: Configured
    ```
    
    ```bash
    $ kubectl get nodes
    NAME       STATUS   ROLES           AGE   VERSION
    minikube   Ready    control-plane   20d   v1.25.3
    ```
    
5. Stop your cluster
    
    ```bash
    $ minikube stop
    ✋  Stopping node "minikube"  ...
    🛑  Powering off "minikube" via SSH ...
    🛑  1 node stopped.
    ```
    
6. Delete cluster (Optional)
    

```bash
$ minikube delete
🔥  Deleting "minikube" in docker ...
🔥  Deleting container "minikube" ...
🔥  Removing /Users/anurag/.minikube/machines/minikube ...
💀  Removed all traces of the "minikube" cluster.
```

## 2\. Kind

* Install Kind
    

### macOS

```bash
$ brew install kind
```

or

```bash
$ # for Intel Macs
[ $(uname -m) = x86_64 ]&& curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.17.0/kind-darwin-amd64

# for M1 / ARM Macs
$ [ $(uname -m) = arm64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.17.0/kind-darwin-arm64

$ chmod +x ./kind
$ mv ./kind /some-dir-in-your-PATH/kind
```

**Linux**

```bash
$ curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.17.0/kind-linux-amd64
$ chmod +x ./kind
$ sudo mv ./kind /usr/local/bin/kind
```

Create a cluster

* Single node cluster
    

```bash
$ kind create cluster --name=k8s-single-node-cluster
Creating cluster "k8s-single-node-cluster" ...
✓ Ensuring node image (kindest/node:v1.25.3) 🖼
✓ Preparing nodes 📦
✓ Writing configuration 📜
✓ Starting control-plane 🕹️
✓ Installing CNI 🔌
✓ Installing StorageClass 💾
Set kubectl context to "kind-k8s-single-node-cluster"
You can now use your cluster with:
  
kubectl cluster-info --context kind-k8s-single-node-cluster

Not sure what to do next? 😅  Check out https://kind.sigs.k8s.io/docs/user/quick-start/
```

* Multi-node cluster
    

Create a config file for your multi-node cluster, you can modify it as per your requirements.

```yaml
# four nodes (three workers) cluster config
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
nodes:
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
```

```bash
  $ kind create cluster --name=k8s-multi-node-cluster --config=multi-nodes-cluster.yaml
  Creating cluster "k8s-multi-node-cluster" ...
 ✓ Ensuring node image (kindest/node:v1.25.3) 🖼
 ✓ Preparing nodes 📦 📦 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-k8s-multi-node-cluster"
You can now use your cluster with:

kubectl cluster-info --context kind-k8s-multi-node-cluster

Have a question, bug, or feature request? Let us know! https://kind.sigs.k8s.io/#community 🙂
```

* Verify the status of your cluster and node
    

```bash
$ kubectl cluster-info
Kubernetes control plane is running at https://127.0.0.1:49493
CoreDNS is running at https://127.0.0.1:49493/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

$ kubectl get nodes
NAME                                   STATUS   ROLES           AGE     VERSION
k8s-multi-node-cluster-control-plane   Ready    control-plane   2m42s   v1.25.3
k8s-multi-node-cluster-worker          Ready    <none>          2m10s   v1.25.3
k8s-multi-node-cluster-worker2         Ready    <none>          2m23s   v1.25.3
k8s-multi-node-cluster-worker3         Ready    <none>          2m23s   v1.25.3
```

As we can see above we've successfully created our multi-node cluster

* Delete cluster (Optional)
    

```bash
$ kind delete cluster --name=k8s-multi-node-cluster
Deleting cluster "k8s-multi-node-cluster" ...
```

## 3\. Kubeadm

We're going to set up three nodes (one control plane node and two worker nodes) cluster on Ubuntu 22.04 machines which can be virtual machines on your local workstation or on any cloud provider.

### Prerequisites

* 2 GB or more of RAM per machine (any less will leave little room for your apps).
    
* 2 CPUs / vCPUs or more.
    
* Full network connectivity between all machines in the cluster (public or private network is fine).
    
* 20 GB free disk space
    
* Root privileges
    

**Follow the below steps to install the Kubernetes cluster properly using kubeadm:**

### On all nodes

* Login as root
    

```bash
sudo su
```

* Disable swap
    

```bash
swapoff -a; sed -i '/swap/d' /etc/fstab
```

* Load the br\_netfilter and overlay kernel module
    

```bash
tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter
```

* Update the sysctl settings for Kubernetes networking
    

```bash
tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sysctl --system
```

* Install containerd runtime
    

```bash
{
apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
apt update
apt install -y containerd.io
}
```

* Configure containerd to use systemd as groups
    

```bash
{
containerd config default | tee /etc/containerd/config.toml >/dev/null 2>&1
sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
systemctl restart containerd
systemctl enable containerd
}
```

* Install kubeadm, kubelet, kubectl
    

```bash
{
curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | tee /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
}
```

### Only on the control plane

* Initialize Kubernetes Cluster
    

```bash
kubeadm init --control-plane-endpoint=192.168.64.31
```

where `--control-plane-endpoint` will be the public IP of the control plane instance

you'll get the output of the above command something like👇

```bash
[init] Using Kubernetes version: v1.26.0
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [control-plane kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.64.31]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [control-plane localhost] and IPs [192.168.64.31 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [control-plane localhost] and IPs [192.168.64.31 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 10.502422 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node control-plane as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node control-plane as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: 1fulgr.nkuxjlz76xyb5bnk
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

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

  kubeadm join 192.168.64.31:6443 --token 1fulgr.nkuxjlz76xyb5bnk \
        --discovery-token-ca-cert-hash sha256:1fb94932236b76c2f65c54a46de83ab56536637a288df1a2e1d8ab3b901e111a \
        --control-plane

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.64.31:6443 --token 1fulgr.nkuxjlz76xyb5bnk \
        --discovery-token-ca-cert-hash sha256:1fb94932236b76c2f65c54a46de83ab56536637a288df1a2e1d8ab3b901e111a
```

Note down the cluster join command

* Export KUBECONFIG as a normal user
    

```bash
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

* Install the Calico pod network
    

```bash
$ kubectl apply -f https://projectcalico.docs.tigera.io/manifests/calico.yaml
```

### On all worker nodes

* Paste the cluster join command copied before. If you lose it you can get it by using the below command
    

```bash
kubeadm token create --print-join-command
```

Join cluster as the root user

```bash
kubeadm join 192.168.64.31:6443 --token 1fulgr.nkuxjlz76xyb5bnk \
        --discovery-token-ca-cert-hash sha256:1fb94932236b76c2f65c54a46de83ab56536637a288df1a2e1d8ab3b901e111a

[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

* Let's check the cluster status
    

```bash
$ kubectl cluster-info
Kubernetes control plane is running at https://192.168.64.31:6443
CoreDNS is running at https://192.168.64.31:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

kubectl can communicate with our cluster.

Check the nodes

```bash
$ kubectl get nodes
NAME            STATUS   ROLES           AGE   VERSION
control-plane   Ready    control-plane   28m   v1.26.0
worker-1        Ready    <none>          17m   v1.26.0
worker-2        Ready    <none>          12m   v1.26.0
```

As you can see above worker nodes have joined the cluster and the cluster is ready to run workloads.

## Shell setup

We'll set up auto-suggestion and alias so that we'll have to type less and increase our productivity.

Let's create an alias, **k** for **kubectl**

```plaintext
alias k=kubectl
```

Add the above line to your `.zshrc` or `.bashrc` file depending on your shell

**Autocompletion**

Add the below line to your `.zshrc` or `.bashr` file

* For zsh shell
    
    ```plaintext
    source <(kubectl completion zsh)
    ```
    
* For bash shell
    
    ```plaintext
    source <(kubectl completion bash)
    complete -o default -F __start_kubectl k
    ```
    

After reloading your shell, kubectl auto-completion should be working.

If you want colorful kubectl output like 👇

![kubectl colored output](https://cdn.hashnode.com/res/hashnode/image/upload/v1671270346574/wFq-oQbOw.png align="center")

* **macOS**
    

```bash
$ brew install kubecolor/tap/kubecolor
```

After installing kubecolor add the following in your `.zshrc` file and reload your shell you should get kubectl colored output

```plaintext
# get zsh complete kubectl
source <(kubectl completion zsh)
alias k=kubecolor
# make completion work with kubecolor
compdef kubecolor=kubectl
```

* **Linux**
    

```bash
$ sudo apt install -y kubecolor
```

If the above command doesn't work then you can manually download kubecolor from its github [release page](https://github.com/hidetatz/kubecolor/releases)

after installing add the following to your `.bashrc` file and reload your shell, you should get kubectl colored output.

```plaintext
alias k=kubecolor
# make completion work with kubecolor
compdef kubecolor=kubectl
```

That's it for the installation and setup of Kubernetes. This is the second part of the [Kubernetes Fundamental series](https://anuragr.hashnode.dev/series/kubernetes-fundamentals). I'll see you in the next tutorial.

Thank you so much for taking your valuable time to read.