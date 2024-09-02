---
title: "Building a Real-world Kubernetes Operator: Part - 2"
datePublished: Mon Sep 02 2024 02:30:21 GMT+0000 (Coordinated Universal Time)
cuid: cm0kdv5ep000308mk3unwfox0
slug: building-a-real-world-kubernetes-operator-part-2
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1721030184285/1899efe9-1fec-4f71-b9f5-71b3a41c38b5.png
tags: kubernetes, kubernetes-operators

---

# Introduction

We'll implement our Custom Resources (CR) types and their controllers in this part.

# Implementation

## Types

### SecurityIntent

Let's first start with `SecurityIntent` CR, the `securityintent_types.go` file in `api/v1alpha1` will be similar to the following:

```go

package v1alpha1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// EDIT THIS FILE!  THIS IS SCAFFOLDING FOR YOU TO OWN!
// NOTE: json tags are required.  Any new fields you add must have json tags for the fields to be serialized.

// SecurityIntentSpec defines the desired state of SecurityIntent
type SecurityIntentSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "make" to regenerate code after modifying this file

	// Foo is an example field of SecurityIntent. Edit securityintent_types.go to remove/update
	Foo string `json:"foo,omitempty"`
}

// SecurityIntentStatus defines the observed state of SecurityIntent
type SecurityIntentStatus struct {
	// INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
	// Important: Run "make" to regenerate code after modifying this file
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:resource:scope=Cluster

// SecurityIntent is the Schema for the securityintents API
type SecurityIntent struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   SecurityIntentSpec   `json:"spec,omitempty"`
	Status SecurityIntentStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true

// SecurityIntentList contains a list of SecurityIntent
type SecurityIntentList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []SecurityIntent `json:"items"`
}

func init() {
	SchemeBuilder.Register(&SecurityIntent{}, &SecurityIntentList{})
}
```

Most of the things are already clear in their comments, however, there are some special things such as:

```go
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:resource:scope=Cluster
```

These are special comments known as Go Markers, to be precise [Kubebuilder Markers](https://book.kubebuilder.io/reference/markers.html). Code generators use them to generate code and configurations.

* The first marker is used to expose the top-level type through the API.
    
* The second marker is used to automatically generate a [status subresource](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#status-subresource). It is used to show the current status of a resource if needed.
    
* The last marker is used to define the scope of CR.
    

Now let's add the fields we want to our `SecurityIntent` CR in the `SecurityIntentSpec` struct. I'll only show the added code and remove unnecessary comments for brevity and space.

```go
// Intent defines the high-level desired intent.
type Intent struct {
	// ID is predefined in adapter ID pool.
	// Used by security engines to generate corresponding security policies.
	//+kubebuilder:validation:Pattern:="^[a-zA-Z0-9]*$"
	ID string `json:"id"`

	// Action defines how the intent will be enforced.
    // Valid actions are "Audit" and "Enforce".
	Action string `json:"action"`

	// Tags are additional metadata for categorization and grouping of intents.
	// Facilitates searching, filtering, and management of security policies.
	Tags []string `json:"tags,omitempty"`
    
    // Params are key-value pairs that allows fine-tuning of intents to specific
	// requirements.
	Params map[string][]string `json:"params,omitempty"`
}

// SecurityIntentSpec defines the desired state of SecurityIntent
type SecurityIntentSpec struct {
	Intent Intent `json:"intent"`
}
```

The `//+kubebuilder:validation:Pattern:="^[a-zA-Z0-9]*$"` marker is a validation marker. It means that the value of the `ID` field must match the given regular expression, i.e., `ID` must be camelCase and may contain digits from 0-9.

### SecurityIntentBinding

As you may have already guessed the `securityintentbinding_types.go` file is similar to `securityintent_types.go` one. Anyway, let's add our required fields in `SecurityIntentBindingSpec` struct just like we did for `SecurityIntent` CR.

```go
// MatchIntent represents an intent definition.
type MatchIntent struct {
	Name string `json:"name"`
}

// WorkloadSelector defines a selector for workloads based on labels.
type WorkloadSelector struct {
	MatchLabels map[string]string `json:"matchLabels"`
}

// SecurityIntentBindingSpec defines the desired state of SecurityIntentBinding
type SecurityIntentBindingSpec struct {
	Intents  []MatchIntent    `json:"intents"`
	Selector WorkloadSelector `json:"selector"`
}
```

Remember from the Nimbus design that SecurityIntentBinding is used to apply SecurityIntents to namespace-level resources, like pods. It's similar to Kubernetes Role and RoleBinding. We might need to bind multiple intents to a workload, which is why we defined intents as a slice (`Intents []MatchIntent`).

`Selector` is self-explanatory. Right now, it only supports labels, but we might add support for operator patterns in the future.

### NimbusPolicy

Do you recall Nimbus's goals? One of its key objectives is to be **generic and independent** of any specific security engine. To achieve this, we introduced this intermediary resource. This resource will hold the **generic representation** of bound SecurityIntent(s) and its corresponding SecurityIntentBinding within a designated namespace.

Since the structure of the `nimbuspolicy_types.go` file in the `api/v1alpha1/` directory is similar to previous ones, we'll directly add the required types there.

```go
// Rule defines a single rule within a NimbusPolicySpec
type Rule struct {
	// ID is a unique identifier for the rule, used by security engine adapters.
	ID string `json:"id"`

	// RuleAction specifies the action to be taken when the rule matches.
	RuleAction string `json:"action"`

	// Params is an optional map of parameters associated with the rule.
	Params map[string][]string `json:"params,omitempty"`
}

// NimbusPolicySpec defines the desired state of NimbusPolicy
type NimbusPolicySpec struct {
	// NimbusRules is a list of rules that define the policy.
	NimbusRules []Rule `json:"rules"`

	// Selector specifies the workload resources that the policy applies to.
	Selector WorkloadSelector `json:"selector"`
}
```

Let's generate the code and Custom Resource Definitions manifests by executing:

```bash
make manifests generate
```

You can check the generated code in the `zz_generated.deepcopy.go` file located in the `api/v1alpha1` directory, and the CRDs in the `config/crd/bases` directory.

Let's try out our SecurityIntent and SecurityIntentBinding, save the following in `pkg-mgrs-intent-and-binding.yaml` the file:

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

Apply it:

```bash
$ k apply -f pkg-mgrs-intent-and-binding.yaml
resource mapping not found for name: "package-mgrs" namespace: "" from "pkg-mgrs-intent-and-binding.yaml": no matches for kind "SecurityIntent" in version "intent.security.nimbus.com/v1alpha1"
ensure CRDs are installed first
resource mapping not found for name: "package-mgrs-binding" namespace: "" from "pkg-mgrs-intent-and-binding.yaml": no matches for kind "SecurityIntentBinding" in version "intent.security.nimbus.com/v1alpha1"
ensure CRDs are installed first
```

We get an error while applying. Do you know why? (Hint: Read the error message carefully.)

If you don't know, let me tell you. We generated the manifests but didn't install them, so Kubernetes didn't recognize them and rejected the creation.

Let's install their definitions so Kubernetes will know about them. Run the following from the root directory of our project:

```bash
make install
```

Start our operator:

```bash
$ make run
/Users/anurag/Projects/oss/tutorials/nimbus/bin/controller-gen-v0.15.0 rbac:roleName=manager-role crd webhook paths="./..." output:crd:artifacts:config=config/crd/bases
/Users/anurag/Projects/oss/tutorials/nimbus/bin/controller-gen-v0.15.0 object:headerFile="hack/boilerplate.go.txt" paths="./..."
go fmt ./...
go vet ./...
go run ./cmd/main.go
2024-07-07T21:50:17+05:30       INFO    setup   starting manager
2024-07-07T21:50:17+05:30       INFO    starting server {"name": "health probe", "addr": "[::]:8081"}
2024-07-07T21:50:17+05:30       INFO    Starting EventSource    {"controller": "securityintent", "controllerGroup": "intent.security.nimbus.com", "controllerKind": "SecurityIntent", "source": "kind source: *v1alpha1.SecurityIntent"}
2024-07-07T21:50:17+05:30       INFO    Starting EventSource    {"controller": "securityintentbinding", "controllerGroup": "intent.security.nimbus.com", "controllerKind": "SecurityIntentBinding", "source": "kind source: *v1alpha1.SecurityIntentBinding"}
2024-07-07T21:50:17+05:30       INFO    Starting Controller     {"controller": "securityintent", "controllerGroup": "intent.security.nimbus.com", "controllerKind": "SecurityIntent"}
2024-07-07T21:50:17+05:30       INFO    Starting Controller     {"controller": "securityintentbinding", "controllerGroup": "intent.security.nimbus.com", "controllerKind": "SecurityIntentBinding"}
2024-07-07T21:50:17+05:30       INFO    Starting workers        {"controller": "securityintent", "controllerGroup": "intent.security.nimbus.com", "controllerKind": "SecurityIntent", "worker count": 1}
2024-07-07T21:50:17+05:30       INFO    Starting workers        {"controller": "securityintentbinding", "controllerGroup": "intent.security.nimbus.com", "controllerKind": "SecurityIntentBinding", "worker count": 1}
```

Let's try again to create our resources:

```bash
$ k apply -f pkg-mgrs-intent-and-binding.yaml
securityintent.intent.security.nimbus.com/package-mgrs created
securityintentbinding.intent.security.nimbus.com/package-mgrs-binding created
```

This time they were created successfully.

```bash
$ k get securityintent
NAME           AGE
package-mgrs   13s
 
$ k get securityintentbinding
NAME                   AGE
package-mgrs-binding   17s

$ k get nimbuspolicy
No resources found in default namespace.
```

Did you notice something?

We created our resources, but Nimbus didn't generate the intermediary resource aka `NimbusPolicy`. Do you know why?

Because Kubernetes don't know how to handle these resources. How can we fix this? By creating controllers that will help achieve the desired state.

## Controllers

Open the `internal/controller/securityintent_controller.go` file which was scaffolded by Kubebuilder. It should be similar to the following:

```go
package controller

import (
	"context"

	"k8s.io/apimachinery/pkg/runtime"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/log"

	intentv1alpha1 "github.com/anurag-rajawat/tutorials/nimbus/api/v1alpha1"
)

// SecurityIntentReconciler reconciles a SecurityIntent object
type SecurityIntentReconciler struct {
	client.Client
	Scheme *runtime.Scheme
}

// +kubebuilder:rbac:groups=intent.security.nimbus.com,resources=securityintents,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=intent.security.nimbus.com,resources=securityintents/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=intent.security.nimbus.com,resources=securityintents/finalizers,verbs=update

// Reconcile is part of the main kubernetes reconciliation loop which aims to
// move the current state of the cluster closer to the desired state.
// TODO(user): Modify the Reconcile function to compare the state specified by
// the SecurityIntent object against the actual cluster state, and then
// perform operations to make the cluster state reflect the state specified by
// the user.
//
// For more details, check Reconcile and its Result here:
// - https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.18.2/pkg/reconcile
func (r *SecurityIntentReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	_ = log.FromContext(ctx)

	// TODO(user): your logic here

	return ctrl.Result{}, nil
}

// SetupWithManager sets up the controller with the Manager.
func (r *SecurityIntentReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&intentv1alpha1.SecurityIntent{}).
		Complete(r)
}
```

Description of the above structs, their methods, and fields:

* `SecurityIntentReconciler` struct includes the `client.Client` interface, a generic Kubernetes client that allows us to perform CRUD and watch operations on Kubernetes objects. And `Scheme` provides methods for serializing and deserializing API objects.
    
* `Reconcile` method, read its comments carefully.
    
* The `Reconcile` method takes two arguments: `ctx` (a standard Go `context.Context` type) and `req` (a `ctrl.Request`). The key point is the `ctrl.Request` object, which only includes the name and namespace of the resource to be reconciled. This means you won't have the full resource object available within the method itself.
    
* The `SetupWithManager` method configures the `SecurityIntent` controller by associating it with the manager object. This manager object takes responsibility for starting the `SecurityIntent` controller when it starts up. Once initiated, the SecurityIntent controller becomes an active listener, specifically for events related to `SecurityIntent` objects, such as creation, updates, and deletion.
    
* Kubebuilder markers:
    
    ```go
    // +kubebuilder:rbac:groups=intent.security.nimbus.com,resources=securityintents,verbs=get;list;watch;create;update;patch;delete
    // +kubebuilder:rbac:groups=intent.security.nimbus.com,resources=securityintents/status,verbs=get;update;patch
    // +kubebuilder:rbac:groups=intent.security.nimbus.com,resources=securityintents/finalizers,verbs=update
    ```
    
    These are Role-Based Access Control ([RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)) markers. They generate ClusterRole and ClusterRoleBinding with the permissions needed for the controller to function in the cluster:
    
* The first marker gives `get`, `list`, `watch`, `create`, `update`, `patch`, and `delete` permissions for the `SecurityIntent` resource.
    
* The second marker gives `get`, `update`, and `patch` permissions for the `status` subresource of `SecurityIntent`.
    
* The third marker gives `update` permission for the [finalizers](https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/) of `SecurityIntent`. Finalizers are special functions that run when a resource is about to be deleted. They can prevent the deletion of resources until certain conditions are met.
    

We'll adjust these permissions as needed.

### Reconciler

The controller comes into play when:

* Changes occur in the resources it manages,
    
* Changes occur in the resources it watches.
    

and calls the `reconcile` method to perform the business logic to align the current state with the desired state.

In Kubernetes controllers, reconciler logic is critical in ensuring the controller loop functions properly. If the reconciler logic is not implemented correctly, the controller can become stuck in an infinite loop, continuously processing requests without ever reaching a successful completion/desired state.

Internally, the controller maintains a queue. It pushes requests requiring action onto this queue. The reconciler then polls the queue, retrieves requests, and performs the necessary actions.

### Reconciler results

The `reconciler` method returns two objects `(ctrl.Result, error)`. Let's see their possible combinations and meanings:

* `ctrl.Result{}, nil`: This indicates the successful processing of the request. No further action is required.
    
* `ctrl.Result{Requeue...}, nil`: The request was processed successfully, but the reconciler needs to check back on it later. You might use this for scenarios where the request depends on external events or requires retries after a certain delay.
    
* `ctrl.Result{}, err`: An error occurred during processing. The controller will typically retry the request later, based on the specific error and your controller's retry logic.
    

Now that we have all the necessary tools and knowledge, let's dive into coding the reconciler. This is the core component responsible for handling changes made to resources managed by its associated controller.

### SecurityIntent

Edit the `securityintent_controller.go` file in `internal/controller` directory as follows:

```go
import (
	"context"

	"k8s.io/apimachinery/pkg/runtime"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/log"

	intentv1alpha1 "github.com/anurag-rajawat/tutorials/nimbus/api/v1alpha1"
)

func (r *SecurityIntentReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	logger := log.FromContext(ctx)

	var securityIntent intentv1alpha1.SecurityIntent
	err := r.Get(ctx, req.NamespacedName, &securityIntent)
	if err != nil {
		if client.IgnoreNotFound(err) != nil {
			logger.Error(err, "failed to fetch SecurityIntent", "securityIntent", req.Name)
			return ctrl.Result{}, err
		}
		logger.Info("SecurityIntent not found. Ignoring since object must be deleted")
		return ctrl.Result{}, nil
	}

	logger.Info("reconciling SecurityIntent", "securityIntent", req.Name)
	return ctrl.Result{}, nil
}
```

The `SecurityIntent` controller doesn't have much to do. It just performs a `get` call using the Kubernetes client's `Get` method and logs a message or an error if something goes wrong during the call.

### SecurityIntentBinding

The `SecurityIntent` controller is straightforward, now let's implement a controller for `SecurityIntentBinding`.

First, let's understand what are the responsibilities of `SecurityIntentBinding` controller:

* If you remember from the design, that `SecurityIntentBinding` references `SecurityIntent`.
    
* It manages an intermediary resource called `NimbusPolicy`, which is the generic representation of bound SecurityIntent(s) and its SecurityIntentBinding within a namespace.
    

Let's tackle these:

Edit the `securityintentbinding_controller.go` file in `internal/controller` directory as follows:

```go
import (
	"context"
	"errors"

	apierrors "k8s.io/apimachinery/pkg/api/errors"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/types"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/log"

	intentv1alpha1 "github.com/anurag-rajawat/tutorials/nimbus/api/v1alpha1"
	"github.com/anurag-rajawat/tutorials/nimbus/pkg/builder"
	buildererrors "github.com/anurag-rajawat/tutorials/nimbus/pkg/utils/errors"
)

func (r *SecurityIntentBindingReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	logger := log.FromContext(ctx)

	var securityIntentBinding intentv1alpha1.SecurityIntentBinding
	err := r.Get(ctx, req.NamespacedName, &securityIntentBinding)
	if err != nil {
		if client.IgnoreNotFound(err) != nil {
			logger.Error(err, "failed to fetch SecurityIntentBinding", "securityIntentBinding.name", req.Name, "securityIntentBinding.namespace", req.Namespace)
			return ctrl.Result{}, err
		}
		logger.Info("SecurityIntentBinding not found. Ignoring since object must be deleted", "securityIntentBinding.name", req.Name, "securityIntentBinding.namespace", req.Namespace)
		return ctrl.Result{}, nil
	}

	logger.Info("reconciling SecurityIntentBinding", "securityIntentBinding.name", req.Name, "securityIntentBinding.namespace", req.Namespace)

	_, err := r.createOrUpdateNimbusPolicy(ctx, securityIntentBinding)
	if err != nil {
		return ctrl.Result{}, err
	}

	return ctrl.Result{}, nil
}

func (r *SecurityIntentBindingReconciler) createOrUpdateNimbusPolicy(ctx context.Context, securityIntentBinding intentv1alpha1.SecurityIntentBinding) (*intentv1alpha1.NimbusPolicy, error) {
	logger := log.FromContext(ctx)

	nimbusPolicyToCreate, err := builder.BuildNimbusPolicy(ctx, r.Client, securityIntentBinding)
	if err != nil {
		if errors.Is(err, buildererrors.ErrSecurityIntentsNotFound) {
			logger.Info("aborted NimbusPolicy creation, since no SecurityIntents were found")
			return nil, nil
		}
		return nil, err
	}

	var nimbusPolicy intentv1alpha1.NimbusPolicy
	err = r.Get(ctx, types.NamespacedName{Name: securityIntentBinding.Name, Namespace: securityIntentBinding.Namespace}, &nimbusPolicy)
	if err != nil {
		if apierrors.IsNotFound(err) {
			return r.createNimbusPolicy(ctx, nimbusPolicyToCreate)
		}
		logger.Error(err, "failed to fetch NimbusPolicy", "nimbusPolicy.name", securityIntentBinding.Name, "nimbusPolicy.namespace", securityIntentBinding.Namespace)
		return nil, err
	}

	return r.updateNimbusPolicy(ctx, &nimbusPolicy, nimbusPolicyToCreate)
}

func (r *SecurityIntentBindingReconciler) createNimbusPolicy(ctx context.Context, nimbusPolicyToCreate *intentv1alpha1.NimbusPolicy) (*intentv1alpha1.NimbusPolicy, error) {
	logger := log.FromContext(ctx)

	err := r.Create(ctx, nimbusPolicyToCreate)
	if err != nil {
		logger.Error(err, "failed to create NimbusPolicy", "nimbusPolicy.name", nimbusPolicyToCreate.Name, "nimbusPolicy.namespace", nimbusPolicyToCreate.Namespace)
		return nil, err
	}

	logger.V(2).Info("nimbusPolicy created", "nimbusPolicy.name", nimbusPolicyToCreate.Name, "nimbusPolicy.namespace", nimbusPolicyToCreate.Namespace)
	return nimbusPolicyToCreate, nil
}

func (r *SecurityIntentBindingReconciler) updateNimbusPolicy(ctx context.Context, existingNimbusPolicy *intentv1alpha1.NimbusPolicy, updatedNimbusPolicy *intentv1alpha1.NimbusPolicy) (*intentv1alpha1.NimbusPolicy, error) {
	logger := log.FromContext(ctx)

    // check the spec, if something changed then only update existing nimbusPolicy.
	existingNimbusPolicySpecBytes, _ := json.Marshal(existingNimbusPolicy.Spec)
	newNimbusPolicySpecBytes, _ := json.Marshal(updatedNimbusPolicy.Spec)
	if bytes.Equal(existingNimbusPolicySpecBytes, newNimbusPolicySpecBytes) {
		return existingNimbusPolicy, nil
	}

	updatedNimbusPolicy.ResourceVersion = existingNimbusPolicy.ResourceVersion
	err := r.Update(ctx, updatedNimbusPolicy)
	if err != nil {
		logger.Error(err, "failed to update NimbusPolicy", "nimbusPolicy.name", updatedNimbusPolicy.Name, "nimbusPolicy.namespace", updatedNimbusPolicy.Namespace)
		return nil, err
	}

	logger.V(2).Info("nimbusPolicy updated", "nimbusPolicy.name", updatedNimbusPolicy.Name, "nimbusPolicy.namespace", updatedNimbusPolicy.Namespace)
	return updatedNimbusPolicy, nil
}
```

The `SecurityIntentBindingReconciler` first checks if the requested `SecurityIntentBinding` exists. If it does, it then creates or updates the `NimbusPolicy`.

You might wonder how it creates the `NimbusPolicy`. Let me explain.

Remember, the `NimbusPolicy` is an intermediary resource that references `SecurityIntentBinding` and its `SecurityIntent` for security engines within a namespace. Now, let's see how to build it.

Create a new file called `nimbus_policy.go` in the `pkg/builder` directory for the nimbusPolicy builder and edit it as follows:

```go
package builder

import (
	"context"

	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/types"
	"sigs.k8s.io/controller-runtime/pkg/client"

	intentv1alpha1 "github.com/anurag-rajawat/tutorials/nimbus/api/v1alpha1"
	buildererrors "github.com/anurag-rajawat/tutorials/nimbus/pkg/utils/errors"
)

func BuildNimbusPolicy(ctx context.Context, k8sClient client.Client, securityIntentBinding intentv1alpha1.SecurityIntentBinding) (*intentv1alpha1.NimbusPolicy, error) {
	intents := extractIntents(ctx, k8sClient, &securityIntentBinding)
	if len(intents) == 0 {
		return nil, buildererrors.ErrSecurityIntentsNotFound
	}

	var nimbusRules []intentv1alpha1.Rule
	for _, intent := range intents {
		nimbusRules = append(nimbusRules, intentv1alpha1.Rule{
			ID:         intent.Spec.Intent.ID,
			RuleAction: intent.Spec.Intent.Action,
			Params:     intent.Spec.Intent.Params,
		})
	}

	nimbusPolicy := &intentv1alpha1.NimbusPolicy{
		TypeMeta: metav1.TypeMeta{
			Kind:       "NimbusPolicy",
			APIVersion: intentv1alpha1.GroupVersion.String(),
		},
		ObjectMeta: metav1.ObjectMeta{
			Name:      securityIntentBinding.Name,
			Namespace: securityIntentBinding.Namespace,
			Labels:    securityIntentBinding.Labels,
		},
		Spec: intentv1alpha1.NimbusPolicySpec{
			NimbusRules: nimbusRules,
			Selector:    securityIntentBinding.Spec.Selector,
		},
	}
	return nimbusPolicy, nil
}

func extractIntents(ctx context.Context, k8sClient client.Client, securityIntentBinding *intentv1alpha1.SecurityIntentBinding) []intentv1alpha1.SecurityIntent {
	var intentsToReturn []intentv1alpha1.SecurityIntent
	for _, intent := range securityIntentBinding.Spec.Intents {
		var currSecurityIntent intentv1alpha1.SecurityIntent
		if err := k8sClient.Get(ctx, types.NamespacedName{Name: intent.Name}, &currSecurityIntent); err != nil {
			continue
		}
		intentsToReturn = append(intentsToReturn, currSecurityIntent)
	}
	return intentsToReturn
}
```

The `nimbusPolicy` builder starts by extracting the `SecurityIntents` from the given `SecurityIntentBinding` object. Then extracts the relevant data from the referenced `SecurityIntents`, such as the action, ID, and parameters used by security engines. More on this later.

Create one more new file called `errors.go` in the `pkg/utils/errors` directory as follows:

```go
package errors

import (
	"errors"
)

var (
	ErrSecurityIntentsNotFound = errors.New("no SecurityIntents found")
)
```

Now once again start the operator:

```bash
make run
```

Open a new terminal and create the same previous sample `SecurityIntent` and `SecurityIntentBinding`:

```bash
$ k apply -f pkg-mgrs-intent-and-binding.yaml
securityintent.intent.security.nimbus.com/package-mgrs created
securityintentbinding.intent.security.nimbus.com/package-mgrs-binding created
```

If you check your operator logs, you should see similar to the following:

```bash
...
...
2024-07-13T20:27:47+05:30       INFO    reconciling SecurityIntent      {"controller": "securityintent", "controllerGroup": "intent.security.nimbus.com", "controllerKind": "SecurityIntent", "SecurityIntent": {"name":"package-mgrs"}, "namespace": "", "name": "package-mgrs", "reconcileID": "49f2e375-145b-45bf-a77c-d46fd8d5a5c4", "securityIntent": "package-mgrs"}
2024-07-13T20:27:47+05:30       INFO    reconciling SecurityIntentBinding       {"controller": "securityintentbinding", "controllerGroup": "intent.security.nimbus.com", "controllerKind": "SecurityIntentBinding", "SecurityIntentBinding": {"name":"package-mgrs-binding","namespace":"default"}, "namespace": "default", "name": "package-mgrs-binding", "reconcileID": "30c530a9-e57f-4d29-a87b-6a4305fab068", "securityIntentBinding.name": "package-mgrs-binding", "securityIntentBinding.namespace": "default"}
```

Let's see what it has created this time:

```bash
$ k get securityintent
NAME           AGE
package-mgrs   47s

$ k get securityintentbinding
NAME                   AGE
package-mgrs-binding   55s

$ k get nimbuspolicy
NAME                   AGE
package-mgrs-binding   65s
```

Hooray 🥳! This time it successfully created the intermediary CR, aka `NimbusPolicy`. Let's take a look at its details:

```bash
$ k get nimbuspolicy package-mgrs-binding -o yaml
apiVersion: intent.security.nimbus.com/v1alpha1
kind: NimbusPolicy
metadata:
  creationTimestamp: "2024-07-13T15:00:38Z"
  generation: 1
  name: package-mgrs-binding
  namespace: default
  resourceVersion: "4235"
  uid: 4d10d65a-3b91-4aa5-83c9-426751a5b0b1
spec:
  rules:
  - action: Enforce
    id: pkgMgrs
  selector:
    matchLabels:
      app: web
      env: prod
```

You may be wondering how in this world the security engines leverage this custom resource to create their policies. We'll explore this in detail in the next part of the series, so stay tuned!

Let's delete the created CRs because we no longer need them:

```bash
$ k delete -f pkg-mgrs-intent-and-binding.yaml
securityintent.intent.security.nimbus.com "package-mgrs" deleted
securityintentbinding.intent.security.nimbus.com "package-mgrs-binding" deleted
```

Check the operator logs:

```bash
2024-07-13T20:38:41+05:30       INFO    SecurityIntent not found. Ignoring since object must be deleted {"controller": "securityintent", "controllerGroup": "intent.security.nimbus.com", "controllerKind": "SecurityIntent", "SecurityIntent": {"name":"package-mgrs"}, "namespace": "", "name": "package-mgrs", "reconcileID": "d2f593cc-64bb-4136-8195-4def0a569193"}
2024-07-13T20:38:41+05:30       INFO    SecurityIntentBinding not found. Ignoring since object must be deleted  {"controller": "securityintentbinding", "controllerGroup": "intent.security.nimbus.com", "controllerKind": "SecurityIntentBinding", "SecurityIntentBinding": {"name":"package-mgrs-binding","namespace":"default"}, "namespace": "default", "name": "package-mgrs-binding", "reconcileID": "d5e49618-1099-4d89-8890-ac94f1b4b2fa", "securityIntentBinding.name": "package-mgrs-binding", "securityIntentBinding.namespace": "default"}
```

Did you notice the following?

1. How does this operator run where is the `main` function? Additionally, how does the operator know which controllers to run?
    
2. Why did the nimbusPolicy not get deleted?
    

Let's unveil the mystery of the first case, by examining the `main.go` file in `cmd` directory. This file typically follows a structure similar to the one below:

```go
/*
Copyright 2024.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

package main

import (
	"crypto/tls"
	"flag"
	"os"

	// Import all Kubernetes client auth plugins (e.g. Azure, GCP, OIDC, etc.)
	// to ensure that exec-entrypoint and run can make use of them.
	_ "k8s.io/client-go/plugin/pkg/client/auth"

	"k8s.io/apimachinery/pkg/runtime"
	utilruntime "k8s.io/apimachinery/pkg/util/runtime"
	clientgoscheme "k8s.io/client-go/kubernetes/scheme"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/healthz"
	"sigs.k8s.io/controller-runtime/pkg/log/zap"
	metricsserver "sigs.k8s.io/controller-runtime/pkg/metrics/server"
	"sigs.k8s.io/controller-runtime/pkg/webhook"

	intentv1alpha1 "github.com/anurag-rajawat/tutorials/nimbus/api/v1alpha1"
	"github.com/anurag-rajawat/tutorials/nimbus/internal/controller"
	// +kubebuilder:scaffold:imports
)

var (
	scheme   = runtime.NewScheme()
	setupLog = ctrl.Log.WithName("setup")
)

func init() {
	utilruntime.Must(clientgoscheme.AddToScheme(scheme))

	utilruntime.Must(intentv1alpha1.AddToScheme(scheme))
	// +kubebuilder:scaffold:scheme
}

func main() {
	var metricsAddr string
	var enableLeaderElection bool
	var probeAddr string
	var secureMetrics bool
	var enableHTTP2 bool
	flag.StringVar(&metricsAddr, "metrics-bind-address", "0", "The address the metric endpoint binds to. "+
		"Use the port :8080. If not set, it will be 0 in order to disable the metrics server")
	flag.StringVar(&probeAddr, "health-probe-bind-address", ":8081", "The address the probe endpoint binds to.")
	flag.BoolVar(&enableLeaderElection, "leader-elect", false,
		"Enable leader election for controller manager. "+
			"Enabling this will ensure there is only one active controller manager.")
	flag.BoolVar(&secureMetrics, "metrics-secure", false,
		"If set the metrics endpoint is served securely")
	flag.BoolVar(&enableHTTP2, "enable-http2", false,
		"If set, HTTP/2 will be enabled for the metrics and webhook servers")
	opts := zap.Options{
		Development: true,
	}
	opts.BindFlags(flag.CommandLine)
	flag.Parse()

	ctrl.SetLogger(zap.New(zap.UseFlagOptions(&opts)))

	// if the enable-http2 flag is false (the default), http/2 should be disabled
	// due to its vulnerabilities. More specifically, disabling http/2 will
	// prevent from being vulnerable to the HTTP/2 Stream Cancellation and
	// Rapid Reset CVEs. For more information see:
	// - https://github.com/advisories/GHSA-qppj-fm5r-hxr3
	// - https://github.com/advisories/GHSA-4374-p667-p6c8
	disableHTTP2 := func(c *tls.Config) {
		setupLog.Info("disabling http/2")
		c.NextProtos = []string{"http/1.1"}
	}

	tlsOpts := []func(*tls.Config){}
	if !enableHTTP2 {
		tlsOpts = append(tlsOpts, disableHTTP2)
	}

	webhookServer := webhook.NewServer(webhook.Options{
		TLSOpts: tlsOpts,
	})

	mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
		Scheme: scheme,
		Metrics: metricsserver.Options{
			BindAddress:   metricsAddr,
			SecureServing: secureMetrics,
			TLSOpts:       tlsOpts,
		},
		WebhookServer:          webhookServer,
		HealthProbeBindAddress: probeAddr,
		LeaderElection:         enableLeaderElection,
		LeaderElectionID:       "9f24daa2.security.nimbus.com",
		// LeaderElectionReleaseOnCancel defines if the leader should step down voluntarily
		// when the Manager ends. This requires the binary to immediately end when the
		// Manager is stopped, otherwise, this setting is unsafe. Setting this significantly
		// speeds up voluntary leader transitions as the new leader don't have to wait
		// LeaseDuration time first.
		//
		// In the default scaffold provided, the program ends immediately after
		// the manager stops, so would be fine to enable this option. However,
		// if you are doing or is intended to do any operation such as perform cleanups
		// after the manager stops then its usage might be unsafe.
		// LeaderElectionReleaseOnCancel: true,
	})
	if err != nil {
		setupLog.Error(err, "unable to start manager")
		os.Exit(1)
	}

	if err = (&controller.SecurityIntentReconciler{
		Client: mgr.GetClient(),
		Scheme: mgr.GetScheme(),
	}).SetupWithManager(mgr); err != nil {
		setupLog.Error(err, "unable to create controller", "controller", "SecurityIntent")
		os.Exit(1)
	}
	if err = (&controller.SecurityIntentBindingReconciler{
		Client: mgr.GetClient(),
		Scheme: mgr.GetScheme(),
	}).SetupWithManager(mgr); err != nil {
		setupLog.Error(err, "unable to create controller", "controller", "SecurityIntentBinding")
		os.Exit(1)
	}
	// +kubebuilder:scaffold:builder

	if err := mgr.AddHealthzCheck("healthz", healthz.Ping); err != nil {
		setupLog.Error(err, "unable to set up health check")
		os.Exit(1)
	}
	if err := mgr.AddReadyzCheck("readyz", healthz.Ping); err != nil {
		setupLog.Error(err, "unable to set up ready check")
		os.Exit(1)
	}

	setupLog.Info("starting manager")
	if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
		setupLog.Error(err, "problem running manager")
		os.Exit(1)
	}
}
```

The things we're interested in are, how does the operator know which controller(s) to run?

In Kubernetes, an operator runs a manager who is responsible for starting and managing the configured controllers.

In our case, Kubebuilder simplifies things by automatically adding the code to the operator that starts the `SecurityIntent` and `SecurityIntentBinding` controllers when the operator itself starts. That's the benefit of using the Kubebuilder framework.

Here's an excerpt from the code:

```go
...
...
	if err = (&controller.SecurityIntentReconciler{
		Client: mgr.GetClient(),
		Scheme: mgr.GetScheme(),
	}).SetupWithManager(mgr); err != nil {
		setupLog.Error(err, "unable to create controller", "controller", "SecurityIntent")
		os.Exit(1)
	}
...
...
```

This covers a lot of ground! I recommend taking some time to absorb this information before moving on. In the next post, we'll tackle how to fix the `NimbusPolicy` deletion issue (second scenario). Stay tuned!

You can find the complete code [here](https://github.com/anurag-rajawat/tutorials/tree/main/nimbus). Please feel free to comment or criticize :)

# Summary

> In this part, we delve into implementing Custom Resource (CR) types and their controllers. We start by defining the SecurityIntent and SecurityIntentBinding CR types using Go structs and Kubebuilder Markers for generating code and configurations. We then enrich these CRs with additional fields and validation rules before generating the necessary code and manifests.
> 
> Once the CRs are defined, we apply them to Kubernetes, resolving errors by properly installing CRDs. Next, we implement controllers for these resources. The SecurityIntent controller simply retrieves and logs SecurityIntent objects. However, the SecurityIntentBinding controller has a more complex task: it manages an intermediary resource called NimbusPolicy, which references both SecurityIntent and SecurityIntentBinding.
> 
> We discuss the details of the Reconcile method, RBAC markers, and the lifecycle of controllers. Additionally, we introduce the concept of a reconciler and its significance in maintaining the desired state of the cluster.
> 
> Finally, we touch on how to start the operator, including necessary setups like manager configuration and controller initialization. The article ends with a preview of the next steps, which will address handling deletions and expanding functionality.

# **References**

* [Kubernetes docs](https://kubernetes.io/docs/home/)
    
* [**5GSec Nimbus**](https://github.com/5GSEC/nimbus)
    
* [**Programming Kubernetes**](https://www.oreilly.com/library/view/programming-kubernetes/9781492047094/)
    
* [**Kubebuilder book**](https://www.kubebuilder.io/introduction)
    
* [API conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)