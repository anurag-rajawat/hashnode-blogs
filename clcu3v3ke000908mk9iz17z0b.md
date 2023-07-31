---
title: "Kubernetes Pod, ReplicaSet & Deployment"
seoTitle: "Hands-on with Kubernetes Pod, Replicaset, and Deployment"
datePublished: Fri Jan 13 2023 05:55:09 GMT+0000 (Coordinated Universal Time)
cuid: clcu3v3ke000908mk9iz17z0b
slug: kubernetes-pod-replicaset-deployment
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1671457508494/3O9r18xS7.png
tags: kubernetes

---

## Introduction

As we learned about the Kubernetes objects in the [previous article](https://anuragr.hashnode.dev/kubernetes-objects), in this tutorial, we'll do hands-on with Pods, ReplicaSets, and Deployments objects, so that we get a better understanding of these objects.

There are several different ways to create and manage Kubernetes objects we'll learn them one by one.

1. [Imperative](https://kubernetes.io/docs/concepts/overview/working-with-objects/object-management/#imperative-commands) **\-** Imperative management means creating Kubernetes objects directly at the command line against a Kubernetes cluster. It is suitable for development projects but not recommended for Production projects.
    
2. [Declarative](https://kubernetes.io/docs/concepts/overview/working-with-objects/object-management/#declarative-object-configuration) - Declarative means we describe the desired state of our cluster in the manifests file without specifying procedures for how that state will be achieved. We can create these manifests either in **YAML** or **JSON** format, since **YAML** is more human-readable we'll write our files in YAML.
    

*We'll do most of the stuff declaratively because it's good practice and we can save them in the version control system.*

## Pod

**Imperative way:**

```bash
$ k run nginx --image nginx:1.22
pod/nginx created
```

**Declarative way:**

Let's create a manifest file for the nginx pod `nginx-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
   - name: nginx
     image: nginx:1.22
```

Let's understand what these attributes mean.

* **apiVersion:** The version of Kubernetes API to create an object.
    
* **kind:** Type of Kubernetes object to be created.
    
* **metadata:** Data that helps uniquely identify the object (name, labels, annotations, and so on)
    
* **spec:** Under this, we define the desired state of the object
    
* **containers:** Under this, we define the container details.
    
* **name:** Unique name for the container in the pod.
    
* **image:** The container image we want to start inside our pod.
    

Now create the pod

```bash
$ k create -f nginx-pod.yaml
pod/nginx created

$ k get pods
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          16s
```

As you can see our pod is successfully created and running.

What if the pod gets deleted due to some reason(s), then what will happen?  
Let's try out:

```bash
$ k delete pod nginx
pod "nginx" deleted

$ k get pods
No resources found in default namespace.
```

Oh no! Our nginx pod has gone completely.

Do you know why?

As we learned in the previous article that **a pod is ephemeral in nature** which means if a pod has gone for any reason it is gone. And we directly created the pod so it will not be replaced by a new one.

Now you're thinking if a pod doesn't get replaced by a new one then how Kubernetes will self-heal?

We need something else which will self-heal and do you know whose job is this?  
The [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) will do that for us. So, now let's take a look at **ReplicaSet.**

## ReplicaSet

The job of ReplicaSet is to make sure that a specified number of pods are always running. Now you're wondering how ReplicaSet knows which pod to manage. ReplicaSet uses a **Label Selector** to identify a set of objects. **Labels** are key-value pairs used to specify attributes of objects that are meaningful and useful to users and the **Selector** matches the object based on the labels.

Let's look at an example:

Create a `nginx-replicaset.yaml` manifest file for our nginx webserver replicaset

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      name: nginx-replicaset
      labels:
        app: nginx
    spec:
      containers:
       - name: nginx
         image: nginx:1.22
```

Let's understand what these attributes mean.

* **spec.replicas:** Desired number of pods for our application.
    
* **spec.selector:** It helps in managing pods by selecting them with the label `app=nginx`.
    
* **spec.template:** Pod template to create new pods.
    

Create the nginx replicaset:

```bash
$ k create -f nginx-replicaset.yaml
replicaset.apps/nginx-replicaset created

$ k get replicasets
NAME               DESIRED   CURRENT   READY   AGE
nginx-replicaset   3         3         3       23s

$ k get pods --show-labels
NAME                     READY   STATUS    RESTARTS   AGE   LABELS
nginx-replicaset-4vhhl   1/1     Running   0          26s   app=nginx
nginx-replicaset-7vhl2   1/1     Running   0          26s   app=nginx
nginx-replicaset-dwtn6   1/1     Running   0          26s   app=nginx
```

As you can see above we've successfully created an nginx replicaset and it has created three pods with label `app=nginx` and all of them are running because we specified that we want three replicas of our pods for our application, so it's doing its job.

Let's try to delete any of these three pods

```bash
$ k delete pod nginx-replicaset-4vhhl
pod "nginx-replicaset-4vhhl" deleted
```

We've deleted the pod, let's check the remaining pods in our cluster

```bash
$ k get pods
NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-replicaset-7vhl2   1/1     Running   0          2m36s
pod/nginx-replicaset-dwtn6   1/1     Running   0          2m36s
pod/nginx-replicaset-xw68w   1/1     Running   0          18s
```

All three pods are running but we've deleted one looks like it didn't get deleted.

Let's try one more time and delete all these three pods so that we'll better understand it:

```bash
$ k delete pods --all
pod "nginx-replicaset-7vhl2" deleted
pod "nginx-replicaset-dwtn6" deleted
pod "nginx-replicaset-xw68w" deleted
```

```bash
$ k get pods --show-labels
NAME                     READY   STATUS    RESTARTS   AGE   LABELS
nginx-replicaset-5842n   1/1     Running   0          14s   app=nginx
nginx-replicaset-lrvdt   1/1     Running   0          14s   app=nginx
nginx-replicaset-qzmmr   1/1     Running   0          14s   app=nginx
```

Still running, did you know what happened here? I think you know.

As you can see above the old pods get deleted and new ones are created, you can also check their age.

Let's take a look at what happened here

```bash
$ k describe pod nginx-replicaset-5842n
Name:             nginx-replicaset-5842n
Namespace:        default
Priority:         0
Service Account:  default
Node:             k8s-multi-node-cluster-worker/172.18.0.2
Start Time:       Fri, 16 Dec 2022 20:52:12 +0530
Labels:           app=nginx
Annotations:      <none>
Status:           Running
IP:               10.244.2.8
IPs:
  IP:           10.244.2.8
Controlled By:  ReplicaSet/nginx-replicaset
...
...
...
```

Did you notice something `Controlled By: ReplicaSet/nginx-replicaset` what does it mean?

Basically, it means that the pod `nginx-replicaset-5842n` is managed by the `nginx-replicaset` replicaset and we know very well that replicaset has to ensure that the specified number of pods are always running at any point in time so when the replicaset noticed that the desired number of replicas are three but none of them are available so it created desired number of replicas, it's doing its job. You can also check other pods.

Also, let's take a look at our replicaset

```bash
$ k describe replicaset nginx-replicaset
Name:         nginx-replicaset
Namespace:    default
Selector:     app=nginx
Labels:       <none>
Annotations:  <none>
Replicas:     3 current / 3 desired
Pods Status:  3 Running / 0 Waiting / 0 Succeeded / 0 Failed
...
...
...
Events:
  Type    Reason            Age    From                   Message
  ----    ------            ----   ----                   -------
  Normal  SuccessfulCreate  6m24s  replicaset-controller  Created pod: nginx-replicaset-7vhl2
  Normal  SuccessfulCreate  6m24s  replicaset-controller  Created pod: nginx-replicaset-4vhhl
  Normal  SuccessfulCreate  6m24s  replicaset-controller  Created pod: nginx-replicaset-dwtn6
  Normal  SuccessfulCreate  4m6s   replicaset-controller  Created pod: nginx-replicaset-xw68w
  Normal  SuccessfulCreate  3m14s  replicaset-controller  Created pod: nginx-replicaset-lrvdt
  Normal  SuccessfulCreate  3m14s  replicaset-controller  Created pod: nginx-replicaset-qzmmr
  Normal  SuccessfulCreate  3m14s  replicaset-controller  Created pod: nginx-replicaset-5842n
```

Did you notice something?  
`Replicas: 3 current / 3 desired` and  
`Pods Status: 3 Running / 0 Waiting / 0 Succeeded / 0 Failed`

It means that we want three replicas of our app and all of them are currently running. No matter what happens with the pods having the label `app=nginx` it will make sure that at any time the desired number of pods are always running and you can also check events that pods are successfully created. Although you can create a ReplicaSet, it's not recommended, the deployment controller will take care of it. So, let's take a look at **Deployment.**

## Deployment

**Imperative way:**

```bash
$ k create deploy nginx-deploy --image=nginx:1.22 --replicas=3
```

**Declarative way:**

Let's create a manifest file `nginx-deploy.yaml` for our nginx webserver deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
       - name: nginx
         image: nginx:1.22
```

```bash
$ k create -f nginx-deploy.yaml
deployment.apps/nginx-deploy created
```

Let's check all the resources created by nginx deployment

```bash
$ k get all
NAME                                READY   STATUS    RESTARTS   AGE
pod/nginx-deploy-7cdf965978-2cvsc   1/1     Running   0          3s
pod/nginx-deploy-7cdf965978-2msnz   1/1     Running   0          3s
pod/nginx-deploy-7cdf965978-6vrsc   1/1     Running   0          3s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   31h

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deploy   3/3     3            3           3s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-deploy-7cdf965978   3         3         3       3s
```

*For now, just ignore the* `service/kubernetes`*, we'll learn about it in the upcoming tutorial.*

As you can see above we've successfully created the nginx deployment and the deployment internally created a replicaset `nginx-deploy-7cdf965978` and that replicaset created the desired number of replicas of pods for our application.

Let's take a look at all of them one by one.

* Deployment
    

```bash
$ k describe deploy nginx-deploy
Name:                   nginx-deploy
Namespace:              default
CreationTimestamp:      Fri, 16 Dec 2022 21:25:01 +0530
Labels:                 app=nginx
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=nginx
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
...
...
...
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-deploy-7cdf965978 (3/3 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  56s   deployment-controller  Scaled up replica set nginx-deploy-7cdf965978 to 3
```

As you can see above that the deployment `nginx-deploy` internally created a replicaset `nginx-deploy-7cdf965978` and all the desired replicas are available

* ReplicaSet
    

```bash
$ k describe rs nginx-deploy-7cdf965978
Name:           nginx-deploy-7cdf965978
Namespace:      default
Selector:       app=nginx,pod-template-hash=7cdf965978
Labels:         app=nginx
                pod-template-hash=7cdf965978
Annotations:    deployment.kubernetes.io/desired-replicas: 3
                deployment.kubernetes.io/max-replicas: 4
                deployment.kubernetes.io/revision: 1
Controlled By:  Deployment/nginx-deploy
Replicas:       3 current / 3 desired
Pods Status:    3 Running / 0 Waiting / 0 Succeeded / 0 Failed
...
...
...
Events:
  Type    Reason            Age   From                   Message
  ----    ------            ----  ----                   -------
  Normal  SuccessfulCreate  7m5s  replicaset-controller  Created pod: nginx-deploy-7cdf965978-2cvsc
  Normal  SuccessfulCreate  7m5s  replicaset-controller  Created pod: nginx-deploy-7cdf965978-2msnz
  Normal  SuccessfulCreate  7m5s  replicaset-controller  Created pod: nginx-deploy-7cdf965978-6vrsc
```

As you can see above that this replicaset is `Controlled By: Deployment/nginx-deploy` so we don't need to worry about the replicaset because our deployment will take care of it and the replicaset has successfully created the desired number of replicas.

* Pods
    

```bash
$ k describe pod nginx-deploy-7cdf965978-2cvsc
Name:             nginx-deploy-7cdf965978-2cvsc
Namespace:        default
Priority:         0
Service Account:  default
Node:             k8s-multi-node-cluster-worker2/172.18.0.5
Start Time:       Fri, 16 Dec 2022 21:25:01 +0530
Labels:           app=nginx
                  pod-template-hash=7cdf965978
Annotations:      <none>
Status:           Running
IP:               10.244.3.16
IPs:
  IP:           10.244.3.16
Controlled By:  ReplicaSet/nginx-deploy-7cdf965978
...
...
...
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  10m   default-scheduler  Successfully assigned default/nginx-deploy-7cdf965978-2cvsc to k8s-multi-node-cluster-worker2
  Normal  Pulled     10m   kubelet            Container image "nginx:1.22" already present on machine
  Normal  Created    10m   kubelet            Created container nginx
  Normal  Started    10m   kubelet            Started container nginx
```

We got a lot of information about this pod but we're interested in `Controlled By: ReplicaSet/nginx-deploy-7cdf965978` , and events that state that the container inside the pod has been started. I think you got an idea of what is the meaning of `Controlled By: ReplicaSet/nginx-deploy-7cdf965978` here.

Now just imagine your application becomes so popular and traffic increases then what will you do?

Simple just scale your deployment

```bash
$ k scale deploy nginx-deploy --replicas=10
deployment.apps/nginx-deploy scaled
```

Now check the pods

```bash
$ k get pods
NAME                            READY   STATUS    RESTARTS   AGE
nginx-deploy-7cdf965978-9mpv4   1/1     Running   0          17m
nginx-deploy-7cdf965978-9r2dn   1/1     Running   0          39s
nginx-deploy-7cdf965978-ctnld   1/1     Running   0          39s
nginx-deploy-7cdf965978-j7dxc   1/1     Running   0          39s
nginx-deploy-7cdf965978-qgfdz   1/1     Running   0          39s
nginx-deploy-7cdf965978-rbr2w   1/1     Running   0          39s
nginx-deploy-7cdf965978-s2l2b   1/1     Running   0          39s
nginx-deploy-7cdf965978-tz9zw   1/1     Running   0          17m
nginx-deploy-7cdf965978-vl8x6   1/1     Running   0          39s
nginx-deploy-7cdf965978-zz7mq   1/1     Running   0          17m
```

Let's check our deployment

```bash
$ k describe deploy nginx-deploy
Name:                   nginx-deploy
Namespace:              default
CreationTimestamp:      Fri, 16 Dec 2022 21:25:01 +0530
Labels:                 app=nginx
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=nginx
Replicas:               10 desired | 10 updated | 10 total | 10 available | 0 unavailable
...
...
...
NewReplicaSet:   nginx-deploy-7cdf965978 (10/10 replicas created)
Events:
  Type    Reason             Age                From                   Message
  ----    ------             ----               ----                   -------
  Normal  ScalingReplicaSet  17m (x2 over 43m)  deployment-controller  Scaled up replica set nginx-deploy-7cdf965978 to 3
  Normal  ScalingReplicaSet  65s                deployment-controller  Scaled up replica set nginx-deploy-7cdf965978 to 10 from 3
```

we've successfully scaled our deployment from 3 replicas to 10 and all of the replicas are available. That's it for deployment.

### Misc

If you're thinking how do I know the **apiVersion**, short name for these objects, etc. Let me tell you it's very easy.

```bash
# list all resources
$ k api-resources

# for a particular kind of object
$ k api-resources | grep -i $name_of_object

# Example: I want to know the apiVersion of Deployment object
$ k api-resources | grep -i deployment
deployments                       deploy       apps/v1                                true         Deployment
```

we get what we want i.e.,

* **apiVersion:** apps/v1
    
* **short name:** deploy
    

That's it for Kubernetes Pod, ReplicaSet, and Deployment. This is the fourth part of the [Kubernetes Fundamental series](https://anuragr.hashnode.dev/series/kubernetes-fundamentals) where we've done hands-on with **Kubernetes Pods, ReplicaSet, and Deployment**. I'll see you in the next tutorial.

Thank you so much for taking your valuable time to read.