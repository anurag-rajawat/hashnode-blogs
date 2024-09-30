---
title: "Building a Real-world Kubernetes Operator: Part 6"
seoTitle: "Kubernetes Operator: Implementation Part 6"
seoDescription: "Establish end-to-end tests for the Nimbus suite using Kyverno's Chainsaw, including creation, update, and deletion workflows."
datePublished: Mon Sep 30 2024 04:34:10 GMT+0000 (Coordinated Universal Time)
cuid: cm1oim7ql000u08jz8ohj1hcz
slug: building-a-real-world-kubernetes-operator-part-6
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1724593197483/7eac19b5-32c5-47b2-b804-0b636670dfcd.png
tags: kubernetes, kubernetes-operators

---

# **Introduction**

We've made significant progress, and in this section, we'll establish end-to-end tests for the entire Nimbus suite using a declarative methodology, mirroring our approach to integration testing. Kyverno's [**Chainsaw**](https://kyverno.github.io/chainsaw/latest/) will be our tool for implementing declarative end-to-end tests across the Nimbus suite, creating one test case for each SecurityIntent SecurityIntent.

In our specific context, end-to-end testing involves the following steps:

* **Create:** When a SecurityIntent and SecurityIntentBinding are created, Nimbus generates a corresponding NimbusPolicy. Subsequently, the Nimbus adapter translates this NimbusPolicy into concrete policy rules to create the necessary policies.
    
* **Update:**
    
    * When a SecurityIntent or SecurityIntentBinding is updated, Nimbus updates the corresponding NimbusPolicy. The Nimbus adapter then updates the corresponding policies accordingly.
        
    * If an adapter policy is manually altered for any reason, those changes are discarded.
        
* **Delete:**
    
    * When a SecurityIntent or SecurityIntentBinding is deleted, Nimbus deletes the corresponding NimbusPolicy. The Nimbus adapter follows by deleting the associated policies.
        
    * If an adapter policy is deleted manually, it is automatically recreated.
        

Let's get started.

# Setup

Create an `e2e` new sub-directory in the `test` directory to keep all e2e tests.

```bash
mkdir -p test/e2e
```

Again create some new sub-directories according to the intent in the `test/e2e` directory. We'll organize tests in separate directories based on their operation, for example, `create`, `update` and `delete`.

```bash
mkdir -p test/e2e/pkg-mgr-execution/{create,update,delete}
```

# Create

Create and edit the `chainsaw-test.yaml` file in `test/e2e/pkg-mgr-execution/create` directory as follows:

```yaml
apiVersion: chainsaw.kyverno.io/v1alpha1
kind: Test
metadata:
  name: pkg-mgr-exec-intent-binding-creation
spec:
  description: |
    This test validates when a SecurityIntent and SecurityIntentBinding are
    created, Nimbus generates a corresponding NimbusPolicy. Subsequently, the
    Nimbus adapter translates this NimbusPolicy into concrete policy rules to
    create the necessary policies.

  steps:
    - name: Create PacakgeManagerExecution SecurityIntent and SecurityIntentBinding
      try:
        - apply:
            file: ../../../controller/manifest/security-intent.yaml
        - apply:
            file: ../../../controller/manifest/security-intent-binding.yaml

    - name: Assert SecurityIntentBinding's status subresource
      try:
        - assert:
            file: ../sib-status.yaml

    - name: Assert NimbusPolicy creation
      try:
        - assert:
            file: ../nimbus-policy.yaml

    - name: Assert KubeArmor policy creation
      try:
        - assert:
            file: ../ksp.yaml

    - name: Assert NimbusPolicy's status subresource
      description: |
        Verify the NimbusPolicy's status subresource includes the number and names of
        created security engine policies.
      try:
        - assert:
            file: ../np-status.yaml
```

The test explains itself, so I don't think it needs further explanation.

Let's create the manifests used by the test in the `test/e2e/pkg-mgr-execution` directory.

* Create a `sib-status.yaml` file as follows:
    
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
    
* Create a `nimbus-policy.yaml` file as follows:
    
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
    
* Create a `ksp.yaml` file as follows:
    
    ```yaml
    apiVersion: security.kubearmor.com/v1
    kind: KubeArmorPolicy
    metadata:
      labels:
        app.kubernetes.io/managed-by: nimbus-kubearmor
        app.kubernetes.io/part-of: package-mgrs-binding-nimbuspolicy
      name: package-mgrs-binding-pkgmgrs
      ownerReferences:
        - apiVersion: intent.security.nimbus.com/v1alpha1
          blockOwnerDeletion: true
          controller: true
          kind: NimbusPolicy
          name: package-mgrs-binding
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
    
* Create a `np-status.yaml` file as follows:
    
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
    status:
      status: Created
      policies: 1
      policiesName:
        - package-mgrs-binding-pkgmgrs
    ```
    

The e2e test workflow needs the full Nimbus suite, including the Nimbus operator and its adapters. Remember to install the security engine CRDs, like KubeArmor CRDs. You can do this by installing the engine or directly installing the CRDs.

Run the operator and its adapter and test in separate terminals:

```bash
$ cd test/e2e/pkg-mgr-execution/create
$ chainsaw test
...
...
--- PASS: chainsaw (0.00s)
    --- PASS: chainsaw/pkg-mgr-exec-intent-binding-creation (30.49s)
PASS
Tests Summary...
- Passed  tests 1
- Failed  tests 0
- Skipped tests 0
Done.
```

Awesome! The test passed on the first try.

# Update

Create and edit the `chainsaw-test.yaml` file in `test/e2e/pkg-mgr-execution/update` directory as follows:

```yaml
apiVersion: chainsaw.kyverno.io/v1alpha1
kind: Test
metadata:
  name: pkg-mgr-exec-intent-binding-update
spec:
  description: |
    This test validates the following:
    - If a SecurityIntent or SecurityIntentBinding is updated, Nimbus updates the
      corresponding NimbusPolicy. The Nimbus adapter then updates the corresponding
      policies accordingly.

    - If an adapter policy is manually altered for any reason, those changes are
      discarded.

  steps:
    - name: Create PacakgeManagerExecution SecurityIntent and SecurityIntentBinding
      try:
        - apply:
            file: ../../../controller/manifest/security-intent.yaml
        - apply:
            file: ../../../controller/manifest/security-intent-binding.yaml

    - name: Assert SecurityIntentBinding's status subresource
      try:
        - assert:
            file: ../sib-status.yaml

    - name: Assert NimbusPolicy creation
      try:
        - assert:
            file: ../nimbus-policy.yaml

    - name: Assert KubeArmor policy creation
      try:
        - assert:
            file: ../ksp.yaml

    - name: Assert NimbusPolicy's status subresource
      description: |
        Verify the NimbusPolicy's status subresource includes the number and names of
        created security engine policies.
      try:
        - assert:
            file: ../np-status.yaml

    - name: Update referenced SecurityIntent
      try:
        - patch:
            file: updated-security-intent.yaml

    - name: Assert NimbusPolicy changes
      try:
        - assert:
            file: updated-nimbus-policy-to-assert.yaml

    - name: Assert KubeArmor policy changes
      try:
        - assert:
            file: updated-ksp-to-assert.yaml

    - name: Manually update the managed KubeArmor policy
      try:
        - patch:
            file: updated-ksp.yaml

    - name: Assert manual changes to KubeArmor policy are discarded
      try:
        - assert:
            file: updated-ksp-to-assert.yaml
```

Again the test explains itself, so I don't think it needs further explanation.

Let's create the manifests used by the test in the `test/e2e/pkg-mgr-execution/update` directory.

* Create `updated-security-intent.yaml` file as follows:
    
    ```yaml
    apiVersion: intent.security.nimbus.com/v1alpha1
    kind: SecurityIntent
    metadata:
      name: package-mgrs
    spec:
      intent:
        action: Audit # Changed from `Enforce` to `Audit`
    ```
    
* Create `updated-nimbus-policy-to-assert.yaml` file as follows:
    
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
    
* Create `updated-ksp-to-assert.yaml` file as follows:
    
    ```yaml
    apiVersion: security.kubearmor.com/v1
    kind: KubeArmorPolicy
    metadata:
      labels:
        app.kubernetes.io/managed-by: nimbus-kubearmor
        app.kubernetes.io/part-of: package-mgrs-binding-nimbuspolicy
      name: package-mgrs-binding-pkgmgrs
      ownerReferences:
        - apiVersion: intent.security.nimbus.com/v1alpha1
          blockOwnerDeletion: true
          controller: true
          kind: NimbusPolicy
          name: package-mgrs-binding
    spec:
      capabilities: {}
      file: {}
      message: Alert! Execution of package management process inside container is detected
      network: {}
      process:
        action: Audit
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
    
* Create `updated-ksp.yaml` file as follows:
    
    ```yaml
    apiVersion: security.kubearmor.com/v1
    kind: KubeArmorPolicy
    metadata:
      name: package-mgrs-binding-pkgmgrs
    spec:
      process:
        action: Allow # Changed to Audit from Block
    ```
    

Let's run the test:

```bash
$ cd test/e2e/pkg-mgr-execution/update
$ chainsaw test
...
...
--- PASS: chainsaw (0.00s)
    --- PASS: chainsaw/pkg-mgr-exec-intent-binding-update (6.20s)
PASS
Tests Summary...
- Passed  tests 1
- Failed  tests 0
- Skipped tests 0
Done.
```

Once again this test also passed on the first try.

# Delete

Create and edit the `chainsaw-test.yaml` file in `test/e2e/pkg-mgr-execution/delete` directory as follows:

```yaml
apiVersion: chainsaw.kyverno.io/v1alpha1
kind: Test
metadata:
  name: pkg-mgr-exec-intent-binding-update
spec:
  description: |
    This test validates the following:
    - If an adapter policy is deleted manually, it is automatically recreated.

    - When a SecurityIntent or SecurityIntentBinding is deleted, Nimbus deletes the
      corresponding NimbusPolicy. The Nimbus adapter follows by deleting the
      associated policies.

  steps:
    - name: Create PacakgeManagerExecution SecurityIntent and SecurityIntentBinding
      try:
        - apply:
            file: ../../../controller/manifest/security-intent.yaml
        - apply:
            file: ../../../controller/manifest/security-intent-binding.yaml

    - name: Assert SecurityIntentBinding's status subresource
      try:
        - assert:
            file: ../sib-status.yaml

    - name: Assert NimbusPolicy creation
      try:
        - assert:
            file: ../nimbus-policy.yaml

    - name: Assert KubeArmor policy creation
      try:
        - assert:
            file: ../ksp.yaml

    - name: Assert NimbusPolicy's status subresource
      description: |
        Verify the NimbusPolicy's status subresource includes the number and names of
        created security engine policies.
      try:
        - assert:
            file: ../np-status.yaml

    - name: Manually delete the managed KubeArmor policy
      try:
        - delete:
            file: ../ksp.yaml

    - name: Assert managed KubeArmor policy re-creation
      try:
        - assert:
            file: ../ksp.yaml

    - name: Delete the created SecurityIntent and SecurityIntentBinding
      try:
        - delete:
            file: ../../../controller/manifest/security-intent.yaml
        - delete:
            file: ../../../controller/manifest/security-intent-binding.yaml

    - name: Assert the NimbusPolicy deletion
      try:
        - script:
            content: kubectl -n $NAMESPACE get np package-mgrs-binding
            check:
              ($error != null): true

    - name: Assert the managed KubeArmor policy deletion
      try:
        - script:
            content: kubectl get ksp package-mgrs-binding-pkgmgrs -n $NAMESPACE
            check:
              ($error != null): true
```

You know what to do next.

Let's create a make target to run all the e2e tests, edit the Nimbus `Makefile` as follows:

```makefile
.PHONY: test-e2e
test-e2e: chainsaw ## Run the e2e tests.
	@$(LOCALBIN)/chainsaw test --test-dir=test/e2e --config test/chainsaw-config.yaml
```

Run the e2e test:

```bash
$ make test-e2e
...
...
--- PASS: chainsaw (0.00s)
    --- PASS: chainsaw/pkg-mgr-exec-intent-binding-creation (5.90s)
    --- PASS: chainsaw/pkg-mgr-exec-intent-binding-update#01 (5.91s)
    --- PASS: chainsaw/pkg-mgr-exec-intent-binding-update (6.34s)
PASS
Tests Summary...
- Passed  tests 3
- Failed  tests 0
- Skipped tests 0
Done.
```

Awesome! All tests passed, generating colourful output. You deserve a pat on your back.

In this part, we implemented an e2e test for a single intent. The process will be similar for future intents. I hope you now understand how to create end-to-end tests for all intents.

That's it for this part. In the next part, we'll create Helm charts for Nimbus and its adapter. Stay tuned!

You can find the complete code [here](https://github.com/anurag-rajawat/tutorials/tree/main/nimbus). Please feel free to comment or criticize :)

# Summary

> In this article, we detail the process of establishing end-to-end tests for the Nimbus suite using a declarative methodology with Kyverno's Chainsaw. We outline the steps for creating, updating, and deleting SecurityIntent and SecurityIntentBindings, and demonstrate how to validate the corresponding NimbusPolicy and adapter policies. The article also includes practical steps, test cases, and scripts to ensure accurate and effective testing, culminating in a successful integration of end-to-end testing across the Nimbus suite.

# References

* [**5GSec Nim**](https://github.com/5GSEC/nimbus)[**bus**](https://github.com/anurag-rajawat/tutorials/tree/main/nimbus)
    
* [**Kyverno Chainsaw**](https://kyverno.github.io/chainsaw/latest/)