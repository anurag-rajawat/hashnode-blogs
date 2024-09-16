---
title: "Building a Real-world Kubernetes Operator: Part - 4"
seoTitle: "Creating a Real-world Kubernetes Operator, Part 4"
seoDescription: "Learn how to write and run declarative tests for Kubernetes operators with Kyverno's Chainsaw"
datePublished: Mon Sep 16 2024 04:49:59 GMT+0000 (Coordinated Universal Time)
cuid: cm14j0mmi005k08jt9xyvhdbe
slug: building-a-real-world-kubernetes-operator-part-4
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1721994631737/88696e7b-7ae0-4578-9312-ef83a57d1f5a.png
tags: kubernetes, testing, testing-tools, kubernetes-operators

---

# Introduction

In this part, we'll write tests for our operator and find bugs. We'll not follow the traditional (aka imperative) approach for testing, we'll use a declarative approach. Since Kubernetes follows a declarative approach, why shouldn't we follow the same approach for testing?

While testing doesn't guarantee a completely bug-free product, it's important to ensure core functionalities remain intact and haven't been unintentionally broken during development. We'll use Kyverno's [Chainsaw](https://kyverno.github.io/chainsaw/latest/) as an alternative to Ginkgo for implementing declarative testing for Kubernetes operators and controllers.

# SecurityIntent

Let's begin by examining SecurityIntent. Unlike most tests, our primary concern with SecurityIntent isn't its functionality, but its overall status. Therefore, we'll focus on writing a **status test** for SecurityIntent.

Create a directory `test/controller` where we'll keep our controllers' test.

```bash
mkdir -p test/controller
```

Since Chainsaw recommends creating one directory per test, with a `chainsaw-test.yaml` file, so let's create a directory for the SecurityIntent's controller.

```bash
mkdir -p test/controller/securityintent
touch test/controller/securityintent/chainsaw-test.yaml
```

Let's write our first declarative test, edit the `chainsaw-test.yaml` file in the `test/controller/securityintent` directory as follows:

```yaml
apiVersion: chainsaw.kyverno.io/v1alpha1
kind: Test
metadata:
  name: securityintent-creation
spec:
  description: |
    This test validates that the created SecurityIntent's status subresource contains the action 
    fields with the corresponding intent action.

  steps:
    - name: Create a SecurityIntent
      try:
        - apply:
            file: ../manifest/security-intent.yaml

    - name: Assert status
      try:
        - assert:
            file: status.yaml
```

Let's break this test, we defined two steps:

* Create a SecurityIntent by applying the `../manifest/security-intent.yaml` file
    
* Assert the created SecurityIntent's state using the `status.yaml` file.
    

Here we just specified **what** we want (creating a SecurityIntent and asserting the status) not **how** (specific implementation details). That's what "declarative" means.

Now, let's create the manifest files for this test. Make a `manifest` directory in `test/controller` to store the manifests that will be shared among tests.

* `security-intent.yaml` file in the `manifest` directory:
    
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
    ```
    
* Since the `status.yaml` file only applies to the SecurityIntent controller, we'll keep it in the `test/controller/securityintent` directory.
    
    ```yaml
    apiVersion: intent.security.nimbus.com/v1alpha1
    kind: SecurityIntent
    metadata:
      name: package-mgrs
    status:
      action: Enforce
      status: Created
    ```
    

I know you're wondering why this `status.yaml` file only contains the status and seems incomplete. Let me explain: we're only interested in checking the `status`, so there's no need to include unnecessary details. Just include what you need to check. You can use a complete file if you prefer.

I know you're thinking, "Wow, tests can be this simple." Yes, the declarative approach makes this possible, thanks to Kyverno for creating Chainsaw.

I'm very much excited to run this test, are you? Anyway, let's run it:

* [Install chainsaw](https://kyverno.github.io/chainsaw/latest/quick-start/install/)
    
    ```bash
    go install github.com/kyverno/chainsaw@latest
    ```
    
* Start the operator and keep it running
    
    ```bash
    make install # Install CRDs if not installed
    make run
    ```
    
* Run test in a new terminal
    
    ```bash
    cd test/controller/securityintent
    chainsaw test
    ```
    

Awesome! The test passed, generating colourful output. If your test ran successfully (which is likely!), pat yourself on the back.

# SecurityIntentBinding

Here we have a couple of cases to test:

* **Create:** When a SecurityIntentBinding is created, a NimbusPolicy should be created if the referenced SecurityIntent(s) exist in the cluster.
    
* **Update:** When a SecurityIntentBinding is updated, the corresponding NimbusPolicy should be updated.
    
* **Delete:** When a SecurityIntentBinding is deleted, the corresponding NimbusPolicy should also be deleted.
    

Let's write tests for these cases. Create a new directory called `securityintentbinding` in the `test/controller` directory to keep the related tests.

We'll organize tests in separate directories based on their operation, for example, `create`, `update` and `delete`.

## Create

Create and edit the `chainsaw-test.yaml` file in `test/controller/securityintentbinding/create` directory as follows:

```yaml
apiVersion: chainsaw.kyverno.io/v1alpha1
kind: Test
metadata:
  name: securityintentbinding-creation
spec:
  description: |
    This test validates the automated creation of a NimbusPolicy when a SecurityIntent 
    and SecurityIntentBinding are created.

  steps:
    - name: Create a SecurityIntent
      try:
        - apply:
            file: ../../manifest/security-intent.yaml

    - name: Create a SecurityIntentBinding
      try:
        - apply:
            file: ../../manifest/security-intent-binding.yaml

    - name: Assert NimbusPolicy creation
      try:
        - assert:
            file: ../nimbus-policy-to-assert.yaml

    - name: Assert SecurityIntentBinding's status subresource
      description: |
        Verify the created SecurityIntentBinding status subresource includes the number and names of bound intents, 
        along with the generated NimbusPolicy name.
      try:
        - assert:
            file: ../sib-status-to-assert.yaml
```

The test explains itself, so I don't think it needs further explanation.

Now let's create the files used in this test:

* Create a `security-intent-binding.yaml` file in the `test/controller/manifest` directory used by this test.
    

```yaml
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

* Create a `nimbus-policy-to-assert.yaml` in the `test/controller/securityintentbinding` directory as follows:
    

```yaml
apiVersion: intent.security.nimbus.com/v1alpha1
kind: NimbusPolicy
metadata:
  name: package-mgrs-binding
  ownerReferences:
    - apiVersion: intent.security.nimbus.com/v1alpha1
      blockOwnerDeletion: true
      controller: true
      kind: SecurityIntentBinding
      name: package-mgrs-binding
spec:
  rules:
    - action: Enforce
      id: pkgMgrs
  selector:
    matchLabels:
      app: web
      env: prod
status:
  status: Created
```

* Create a `sib-status-to-assert.yaml` in the same directory as the previous file as follows:
    

```yaml
apiVersion: intent.security.nimbus.com/v1alpha1
kind: SecurityIntentBinding
metadata:
  name: package-mgrs-binding
spec:
  intents:
    - name: package-mgrs
  selector:
    matchLabels:
      app: web
      env: prod
status:
  boundIntents:
    - package-mgrs
  countOfBoundIntents: 1
  nimbusPolicy: package-mgrs-binding
  status: Created
```

Now, run the test:

```bash
chainsaw test
```

The test passed again.

## Update

Create and edit the `chainsaw-test.yaml` file in the `test/controller/securityintentbinding/update` directory as follows:

```yaml
apiVersion: chainsaw.kyverno.io/v1alpha1
kind: Test
metadata:
  name: securityintentbinding-update
spec:
  description: |
    This test validates the propagation of changes from a SecurityIntentBinding to the corresponding NimbusPolicy.

  steps:
    - name: Create a SecurityIntent
      try:
        - apply:
            file: ../../manifest/security-intent.yaml

    - name: Create a SecurityIntentBinding
      try:
        - apply:
            file: ../../manifest/security-intent-binding.yaml

    - name: Assert NimbusPolicy creation
      try:
        - assert:
            file: ../nimbus-policy-to-assert.yaml

    - name: Assert SecurityIntentBinding's status subresource
      description: |
        Verify the created SecurityIntentBinding status subresource includes the number and names of bound intents, 
        along with the generated NimbusPolicy name.
      try:
        - assert:
            file: ../sib-status-to-assert.yaml

    - name: Update existing SecurityIntentBinding
      try:
        - patch:
            file: updated-sib.yaml

    - name: Assert the updated NimbusPolicy
      try:
        - assert:
            file: updated-np.yaml
```

This test builds on the previous one by adding two new steps:

* **Update SecurityIntentBinding:** I used the `patch` operation in the `try` block to update the existing `SecurityIntentBinding`. This way, I don't have to provide the entire `NimbusPolicy` object. If you use the `update` operation instead, you would need to provide the complete updated `NimbusPolicy`.
    
* **Assert updated NimbusPolicy:** After updating the SecurityIntentBinding, we check the updated `NimbusPolicy` to make sure the changes were successful.
    

Create the files used in this test.

* `updated-sib.yaml`
    

```yaml
apiVersion: intent.security.nimbus.com/v1alpha1
kind: SecurityIntentBinding
metadata:
  name: package-mgrs-binding
spec:
  selector:
    matchLabels:
      env: demo
```

* `updated-np.yaml`
    

```yaml
apiVersion: intent.security.nimbus.com/v1alpha1
kind: NimbusPolicy
metadata:
  name: package-mgrs-binding
  ownerReferences:
    - apiVersion: intent.security.nimbus.com/v1alpha1
      blockOwnerDeletion: true
      controller: true
      kind: SecurityIntentBinding
      name: package-mgrs-binding
spec:
  rules:
    - action: Enforce
      id: pkgMgrs
  selector:
    matchLabels:
      env: demo
status:
  status: Created
```

Now, run the test:

```bash
chainsaw test
```

The test passed again.

## Delete

Create and edit the `chainsaw-test.yaml` file in `test/controller/securityintentbinding/delete` directory as follows:

```yaml
apiVersion: chainsaw.kyverno.io/v1alpha1
kind: Test
metadata:
  name: securityintentbinding-delete
spec:
  description: |
    This test validates NimbusPolicy deletion when the parent SecurityIntentBinding gets deleted.

  steps:
    - name: Create a SecurityIntent
      try:
        - apply:
            file: ../../manifest/security-intent.yaml

    - name: Create a SecurityIntentBinding
      try:
        - apply:
            file: ../../manifest/security-intent-binding.yaml

    - name: Assert NimbusPolicy creation
      try:
        - assert:
            file: ../nimbus-policy-to-assert.yaml

    - name: Assert SecurityIntentBinding's status subresource
      description: |
        Verify the created SecurityIntentBinding status subresource includes the number and names of bound intents, 
        along with the generated NimbusPolicy name.
      try:
        - assert:
            file: ../sib-status-to-assert.yaml

    - name: Delete the created SecurityIntentBinding
      try:
        - delete:
            file: ../../manifest/security-intent-binding.yaml

    - name: Assert the NimbusPolicy deletion
      try:
        - script:
            content: kubectl get np package-mgrs-binding -n $NAMESPACE
            check:
              ($error != null): true
```

While most of this test is straightforward, the final step might seem a bit complex. Let me explain:

* We use the `script` operation in a `try` block. This operation runs a script with the command provided in the `content` field.
    
* The `check` assertion verifies the expected outcome of the script. In this case, we expect the script to run a `kubectl get` command and return an error.
    

Do you know why we're expecting an error?

Because the deletion of a SecurityIntentBinding should also trigger the deletion of the corresponding NimbusPolicy. We run `kubectl get` to see if the NimbusPolicy still exists. If the command doesn't return an error (indicating the policy still exists), the deletion fails.

Let's run this test as well:

```bash
chainsaw test
```

The test also passed. Seems our k8s operator is officially bug-free...

# NimbusPolicy

Here's what we'll test for NimbusPolicy:

* **Update:** Manual updates to a NimbusPolicy should be ignored.
    
* **Deletion:** If a NimbusPolicy is manually deleted, it should be recreated.
    

## Update

At this step, you should know which and where to create the file or directory. If not then create a new directory called `nimbuspolicy/update` in `test/controller` directory.

Edit the `chainsaw-test.yaml` file as follows:

```yaml
apiVersion: chainsaw.kyverno.io/v1alpha1
kind: Test
metadata:
  name: nimbuspolicy-update
spec:
  description: This test validates that direct updates to a NimbusPolicy are ignored to prevent unintended modifications.

  steps:
    - name: Create a SecurityIntent and SecurityIntentBinding
      try:
        - apply:
            file: ../../manifest/security-intent.yaml
        - apply:
            file: ../../manifest/security-intent-binding.yaml

    - name: Assert NimbusPolicy creation
      try:
        - assert:
            file: ../original-np.yaml

    - name: Update existing NimbusPolicy
      try:
        - patch:
            file: updated-np.yaml

    - name: Assert direct changes to NimbusPolicy are discarded
      try:
        - assert:
            file: ../original-np.yaml
```

Create the files used in this test:

* `original-np.yaml` in `test/controller/nimbuspolicy` directory.
    

```yaml
apiVersion: intent.security.nimbus.com/v1alpha1
kind: NimbusPolicy
metadata:
  name: package-mgrs-binding
  ownerReferences:
    - apiVersion: intent.security.nimbus.com/v1alpha1
      blockOwnerDeletion: true
      controller: true
      kind: SecurityIntentBinding
      name: package-mgrs-binding
spec:
  rules:
    - action: Enforce
      id: pkgMgrs
  selector:
    matchLabels:
      app: web
      env: prod
status:
  status: Created
```

* `updated-np.yaml` in the same directory as the test itself.
    

```yaml
apiVersion: intent.security.nimbus.com/v1alpha1
kind: NimbusPolicy
metadata:
  name: package-mgrs-binding
spec:
  rules:
    - action: Audit
      id: pkgMgrs
  selector:
    matchLabels:
      env: demo
```

## Delete

Edit the `chainsaw-test.yaml` file in `test/controller/nimbuspolicy/update` directory as follows:

```yaml
apiVersion: chainsaw.kyverno.io/v1alpha1
kind: Test
metadata:
  name: nimbuspolicy-delete
spec:
  description: This test validates whether a NimbusPolicy is re-created on manually deletion.

  steps:
    - name: Create a SecurityIntent and SecurityIntentBinding
      try:
        - apply:
            file: ../../manifest/security-intent.yaml
        - apply:
            file: ../../manifest/security-intent-binding.yaml

    - name: Assert NimbusPolicy creation
      try:
        - assert:
            file: ../original-np.yaml

    - name: Delete existing NimbusPolicy
      try:
        - delete:
            file: ../original-np.yaml

    - name: Assert NimbusPolicy re-creation after deletion
      try:
        - assert:
            file: ../original-np.yaml
```

Let's create a make target to run all the tests so that we don't have to run each controller's test manually.

Update the make `test` target as follows:

```makefile
...
.PHONY: test
test: chainsaw ## Run tests.
	@$(LOCALBIN)/chainsaw test --test-dir=test/controller/

.PHONY: chainsaw
chainsaw: ## Download chainsaw locally if necessary.
	@test -s $(LOCALBIN)/chainsaw || GOBIN=$(LOCALBIN) go install github.com/kyverno/chainsaw@latest
...
```

Now, run the tests by executing the following command from the root directory of our operator:

```bash
make test
```

This time, 2 tests failed, but when we ran them individually, they passed. What happened?

```bash
...
...
--- FAIL: chainsaw (0.00s)
    --- PASS: chainsaw/securityintent-creation (5.27s)
    --- PASS: chainsaw/securityintentbinding-creation (5.39s)
    --- PASS: chainsaw/nimbuspolicy-delete (5.44s)
    --- PASS: chainsaw/securityintentbinding-delete (5.60s)
    --- FAIL: chainsaw/securityintentbinding-update (35.45s)
    --- FAIL: chainsaw/nimbuspolicy-update (35.45s)
FAIL
Tests Summary...
- Passed  tests 4
- Failed  tests 2
- Skipped tests 0
Done with failures.
Error: some tests failed
make: *** [test] Error 1
```

By default, Chainsaw runs tests in parallel, which is causing the tests to fail. We can change this behaviour by providing a config to run tests sequentially.

Create a `chainsaw-config.yaml` file in the `test` directory as follows:

```yaml
apiVersion: chainsaw.kyverno.io/v1alpha1
kind: Configuration
metadata:
  name: default-configuration
spec:
  parallel: 1 # The maximum number of tests to run at once.
```

Update the make `test` target:

```makefile
.PHONY: test
test: chainsaw ## Run tests.
	@$(LOCALBIN)/chainsaw test --test-dir=test/controller/ --config test/chainsaw-config.yaml
```

Finally re-run the tests:

```bash
$ make test
...
...
--- PASS: chainsaw (0.00s)
    --- PASS: chainsaw/nimbuspolicy-delete (5.40s)
    --- PASS: chainsaw/securityintentbinding-creation (5.36s)
    --- PASS: chainsaw/securityintentbinding-update (5.51s)
    --- PASS: chainsaw/securityintentbinding-delete (5.57s)
    --- PASS: chainsaw/securityintent-creation (5.24s)
    --- PASS: chainsaw/nimbuspolicy-update (5.44s)
PASS
Tests Summary...
- Passed  tests 6
- Failed  tests 0
- Skipped tests 0
Done.
```

All green this time.

# Edge cases

Here are some edge cases we should test:

* **Create:** Create individual resources separately to make sure we can create these custom resources without needing another one to exist first.
    
* **Update:** When SecurityIntent's action is updated, make sure NimbusPolicy rules are updated too.
    
* **Delete:** If the referenced SecurityIntents don't exist in the cluster or deleted, stop NimbusPolicy creation.
    

Create a new directory called `edge-cases` in the `test/controller/` directory and separate the tests by their operations.

## Create

Create `chainsaw-test.yaml` in `test/controller/edge-cases/create` directory as follows:

```yaml
apiVersion: chainsaw.kyverno.io/v1alpha1
kind: Test
metadata:
  name: securityintentbinding-and-securityintent-independent-creation
spec:
  description: |
    This test verifies the independent creation of SecurityIntent and SecurityIntentBinding custom resources.
    To make sure we can create these custom resources without needing another one to exist first.

  steps:
    - name: Create a SecurityIntentBinding
      try:
        - apply:
            file: ../../manifest/security-intent-binding.yaml

    - name: Create a SecurityIntent
      try:
        - apply:
            file: ../../manifest/security-intent.yaml

    - name: Assert NimbusPolicy creation
      try:
        - assert:
            file: ../nimbus-policy-to-assert.yaml

    - name: Assert SecurityIntentBinding's status subresource
      description: |
        Verify the created SecurityIntentBinding status subresource includes the number and names of bound intents, 
        along with the generated NimbusPolicy name.
      try:
        - assert:
            file: ../sib-status-to-assert.yaml
```

In this test, we first create `SecurityIntentBinding`, then `SecurityIntent`, and finally, check that `NimbusPolic` exists. This ensures we can create resources without needing other resources to exist first.

Run the test:

```bash
$ chainsaw test
...
...
- securityintentbinding-and-securityintent-independent-creation (.)
    | 13:01:40 | securityintentbinding-and-securityintent-independent-creation | Assert NimbusPolicy creation                      | ASSERT    | ERROR | intent.security.nimbus.com/v1alpha1/NimbusPolicy @ chainsaw-musical-humpback/package-mgrs-binding
        === ERROR
        actual resource not found
...
...
--- FAIL: chainsaw (0.00s)
    --- FAIL: chainsaw/securityintentbinding-and-securityintent-independent-creation (35.33s)
FAIL
Tests Summary...
- Passed  tests 0
- Failed  tests 1
- Skipped tests 0
Done with failures.
Error: some tests failed
```

The test failed with an error, `actual resource not found`.

## Delete

```yaml
apiVersion: chainsaw.kyverno.io/v1alpha1
kind: Test
metadata:
  name: securityintent-deletion-after-creation-of-nimbuspolicy
spec:
  description: |
    This test verifies that if a SecurityIntent is the only one referenced in a SecurityIntentBinding, and that
    SecurityIntent is deleted, the related NimbusPolicy is also automatically deleted.

  steps:
    - name: Create a SecurityIntent
      try:
        - apply:
            file: ../../manifest/security-intent.yaml

    - name: Create a SecurityIntentBinding
      try:
        - apply:
            file: ../../manifest/security-intent-binding.yaml

    - name: Assert NimbusPolicy creation
      try:
        - assert:
            file: ../nimbus-policy-to-assert.yaml

    - name: Delete referenced created SecurityIntent
      try:
        - delete:
            file: ../../manifest/security-intent.yaml

    - name: Assert NimbusPolicy deletion
      try:
        - script:
            content: kubectl get np package-mgrs-binding -n $NAMESPACE
            check:
              ($error != null): true

    - name: Assert SecurityIntentBinding's status subresource
      description: |
        Verify that the created SecurityIntentBinding's status subresource does not include the number
        and names of bound intents, and does not have a NimbusPolicy name.
      try:
        - assert:
            file: sib-status-after-si-deletion.yaml
```

Let's create the required files used by the test.

* `nimbus-policy-to-assert.yaml` in `test/controller/edge-cases` directory:
    

```yaml
apiVersion: intent.security.nimbus.com/v1alpha1
kind: NimbusPolicy
metadata:
  name: package-mgrs-binding
  ownerReferences:
    - apiVersion: intent.security.nimbus.com/v1alpha1
      blockOwnerDeletion: true
      controller: true
      kind: SecurityIntentBinding
      name: package-mgrs-binding
spec:
  rules:
    - action: Enforce
      id: pkgMgrs
  selector:
    matchLabels:
      app: web
      env: prod
status:
  status: Created
```

* `sib-status-after-si-deletion.yaml` in the same directory as the test itself:
    

```yaml
apiVersion: intent.security.nimbus.com/v1alpha1
kind: SecurityIntentBinding
metadata:
  name: package-mgrs-binding
status:
  nimbusPolicy: ""
  countOfBoundIntents: 0
  status: Created
```

Run the test:

```bash
$ chainsaw test
...
Loading tests...
- securityintent-deletion-after-creation-of-nimbuspolicy (.)
...
...
    | 14:34:59 | securityintent-deletion-after-creation-of-nimbuspolicy | Assert NimbusPolicy deletion                      | SCRIPT    | RUN   |
        === COMMAND
        /bin/sh -c kubectl get np package-mgrs-binding -n $NAMESPACE
    | 14:34:59 | securityintent-deletion-after-creation-of-nimbuspolicy | Assert NimbusPolicy deletion                      | SCRIPT    | LOG   |
        === STDOUT
        NAME                   STATUS    AGE   POLICIES
        package-mgrs-binding   Created   0s
    | 14:34:59 | securityintent-deletion-after-creation-of-nimbuspolicy | Assert NimbusPolicy deletion                      | SCRIPT    | ERROR |
        === ERROR
        ($error != null): Invalid value: false: Expected value: true
...
...
--- FAIL: chainsaw (0.00s)
    --- FAIL: chainsaw/securityintent-deletion-after-creation-of-nimbuspolicy (5.50s)
FAIL
Tests Summary...
- Passed  tests 0
- Failed  tests 1
- Skipped tests 0
Done with failures.
Error: some tests failed
```

This test failed because the `NimbusPolicy` was not deleted when the referenced `SecurityIntent` was deleted.

## Update

Create `chainsaw-test.yaml` test file in `test/controller/edge-cases/update` directory as follows:

```yaml
apiVersion: chainsaw.kyverno.io/v1alpha1
kind: Test
metadata:
  name: securityintentbinding-and-securityintent-update
spec:
  description: |
    This test verifies that modifying a SecurityIntent triggers the updates in corresponding SecurityIntentBinding's 
    status subresource and related NimbusPolicy.

  steps:
    - name: Create a SecurityIntent
      try:
        - apply:
            file: ../../manifest/security-intent.yaml

    - name: Create a SecurityIntentBinding
      try:
        - apply:
            file: ../../manifest/security-intent-binding.yaml

    - name: Assert SecurityIntentBinding's status subresource
      description: |
        Verify the created SecurityIntentBinding status subresource includes the number and names of bound intents, 
        along with the generated NimbusPolicy name.
      try:
        - assert:
            file: ../sib-status-to-assert.yaml

    - name: Assert NimbusPolicy creation
      try:
        - assert:
            file: ../nimbus-policy-to-assert.yaml

    - name: Update referenced SecurityIntent
      try:
        - patch:
            file: updated-security-intent.yaml

    - name: Assert changes to NimbusPolicy
      try:
        - assert:
            file: updated-nimbus-policy-to-assert.yaml
```

* `updated-security-intent.yaml` in the same directory as the test:
    

```yaml
apiVersion: intent.security.nimbus.com/v1alpha1
kind: SecurityIntent
metadata:
  name: package-mgrs
spec:
  intent:
    action: Audit # Changed the action from `Enforce` to `Audit`
```

* `updated-nimbus-policy-to-assert.yaml` in the same directory as the test:
    

```yaml
apiVersion: intent.security.nimbus.com/v1alpha1
kind: NimbusPolicy
metadata:
  name: package-mgrs-binding
  ownerReferences:
    - apiVersion: intent.security.nimbus.com/v1alpha1
      blockOwnerDeletion: true
      controller: true
      kind: SecurityIntentBinding
      name: package-mgrs-binding
spec:
  rules:
    - action: Audit
      id: pkgMgrs
  selector:
    matchLabels:
      app: web
      env: prod
```

Run the test:

```bash
$ chainsaw test
Loading tests...
- securityintentbinding-and-securityintent-update (.)
...
...
    | 14:56:43 | securityintentbinding-and-securityintent-update | Assert changes to NimbusPolicy                    | ASSERT    | ERROR | intent.security.nimbus.com/v1alpha1/NimbusPolicy @ chainsaw-enough-anchovy/package-mgrs-binding
        === ERROR
        ---------------------------------------------------------------------------------------------
        intent.security.nimbus.com/v1alpha1/NimbusPolicy/chainsaw-enough-anchovy/package-mgrs-binding
        ---------------------------------------------------------------------------------------------
        * spec.rules[0].action: Invalid value: "Enforce": Expected value: "Audit"
        
        --- expected
        +++ actual
        @@ -9,9 +9,10 @@
             controller: true
             kind: SecurityIntentBinding
             name: package-mgrs-binding
        +    uid: 827ce9e7-a03c-4e9f-8728-be13090a9b76
         spec:
           rules:
        -  - action: Audit
        +  - action: Enforce
             id: pkgMgrs
           selector:
             matchLabels:
...
...
   --- FAIL: chainsaw (0.00s)
    --- FAIL: chainsaw/securityintentbinding-and-securityintent-update (35.51s)
FAIL
Tests Summary...
- Passed  tests 0
- Failed  tests 1
- Skipped tests 0
Done with failures.
Error: some tests failed
```

This test failed because updating the referenced `SecurityIntent` did not trigger any updates.

The importance of testing is clear in this instance. Our tests uncovered several bugs in the operator. While the core functionalities (likely represented by the basic cases) were working correctly, however, the edge tests designed for less common scenarios were failing and revealed issues.

Let's fix these bugs.

# Bugs

## Create & Update

Let's fix the first issue, i.e., independent creation of resources.

If we think about how SecurityIntentBinding works, we'll see it depends on SecurityIntent. It only creates NimbusPolicy if the referenced SecurityIntent exists. We can't just say to create SecurityIntent first and then SecurityIntentBinding, that's not how the real world works. So, how can we fix this?

The solution is to configure SecurityIntentBinding's controller to watch SecurityIntent's CRUD operations. This allows the controller to react appropriately based on changes to SecurityIntents. Let's get into business.

Edit the `securityintentbinding_controller.go` file in the `internal/controller` directory as follows:

```go
...
...
// SetupWithManager sets up the controller with the Manager.
func (r *SecurityIntentBindingReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&intentv1alpha1.SecurityIntentBinding{}).
		Owns(&intentv1alpha1.NimbusPolicy{}).
		Watches(&intentv1alpha1.SecurityIntent{},
			handler.EnqueueRequestsFromMapFunc(r.findBindingsMatchingWithIntent),
		).
		Complete(r)
}
...
...

// findBindingsMatchingWithIntent finds SecurityIntentBindings that reference given SecurityIntent.
func (r *SecurityIntentBindingReconciler) findBindingsMatchingWithIntent(ctx context.Context, securityIntent client.Object) []reconcile.Request {
	logger := log.FromContext(ctx)
	var requests []reconcile.Request

	var bindings intentv1alpha1.SecurityIntentBindingList
	if err := r.List(ctx, &bindings); err != nil {
		logger.Error(err, "failed to list SecurityIntentBinding")
		return requests
	}

	for _, binding := range bindings.Items {
		for _, intent := range binding.Spec.Intents {
			if intent.Name == securityIntent.GetName() {
				requests = append(requests, reconcile.Request{
					NamespacedName: types.NamespacedName{
						Name:      binding.Name,
						Namespace: binding.Namespace,
					},
				})
                break
			}
		}
	}

	return requests
}
```

Here we configured the controller to also watch for create, update, and delete events of the SecurityIntent object. We provided an event handler callback `findBindingsMatchingWithIntent` that returns SecurityIntentBindings referencing the given intent to reconcile using the `Reconcile` method.

Let's re-run the `test/controller/edge-cases/create` test case to verify the independent resource creation.

Restart the operator:

```bash
$ make run
```

Re-run the `create` test (in a new terminal):

```bash
$ chainsaw test
...
...
--- PASS: chainsaw (0.00s)
    --- PASS: chainsaw/securityintentbinding-and-securityintent-independent-creation (16.11s)
PASS
Tests Summary...
- Passed  tests 1
- Failed  tests 0
- Skipped tests 0
Done.
```

Awesome! The test passed. Pat yourself on the back for fixing this bug.

Let's also run the `update` edge case test in the `test/controller/edge-cases/update` directory:

```bash
$ chainsaw test
...
...
--- PASS: chainsaw (0.00s)
    --- PASS: chainsaw/securityintentbinding-and-securityintent-update (5.55s)
PASS
Tests Summary...
- Passed  tests 1
- Failed  tests 0
- Skipped tests 0
Done.
```

Wow! You fixed two bugs at once. Give yourself another pat on the back.

## Delete

Before fixing this let's run all the tests, to make sure that we haven't broken anything.

```bash
# From root directory of our operator
$ make test
...
...
--- FAIL: chainsaw (0.00s)
    --- PASS: chainsaw/securityintentbinding-and-securityintent-independent-creation (5.40s)
    --- PASS: chainsaw/securityintent-creation (5.24s)
    --- PASS: chainsaw/securityintentbinding-update (5.49s)
    --- PASS: chainsaw/securityintentbinding-delete (5.52s)
    --- PASS: chainsaw/securityintentbinding-creation (5.36s)
    --- PASS: chainsaw/nimbuspolicy-update (5.43s)
    --- PASS: chainsaw/securityintentbinding-and-securityintent-update (5.50s)
    --- FAIL: chainsaw/securityintent-deletion-after-creation-of-nimbuspolicy (5.51s)
    --- PASS: chainsaw/nimbuspolicy-delete (5.36s)
FAIL
Tests Summary...
- Passed  tests 8
- Failed  tests 1
- Skipped tests 0
Done with failures.
Error: some tests failed
make: *** [test] Error 1
```

One test failed, but don't worry, we'll fix it.

First, let's break down the issue and figure out how to fix it.

**Issue:** If the referenced SecurityIntents don't exist in the cluster or are deleted, then stop creating NimbusPolicy or delete the existing NimbusPolicy.

Further breakdown:

* If a SecurityIntentBinding references SecurityIntents that don't exist in the cluster, stop creating `NimbusPolicy`.
    
* If there is only one referenced SecurityIntent and it gets deleted, then delete the existing related `NimbusPolicy`.
    

Now the problem statement is much clearer, so let's write code to fix this.

Since SecurityIntentBinding's controller is responsible for NimbusPolicy management, so edit the `securityintentbinding_controller.go` file in the `internal/controller` directory as follows:

```go
func (r *SecurityIntentBindingReconciler) createOrUpdateNimbusPolicy(ctx context.Context, securityIntentBinding intentv1alpha1.SecurityIntentBinding) (*intentv1alpha1.NimbusPolicy, error) {
    ...
    nimbusPolicyToCreate, err := builder.BuildNimbusPolicy(ctx, r.Client, securityIntentBinding)
	if err != nil {
		if errors.Is(err, buildererrors.ErrSecurityIntentsNotFound) {
			// Since the SecurityIntent(s) referenced in SecurityIntentBinding spec don't
			// exist, so delete NimbusPolicy if it exists.
			if err = r.deleteNimbusPolicyIfExists(ctx, securityIntentBinding.Name, securityIntentBinding.Namespace); err != nil {
				return nil, err
			}

			// When a NimbusPolicy is deleted, it implies the referenced SecurityIntent(s)
			// is(are) no longer exist. Therefore, update the status subresource of the
			// associated SecurityIntentBinding to reflect the latest details.
			if err = r.removeNpAndSisDetailsFromSibStatus(ctx, securityIntentBinding.Name, securityIntentBinding.Namespace); err != nil {
				return nil, err
			}

			return nil, nil
		}
		return nil, err
	}
    ...
}

func (r *SecurityIntentBindingReconciler) deleteNimbusPolicyIfExists(ctx context.Context, name, namespace string) error {
	var nimbusPolicyToDelete intentv1alpha1.NimbusPolicy
	if err := r.Get(ctx, types.NamespacedName{Name: name, Namespace: namespace}, &nimbusPolicyToDelete); err != nil {
		if apierrors.IsNotFound(err) {
			return nil
		}
		return err
	}

	if err := r.Delete(ctx, &nimbusPolicyToDelete); err != nil {
		return err
	}

	return nil
}

func (r *SecurityIntentBindingReconciler) removeNpAndSisDetailsFromSibStatus(ctx context.Context, bindingName, namespace string) error {
	var securityIntentBinding intentv1alpha1.SecurityIntentBinding
	if err := r.Get(ctx, types.NamespacedName{Name: bindingName, Namespace: namespace}, &securityIntentBinding); err != nil {
		return err
	}

	securityIntentBinding.Status.NimbusPolicy = ""
	securityIntentBinding.Status.CountOfBoundIntents = 0
	securityIntentBinding.Status.BoundIntents = []string{}

	return r.Status().Update(ctx, &securityIntentBinding)
}
```

Now re-run the `delete` edge-case test (restart operator):

```bash
$ chainsaw test
...
...
Loading tests...
- securityintent-deletion-after-creation-of-nimbuspolicy (.)
...
...
    | 21:22:21 | securityintent-deletion-after-creation-of-nimbuspolicy | Assert NimbusPolicy deletion                      | SCRIPT    | RUN   |
        === COMMAND
        /bin/sh -c kubectl get np package-mgrs-binding -n $NAMESPACE
    | 21:22:21 | securityintent-deletion-after-creation-of-nimbuspolicy | Assert NimbusPolicy deletion                      | SCRIPT    | LOG   |
        === STDERR
        Error from server (NotFound): nimbuspolicies.intent.security.nimbus.com "package-mgrs-binding" not found
    | 21:22:21 | securityintent-deletion-after-creation-of-nimbuspolicy | Assert NimbusPolicy deletion                      | SCRIPT    | DONE  |
    | 21:22:21 | securityintent-deletion-after-creation-of-nimbuspolicy | Assert NimbusPolicy deletion                      | TRY       | DONE  |
    | 21:22:21 | securityintent-deletion-after-creation-of-nimbuspolicy | Assert SecurityIntentBinding's status subresource | TRY       | RUN   |
    | 21:22:21 | securityintent-deletion-after-creation-of-nimbuspolicy | Assert SecurityIntentBinding's status subresource | ASSERT    | RUN   | intent.security.nimbus.com/v1alpha1/SecurityIntentBinding @ chainsaw-relaxing-skylark/package-mgrs-binding
    | 21:22:51 | securityintent-deletion-after-creation-of-nimbuspolicy | Assert SecurityIntentBinding's status subresource | ASSERT    | ERROR | intent.security.nimbus.com/v1alpha1/SecurityIntentBinding @ chainsaw-relaxing-skylark/package-mgrs-binding
        === ERROR
        ---------------------------------------------------------------------------------------------------
        intent.security.nimbus.com/v1alpha1/SecurityIntentBinding/chainsaw-cool-mammal/package-mgrs-binding
        ---------------------------------------------------------------------------------------------------
        * status.countOfBoundIntents: Invalid value: "null": Expected value: 0
        * status.nimbusPolicy: Invalid value: "null": Expected value: ""
        
        --- expected
        +++ actual
        @@ -4,7 +4,5 @@
           name: package-mgrs-binding
           namespace: chainsaw-cool-mammal
         status:
        -  countOfBoundIntents: 0
        -  nimbusPolicy: ""
           status: Created
    | 21:22:51 | securityintent-deletion-after-creation-of-nimbuspolicy | Assert SecurityIntentBinding's status subresource | TRY       | DONE  |
...
...
--- FAIL: chainsaw (0.00s)
    --- FAIL: chainsaw/securityintent-deletion-after-creation-of-nimbuspolicy (35.43s)
FAIL
Tests Summary...
- Passed  tests 0
- Failed  tests 1
- Skipped tests 0
Done with failures.
Error: some tests failed
```

This test is still failing because of an error in the status subresource fields. The error shows that the fields have null values.

Do you know where the problem is? Think about it before continuing, I'll wait for you.

This issue is due to the `SecurityIntentBindingStatus` struct's fields in the `securityintentbinding_types.go` file located in the `api/v1alpha1` directory, specifically in the `CountOfBoundIntents` and `NimbusPolicy` fields.

```go
// SecurityIntentBindingStatus defines the observed state of SecurityIntentBinding
type SecurityIntentBindingStatus struct {
    ...
	CountOfBoundIntents int32    `json:"countOfBoundIntents,omitempty"`
	NimbusPolicy        string   `json:"nimbusPolicy,omitempty"`
}
```

I told you where the issue is, but do you know what the issue is? Think about it and then continue.

The problem is with the JSON tags, specifically the `omitempty` tag option. This means if the field's value is the zero value for its type, it will be omitted from the JSON output. Now you know what exactly the issue is, so let's fix it.

Update the struct as follows:

```go
// SecurityIntentBindingStatus defines the observed state of SecurityIntentBinding
type SecurityIntentBindingStatus struct {
    ...
	CountOfBoundIntents int32    `json:"countOfBoundIntents"`
	NimbusPolicy        string   `json:"nimbusPolicy"`
}
```

Generate and re-install the CRDs:

```bash
make manifests install
```

Finally re-run the test after restarting the operator:

```bash
$ chainsaw test
...
--- PASS: chainsaw (0.00s)
    --- PASS: chainsaw/securityintent-deletion-after-creation-of-nimbuspolicy (5.64s)
PASS
Tests Summary...
- Passed  tests 1
- Failed  tests 0
- Skipped tests 0
Done.
```

Finally, this test passed.

Now let's run all the tests

```bash
# From root directory of operator
$ make test
...
...
--- PASS: chainsaw (0.00s)
    --- PASS: chainsaw/securityintentbinding-and-securityintent-independent-creation (5.41s)
    --- PASS: chainsaw/securityintent-creation (5.24s)
    --- PASS: chainsaw/securityintentbinding-update (5.48s)
    --- PASS: chainsaw/securityintentbinding-delete (5.56s)
    --- PASS: chainsaw/securityintentbinding-creation (5.37s)
    --- PASS: chainsaw/nimbuspolicy-delete (5.37s)
    --- PASS: chainsaw/nimbuspolicy-update (5.44s)
    --- PASS: chainsaw/securityintentbinding-and-securityintent-update (5.48s)
    --- PASS: chainsaw/securityintent-deletion-after-creation-of-nimbuspolicy (5.59s)
PASS
Tests Summary...
- Passed  tests 9
- Failed  tests 0
- Skipped tests 0
Done.
```

All the tests have passed, and we haven't broken anything.

The last edge case was nontrivial, but we fixed it. So, you have to give yourself a pat on the back.

Here's the key: by identifying and handling edge cases without even realizing it, we were actually following the principles of Test-Driven Development (TDD). In TDD, you write tests that define the expected behaviour of your code before you write the actual code itself.

# Event Filters

There is one last issue: the `SecurityIntentBinding` controller gets unnecessary events. For example, it gets events when a `SecurityIntentBinding` or `SecurityIntent` is created, and when their spec or status subresource is updated. We don't need the update event from the status subresource. So let's fix this so the controller only gets relevant events.

Edit the `securityintentbinding_controller.go` file in the `internal/controller` directory as follows:

```go
...
// SetupWithManager sets up the controller with the Manager.
func (r *SecurityIntentBindingReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
        ...
        ...
		WithEventFilter(predicate.GenerationChangedPredicate{}).
		Complete(r)
}
...
```

Edit the `securityintent_controller.go` file in the `internal/controller` directory as follows:

```go
...
// SetupWithManager sets up the controller with the Manager.
func (r *SecurityIntentReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&intentv1alpha1.SecurityIntent{}).
		WithEventFilter(predicate.GenerationChangedPredicate{}).
		Complete(r)
}
...
```

Here we set up the controllers to use the [GenerationChangedPredicate](https://github.com/kubernetes-sigs/controller-runtime/blob/1ed345090869edc4bd94fe220386cb7fa5df745f/pkg/predicate/predicate.go#L183) event filter. This filter ignores events that aren't caused by changes to the generation field of resources. A resource's generation only increases when its spec field is changed, which is the change we care about.

Take your time to understand this information. In the next part, we'll delve into operator adapters and plugins. Stay tuned!

You can find the complete code [here](https://github.com/anurag-rajawat/tutorials/tree/main/nimbus). Please feel free to comment or criticize :)

# Summary

> This article demonstrates how to write declarative tests for Kubernetes operators and controllers using Kyverno's Chainsaw. It covers creating and organizing test directories and files, writing and breaking down declarative tests, and running them. Key focus areas include testing resource creation, update, and deletion, handling edge cases, and ensuring core functionalities remain intact. Detailed code examples and explanations help in identifying and fixing bugs using a declarative approach. The article also explores event filters to optimize controller performance and concludes with the importance of Test-Driven Development (TDD).

# **References**

* [**5GSec Nimbus**](https://github.com/5GSEC/nimbus)
    
* [Kyverno Chainsaw](https://kyverno.github.io/chainsaw/latest/)