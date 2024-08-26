---
title: "Building a Real-world Kubernetes Operator"
seoTitle: "Creating a Kubernetes Operator"
seoDescription: "Build a real-world Kubernetes Operator in Golang with hands-on Kubebuilder tutorials. Ideal for developers with Kubernetes experience."
datePublished: Mon Aug 26 2024 06:53:25 GMT+0000 (Coordinated Universal Time)
cuid: cm0an6hgt000108mo940ee8us
slug: building-a-real-world-kubernetes-operator
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1720330926463/ed03eea7-498c-4058-817c-9534c83174bf.png
tags: kubernetes, kubernetes-operators

---

# Introduction

In this series of in-depth tutorials, you'll learn how to build a real-world Kubernetes Operator in Golang with integration and end-to-end testing. We'll focus on hands-on coding to give you practical experience. So, roll up your sleeves and get ready to write code with me. We'll build an operator similar to [Nimbus](https://github.com/5GSEC/nimbus/) from scratch, just like my team did!

# Prerequisites

* Working knowledge of Kubernetes,
    
* Access to a Kubernetes cluster,
    
* Working Go SDK,
    
* And of course, your favourite IDE to write code (Tone of code) 😎.
    

# Operator

First, let's understand what an operator is. The Kubernetes documentation describes the [Operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) as follows:

***Operators are software extensions to Kubernetes that make use of custom resources to manage applications and their components. Operators follow Kubernetes principles, notably the control loop.***

Another good definition from [CoreOS:](https://coreos.com/blog/introducing-operators.html)

***An operator represents human operational knowledge in software, to reliably manage an application.***

There are many different use cases spanning different domains, but the general idea is:

***Manage some resources (that reside inside or outside the cluster), using Kubernetes native manifests and tooling.***

In simple terms, an operator includes [Custom Resource Definitions](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#customresourcedefinitions) (CRDs) and Controllers to manage those [Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) (CR).

You might be wondering, what is a [controller](https://kubernetes.io/docs/concepts/architecture/controller/) now?

***Controllers are control loops that monitor your cluster's state and make changes to align the current state with the desired one.***

# Design

Before writing any code, we need a high-level design to understand the product's purpose and the problem it will solve. Let's look at what this operator is and its design.

## Purpose

> Kubernetes is hard and securing it is even much harder.

The problem we're tackling: Today, securing your workloads requires manually creating various policies, e.g., KubeArmor policies for runtime security, network policies for network security, Kyverno policies and so on. Wouldn't it be nice if you just specify your desired security state, and our operator automatically will figure out how to achieve it in the best way (when possible)?

At first, this might seem like a new thing but we're not inventing a new wheel, this is a well-known pattern in Kubernetes. If you know how Kubernetes handles storage then you can easily understand this.

The name of our operator is **Nimbus** and it will simplify security using an intent-driven approach, i.e., users will define their desired security state as intent and intent-binding to bind (apply) the intent on resources, and the operator automatically translates it into the necessary security engine policies to achieve that state.

## Architecture

Goals:

* It should be declarative.
    
* It should be extensible to support future security engine(s).
    
* It should be generic and not tied to any specific security engine.
    

## High Level

Here is the bird's eye view of our operator:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1719850525603/89db8ffb-2b26-4a31-9a20-ec7984b06fab.png align="center")

1. A user will create an intent and intent binding.
    
2. Nimbus will create and manage different (if needed) policies as a result of created intent and intentbinding.
    

Easy, isn't it?

## Low Level

That high-level architecture is great for everyone else, but we engineers need low-level details, aka implementation details. So here they are.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1719853459822/8a715fab-c337-4faf-a05b-1d095af961d3.png align="center")

### Components

1. **Core** - The Nimbus core is composed of a set of dedicated controllers and processors for handling custom resources.
    
    Nimbus defines intent and intent-binding by following Kubernetes custom resources:
    
    * **SecurityIntent:** Defines the security intent itself.
        
    * **SecurityIntentBinding:** Binds SecurityIntents to namespace-level resources, like pods.
        
    * **NimbusPolicy:** Representation of SecurityIntent(s) and its SecurityIntentBinding for security engines within a namespace.
        
    
    And controllers for:
    
    * **SecurityIntent:** Manages intent definitions for desired security configurations.
        
    * **SecurityIntentBinding:** Binds SecurityIntents to specific Kubernetes resources.
        
    
    **Processors:** Perform specific tasks on resources, such as validation, transformation, or translation into platform-specific security configurations.
    
2. **Security Engine Adapter** - Nimbus security engine adapters act as plugins enabling integration with various security engines like KubeArmor, NetworkPolicy, Kyverno, Istio, etc. These adapters translate our intent defined within our system into native rules understood by the target security engine. Don't worry we'll revisit adapter/plugins with greater details once we're done with the core.
    

With this, we completed the design and architecture.

# Implementation

Are you excited to write code?

We'll use [Kubebuilder](https://www.kubebuilder.io/quick-start.html) to build our operator although you can use [Operator SDK](https://sdk.operatorframework.io) if you want.

Install [Kubebuilder](https://www.kubebuilder.io/quick-start.html#installation)

```bash
# download kubebuilder and install locally.
curl -L -o kubebuilder "https://go.kubebuilder.io/dl/latest/$(go env GOOS)/$(go env GOARCH)"
chmod +x kubebuilder && sudo mv kubebuilder /usr/local/bin/
```

Let's initialize our project:

```bash
kubebuilder init --plugins go/v4 --domain security.nimbus.com  --repo github.com/anurag-rajawat/tutorials/nimbus
```

Change the value of `--domain` and `--repo` flag accordingly.

You should have a similar structure to:

```bash
$ tree
.
├── Dockerfile
├── Makefile
├── PROJECT
├── README.md
├── cmd
│   └── main.go
├── config
│   ├── default
│   │   ├── kustomization.yaml
│   │   ├── manager_metrics_patch.yaml
│   │   └── metrics_service.yaml
│   ├── manager
│   │   ├── kustomization.yaml
│   │   └── manager.yaml
│   ├── prometheus
│   │   ├── kustomization.yaml
│   │   └── monitor.yaml
│   └── rbac
│       ├── kustomization.yaml
│       ├── leader_election_role.yaml
│       ├── leader_election_role_binding.yaml
│       ├── role.yaml
│       ├── role_binding.yaml
│       └── service_account.yaml
├── go.mod
├── go.sum
├── hack
│   └── boilerplate.go.txt
└── test
    ├── e2e
    │   ├── e2e_suite_test.go
    │   └── e2e_test.go
    └── utils
        └── utils.go

11 directories, 24 files
```

Kubebuilder created scaffolding for us, we just need to add our business code rest will be handled by Kubebuilder.

* `cmd` - directory contains the main package for running the operator.
    
* `config` - directory contains the configurations aka launch configuration in subdirectories as follows:
    
    * `default` - manifests to directly run the operator with default configs.
        
    * `manager` - manifests to run the operator as a pod in a cluster.
        
    * `prometheus` - manifests to monitor our operator using Prometheus.
        
    * `rbac` - manifests related to the permission required to run the operator in a cluster.
        
* `hack` - directory is mostly used for scripts, and license headers and is a well-known directory in the Kubernetes ecosystem.
    
* `test` - directory for tests.
    
* `PROJECT` - this file is used by Kubebuilder to track the project and scaffolding new components.
    

Feel free to explore the generated code.

Now let's scaffold API for our SecurityIntent and SecurityIntentBinding custom resources.

```bash
kubebuilder create api --controller=true --resource=true --namespaced=false --group=intent --version=v1alpha1 --kind=SecurityIntent
kubebuilder create api --group=intent --version=v1alpha1 --kind=SecurityIntentBinding --controller=true --resource=true
kubebuilder create api --controller=false --resource=true --group=intent --version=v1alpha1 --kind=NimbusPolicy
```

The first and second commands create the `SecurityIntent` CR, which is global, and the `SecurityIntentBinding` CR, which is namespace-scoped, along with their controllers. The last command creates the `NimbusPolicy` CR without a controller, and it is also namespace-scoped.

You should see that it created two new directories `api` and `internal/controller` to keep CRs and their controllers, similar to:

```bash
$ tree api internal
api
└── v1alpha1
    ├── groupversion_info.go
    ├── nimbuspolicy_types.go
    ├── securityintent_types.go
    ├── securityintentbinding_types.go
    └── zz_generated.deepcopy.go
internal
└── controller
    ├── securityintent_controller.go
    ├── securityintent_controller_test.go
    ├── securityintentbinding_controller.go
    ├── securityintentbinding_controller_test.go
    └── suite_test.go

4 directories, 10 files
```

* `groupversion_info.go` - contains our API metadata such as group (`intent.security.nimbus.com`) and its version (`v1alpha1`).
    
* `nimbuspolicy_types.go` - contains type (structs) for our `NimbusPolicy` CR.
    
* `securityintent_types.go` - contains types (structs) for our `SecurityIntent` CR.
    
* `securityintentbinding_types.go` - contains types (structs) for our `SecurityIntentBinding` CR.
    
* `zz_generated.deepcopy.go` - contains the autogenerated code for the `runtime.Object` interface, marking all our main types as representing Kinds.
    

This is a lot for this part since I want to keep my posts concise and focused on specific functionalities. We'll implement types and controllers in the next post. Stay tuned!

You can find the complete code [here](https://github.com/anurag-rajawat/tutorials/tree/main/nimbus). Please feel free to comment or criticize :)

# Summary

> *This series of tutorials covers how to build a Kubernetes Operator in Golang, focusing on coding, integration, and end-to-end testing. We'll develop the Nimbus operator from scratch, which simplifies security through an intent-driven approach, automatically translating user-defined security intents into necessary policies. The tutorial includes setting up prerequisites, understanding operators and controllers, designing the operator’s architecture, and implementing it using Kubebuilder. You'll learn to scaffold APIs and controllers for custom resources like SecurityIntent and SecurityIntentBinding, laying the groundwork for future detailed implementations.*

# References

* [5GSec Nimbus](https://github.com/5GSEC/nimbus)
    
* [Programming Kubernetes](https://www.oreilly.com/library/view/programming-kubernetes/9781492047094/)
    
* [Kubebuilder book](https://www.kubebuilder.io/introduction)