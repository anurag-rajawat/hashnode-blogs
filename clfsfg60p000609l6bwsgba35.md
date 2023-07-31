---
title: "Deploy Java app on Kubernetes with JKube"
seoTitle: "Build and Deploy java applications on Kubernetes without any hassle"
datePublished: Tue Mar 28 2023 15:43:02 GMT+0000 (Coordinated Universal Time)
cuid: clfsfg60p000609l6bwsgba35
slug: deploy-java-app-on-kubernetes-with-jkube
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1679992230338/311eeeba-8c18-4c36-99d3-1f152abe2e27.png
tags: java, kubernetes, jkube

---

Welcome everyone. Today we will deploy a java spring-boot web app on the Kubernetes cluster using the [Eclipse JKube](https://github.com/eclipse/jkube) maven plugin.

Now you're thinking what is Eclipse JKube?

[Eclipse JKube](https://github.com/eclipse/jkube) is a maven plugin to build container images and deploy Java applications on the Kubernetes/Openshift cluster without writing any Kubernetes manifests file. It automatically detects the framework used in the application and generates cluster configuration files and deploys them on Kubernetes/Openshift cluster. Not only this it supports various configurations to know more about it please visit [https://www.eclipse.org/jkube/](https://www.eclipse.org/jkube/)

## Requirements

* Jdk 17+
    
* Docker
    
* Kubernetes cluster
    

That's all you need.

Clone the repository [https://github.com/anurag-rajawat/spring-boot-hello-world.git](https://github.com/anurag-rajawat/spring-boot-hello-world.git) and change the directory to `spring-boot-hello-world`

Start your cluster and expose your docker-daemon to build docker images directly inside the minikube cluster.

```bash
$ minikube start
$ eval $(minikube -p minikube docker-env)
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1679984862105/c7aa7e0f-6351-4fa8-b41e-3b9eb4a6b1a9.png align="center")

Package the application

```bash
$ ./mvnw clean package
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1679985332540/00dc3b38-022f-4751-b558-b80386fea5a5.png align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1679985350249/b5f813c8-51c9-4dce-b125-f0ab872761f9.png align="center")

Our application is successfully packaged.

Now comes the interesting part which is deploying the app on the cluster. So let's deploy

```bash
$ ./mvnw k8s:build k8s:resource k8s:apply
```

By default, JKube will generate resource descriptors for Kubernetes deployment. In our case not only deployment but `NodePort` service resource descriptors will be generated as well because I explicitly told JKube to generate `NodePort` service by configuring service type, otherwise we've to manually expose our deployment or do port-forwarding to access our application.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1679986467972/be78c516-95c9-49ca-956c-7e53f251ec9f.png align="center")

As we can see the build is successful. So, let's check the application deployment, pod, and service status, and then we see what are these goals.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1679987804441/28538fef-1010-4173-ae72-bed7dc4fa239.png align="center")

Voilà our application is successfully running 🥳

Now, let's discuss what are those goals:

* `k8s: build`: This goal is responsible for creating the Docker images containing the actual application. These then can be deployed later on Kubernetes or OpenShift cluster.
    
* `k8s:resource`: This goal generates the Kubernetes resource descriptors which are packaged within the maven artifacts and can be deployed to the Kubernetes or Openshift cluster.
    
* `k8s:apply`: This goal applies the resource descriptors generated previously to a connected Kubernetes or Openshift cluster.
    

If you're curious you can check the resource descriptors in the `target/classes/META-INF/jkube` directory

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1679987540821/2a46bc8f-46b4-4fab-9337-2c68b6f65c96.png align="center")

So far so good, let's access the application

```bash
$ minikube service spring-boot-hello-world --url
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1679989181475/259ce473-c9f0-4663-a27e-7a20ba4e03a0.png align="center")

Invoke the service URL in your favorite browser and you can see like 👇

* `/` endpoint
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1679989243788/e92b5103-3544-4e61-91d0-1d83fa48ac47.png align="center")

* `<url>/greeting?name=User` endpoint
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1679989421555/b9aaf49a-7651-429e-b98b-3bb2a5b2eb00.png align="center")

As we can see our application is working properly.

We deployed our app using JKube on Kubernetes without any hassle. If we try to deploy it without JKube then we have to write a Dockerfile, build the image, and then resource descriptors which are quite complex.

JKube made our life so much easier and enhances our productivity by helping us to focus only on building our application rest will be handled by JKube.

JKube also offers Kubernetes Remote Development feature which is very interesting. You can check more about it at [JKube Remote Development](https://www.eclipse.org/jkube/docs/kubernetes-maven-plugin/#jkube:remote-dev). If you want more information about JKube please [click here](http://github.com/eclipse/jkube).

I hope you find this useful. Thank you so much for taking your valuable time to read.