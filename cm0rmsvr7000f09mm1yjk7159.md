---
title: "Building a Real-world Kubernetes Operator: Part - 3"
seoTitle: "Real-world Kubernetes Operator: Part 3"
seoDescription: "Manage lifecycle with Kubernetes OwnerReferences; add status fields and printer columns to custom resources."
datePublished: Sat Sep 07 2024 04:14:55 GMT+0000 (Coordinated Universal Time)
cuid: cm0rmsvr7000f09mm1yjk7159
slug: building-a-real-world-kubernetes-operator-part-3
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1721994590044/ea7c5099-96a0-49ba-bf0c-75107e35d212.png
tags: kubernetes, kubernetes-operators

---

# Introduction

This part continues from the previous section where we developed the controllers. Here, we'll fix the `NimbusPolicy` deletion issue and make a few other improvements.

There's another issue with `NimbusPolicy`: if you manually delete or update it, the `SecurityIntentBinding` controller neither recreates it nor discards those manual changes.

Right now, the `SecurityIntentBinding` controller only creates or updates the `NimbusPolicy` when it gets an event related to `SecurityIntentBinding`. Ideally, it should manage the entire lifecycle of a NimbusPolicy, similar to how a ReplicaSet controller manages Pods.

There are two ways to fix these:

* **Manual Lifecycle Management (Not Recommended):** This involves manually handling the creation, deletion, and updates of NimbusPolicy objects. However, this approach can quickly become cumbersome and error-prone. Kubernetes already provides built-in mechanisms for managing resources, so this method is generally discouraged.
    
* **Leveraging Kubernetes' OwnerReferences:** Kubernetes way, by utilizing a built-in Kubernetes feature called OwnerReferences.
    

Let's get into business.

# Owners and Dependents

The Kubernetes documentation provides a clear explanation of [ownership and dependencies](https://kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/) between resources.

In the context of our operator, let's break down the concept of owner and child resources.

Recall that in our design, a `SecurityIntentBinding` resource manages a `NimbusPolicy` resource. Therefore, `SecurityIntentBinding` acts as the owner, while `NimbusPolicy` is the child resource.

With this understanding, we'll configure our `SecurityIntentBinding` controller to be aware of its owned objects, specifically `NimbusPolicy` resources. This ensures that any changes to the `SecurityIntentBinding` are reflected in the corresponding `NimbusPolicy`. Additionally, the controller will recreate any deleted `NimbusPolicy` and discard any manual changes made directly to them.

Edit the `securityintentbinding_controller.go` file in the `internal/controller` directory as follows:

```go
...
// SetupWithManager sets up the controller with the Manager.
func (r *SecurityIntentBindingReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&intentv1alpha1.SecurityIntentBinding{}).
		Owns(&intentv1alpha1.NimbusPolicy{}).
		Complete(r)
}
...
```

Configured the controller to own `NimbusPolicy` i.e., it will manage its complete lifecycle.

Edit the `nimbus_policy.go` file in `pkg/builder` as follows:

```go
import (
    ...
    ctrl "sigs.k8s.io/controller-runtime"
)

func BuildNimbusPolicy(ctx context.Context, k8sClient client.Client, securityIntentBinding intentv1alpha1.SecurityIntentBinding) (*intentv1alpha1.NimbusPolicy, error) {
    ...
    if err := ctrl.SetControllerReference(&securityIntentBinding, nimbusPolicy, k8sClient.Scheme()); err != nil {
		return nil, err
	}
    
    return nimbusPolicy, nil
}
```

Set the `SecurityIntentBinding` as the owner of the `NimbusPolicy`.This tells the controller, "*Hey, you're responsible for handling this*`NimbusPolicy`".

Let's verify whether things are working as intended.

Create sample intent and binding:

```bash
$ k apply -f pkg-mgrs-intent-and-binding.yaml
securityintent.intent.security.nimbus.com/package-mgrs created
securityintentbinding.intent.security.nimbus.com/package-mgrs-binding created
```

check operator logs:

```bash
...
2024-07-13T23:08:02+05:30       INFO    reconciling SecurityIntent      {"controller": "securityintent", "controllerGroup": "intent.security.nimbus.com", "controllerKind": "SecurityIntent", "SecurityIntent": {"name":"package-mgrs"}, "namespace": "", "name": "package-mgrs", "reconcileID": "7dcbcd8f-70a7-419f-be5c-4b775bdba124", "securityIntent": "package-mgrs"}
2024-07-13T23:08:02+05:30       INFO    reconciling SecurityIntentBinding       {"controller": "securityintentbinding", "controllerGroup": "intent.security.nimbus.com", "controllerKind": "SecurityIntentBinding", "SecurityIntentBinding": {"name":"package-mgrs-binding","namespace":"default"}, "namespace": "default", "name": "package-mgrs-binding", "reconcileID": "8452da71-11a4-4e62-a60f-9d27ca74e6fc", "securityIntentBinding.name": "package-mgrs-binding", "securityIntentBinding.namespace": "default"}
2024-07-13T23:08:02+05:30       INFO    reconciling SecurityIntentBinding       {"controller": "securityintentbinding", "controllerGroup": "intent.security.nimbus.com", "controllerKind": "SecurityIntentBinding", "SecurityIntentBinding": {"name":"package-mgrs-binding","namespace":"default"}, "namespace": "default", "name": "package-mgrs-binding", "reconcileID": "fea993ee-101f-4285-a730-251b45799160", "securityIntentBinding.name": "package-mgrs-binding", "securityIntentBinding.namespace": "default"}
...
```

Check the resources and their detail:

### SecurityIntentBinding

```bash
$ k get securityintentbinding
NAME                   AGE
package-mgrs-binding   4m16s

$ k get securityintentbinding package-mgrs-binding -o yaml
apiVersion: intent.security.nimbus.com/v1alpha1
kind: SecurityIntentBinding
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"intent.security.nimbus.com/v1alpha1","kind":"SecurityIntentBinding","metadata":{"annotations":{},"name":"package-mgrs-binding","namespace":"default"},"spec":{"intents":[{"name":"package-mgrs"}],"selector":{"matchLabels":{"app":"web","env":"prod"}}}}
  creationTimestamp: "2024-07-13T17:38:02Z"
  generation: 1
  name: package-mgrs-binding
  namespace: default
  resourceVersion: "4227"
  uid: ab7d399c-67ef-401a-87f9-e94257970f34
spec:
  intents:
  - name: package-mgrs
  selector:
    matchLabels:
      app: web
      env: prod
```

### NimbusPolicy

```bash
$ k get nimbuspolicy
NAME                   AGE
package-mgrs-binding   4m36s

$ k get nimbuspolicy -o yaml
apiVersion: intent.security.nimbus.com/v1alpha1
kind: NimbusPolicy
metadata:
  creationTimestamp: "2024-07-13T17:38:02Z"
  generation: 1
  name: package-mgrs-binding
  namespace: default
  ownerReferences:
  - apiVersion: intent.security.nimbus.com/v1alpha1
    blockOwnerDeletion: true
    controller: true
    kind: SecurityIntentBinding
    name: package-mgrs-binding
    uid: ab7d399c-67ef-401a-87f9-e94257970f34
  resourceVersion: "4228"
  uid: 9996fe62-311c-42e7-b4ae-e2624d9d605f
spec:
  rules:
  - action: Enforce
    id: pkgMgrs
  selector:
    matchLabels:
      app: web
      env: prod
```

Check the `metadata.ownerReferences` field; it has an entry for its owner with its kind, name, and uid.

Let's delete the resources we created earlier:

```bash
$ k delete -f pkg-mgrs-intent-and-binding.yaml
securityintent.intent.security.nimbus.com "package-mgrs" deleted
securityintentbinding.intent.security.nimbus.com "package-mgrs-binding" deleted
```

Check if the owned `NimbusPolicy` has been deleted.

```bash
$ k get nimbuspolicy
No resources found in default namespace.
```

Things are working as expected. Try manually editing the `NimbusPolicy`, then delete it and check if the changes are discarded and if it gets recreated.

That's how to add the owner and dependents. Easy, isn't it?

# Short name

Shouldn't we give short names or aliases to resources so we have to type less? Just like we do for other core resources, for example, `deploy` for deployment, `rs` for replicaset, and `sts` for statefulset.

Edit the Kubebuilder markers in `securityintent_types.go` file in `api/v1alpha1` directory as follows:

```go
...
// +kubebuilder:resource:scope=Cluster,shortName=si

// SecurityIntent is the Schema for the securityintents API
type SecurityIntent struct {
    ...
}
...
```

Similarly, edit other files:

* `securityintentbinding_types.go`
    

```go
...
// +kubebuilder:resource:shortName=sib

// SecurityIntentBinding is the Schema for the securityintentbindings API
type SecurityIntentBinding struct {
    ...
}
...
```

* `nimbuspolicy_types.go`
    

```go
...
// +kubebuilder:resource:shortName=np

// NimbusPolicy is the Schema for the nimbuspolicies API
type NimbusPolicy struct {
    ...
}
...
```

We gave `si` for `SecurityIntent`, `sib` for `SecurityIntentBinding`, and `np` for `NimbusPolicy`.

Finally, install the updated CRDs:

```bash
make manifests install
```

Try `kubectl get si/sib/np`.

# Status

We've addressed edit and deletion issues, and even added short names for our resources. But there's one more step!

Have you noticed that running `kubectl get [kind] [name]` or `kubectl get [kind] [name] -o yaml` doesn't show the current state of your resources? Let's explore why.

This information is missing because we haven't defined the [`.status` subresource](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#status-subresource). The `.status` subresource is a dedicated field within Kubernetes resources where you can store details about their current/observed state.

So let's add it to all resources.

## SecurityIntent

Since we're only concerned with the action (`Enforce` / `Audit`) and status (`Created`) of a `SecurityIntent` resource, we can add these fields by modifying the `securityintent_types.go` file located within the `api/v1alpha1` directory as follows:

```go
...
// SecurityIntentStatus defines the observed state of SecurityIntent
type SecurityIntentStatus struct {
	Action string `json:"action"`
	Status string `json:"status"`
}
...
```

You can add additional fields as needed to suit your specific requirements.

Let's update the `SecurityIntent` controller to update the status subresource accordingly.

Edit the `securityintent_controller.go` in `internal/controller` directory as follows:

```go
...
func (r *SecurityIntentReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    ...
    logger.Info("reconciling SecurityIntent", "securityIntent", req.Name)
	if err = r.updateStatus(ctx, &securityIntent); err != nil {
		logger.Error(err, "failed to update SecurityIntent status", "securityIntent.name", req.Name)
		return ctrl.Result{}, err
	}
    ...
}

func (r *SecurityIntentReconciler) updateStatus(ctx context.Context, existingSecurityIntent *intentv1alpha1.SecurityIntent) error {
	existingSecurityIntent.Status = intentv1alpha1.SecurityIntentStatus{
		Action: existingSecurityIntent.Spec.Intent.Action,
		Status: consts.StatusCreated,
	}
	return r.Status().Update(ctx, existingSecurityIntent)
}
```

Here we're leveraging `Status` part of the client, with the Update method. The status subresource ignores changes to the `spec` field, so it’s less likely to conflict with any other updates and can have separate permissions.

## SecurityIntentBinding

For `SecurityIntentBinding`, we need the status, the names of bound security intents, the count of those bound intents, and the generated nimbuspolicy name. Let's add these fields to the `SecurityIntentBindingStatus` struct in the `securityintentbinding_types.go` file in the `api/v1alpha1` directory as follows:

```go
...
// SecurityIntentBindingStatus defines the observed state of SecurityIntentBinding
type SecurityIntentBindingStatus struct {
	Status              string   `json:"status"`
	BoundIntents        []string `json:"boundIntents,omitempty"`
	CountOfBoundIntents int32    `json:"countOfBoundIntents,omitempty"`
    NimbusPolicy        string   `json:"nimbusPolicy,omitempty"`
}
...
```

## NimbusPolicy

In `NimbusPolicy`, we need the status, the names of generated security policies, and their count. Let's add these fields too. Edit the `nimbuspolicy_types.go` file in the `api/v1alpha1` directory as follows:

```go
...
// NimbusPolicyStatus defines the observed state of NimbusPolicy
type NimbusPolicyStatus struct {
	Status                string   `json:"status"`
	GeneratedPoliciesName []string `json:"policiesName,omitempty"`
	CountOfPolicies       int32     `json:"policies,omitempty"`
}
...
```

Finally, update the `SecurityIntentBinding` controller to update the status subresource for both resources.

Edit the `securityintentbinding_controller.go` file in `internal/controller` directory as follows:

```go
...
func (r *SecurityIntentBindingReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    ...
    nimbusPolicy, err := r.createOrUpdateNimbusPolicy(ctx, securityIntentBinding)
	if err != nil {
		return ctrl.Result{}, err
	}

	if nimbusPolicy != nil {
		if err = r.updateNpStatus(ctx, nimbusPolicy); err != nil {
			logger.Error(err, "failed to update NimbusPolicy status", "nimbusPolicy.Name", nimbusPolicy.Name, "nimbusPolicy.Namespace", nimbusPolicy.Namespace)
			return ctrl.Result{}, err
		}

		if err = r.updateSibStatusWithBoundNpAndSisInfo(ctx, &securityIntentBinding, nimbusPolicy); err != nil {
			logger.Error(err, "failed to update SecurityIntentBinding status", "securityIntentBinding.name", req.Name, "securityIntentBinding.namespace", req.Namespace)
			return ctrl.Result{}, err
		}
	}

    return ctrl.Result{}, nil
}

func (r *SecurityIntentBindingReconciler) updateNpStatus(ctx context.Context, nimbusPolicy *intentv1alpha1.NimbusPolicy) error {
	nimbusPolicy.Status = intentv1alpha1.NimbusPolicyStatus{
		Status: StatusCreated,
	}
	return r.Status().Update(ctx, nimbusPolicy)
}

func (r *SecurityIntentBindingReconciler) updateSibStatusWithBoundNpAndSisInfo(ctx context.Context, existingSib *intentv1alpha1.SecurityIntentBinding, existingNp *intentv1alpha1.NimbusPolicy) error {
	existingSib.Status.Status = StatusCreated
	existingSib.Status.NimbusPolicy = existingNp.Name
	existingSib.Status.CountOfBoundIntents = int32(len(existingNp.Spec.NimbusRules))
	existingSib.Status.BoundIntents = r.getBoundIntents(ctx, existingSib.Spec.Intents)
	return r.Status().Update(ctx, existingSib)
}

func (r *SecurityIntentBindingReconciler) getBoundIntents(ctx context.Context, intents []intentv1alpha1.MatchIntent) []string {
	var boundIntentsName []string
	for _, intent := range intents {
		var currIntent intentv1alpha1.SecurityIntent
		if err := r.Get(ctx, types.NamespacedName{Name: intent.Name}, &currIntent); err != nil {
			continue
		}
		boundIntentsName = append(boundIntentsName, currIntent.Name)
	}
	return boundIntentsName
}
```

After updating the types and controllers, let's quickly install and verify the changes.

```bash
make manifests generate install
```

to run:

```bash
make run
```

create the same intent and binding as before:

```bash
$ k apply -f pkg-mgrs-intent-and-binding.yaml
securityintent.intent.security.nimbus.com/package-mgrs created
securityintentbinding.intent.security.nimbus.com/package-mgrs-binding created
```

verify the status subresource:

* SecurityIntent
    
    ```bash
    $ k get si package-mgrs -o wide
    NAME           AGE
    package-mgrs   69s
    
    $ k get si package-mgrs -o yaml
    apiVersion: intent.security.nimbus.com/v1alpha1
    kind: SecurityIntent
    # Fields removed for brevity
    ...
    ...
    status:
      action: Enforce
      status: Created
    ```
    
* SecurityIntentBinding
    
    ```bash
    $ k get sib package-mgrs-binding -o wide
    NAME                   AGE
    package-mgrs-binding   2m21s
    
    $ k get sib package-mgrs-binding -o yaml
    apiVersion: intent.security.nimbus.com/v1alpha1
    kind: SecurityIntentBinding
    # Fields removed for brevity
    ...
    ...
    status:
      boundIntents:
      - package-mgrs
      countOfBoundIntents: 1
      nimbusPolicy: package-mgrs-binding
      status: Created
    ```
    
* NimbusPolicy
    
    ```bash
    $ k get np package-mgrs-binding -o wide
    NAME                   AGE
    package-mgrs-binding   4m11s
    
    $ k get np package-mgrs-binding -o yaml
    apiVersion: intent.security.nimbus.com/v1alpha1
    kind: NimbusPolicy
    # Fields removed for brevit
    ...
    ...
    status:
      status: Created
    ```
    

As you can see, the status field is now populated by the controllers, so things are working as expected.

# Printer Columns

I feel like we're still missing something. Do you know what it is?

When we run `kubectl get [kind] [name] -o wide`, we don't see extra info. I know we can get that info by running `kubectl get [kind] [name] -o yaml`. Think about other core Kubernetes resources. For example:

```bash
$ k get deploy -o wide
NAME    READY   UP-TO-DATE   AVAILABLE   AGE    CONTAINERS   IMAGES                SELECTOR
httpd   10/10   10           10          104s   httpd        httpd:2.4.53-alpine   app=httpd
```

These are called [additional printer columns](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#additional-printer-columns). They help the API server decide which columns to show when we use the `kubectl get` command. Let's add these columns based on what we need.

## SecurityIntent

Add the Kubebuilder print column markers in the `securityintent_types.go` file located in the `api/v1alpha1` directory as follows:

```go
...
// +kubebuilder:printcolumn:name="Status",type="string",JSONPath=".status.status"
// +kubebuilder:printcolumn:name="Age",type="date",JSONPath=".metadata.creationTimestamp"
// +kubebuilder:printcolumn:name="Action",type="string",JSONPath=".spec.intent.action",priority=1

// SecurityIntent is the Schema for the securityintents API
type SecurityIntent struct {
    ...
}
...
```

Make sure to specify the correct `JSONPath` field value, or you'll get unexpected results.

The [priority](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#priority) field decides what to show in wide view when you run `kubectl get [kind] [name] -o wide`.

## SecurityIntentBinding

Similarly, edit the `securityintentbinding_types.go` file in the `api/v1alpha1` directory as follows:

```go
...
// +kubebuilder:printcolumn:name="Status",type="string",JSONPath=".status.status"
// +kubebuilder:printcolumn:name="Age",type="date",JSONPath=".metadata.creationTimestamp"
// +kubebuilder:printcolumn:name="Intents",type="integer",JSONPath=".status.countOfBoundIntents"
// +kubebuilder:printcolumn:name="NimbusPolicy",type="string",JSONPath=".status.nimbusPolicy"

// SecurityIntentBinding is the Schema for the securityintentbindings API
type SecurityIntentBinding struct {
    ...
}
...
```

Here, we're not using the `priority` field, so the standard view and the wide view will look the same. Since the names of bound security intents can be long, we're not showing their names in any view. You can get their details when you describe `SecurityIntentBinding`.

## NimbusPolicy

Edit the `nimbuspolicy_types.go` file located in `api/v1alpha1` directory as follows:

```go
...
// +kubebuilder:printcolumn:name="Status",type="string",JSONPath=".status.status"
// +kubebuilder:printcolumn:name="Age",type="date",JSONPath=".metadata.creationTimestamp"
// +kubebuilder:printcolumn:name="Policies",type="integer",JSONPath=".status.policies"

// NimbusPolicy is the Schema for the nimbuspolicies API
type NimbusPolicy struct {
    ...
}
...
```

This time we don't need to make any changes in any controller, just install the updated CRDs.

```bash
make manifests install
```

Run the operator:

```bash
make run
```

Apply the intent and binding:

```bash
$ k apply -f pkg-mgrs-intent-and-binding.yaml
securityintent.intent.security.nimbus.com/package-mgrs created
securityintentbinding.intent.security.nimbus.com/package-mgrs-binding created
```

verify the output:

* SecurityIntent
    
    ```bash
    $ k get si
    NAME           STATUS    AGE
    package-mgrs   Created   44s
    
    $ k get si -o wide
    NAME           STATUS    AGE   ACTION
    package-mgrs   Created   46s   Enforce
    ```
    
* SecurityIntentBinding
    
    ```bash
    $ k get sib
    NAME                   STATUS    AGE   INTENTS   NIMBUSPOLICY
    package-mgrs-binding   Created   87s   1         package-mgrs-binding
    
    $ k get sib -o wide
    NAME                   STATUS    AGE   INTENTS   NIMBUSPOLICY
    package-mgrs-binding   Created   92s   1         package-mgrs-binding
    ```
    
    You know why there's no difference between these views.
    
* NimbusPolicy
    
    ```bash
    $ k get np
    NAME                   STATUS    AGE     POLICIES
    package-mgrs-binding   Created   2m28s   
    
    $ k get np -o wide
    NAME                   STATUS    AGE     POLICIES
    package-mgrs-binding   Created   2m33s
    ```
    
    I know you're wondering why the `POLICIES` column is empty. Stay tuned!
    

That's it for this part. In the next part, we'll write tests for our controllers. Stay tuned!

You can find the complete code [**here**](https://github.com/anurag-rajawat/tutorials/tree/main/nimbus). Please feel free to comment or criticize :)

# Summary

> In this article, we address the lifecycle management of the NimbusPolicy resource in our Kubernetes operator via OwnerReferences. We configure our SecurityIntentBinding controller to manage NimbusPolicy, ensuring it handles creation, deletion, and updates effectively. Additionally, we add short names and status subresources for better usability, and enhance resource visibility using printer columns. This guide not only improves resource management but also enhances kubectl output for a streamlined operator experience.

# References

* [Kubernetes docs](https://kubernetes.io/docs/home/)
    
* [**5GSec Nimbus**](https://github.com/5GSEC/nimbus)
    
* [**Kubebuilder book**](https://www.kubebuilder.io/introduction)
    
* [API conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)