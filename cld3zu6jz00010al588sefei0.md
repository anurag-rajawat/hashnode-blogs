---
title: "Kubernetes DaemonSet (Part - 5)"
seoTitle: "Introduction to Kubernetes daemonset"
seoDescription: "Kubernetes Daemonset ensures that a single copy of your pod runs on each of your nodes."
datePublished: Fri Jan 20 2023 04:00:09 GMT+0000 (Coordinated Universal Time)
cuid: cld3zu6jz00010al588sefei0
slug: kubernetes-daemonset-part-5
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1672071232038/cc710341-a4f4-4aaf-91b7-aa8442bd6761.png
tags: kubernetes

---

### Introduction

After learning about Deployment, Replicaset, etc in the [previous tutorial](https://anuragr.hashnode.dev/kubernetes-pod-replicaset-deployment). In this tutorial, we will learn about [Kubernetes DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/), what are these and why we need them, and will do hands-on with them.

Let's say you want to run a single copy of your pod on each of your nodes, what will you do to achieve it? And what if you want that when a new node is added to your cluster that pod automatically deployed to that node?

Probably you will create a deployment with replicas equal to the number of your nodes and the replicaset will create pods, but what will you do when a new node is added to your cluster, and can you make sure that each of your nodes runs only a single copy of the pod because it may be possible that one or more than one pods got deployed on a single node. After all, many factors are taken into account when the scheduling happened by the scheduler.

Now you're thinking ok fine but why do we need to run only a single copy of the pod on each node?

Well, what if you want to collect logs from each of your nodes or monitor them, do you need two or three pods to do so, absolutely not. So, It depends on the use case.

That's where the **DaemonSet** comes in. A DaemonSet ensures that one pod gets deployed per node (or on a subset of nodes) and as the nodes are added to the cluster that pod gets automatically deployed to that node and as the nodes are removed the pod also gets removed from that node.

Typical uses:

* A node monitoring daemon on every node
    
* Logs collection daemon on every node
    
* Cluster storage daemon
    

Enough talking let's create a manifest file `daemonset.yaml` for logging

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-ds
spec:
  selector:
    matchLabels:
      name: fluentd-ds
  template:
    metadata:
      labels:
        name: fluentd-ds
    spec:
      containers:
       - name: fluentd
         image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
```

```bash
$ k get no
NAME                         STATUS   ROLES           AGE   VERSION
k8s-learning-control-plane   Ready    control-plane   57s   v1.25.3
k8s-learning-worker          Ready    <none>          31s   v1.25.3
k8s-learning-worker2         Ready    <none>          32s   v1.25.3

$ k create -f daemonset.yaml
daemonset.apps/fluentd-ds created
```

As you can see we've successfully created it.

Let's take a look at the pods and where they get deployed

```bash
$ k get po -o wide
NAME               READY   STATUS    RESTARTS   AGE   IP           NODE                   NOMINATED NODE   READINESS GATES
fluentd-ds-992ss   1/1     Running   0          92s   10.244.1.2   k8s-learning-worker2   <none>           <none>
fluentd-ds-wfrv2   1/1     Running   0          92s   10.244.2.2   k8s-learning-worker    <none>           <none>
```

As you can see one pod per node is running

```bash
$ k get ds fluentd-ds
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
fluentd-ds   2         2         2       2            2           <none>          3m5s
```

From above it's confirmed that 2 pods are currently running.

Let's get the details of `fluent-ds` daemonset

```bash
$ k describe ds fluentd-ds
Selector:       name=fluentd-ds
Node-Selector:  <none>
Labels:         <none>
Annotations:    deprecated.daemonset.template.generation: 1
Desired Number of Nodes Scheduled: 2
Current Number of Nodes Scheduled: 2
Number of Nodes Scheduled with Up-to-date Pods: 2
Number of Nodes Scheduled with Available Pods: 2
Number of Nodes Misscheduled: 0
Pods Status:  2 Running / 0 Waiting / 0 Succeeded / 0 Failed
...
...
...
Events:
  Type    Reason            Age    From                  Message
  ----    ------            ----   ----                  -------
  Normal  SuccessfulCreate  4m54s  daemonset-controller  Created pod: fluentd-ds-wfrv2
  Normal  SuccessfulCreate  4m54s  daemonset-controller  Created pod: fluentd-ds-992ss
```

One noticeable thing to note here is,  
`Desired Number of Nodes Scheduled: 2` and `Current Number of Nodes Scheduled: 2` and you can also see pod status and events.

***NOTE*:** *Since Kubernetes 1.6, DaemonSets do not schedule on master nodes by default. To schedule it on master, you have to add toleration into the Pod spec section:*

```yaml
...
tolerations:
      # these tolerations are to have the daemonset runnable on control plane nodes
      # remove them if your control plane nodes should not run pods
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
...
```

complete manifest file `daemonset-all-nodes.yaml` after adding toleration looks like 👇

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-ds
spec:
  selector:
    matchLabels:
      name: fluentd-ds
  template:
    metadata:
      labels:
        name: fluentd-ds
    spec:
      containers:
       - name: fluentd
         image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
      tolerations:
      # these tolerations are to have the daemonset runnable on control plane nodes
      # remove them if your control plane nodes should not run pods
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
```

Let's quickly delete the old daemonset and create this new one

```bash
$ k delete -f daemonset.yaml
daemonset.apps "fluentd-ds" deleted

$ k create -f daemonset-all-nodes.yaml
daemonset.apps/fluentd-ds-all-nodes created
```

and validate it

```bash
$ k get po -o wide
NAME                         READY   STATUS    RESTARTS   AGE   IP           NODE                         NOMINATED NODE   READINESS GATES
fluentd-ds-all-nodes-gscs6   1/1     Running   0          32s   10.244.1.9   k8s-learning-worker2         <none>           <none>
fluentd-ds-all-nodes-n6tpz   1/1     Running   0          32s   10.244.0.7   k8s-learning-control-plane   <none>           <none>
fluentd-ds-all-nodes-zjgxt   1/1     Running   0          32s   10.244.2.9   k8s-learning-worker          <none>           <none>
```

As you can see above fluentd has been successfully running on all (3) of our nodes.

### Cleanup

* Delete daemonset
    

```bash
$ k delete -f daemonset-all-nodes.yaml
daemonset.apps "fluentd-ds-all-nodes" deleted
```

* Delete cluster (Optional)
    

```bash
$ kind delete cluster --name=k8s-multi-node-cluster
Deleting cluster "k8s-multi-node-cluster" ...
```

That's all for [Kubernetes DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/), I'll see you at the next one. This is the fifth part of our [Kubernetes Fundamental series](https://anuragr.hashnode.dev/series/kubernetes-fundamentals).

Thank you so much for taking your valuable time to read.