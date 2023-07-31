---
title: "Kubernetes Objects"
seoTitle: "Kubernetes objects"
datePublished: Tue Jan 10 2023 03:26:04 GMT+0000 (Coordinated Universal Time)
cuid: clcpo7ttg000208ldbebd3ida
slug: kubernetes-objects
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1671385314691/q28UWMbYS.png
tags: kubernetes

---

**Kubernetes objects** are persistent entities in the Kubernetes system. Kubernetes uses these entities to represent the state of your cluster. In this tutorial, we will learn about different objects.

* **Pod** - Pod is an atomic deployable workload unit. It encapsulates an application's container. They are ephemeral in nature and they can have one or more than one container. If a pod fails it is replaced with a new pod with a shiny new IP address. You don't update a pod, you replace it with an updated version. All containers of a pod run on the same node. A pod never spans two nodes.
    
    Different lifecycles of a pod.  
    
    **Pod Lifecycle**
    
    * **Pending** - The pod is accepted by the cluster but has not yet been created may be due to insufficient resources.
        
    * **Running** - The Pod has been bound to a node, and all of the container(s) have been created.
        
    * **Succeeded** - The container(s) in the pod have exited with status 0.
        
    * **Failed** - All containers and at least one container exited with the non-zero status.
        
    * **Unknown** - For some reason, the state of the pod cannot be obtained. It may be due to communication issues with the node.
        
    * **CrashLoopBackOff** - The pod started, crashed, started again, and then crashed again, and so on unless RestartPolicy is set to Never. It typically occurs due to misconfiguration, an unavailable resource like PersistentVolume that is not mounted or very specific to your application.
        
* **ReplicaSet** - It is a primary method of managing Pod replicas (copy of pod) and their lifecycle to provide self-healing capabilities. Their job is to always ensure that the desired number of pods are always running. While you can create ReplicaSet but the recommended way is to create Deployment which internally manages Replicaset.
    
* **Deployment** - A deployment manages a single pod template. You create a deployment for each of your microservices like frontend, backend, payment processor, etc and Deployments create ReplicaSets in the background we don't need to manage ReplicaSets directly the Deployment controller manages itself. Deployments are used to provide rolling updates and rollbacks, scaling up the deployment to facilitate more load. Since Deployment internally creates ReplicaSets so a Deployment can self-heal, scale, update, and rollback.
    
* **DaemonSet** - A DaemonSet is a specialized workload whose role is to ensure that all the nodes (or a subset) run only an instance of a pod and the pods are scheduled by the scheduler controller and run by the daemon controller. DaemonSet ensures as the nodes are added to the cluster the pod got automatically deployed to that node and as the nodes are removed the pod also gets removed from that node.
    
    Typical uses:
    
    * Running a cluster storage daemon
        
    * Running a logs collection daemon on every node
        
    * Running a node monitoring daemon on every node
        
* **StatefulSet** - StatefulSets are used when the pods need to persist or maintain the state of the application. Unlike a Deployment, a StatefulSet maintains a sticky identity for each of its pods. Each pod has a persistent identifier. If a pod dies, it is replaced with another one using the identifier. It creates a series of pods in sequence from 0 to N and deletes them in reverse order.
    
* **Volumes** - By default when the Pods are restarted or the container within the pod crashes all the data stored in it will be lost because of ephemeral storage provided by the File Systems of Kubernetes. This form of storage will remove all the data stored in such containers when the Pod is deleted. The Kubernetes volume abstraction solves these problems. Kubernetes supports several types of volumes. Kubernetes Volume will provide persistent storage such that the data exists for the whole lifetime of the Pod.
    
* **Service** - A Service is used to expose an application deployed on a set of pods using a single endpoint. Services get static IP addresses and DNS names, unlike pods with ephemeral IP addresses. They serve as a way to access pods, they're kind of a middleman and they target pods using selectors.
    
* **ConfigMap & Secrets** - ConfigMap object allows you to decouple and externalize configuration so that your applications are easily portable. They are static meaning that if you change values, the container will have to be restarted to get them.
    
    Like ConfigMaps Secrets are used to store confidential data so they are somewhat identical to ConfigMaps except that they store values as [base64](https://en.wikipedia.org/wiki/Base64) encoded strings, base64 is a way to encode a string, not an encryption algorithm. Both methods permit changes in the configuration without rebuilding your application.
    
* **Namespace** - Kubernetes Namespace allows us to group resources. They are logically separated from each other and are generally used when a large number of users exist in the form of multiple teams or projects e.g., dev, test, prod, etc. Names of resources need to be unique within a namespace, but not across namespaces. Namespaces cannot be nested inside one another and each Kubernetes resource can only be in one namespace but the objects in one namespace can access objects in a different one. Deleting a namespace will delete all its child objects.
    
* **Job** - Kubernetes Jobs is the workload for short-lived tasks. They create one or more pods and ensure that a specified number of them are successfully terminated. They run to completion which means that when we create a job the API server will create a pod and that pod will execute its task, and when the task is completed the pod will shut down by itself. As pods successfully completed, the Job tracks the successful completions and when a specified number of successful completions is reached, the Job completes. By default, jobs with more than 1 pod will create them one after the other.
    
* **CronJob** - A cronjob is a workload that runs a job on schedule. It’s an extension of the Job and provides a method of executing jobs on a cron-like schedule. The schedule is defined using cron-like syntax in UTC.
    

That's all for the Kubernetes object. This is the third part of the [Kubernetes Fundamental series](https://anuragr.hashnode.dev/series/kubernetes-fundamentals) where we learned about the different Kubernetes objects. In the upcoming tutorial, we'll do hands-on with these objects so stay tuned.

Thank you so much for taking your valuable time to read.