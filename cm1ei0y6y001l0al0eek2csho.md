---
title: "Building a Real-world Kubernetes Operator: Part 5"
seoTitle: "Kubernetes Operator: Real-world Guide Part 5"
seoDescription: "Build SecurityEngine adapters to translate security intents to native rules."
datePublished: Mon Sep 23 2024 04:19:56 GMT+0000 (Coordinated Universal Time)
cuid: cm1ei0y6y001l0al0eek2csho
slug: building-a-real-world-kubernetes-operator-part-5
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1724593170284/691a99f3-c254-4c51-898a-59a76170adf5.png
tags: kubernetes, kubernetes-operators

---

# Introduction

In the previous part, we completed the core of our operator. From this part, we start with the next component i.e., SecurityEngine adapters/plugins which will translate the defined intent into native rules understood by the target security engine.

# Architecture

As usual, let's look at the architecture of adapters.

## High Level

Here is the bird's eye view:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1722002165823/0aa756ec-ff4b-4b78-b340-e0249866258e.png align="center")

Adapters will watch for NimbusPolicy and manage corresponding security engine policies.

## Low Level

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1722008947182/429ba305-a298-4906-9b88-35a2bce1a932.png align="center")

### Components

* **Watcher:** Watches NimbusPolicy objects within the Kubernetes cluster.
    
* **Builder:** Analyzes the watched NimbusPolicy, and translates them into the native security engine policy format of the target security engine. Generating the adapter policies based on the processed information.
    
* **Manager:** Takes ownership of the generated adapter policies. This includes deploying them and handling updates or deletions triggered by changes in the original NimbusPolicy.
    

The adapters are responsible for:

* **Filtering:** Skipping orphaned policies (without OwnerReferences) in NimbusPolicy to prevent unintended policy creation.
    
* **Policy Management:** Performing CRUD operations (Create, Read, Update, Delete) on policies within their scope.
    
* **State Reconciliation:** Ensuring the actual state of policies matches the desired state defined in the corresponding NimbusPolicy custom resource, even if the adapter policies are modified manually by an administrator. This guarantees consistent security enforcement.
    

### Message Sequence Diagram

Message Sequence Diagram to make things crystal clear:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1722011769617/2d425343-e1e5-4f7b-9279-0b70103d019e.png align="center")

So this is the architecture of the Nimbus adapter. Let's implement those.

# Implementation

First, we'll create an adapter for [KubeArmor](https://kubearmor.io), a tool that enforces security at runtime. We won't cover runtime security in detail here, as it deserves its own post.

Create a new directory called `adapter/nimbus-kubearmor` in the `pkg` directory where we will keep the adapter.

```bash
mkdir -p pkg/adapter/nimbus-kubearmor && cd "$_"
```

Initialize its go module:

```bash
go mod init github.com/anurag-rajawat/tutorials/nimbus/adapter/nimbus-kubearmor
```

Change the module according to your github.

Create the directory structure as follows for the adapters:

```bash
$ tree pkg/adapter
pkg/adapter
├── idpool
│   └── idpool.go
├── k8s
│   └── client.go
└── nimbus-kubearmor
    ├── Dockerfile
    ├── Makefile
    ├── builder
    │   ├── builder.go
    │   └── ksp.go
    ├── cmd
    │   └── main.go
    ├── go.mod
    ├── manager
    │   ├── ksp.go
    │   └── manager.go
    └── watcher
        └── watcher.go

8 directories, 11 files
```

Our project now consists of two Go modules, so we need to establish a multi-module workspace. Let's initialize it, or you might not be able to run the adapter.

```bash
# In the project's root directory
go work init
go work use ./
go work use pkg/adapter/nimbus-kubearmor
```

For more info please check [this](https://go.dev/doc/tutorial/workspaces).

## Watcher

Let's start with the watcher, which will watch NimbusPolicy.

To be notified of state changes (CRUD operations) on NimbusPolicy CR, we'll utilize Kubernetes informers.

Informers provide a higher-level programming interface optimized for the common use case of watching resources. They offer in-memory caching and efficient indexed lookups of objects by name or other properties, significantly improving performance compared to directly polling the API server. By caching data locally, informers reduce the load on the API server and enable near real-time responses to object changes.

Let's create the informer for NimbusPolicy CR, edit the `watcher.go` file in `watcher` directory as follows:

```go
package watcher

import (
	"github.com/go-logr/logr"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/dynamic/dynamicinformer"
	"k8s.io/client-go/tools/cache"
)

type Watcher struct {
	dynamicClient dynamic.Interface
	factory       dynamicinformer.DynamicSharedInformerFactory
	logger        logr.Logger
}

func (w *Watcher) getNpInformer() cache.SharedIndexInformer {
	nimbusPolicyGvr := schema.GroupVersionResource{
		Group:    "intent.security.nimbus.com",
		Version:  "v1alpha1",
		Resource: "nimbuspolicies",
	}
	return w.factory.ForResource(nimbusPolicyGvr).Informer()
}
```

Let's break down the purpose of each field within `Watcher` struct:

* **dynamicClient**: Kubernetes offers a typed client (`kubernetes.ClientSet`) for interacting with its native resources. However, in this case, we're dealing with **Custom Resources**, which are user-defined resources not natively supported by Kubernetes. To work with these CRs, we use a `dynamic.Interface` client. This dynamic client allows us to interact with any resource type, including built-in and custom ones.
    
* **factory**: While informers are preferred over polling, they create a load on the API server. One binary should instantiate only one informer per `GroupVersionResource`. To make sharing of informers easy, we can instantiate an informer by using the shared informer factory. The shared informer factory allows informers to be shared for the same resource in an application. In other words, different control loops can use the same watch connection to the API server under the hood. Since we're using a dynamic client, so we've to use `dynamicinformer.DynamicSharedInformerFactory`.
    
* **logger:** This is self-explanatory.
    

The `getNpInformer` method instantiates an informer for the NimbusPolicy CR. Since the dynamic client does not know the resource you want to consume, so, you need to first provide a `schema.GroupVersionResource`, which is a Golang type that provides the necessary information to construct an HTTP request to the API Server.

Now that we've established the informer, we can proceed to create a watcher who will watch NimbusPolicy for state changes by leveraging the informer.

Edit the `watcher.go` file as follows:

```go
package watcher

import (
	"context"

	"github.com/go-logr/logr"
	"k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/dynamic/dynamicinformer"
	k8sscheme "k8s.io/client-go/kubernetes/scheme"
	"k8s.io/client-go/tools/cache"

	intentv1alpha1 "github.com/anurag-rajawat/tutorials/nimbus/api/v1alpha1"
)
...
...
func (w *Watcher) watchNimbusPolicies(ctx context.Context, nimbusPolicyChan chan *intentv1alpha1.NimbusPolicy) {
	informer := w.getNpInformer()
	handlers := cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			u := obj.(*unstructured.Unstructured)
			w.logger.V(4).Info("NimbusPolicy found", "nimbusPolicy.Name", u.GetName(), "nimbusPolicy.Namespace", u.GetNamespace())
			nimbusPolicyChan <- w.getDecodedNimbusPolicy(u)
		},
		UpdateFunc: func(oldObj, newObj interface{}) {
			oldU := oldObj.(*unstructured.Unstructured)
			newU := newObj.(*unstructured.Unstructured)
			if oldU.GetGeneration() != newU.GetGeneration() {
				w.logger.V(4).Info("NimbusPolicy updated", "nimbusPolicy.Name", newU.GetName(), "nimbusPolicy.Namespace", newU.GetNamespace())
				nimbusPolicyChan <- w.getDecodedNimbusPolicy(newU)
			}
		},
	}

	_, err := informer.AddEventHandler(handlers)
	if err != nil {
		w.logger.Error(err, "failed to add nimbus policy handler")
		return
	}

	w.logger.Info("Started NimbusPolicy watcher")
	informer.Run(ctx.Done())
	close(nimbusPolicyChan)
	w.logger.Info("Stopped NimbusPolicy watcher")
}

func (w *Watcher) getDecodedNimbusPolicy(obj *unstructured.Unstructured) *intentv1alpha1.NimbusPolicy {
	bytes, err := obj.MarshalJSON()
	if err != nil {
		w.logger.Error(err, "failed to marshal nimbusPolicy")
		return nil
	}

	nimbusPolicy := &intentv1alpha1.NimbusPolicy{}
	decoder := k8sscheme.Codecs.UniversalDeserializer()
	_, _, err = decoder.Decode(bytes, nil, nimbusPolicy)
	if err != nil {
		w.logger.Error(err, "failed to decode nimbusPolicy")
		return nil
	}

	return nimbusPolicy
}

func (w *Watcher) Run(ctx context.Context, nimbusPolicyChan chan *intentv1alpha1.NimbusPolicy) {
	w.logger.Info("Starting NimbusPolicy watcher")
	w.watchNimbusPolicies(ctx, nimbusPolicyChan)
}
```

In `watchNimbusPolicies` method we define handlers for *add, and update* events and finally start the informer. These are usually used to trigger the business logic, in our case they decode the NimbusPolicy from `unstructured` object and then put it onto `nimbusPolicyChan` channel.

The `Run` method just wraps the `watchNimbusPolicies` method.

Finally, a function to get a new instance of the `Watcher` in `watcher.go` file:

```go
package watcher

import (
	"context"
	"time"

	"github.com/go-logr/logr"
	"k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/dynamic/dynamicinformer"
	k8sscheme "k8s.io/client-go/kubernetes/scheme"
	"k8s.io/client-go/tools/cache"
	"sigs.k8s.io/controller-runtime/pkg/log"

	intentv1alpha1 "github.com/anurag-rajawat/tutorials/nimbus/api/v1alpha1"
	"github.com/anurag-rajawat/tutorials/nimbus/pkg/adapter/k8s"
)
...
...
func NewWatcher(ctx context.Context) *Watcher {
	logger := log.FromContext(ctx)
	dynamicClient, err := k8s.NewDynamicClient()
	if err != nil {
		logger.Error(err, "failed to create kubernetes client")
		return nil
	}
	return &Watcher{
		dynamicClient: dynamicClient,
		logger:        logger,
		factory:       dynamicinformer.NewDynamicSharedInformerFactory(dynamicClient, time.Minute),
	}
}
```

The `NewWatcher` function makes use of `k8s.NewDynamicClient` function so let's create it as well.

Edit the `client.go` file in `pkg/adapter/k8s` directory as follows:

```go
package k8s

import (
	"errors"
	"fmt"
	"os"
	"path/filepath"

	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
)

func NewDynamicClient() (dynamic.Interface, error) {
	config, err := getConfig()
	if err != nil {
		return nil, fmt.Errorf("failed to get config: %v", err)
	}
	return dynamic.NewForConfig(config)
}

func getConfig() (*rest.Config, error) {
	config, err := rest.InClusterConfig()
	if err != nil && errors.Is(err, rest.ErrNotInCluster) {
		kubeConfig := filepath.Join(os.Getenv("HOME"), ".kube", "config")
		config, err = clientcmd.BuildConfigFromFlags("", kubeConfig)
		if err != nil {
			return nil, err
		}
	}
	return config, nil
}
```

The `NewDynamicClient` function creates a dynamic client by first using the `rest.InClusterConfig` from the service account token mounted in the pod at `/var/run/secrets/kubernetes.io/serviceaccount/token`. If it doesn't find `rest.InClusterConfig`, it then uses the `kubeconfig` file located at `$HOME/.kube/config`.

Now we're done with the watcher.

## Manager

Remember the responsibilities of the manager? The manager will manage the watcher and the generated security-engine-specific policies.

So let's, edit the `manager.go` file as follows:

```go
package manager

import (
	"context"
	"sync"

	"github.com/go-logr/logr"
	kubearmorv1 "github.com/kubearmor/KubeArmor/pkg/KubeArmorController/api/security.kubearmor.com/v1"
	"k8s.io/apimachinery/pkg/runtime"
	utilruntime "k8s.io/apimachinery/pkg/util/runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/log"

	"github.com/anurag-rajawat/tutorials/nimbus/adapter/nimbus-kubearmor/watcher"
	intentv1alpha1 "github.com/anurag-rajawat/tutorials/nimbus/api/v1alpha1"
	"github.com/anurag-rajawat/tutorials/nimbus/pkg/adapter/k8s"
)

type manager struct {
	logger    logr.Logger
	k8sClient client.Client
	scheme    *runtime.Scheme
	wg        *sync.WaitGroup
}

func (m *manager) run(ctx context.Context) {
	m.logger.Info("Starting manager")
	nimbusPolicyChan := make(chan *intentv1alpha1.NimbusPolicy)

	m.wg.Add(1)
	go func() {
		defer m.wg.Done()
		npWatcher := watcher.NewWatcher(ctx)
		npWatcher.Run(ctx, nimbusPolicyChan)
	}()

	m.wg.Add(1)
	go func() {
		defer m.wg.Done()
		m.managePolicies(ctx, nimbusPolicyChan)
	}()

	m.logger.Info("Started manager")
	m.wg.Wait()
}

func Run(ctx context.Context) {
	logger := log.FromContext(ctx)
	scheme := runtime.NewScheme()

	utilruntime.Must(intentv1alpha1.AddToScheme(scheme))
	utilruntime.Must(kubearmorv1.AddToScheme(scheme))

	k8sClient, err := k8s.NewClient(scheme)
	if err != nil {
		logger.Error(err, "failed to initialize Kubernetes client")
		return
	}

	mgr := &manager{
		logger:    logger,
		k8sClient: k8sClient,
		wg:        &sync.WaitGroup{},
		scheme:    scheme,
	}

	mgr.run(ctx)
}
```

Let's break down the above code snippet:

* **Manager struct:**
    
    * **logger:** Self-explanatory.
        
    * **k8sClient:** A generic Kubernetes client that allows working with multiple resources.
        
    * **scheme:** The main feature of a scheme is the mapping of Golang types to possible GVKs. Here is a nice [Twitter thread](https://x.com/iximiuz/status/1485704726595477515?lang=en) around this.
        
    * **wg:** Waitgroup for go routines.
        
* The `run` method spawns two goroutines one for the NimbusPolicy watcher and another one to manage the generated security-engine-specific policies.
    
* `Run` function registers KubeArmor and Nimbus types to the `scheme`, so that the k8sClient can build up the REST mapping. And finally starts the manager.
    

The `Run` function makes use of `k8s.NewClient(scheme)` function so let's create it as well.

Edit the `client.go` file in `pkg/adapter/k8s` directory as follows:

```go
package k8s

import (
	"errors"
	"fmt"
	"os"
	"path/filepath"

	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
	"sigs.k8s.io/controller-runtime/pkg/client"
)
...
...
// NewClient returns a new Client using the provided scheme to map go structs to
// GroupVersionKinds.
func NewClient(scheme *runtime.Scheme) (client.Client, error) {
	config, err := getConfig()
	if err != nil {
		return nil, fmt.Errorf("failed to get config: %v", err)
	}
	return client.New(config, client.Options{
		Scheme: scheme,
	})
}
```

Let's tackle the second responsibility of the manager, i.e., managing security-engine-specific policies.

Edit the `ksp.go` file in the `manager` directory as follows:

```go
package manager

import (
	"context"

	kubearmorv1 "github.com/kubearmor/KubeArmor/pkg/KubeArmorController/api/security.kubearmor.com/v1"
	apierrors "k8s.io/apimachinery/pkg/api/errors"
	"k8s.io/apimachinery/pkg/types"
	"sigs.k8s.io/controller-runtime/pkg/client"

	"github.com/anurag-rajawat/tutorials/nimbus/adapter/nimbus-kubearmor/builder"
	intentv1alpha1 "github.com/anurag-rajawat/tutorials/nimbus/api/v1alpha1"
)

func (m *manager) managePolicies(ctx context.Context, nimbusPolicyChan chan *intentv1alpha1.NimbusPolicy) {
	for {
		select {
		case <-ctx.Done():
			return
		case nimbusPolicy := <-nimbusPolicyChan:
			m.createOrUpdatePolicies(ctx, nimbusPolicy)
		}
	}
}

func (m *manager) createOrUpdatePolicies(ctx context.Context, nimbusPolicy *intentv1alpha1.NimbusPolicy) {
	ksps := builder.BuildPolicy(m.logger, m.scheme, nimbusPolicy)
	// Iterate using a separate index variable to avoid aliasing
	for idx := range ksps {
		ksp := &ksps[idx]

		var existingKsp kubearmorv1.KubeArmorPolicy
		err := m.k8sClient.Get(ctx, types.NamespacedName{Name: ksp.Name, Namespace: ksp.Namespace}, &existingKsp)
		if err != nil && !apierrors.IsNotFound(err) {
			m.logger.Error(err, "failed to get existing KubeArmorPolicy", "kubeArmorPolicy.Name", nimbusPolicy.Name, "KubeArmorPolicy.Namespace", nimbusPolicy.Namespace)
			return
		}

		if err != nil {
			if apierrors.IsNotFound(err) {
				if err = m.k8sClient.Create(ctx, ksp); err != nil {
					m.logger.Error(err, "failed to create KubeArmorPolicy", "kubeArmorPolicy.Name", ksp.Name, "kubeArmorPolicy.Namespace", ksp.Namespace)
					return
				}
				m.logger.Info("Successfully created KubeArmorPolicy", "kubeArmorPolicy.Name", ksp.Name, "kubeArmorPolicy.Namespace", ksp.Namespace)
			}
		} else {
			ksp.ObjectMeta.ResourceVersion = existingKsp.ObjectMeta.ResourceVersion
			if err = m.k8sClient.Update(ctx, ksp); err != nil {
				m.logger.Error(err, "failed to configure existing KubeArmorPolicy", "kubeArmorPolicy.Name", ksp.Name, "kubeArmorPolicy.Namespace", ksp.Namespace)
				return
			}
			m.logger.Info("Configured KubeArmorPolicy", "kubeArmorPolicy.Name", ksp.Name, "kubeArmorPolicy.Namespace", ksp.Namespace)
		}
	}
}
```

* `managePolicies` - This method continuously listens on `nimbusPolicyChan` channel and manage policies accordingly.
    
* `createOrUpdatePolicies` - This method creates or updates KubeArmor policies based on received NimbusPolicies derived from SecurityIntent and its SecurityIntentBinding. It utilizes a policy builder to construct KubeArmor policies from NimbusPolicies. If a KubeArmor policy already exists, it's updated if necessary; otherwise, it's created.
    
    The important thing to note here is the update process: the existing KubeArmor policy's `resourceVersion` field must be set in the newly constructed KubeArmor policy to ensure the API server successfully updates the resource. While the `resourceVersion` field is rarely accessed directly in client-go code, it's a fundamental component of Kubernetes that guarantees system integrity. As part of the `ObjectMeta`, this field is associated with an etcd key where the `resourceVersion` value originates.
    

You may wonder why we don't delete the created policies ourselves. It's because Kubernetes' garbage collector handles it since we set the NimbusPolicy as the owner of these policies.

## Builder

Let's create the builder that will construct KubeArmorPolicy objects from `nimbusRules`, edit the `builder.go` file as follows:

```go
package builder

import (
	"github.com/go-logr/logr"
	kubearmorv1 "github.com/kubearmor/KubeArmor/pkg/KubeArmorController/api/security.kubearmor.com/v1"
	"k8s.io/apimachinery/pkg/runtime"
	ctrl "sigs.k8s.io/controller-runtime"

	intentv1alpha1 "github.com/anurag-rajawat/tutorials/nimbus/api/v1alpha1"
	"github.com/anurag-rajawat/tutorials/nimbus/pkg/adapter/idpool"
)

func BuildPolicy(logger logr.Logger, scheme *runtime.Scheme, nimbusPolicy *intentv1alpha1.NimbusPolicy) []kubearmorv1.KubeArmorPolicy {
	var ksps []kubearmorv1.KubeArmorPolicy
	for _, nimbusRule := range nimbusPolicy.Spec.NimbusRules {
		if !idpool.IsSupportedId(nimbusRule.ID, "kubearmor") {
			logger.Info("KubeArmor adapter doesn't support this ID", "ID", nimbusRule.ID)
			continue
		}
		actualKsps := buildPolicy(logger, nimbusPolicy, nimbusRule, scheme)
		ksps = append(ksps, actualKsps...)
	}
	return ksps
}

func buildPolicy(logger logr.Logger, np *intentv1alpha1.NimbusPolicy, rule intentv1alpha1.Rule, scheme *runtime.Scheme) []kubearmorv1.KubeArmorPolicy {
	switch rule.ID {
	case idpool.PkgManagerExecution:
		ksp := pkgMgrPolicy(logger, np, rule)
		//Set NimbusPolicy as the owner of the KSP
		if err := ctrl.SetControllerReference(np, ksp, scheme); err != nil {
			logger.Error(err, "failed to set controller reference on policy")
		}
		return []kubearmorv1.KubeArmorPolicy{*ksp}
	default:
		return nil
	}
}
```

The `BuildPolicy` function builds policies if it supports the given SecurityIntent ID (E.g., `pkgMgrs`) in `nimbusRules` of NimbusPolicy. And sets the NimbusPolicy as the parent of generated policies to track the policies created as part of the applied SecurityIntent and its SecurityIntentBinding.

I know you are wondering why `BuildPolicy` returns a list of policies. The reason is that for a particular intent ID, there might be more than one policy. That's why it returns a list instead of just one policy. Here is an [example](https://github.com/5GSEC/nimbus/blob/3e4fbc2d14b56122a36207bdae5ada4b28f7187e/pkg/adapter/idpool/idpool.go#L30).

Also, you may think how does it know about the supported IDs? The adapter has an idpool that contains supported IDs and they make use of it (`idpool.IsSupportedId(id, "kubearmor")`) so let's create the ID pool.

Edit the `idpool.go` file in `/pkg/adapter/idpool` directory as follows:

```go
package idpool

import (
	"slices"
)

const (
	PkgManagerExecution = "pkgMgrs"
)

var IdsSupportedByKubeArmor = []string{
	PkgManagerExecution,
}

func IsSupportedId(id, engine string) bool {
	switch engine {
	case "kubearmor":
		return slices.Contains(IdsSupportedByKubeArmor, id)
	default:
		return false
	}
}
```

and edit the `ksp.go` file in `builder` directory as follows that will construct the actual KubeArmor policies.

```go
package builder

import (
	"io"
	"net/http"
	"strings"

	"github.com/go-logr/logr"
	kubearmorv1 "github.com/kubearmor/KubeArmor/pkg/KubeArmorController/api/security.kubearmor.com/v1"
	"sigs.k8s.io/yaml"

	intentv1alpha1 "github.com/anurag-rajawat/tutorials/nimbus/api/v1alpha1"
)

const (
	ActionEnforce  = "Enforce"
	ActionAudit    = "Audit"
	LabelPartOf    = "app.kubernetes.io/part-of"
	LabelManagedBy = "app.kubernetes.io/managed-by"
)

func pkgMgrPolicy(logger logr.Logger, np *intentv1alpha1.NimbusPolicy, rule intentv1alpha1.Rule) *kubearmorv1.KubeArmorPolicy {
	ksp, err := downloadKsp(
		"https://raw.githubusercontent.com/kubearmor/policy-templates/main/nist/system/ksp-nist-si-4-execute-package-management-process-in-container.yaml",
	)
	if err != nil {
		logger.Error(err, "failed to download KubeArmor policy")
		return nil
	}

	ksp.ObjectMeta.Name = np.Name + "-" + strings.ToLower(rule.ID)
	ksp.ObjectMeta.Namespace = np.Namespace
	ksp.Spec.Selector.MatchLabels = np.Spec.Selector.MatchLabels

	if ksp.Labels == nil {
		ksp.Labels = map[string]string{}
	}
	ksp.Labels[LabelPartOf] = np.Name + "-" + "nimbuspolicy"
	ksp.Labels[LabelManagedBy] = "nimbus-kubearmor"

	if rule.RuleAction == ActionEnforce {
		ksp.Spec.Process.Action = "Block"
	} else {
		ksp.Spec.Process.Action = ActionAudit
	}

	return ksp
}

func downloadKsp(fileUrl string) (*kubearmorv1.KubeArmorPolicy, error) {
	ksp := &kubearmorv1.KubeArmorPolicy{}
	response, err := http.Get(fileUrl)
	if err != nil {
		return nil, err
	}
	defer response.Body.Close()

	data, err := io.ReadAll(response.Body)
	if err != nil {
		return nil, err
	}

	err = yaml.Unmarshal(data, &ksp)
	return ksp, err
}
```

KubeArmor maintains a community-curated policy called [policy-templates](https://github.com/kubearmor/policy-templates) so we're making use of those policies here.

The `pkgMgrPolicy` function just enriches and updates the downloaded KubeArmor policy to match the NimbusPolicy.

Finally, create the execution point aka the main function for the adapter, by editing the `main.go` file as follows:

```go
package main

import (
	"context"

	"github.com/go-logr/logr"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/log/zap"

	"github.com/anurag-rajawat/tutorials/nimbus/adapter/nimbus-kubearmor/manager"
)

func main() {
	ctx := ctrl.SetupSignalHandler()
	logger := setupLogger(ctx)
	manager.Run(ctx)
	<-ctx.Done()
	logger.Info("Shutting down")
	logger.Info("Shutdown complete")
}

func setupLogger(ctx context.Context) logr.Logger {
	ctrl.SetLogger(zap.New())
	logger := ctrl.Log
	ctrl.LoggerInto(ctx, logger)
	return logger
}
```

Here, `ctrl.SetupSignalHandler` establishes a signal handler for the `SIGTERM` and `SIGINT` signals. It returns a context that is cancelled when either of these signals is received. Additionally, the manager is started and provided with the parent context to ensure a graceful shutdown of the adapter and proper resource release.

Let's create some make targets to make our life easier, edit the Makefile as follows:

```makefile
# Image URL to use all building/pushing image targets
IMG ?= nimbus-kubearmor
# Image Tag to use all building/pushing image targets
TAG ?= latest

CONTAINER_TOOL ?= docker
BINARY ?= bin/nimbus-kubearmor

.PHONY: help
help: ## Display this help.
	@awk 'BEGIN {FS = ":.*##"; printf "\nUsage:\n  make \033[36m<target>\033[0m\n"} /^[a-zA-Z_0-9-]+:.*?##/ { printf "  \033[36m%-15s\033[0m %s\n", $$1, $$2 } /^##@/ { printf "\n\033[1m%s\033[0m\n", substr($$0, 5) } ' $(MAKEFILE_LIST)

build: ## Build nimbus-kubearmor executable.
	@go build -ldflags="-s" -o ${BINARY} cmd/main.go

run: build ## Run nimbus-kubearmor.
	@./${BINARY}

.PHONY: image
image: ## Build nimbus-kubearmor container image.
	$(CONTAINER_TOOL) build -t ${IMG}:${TAG} --build-arg VERSION=${TAG} -f ./Dockerfile ../../../

.PHONY: push-image
push-image: ## Push nimbus-kubearmor container image.
	$(CONTAINER_TOOL) push ${IMG}:${TAG}

PLATFORMS ?= linux/arm64,linux/amd64
.PHONY: imagex
imagex: ## Build and push container image for cross-platform support
	sed -e '1 s/\(^FROM\)/FROM --platform=\$$\{BUILDPLATFORM\}/; t' -e ' 1,// s//FROM --platform=\$$\{BUILDPLATFORM\}/' Dockerfile > Dockerfile.cross
	- $(CONTAINER_TOOL) buildx create --name project-v3-builder
	$(CONTAINER_TOOL) buildx use project-v3-builder
	- $(CONTAINER_TOOL) buildx build --push --platform=$(PLATFORMS) --build-arg VERSION=${TAG} --tag ${IMG}:${TAG} -f Dockerfile.cross ../../../ || { $(CONTAINER_TOOL) buildx rm project-v3-builder; rm Dockerfile.cross; exit 1; }
	- $(CONTAINER_TOOL) buildx rm project-v3-builder
	rm Dockerfile.cross
```

Since the adapter depends on KubeArmor, so install it by following [this](https://docs.kubearmor.io/kubearmor/quick-links/deployment_guide) and then start it:

```bash
$ make run
{"level":"info","ts":"2024-08-12T22:13:05+05:30","msg":"Starting manager"}
{"level":"info","ts":"2024-08-12T22:13:05+05:30","msg":"Started manager"}
{"level":"info","ts":"2024-08-12T22:13:05+05:30","msg":"Starting NimbusPolicy watcher"}
{"level":"info","ts":"2024-08-12T22:13:05+05:30","msg":"Started NimbusPolicy watcher"}
```

Don't forget to start the nimbus operator otherwise creating intent and binding would have no effect.

Let's apply the following SecurityIntent and SecurityIntentBinding:

```yaml
apiVersion: intent.security.nimbus.com/v1alpha1
kind: SecurityIntent
metadata:
  name: package-mgrs
  annotations:
    intent.nimbus.io/title: Package Manager Execution Prevention
    # Severity should be a standard threat severity level (e.g., Low, Medium, High, Critical)
    intent.nimbus.io/severity: Medium
    # Description should clearly explain the intent and its security implications
    intent.nimbus.io/description: |
      This SecurityIntent aims to prevent adversaries from exploiting
      third-party software suites (administration, monitoring, deployment tools)
      within the network to achieve lateral movement. It enforces restrictions
      on the execution of package managers.
spec:
  intent:
    action: Enforce
    id: pkgMgrs
---
apiVersion: intent.security.nimbus.com/v1alpha1
kind: SecurityIntentBinding
metadata:
  name: package-mgrs-binding
spec:
  # Names of SecurityIntents to be applied
  intents:
    - name: package-mgrs # Reference the intended SecurityIntent resource
  selector:
    matchLabels:
      env: prod
      app: web
```

```bash
$ k apply -f pkg-mgrs-intent-and-binding.yaml
securityintent.intent.security.nimbus.com/package-mgrs created
securityintentbinding.intent.security.nimbus.com/package-mgrs-binding created
```

Check the adapter logs you should have a similar output:

```bash
$ make run
{"level":"info","ts":"2024-08-12T22:13:05+05:30","msg":"Starting manager"}
{"level":"info","ts":"2024-08-12T22:13:05+05:30","msg":"Started manager"}
{"level":"info","ts":"2024-08-12T22:13:05+05:30","msg":"Starting NimbusPolicy watcher"}
{"level":"info","ts":"2024-08-12T22:13:05+05:30","msg":"Started NimbusPolicy watcher"}
{"level":"info","ts":"2024-08-12T22:18:57+05:30","msg":"Successfully created KubeArmorPolicy","kubeArmorPolicy.Name":"package-mgrs-binding-pkgmgrs","kubeArmorPolicy.Namespace":"default"}
```

Adapters says it created a KubeArmorPolicy, let's verify whether the policy is created:

```bash
$ k get ksp
NAME                           AGE
package-mgrs-binding-pkgmgrs   2m13s
```

Check the generated KubeArmor policy:

```bash
$ k get ksp package-mgrs-binding-pkgmgrs -o yaml
```

```yaml
apiVersion: security.kubearmor.com/v1
kind: KubeArmorPolicy
metadata:
  creationTimestamp: "2024-08-12T22:18:57Z"
  generation: 1
  labels:
    app.kubernetes.io/managed-by: nimbus-kubearmor
    app.kubernetes.io/part-of: package-mgrs-binding-nimbuspolicy
  name: package-mgrs-binding-pkgmgrs
  namespace: default
  ownerReferences:
  - apiVersion: intent.security.nimbus.com/v1alpha1
    blockOwnerDeletion: true
    controller: true
    kind: NimbusPolicy
    name: package-mgrs-binding
    uid: 6c364de5-9567-45f4-9ba1-2a70a64d6ae9
  resourceVersion: "56395"
  uid: 9472ff45-2eef-4816-856c-863395761efa
spec:
  capabilities: {}
  file: {}
  message: Alert! Execution of package management process inside container is detected
  network: {}
  process:
    action: Block
    matchPaths:
    - path: /usr/bin/apt
    - path: /usr/bin/apt-get
    - path: /bin/apt-get
    - path: /bin/apt
    - path: /sbin/apk
    - path: /usr/bin/dpkg
    - path: /bin/dpkg
    - path: /usr/bin/gdebi
    - path: /bin/gdebi
    - path: /usr/bin/make
    - path: /bin/make
    - path: /usr/bin/yum
    - path: /bin/yum
    - path: /usr/bin/rpm
    - path: /bin/rpm
    - path: /usr/bin/dnf
    - path: /bin/dnf
    - path: /usr/bin/pacman
    - path: /usr/sbin/pacman
    - path: /bin/pacman
    - path: /sbin/pacman
    - path: /usr/bin/makepkg
    - path: /usr/sbin/makepkg
    - path: /bin/makepkg
    - path: /sbin/makepkg
    - path: /usr/bin/yaourt
    - path: /usr/sbin/yaourt
    - path: /bin/yaourt
    - path: /sbin/yaourt
    - path: /usr/bin/zypper
    - path: /bin/zypper
    severity: 5
  selector:
    matchLabels:
      app: web
      env: prod
  syscalls: {}
  tags:
  - NIST
  - CM-7(5)
  - SI-4
  - Package Manager
```

So this is the generated KubeArmor Policy from the applied SecurityIntent and SecurityIntentBinding.

> Actual Enforcement depends on the corresponding security engine, Nimbus is not responsible for security engine specific policies enforcement.

Let's delete the applied SecurityIntent and SecurityIntentBinding:

```bash
$ k delete -f pkg-mgrs-intent-and-binding.yaml
securityintent.intent.security.nimbus.com "package-mgrs" deleted
securityintentbinding.intent.security.nimbus.com "package-mgrs-binding" deleted
```

Let's confirm if the policy has been deleted:

```bash
$ k get ksp
No resources found in default namespace.
```

Yeah, it worked.

So are we done?

No, there are some issues:

* How would an end-user know about the created policies?
    
* What will happen if a policy gets modified or deleted?
    

## NimbusPolicy Status

Let's tackle the first issue.

Here we'll make use of NimbusPolicy's status subresource. Remember the `POLICIES` and `policiesName` field?

That's what we're going to use so that when an end-user describes or gets NimbusPolicy, they will know about the created policies.

Edit the `ksp.go` file in the `manager` directory as follows:

```go
...
func (m *manager) createOrUpdatePolicies(ctx context.Context, nimbusPolicy *intentv1alpha1.NimbusPolicy) {
    ...
	for idx := range ksps {
		ksp := &ksps[idx]

		var modified bool
		var existingKsp kubearmorv1.KubeArmorPolicy
		...
		if err != nil {
			if apierrors.IsNotFound(err) {
				...
				modified = true
				m.logger.Info("Successfully created KubeArmorPolicy", "kubeArmorPolicy.Name", ksp.Name, "kubeArmorPolicy.Namespace", ksp.Namespace)
			}
		} else {
            ...
			modified = true
			m.logger.Info("Configured KubeArmorPolicy", "kubeArmorPolicy.Name", ksp.Name, "kubeArmorPolicy.Namespace", ksp.Namespace)
		}

		if modified {
			nimbusPolicy.Status.CountOfPolicies++
			nimbusPolicy.Status.GeneratedPoliciesName = append(nimbusPolicy.Status.GeneratedPoliciesName, ksp.Name)
			if err = m.k8sClient.Status().Update(ctx, nimbusPolicy); err != nil {
				m.logger.Error(err, "failed to update KubeArmorPolicies info in NimbusPolicy")
			}
		}
	}
}
```

Let's re-run and re-apply the intent and binding and check the status of NimbusPolicy:

```bash
# From pkg/adapter/nimbus-kubearmor directory
$ make run
...
...
{"level":"info","ts":"2024-08-13T20:50:53+05:30","msg":"Successfully created KubeArmorPolicy","kubeArmorPolicy.Name":"package-mgrs-binding-pkgmgrs","kubeArmorPolicy.Namespace":"default"}
{"level":"error","ts":"2024-08-13T20:50:53+05:30","msg":"failed to update KubeArmorPolicies info in NimbusPolicy","error":"Operation cannot be fulfilled on nimbuspolicies.intent.security.nimbus.com \"package-mgrs-binding\": the object has been modified; please apply your changes to the latest version and try again","stacktrace":"github.com/anurag-rajawat/tutorials/nimbus/adapter/nimbus-kubearmor/manager.(*manager).createOrUpdatePolicies\n\t/Users/anurag/Projects/oss/tutorials/nimbus/pkg/adapter/nimbus-kubearmor/manager/ksp.go:85\ngithub.com/anurag-rajawat/tutorials/nimbus/adapter/nimbus-kubearmor/manager.(*manager).managePolicies\n\t/Users/anurag/Projects/oss/tutorials/nimbus/pkg/adapter/nimbus-kubearmor/manager/ksp.go:21\ngithub.com/anurag-rajawat/tutorials/nimbus/adapter/nimbus-kubearmor/manager.(*manager).run.func2\n\t/Users/anurag/Projects/oss/tutorials/nimbus/pkg/adapter/nimbus-kubearmor/manager/manager.go:41"}
```

The adapter failed to update the status, and the error says `"Operation cannot be fulfilled on` [`nimbuspolicies.intent.security.nimbus.com`](http://nimbuspolicies.intent.security.nimbus.com) `"package-mgrs-binding": the object has been modified; please apply your changes to the latest version and try again"`. So let's do the same.

Edit the `ksp.go` file in the `manager` directory as follows:

```go
func (m *manager) createOrUpdatePolicies(ctx context.Context, nimbusPolicy *intentv1alpha1.NimbusPolicy) {
	...
	for idx := range ksps {
		...
		if modified {
			var latestNimbusPolicy intentv1alpha1.NimbusPolicy
			if err = m.k8sClient.Get(ctx, types.NamespacedName{Name: nimbusPolicy.Name, Namespace: nimbusPolicy.Namespace}, &latestNimbusPolicy); err != nil {
				m.logger.Error(err, "failed to get existing NimbusPolicy", "nimbusPolicy.name", nimbusPolicy.Name, "nimbusPolicy.Namespace", nimbusPolicy.Namespace)
				return
			}
			latestNimbusPolicy.Status.CountOfPolicies++
			latestNimbusPolicy.Status.GeneratedPoliciesName = append(latestNimbusPolicy.Status.GeneratedPoliciesName, ksp.Name)
			if err = m.k8sClient.Status().Update(ctx, &latestNimbusPolicy); err != nil {
				m.logger.Error(err, "failed to update KubeArmorPolicies info in NimbusPolicy")
			}
		}
	}
}
```

this time we're making a get request to get the latest NimbusPolicy and applying changes against it.

Let's re-run and check whether it is working this time:

```bash
$ make run
...
...
{"level":"info","ts":"2024-08-13T20:58:59+05:30","msg":"Successfully created KubeArmorPolicy","kubeArmorPolicy.Name":"package-mgrs-binding-pkgmgrs","kubeArmorPolicy.Namespace":"default"}
```

check the NimbusPolicy:

```bash
$ k get np
NAME                   STATUS    AGE   POLICIES
package-mgrs-binding   Created   63s   1
 
$ k describe np package-mgrs-binding
Name:         package-mgrs-binding
Namespace:    default
...
...
Status:
  Policies:  1
  Policies Name:
    package-mgrs-binding-pkgmgrs
  Status:  Created
Events:    <none>
```

Let's restart the adapter and check the status again:

```bash
$ k get np
NAME                   STATUS    AGE   POLICIES
package-mgrs-binding   Created   30s   2
 
$ k describe np package-mgrs-binding
Name:         package-mgrs-binding
Namespace:    default
...
...
Status:
  Policies:  2
  Policies Name:
    package-mgrs-binding-pkgmgrs
    package-mgrs-binding-pkgmgrs
  Status:  Created
Events:    <none>
```

this time the status contains duplicate entries. Let's restart one more time.

```bash
$ k get np
NAME                   STATUS    AGE    POLICIES
package-mgrs-binding   Created   2m3s   3

$ k describe np package-mgrs-binding
Name:         package-mgrs-binding
Namespace:    default
...
...
Status:
  Policies:  3
  Policies Name:
    package-mgrs-binding-pkgmgrs
    package-mgrs-binding-pkgmgrs
    package-mgrs-binding-pkgmgrs
  Status:  Created
Events:    <none>
```

Now the count is 3 and there is one more duplicate entry, why?

Because we're updating the count and policy names without checking whether a policy is already there. So let's fix it, edit the `ksp.go` file in the `manager` directory as follows:

```go
...
func (m *manager) createOrUpdatePolicies(ctx context.Context, nimbusPolicy *intentv1alpha1.NimbusPolicy) {
	...
	for idx := range ksps {
        ...
        ...
		if modified {
            ...
            ...
			if !slices.Contains(latestNimbusPolicy.Status.GeneratedPoliciesName, ksp.Name) {
				latestNimbusPolicy.Status.CountOfPolicies++
				latestNimbusPolicy.Status.GeneratedPoliciesName = append(latestNimbusPolicy.Status.GeneratedPoliciesName, ksp.Name)
				if err = m.k8sClient.Status().Update(ctx, &latestNimbusPolicy); err != nil {
					m.logger.Error(err, "failed to update KubeArmorPolicies info in NimbusPolicy")
				}
			}
		}
	}
}
...
```

We added a check to update the status only if it doesn't already have the current policy. Now it works as expected; we get the count and names of the created policies without duplicates.

## Policies Reconciliation

Let's tackle the second issue. Here we'll follow the same pattern that a Kubernetes controller follows, i.e., reconciliation.

The manager will also reconcile policies if they don't match the desired state. To do so, we need a watcher for KubeArmorPolicy. So let's set it up:

Create a new file `ksp.go` file in the `watcher` directory as follows:

```go
package watcher

import (
	"context"

	kubearmorv1 "github.com/kubearmor/KubeArmor/pkg/KubeArmorController/api/security.kubearmor.com/v1"
	"k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
	"k8s.io/apimachinery/pkg/runtime/schema"
	k8sscheme "k8s.io/client-go/kubernetes/scheme"
	"k8s.io/client-go/tools/cache"
)

func (w *Watcher) getKspInformer() cache.SharedIndexInformer {
	kspGvr := schema.GroupVersionResource{
		Group:    "security.kubearmor.com",
		Version:  "v1",
		Resource: "kubearmorpolicies",
	}
	return w.factory.ForResource(kspGvr).Informer()
}

func (w *Watcher) watchKsp(ctx context.Context, kspChan chan *kubearmorv1.KubeArmorPolicy) {
	informer := w.getKspInformer()
	_, err := informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
		UpdateFunc: func(oldObj, newObj interface{}) {
			oldU := oldObj.(*unstructured.Unstructured)
			newU := newObj.(*unstructured.Unstructured)

			if oldU.GetGeneration() != newU.GetGeneration() {
				w.logger.Info("KubeArmorPolicy modified", "kubeArmorPolicy.Name", newU.GetName(), "kubeArmorPolicy.Namespace", newU.GetNamespace())
				kspChan <- w.getDecodedKsp(newU)
			}
		},
		DeleteFunc: func(obj interface{}) {
			u := obj.(*unstructured.Unstructured)
			w.logger.Info("KubeArmorPolicy deleted", "kubeArmorPolicy.Name", u.GetName(), "kubeArmorPolicy.Namespace", u.GetNamespace())
			kspChan <- w.getDecodedKsp(u)
		},
	})
	if err != nil {
		w.logger.Error(err, "failed to add KubeArmorPolicy event handler")
		return
	}

	w.logger.Info("Started KubeArmorPolicy watcher")
	informer.Run(ctx.Done())

	close(kspChan)
	w.logger.Info("Stopped KubeArmorPolicy watcher")
}

func (w *Watcher) RunKspWatcher(ctx context.Context, kspChan chan *kubearmorv1.KubeArmorPolicy) {
	w.logger.Info("Starting KubeArmorPolicy watcher")
	w.watchKsp(ctx, kspChan)
}

func (w *Watcher) getDecodedKsp(obj *unstructured.Unstructured) *kubearmorv1.KubeArmorPolicy {
	bytes, err := obj.MarshalJSON()
	if err != nil {
		w.logger.Error(err, "failed to marshal KubeArmorPolicy")
		return nil
	}

	ksp := &kubearmorv1.KubeArmorPolicy{}
	decoder := k8sscheme.Codecs.UniversalDeserializer()
	_, _, err = decoder.Decode(bytes, nil, ksp)
	if err != nil {
		w.logger.Error(err, "failed to decode KubeArmorPolicy")
		return nil
	}
	return ksp
}
```

Let's manager know about this watcher:

```go
func (m *manager) run(ctx context.Context) {
	m.logger.Info("Starting manager")
    ...
    ...
	kspChan := make(chan *kubearmorv1.KubeArmorPolicy)

	watchr := watcher.NewWatcher(ctx)

	m.wg.Add(1)
	go func() {
		defer m.wg.Done()
		watchr.RunNpWatcher(ctx, nimbusPolicyChan)
	}()

	m.wg.Add(1)
	go func() {
		defer m.wg.Done()
		watchr.RunKspWatcher(ctx, kspChan)
	}()

	m.wg.Add(1)
	go func() {
		defer m.wg.Done()
		m.managePolicies(ctx, nimbusPolicyChan, kspChan)
	}()

	m.logger.Info("Started manager")
	m.wg.Wait()
}
```

Don't forget to update the NimbusPolicy watcher's method name, in `watcher.go` file

```go
package watcher
...
...
func (w *Watcher) RunNpWatcher(ctx context.Context, nimbusPolicyChan chan *intentv1alpha1.NimbusPolicy) {
	w.logger.Info("Starting NimbusPolicy watcher")
	w.watchNimbusPolicies(ctx, nimbusPolicyChan)
}
...
...
```

Update the `ksp.go` file in `manager` directory as follows, to reconcile the deleted or edited policies:

```go
package manager
...
...
func (m *manager) managePolicies(ctx context.Context, nimbusPolicyChan chan *intentv1alpha1.NimbusPolicy, kspChan chan *kubearmorv1.KubeArmorPolicy) {
	for {
		select {
		case <-ctx.Done():
			return
		case nimbusPolicy := <-nimbusPolicyChan:
			m.createOrUpdatePolicies(ctx, nimbusPolicy)
		case ksp := <-kspChan:
			m.reconcileKsp(ctx, ksp.Name, ksp.Namespace)
		}
	}
}

func (m *manager) reconcileKsp(ctx context.Context, name, namespace string) {
	npName := extractNpName(name)
	nimbusPolicy := &intentv1alpha1.NimbusPolicy{}
	if err := m.k8sClient.Get(ctx, types.NamespacedName{Name: npName, Namespace: namespace}, nimbusPolicy); err != nil {
		m.logger.Error(err, "failed to get NimbusPolicy", "nimbusPolicy.Name", nimbusPolicy.Name, "nimbusPolicy.Namespace", namespace)
		return
	}
	m.createOrUpdatePolicies(ctx, nimbusPolicy)
}

func extractNpName(name string) string {
	words := strings.Split(name, "-")
	return strings.Join(words[:len(words)-1], "-")
}
...
...
```

Let's run and check the reconciliation:

```bash
$ make run
{"level":"info","ts":"2024-08-13T21:56:36+05:30","msg":"Starting manager"}
{"level":"info","ts":"2024-08-13T21:56:36+05:30","msg":"Started manager"}
{"level":"info","ts":"2024-08-13T21:56:36+05:30","msg":"Starting NimbusPolicy watcher"}
{"level":"info","ts":"2024-08-13T21:56:36+05:30","msg":"Starting KubeArmorPolicy watcher"}
{"level":"info","ts":"2024-08-13T21:56:36+05:30","msg":"Started NimbusPolicy watcher"}
{"level":"info","ts":"2024-08-13T21:56:36+05:30","msg":"Started KubeArmorPolicy watcher"}
{"level":"info","ts":"2024-08-13T21:58:11+05:30","msg":"Successfully created KubeArmorPolicy","kubeArmorPolicy.Name":"package-mgrs-binding-pkgmgrs","kubeArmorPolicy.Namespace":"default"}
```

try deleting the created KubeArmor policy:

```bash
$ k delete ksp package-mgrs-binding-pkgmgrs
kubearmorpolicy.security.kubearmor.com "package-mgrs-binding-pkgmgrs" deleted

$ k get ksp
NAME                           AGE
package-mgrs-binding-pkgmgrs   6s
```

check the adapter logs:

```bash
...
...
{"level":"info","ts":"2024-08-13T21:58:51+05:30","msg":"KubeArmorPolicy deleted","kubeArmorPolicy.Name":"package-mgrs-binding-pkgmgrs","kubeArmorPolicy.Namespace":"default"}
{"level":"info","ts":"2024-08-13T21:58:51+05:30","msg":"Successfully created KubeArmorPolicy","kubeArmorPolicy.Name":"package-mgrs-binding-pkgmgrs","kubeArmorPolicy.Namespace":"default"}
```

modify the policy, see if the changes are discarded, and check the adapter logs:

```bash
...
...
{"level":"info","ts":"2024-08-13T22:00:50+05:30","msg":"KubeArmorPolicy modified","kubeArmorPolicy.Name":"package-mgrs-binding-pkgmgrs","kubeArmorPolicy.Namespace":"default"}
{"level":"info","ts":"2024-08-13T22:00:51+05:30","msg":"Configured KubeArmorPolicy","kubeArmorPolicy.Name":"package-mgrs-binding-pkgmgrs","kubeArmorPolicy.Namespace":"default"}
{"level":"info","ts":"2024-08-13T22:00:51+05:30","msg":"KubeArmorPolicy modified","kubeArmorPolicy.Name":"package-mgrs-binding-pkgmgrs","kubeArmorPolicy.Namespace":"default"}
{"level":"info","ts":"2024-08-13T22:00:51+05:30","msg":"Configured KubeArmorPolicy","kubeArmorPolicy.Name":"package-mgrs-binding-pkgmgrs","kubeArmorPolicy.Namespace":"default"}
```

looks like reconciliation is working as intended.

Let's delete the applied SecurityIntent and SecurityIntentBinding:

```bash
$ k delete -f pkg-mgrs-intent-and-binding.yaml
securityintent.intent.security.nimbus.com "package-mgrs" deleted
securityintentbinding.intent.security.nimbus.com "package-mgrs-binding" deleted
```

but on deletion, the adapter reports an error as follows:

```bash
...
...
{"level":"info","ts":"2024-08-13T22:02:26+05:30","msg":"KubeArmorPolicy deleted","kubeArmorPolicy.Name":"package-mgrs-binding-pkgmgrs","kubeArmorPolicy.Namespace":"default"}
{"level":"error","ts":"2024-08-13T22:02:26+05:30","msg":"failed to get NimbusPolicy","nimbusPolicy.Name":"","nimbusPolicy.Namespace":"default","error":"nimbuspolicies.intent.security.nimbus.com \"package-mgrs-binding\" not found","stacktrace":"github.com/anurag-rajawat/tutorials/nimbus/adapter/nimbus-kubearmor/manager.(*manager).reconcileKsp\n\t/Users/anurag/Projects/oss/tutorials/nimbus/pkg/adapter/nimbus-kubearmor/manager/ksp.go:36\ngithub.com/anurag-rajawat/tutorials/nimbus/adapter/nimbus-kubearmor/manager.(*manager).managePolicies\n\t/Users/anurag/Projects/oss/tutorials/nimbus/pkg/adapter/nimbus-kubearmor/manager/ksp.go:27\ngithub.com/anurag-rajawat/tutorials/nimbus/adapter/nimbus-kubearmor/manager.(*manager).run.func3\n\t/Users/anurag/Projects/oss/tutorials/nimbus/pkg/adapter/nimbus-kubearmor/manager/manager.go:50"}
```

the error says, `"failed to get NimbusPolicy","`[`nimbusPolicy.Name`](http://nimbusPolicy.Name)`":"","nimbusPolicy.Namespace":"default","error":"`[`nimbuspolicies.intent.security.nimbus.com`](http://nimbuspolicies.intent.security.nimbus.com) `"package-mgrs-binding" not found"`.

Why did it try to retrieve NimbusPolicy when SecurityIntent and SecurityIntentBinding were deleted? Do you know why?

Because the manager is responsible for watching both KubeArmor and Nimbus policies. When it detects the deletion of either, it initiates a reconciliation process. However, upon detecting the KubeArmor policy deletion, the manager attempts to recreate the deleted policy. This action triggers a retrieval of the corresponding NimbusPolicy, which has also been deleted, resulting in a `not found` error.

Let's fix it, edit the `ksp.go` file in `watcher` directory as follows:

```go
package watcher
...
func (w *Watcher) watchKsp(ctx context.Context, kspChan chan *kubearmorv1.KubeArmorPolicy) {
	...
	_, err := informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
		UpdateFunc: func(oldObj, newObj interface{}) {
			oldU := oldObj.(*unstructured.Unstructured)
			newU := newObj.(*unstructured.Unstructured)

			if w.isOrphan(ctx, newU.GetOwnerReferences(), "NimbusPolicy", newU.GetNamespace()) {
				w.logger.Info("Ignoring orphan Policy", "name", oldU.GetName(), "namespace", oldU.GetNamespace(), "operation", "update")
				return
			}
			...
		},
		DeleteFunc: func(obj interface{}) {
			u := obj.(*unstructured.Unstructured)
			if w.isOrphan(ctx, u.GetOwnerReferences(), "NimbusPolicy", u.GetNamespace()) {
				w.logger.Info("Ignoring orphan Policy", "name", u.GetName(), "namespace", u.GetNamespace(), "operation", "delete")
				return
			}
            ...
		},
	})
    ...
}
```

Here we added a check to ensure managers only receive relevant policies on their designated channels. Since Nimbus suite policies include owner references, we leverage this information to differentiate between relevant and irrelevant policies.

Also, update the NimbusPolicy watcher to filter the relevant NimbusPolicies by following the same pattern as the KubeArmor policy watcher. Edit the `watcher.go` file as follows:

```go
package watcher

import (
	"context"
	"time"

	"github.com/go-logr/logr"
	apierrors "k8s.io/apimachinery/pkg/api/errors"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/dynamic/dynamicinformer"
	k8sscheme "k8s.io/client-go/kubernetes/scheme"
	"k8s.io/client-go/tools/cache"
	"sigs.k8s.io/controller-runtime/pkg/log"

	intentv1alpha1 "github.com/anurag-rajawat/tutorials/nimbus/api/v1alpha1"
	"github.com/anurag-rajawat/tutorials/nimbus/pkg/adapter/k8s"
)

...
func (w *Watcher) watchNimbusPolicies(ctx context.Context, nimbusPolicyChan chan *intentv1alpha1.NimbusPolicy) {
	...
	handlers := cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			u := obj.(*unstructured.Unstructured)
			if w.isOrphan(ctx, u.GetOwnerReferences(), "SecurityIntentBinding", u.GetNamespace()) {
				w.logger.V(4).Info("Ignoring orphan Policy", "name", u.GetName(), "namespace", u.GetNamespace(), "operation", "add")
				return
			}
			...
		},
		UpdateFunc: func(oldObj, newObj interface{}) {
			oldU := oldObj.(*unstructured.Unstructured)
			newU := newObj.(*unstructured.Unstructured)
			if w.isOrphan(ctx, newU.GetOwnerReferences(), "SecurityIntentBinding", newU.GetNamespace()) {
				w.logger.V(4).Info("Ignoring orphan Policy", "name", oldU.GetName(), "namespace", oldU.GetNamespace(), "operation", "update")
				return
			}
            ...
		},
	}
    ...
}

func (w *Watcher) isOrphan(ctx context.Context, ownerReferences []metav1.OwnerReference, owner string, namespace string) bool {
	if len(ownerReferences) == 0 {
		return true
	}

	for _, ownerReference := range ownerReferences {
		if ownerReference.Kind == owner {
			return !w.isOwnerExist(ctx, ownerReference, namespace)
		}
	}

	return true
}

func (w *Watcher) isOwnerExist(ctx context.Context, ownerReference metav1.OwnerReference, namespace string) bool {
	switch ownerReference.Kind {
	case "SecurityIntentBinding":
		sibGvr := schema.GroupVersionResource{
			Group:    "intent.security.nimbus.com",
			Version:  "v1alpha1",
			Resource: "securityintentbindings",
		}
		return w.isExist(ctx, sibGvr, ownerReference.Name, namespace)
	case "NimbusPolicy":
		npGvr := schema.GroupVersionResource{
			Group:    "intent.security.nimbus.com",
			Version:  "v1alpha1",
			Resource: "nimbuspolicies",
		}
		return w.isExist(ctx, npGvr, ownerReference.Name, namespace)
	default:
		return false
	}
}

func (w *Watcher) isExist(ctx context.Context, gvr schema.GroupVersionResource, name, namespace string) bool {
	_, err := w.dynamicClient.Resource(gvr).Namespace(namespace).Get(ctx, name, metav1.GetOptions{})
	if err != nil {
		if apierrors.IsNotFound(err) {
			return false
		}
		w.logger.Error(err, "failed to get Policy", "name", name, "namespace", namespace)
		return false
	}
	return true
}
...
```

Let's verify the changes:

```bash
$ k apply -f pkg-mgrs-intent-and-binding.yaml
securityintent.intent.security.nimbus.com/package-mgrs created
securityintentbinding.intent.security.nimbus.com/package-mgrs-binding created

$ k delete -f pkg-mgrs-intent-and-binding.yaml
securityintent.intent.security.nimbus.com "package-mgrs" deleted
securityintentbinding.intent.security.nimbus.com "package-mgrs-binding" deleted

$ k get si,sib,np,ksp
No resources found

$ make run
...
{"level":"info","ts":"2024-08-13T23:03:07+05:30","msg":"Successfully created KubeArmorPolicy","kubeArmorPolicy.Name":"package-mgrs-binding-pkgmgrs","kubeArmorPolicy.Namespace":"default"}
{"level":"info","ts":"2024-08-13T23:03:14+05:30","msg":"Ignoring orphan Policy","name":"package-mgrs-binding-pkgmgrs","namespace":"default","operation":"delete"}
```

Creating a new adapter will follow a similar process to the one just outlined. I hope you now understand the steps involved in adapter creation. A key benefit of this model is the ability to develop your own commercial adapter. You can check other adapters [here](https://github.com/5GSEC/nimbus/tree/3e4fbc2d14b56122a36207bdae5ada4b28f7187e/pkg/adapter).

That's it for this part take your time to understand this information. Stay tuned!

You can find the complete code [here](https://github.com/anurag-rajawat/tutorials/tree/main/nimbus). Please feel free to comment or criticize :)

# Summary

> This part of the tutorial focuses on implementing SecurityEngine adapters for translating SecurityIntents into native security policies for target engines. The content covers the architecture, components, and message sequences of adapters. Detailed implementation steps are provided for creating an adapter for KubeArmor, including setting up watchers, managing policies, building security policies, handling status updates, and ensuring policy reconciliation. The tutorial also addresses potential issues and suggests solutions while verifying the implementation with practical examples. The goal is to ensure consistent security enforcement through Kubernetes-native CRD mechanisms.

# **References**

* [**5GSec Nimbus**](https://github.com/5GSEC/nimbus)
    
* [**Programming Kubernetes**](https://www.oreilly.com/library/view/programming-kubernetes/9781492047094/)
    
* [**Kubebuilder book**](https://www.kubebuilder.io/introduction)