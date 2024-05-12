---
title: "Integrating Helm Hooks with HashiCorp Vault for Cloud Migrations"
date: 2024-05-12
author: "Matt Popa"
---

![helm_hooks](/images/helm_hooks.jpg)

## Managing Secrets in Kubernetes with Helm and HashiCorp Vault

In the dynamic landscape of cloud migrations, securely and efficiently managing application 
configurations, particularly secrets, is a critical task. This article explores a practical scenario 
where an organization is migrating its applications from on-premises infrastructure to Amazon EKS, 
using HashiCorp Vault's CSI driver to handle secrets. Here, we'll dive into the challenges and solutions 
of using environment variables to access these secrets—a common yet sensitive practice—and provide a 
step-by-step guide to implementing robust solutions using Helm hooks.

## Context: Migrating to EKS with Vault

As organizations shift to cloud environments like Amazon EKS, managing secrets becomes a focal point 
of security and operational efficiency. For this migration, we're utilizing HashiCorp Vault — pretty
much the de facto tool for managing secrets. Initially, applications will access these secrets 
through environment variables, a widely adopted method due to its straightforward implementation,
even though it's not the most secure.

## Challenges in Secret Management

One major hurdle in this migration is ensuring that updates to secrets in Vault propagate promptly 
to Kubernetes, maintaining the seamless functionality of applications. The typical challenge here is 
that when a secret is updated in Vault, the corresponding secret in Kubernetes doesn't automatically 
update, and deleting the Kubernetes secret doesn’t trigger its recreation by `secretStorageClass` &
Vault's CSI driver.

## Tactical Fix: Using Helm Hooks

To address the immediate challenge without refactoring the application, we employ Helm hooks to 
manage the lifecycle of secrets. Helm hooks allow us to execute scripts at various points in the 
application deployment process, which we use here to ensure secrets are correctly managed.

### Permissions Setup with RBAC:

First, ensure the Helm hook has the necessary permissions by setting up RBAC:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: spc-manager

---

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: spc-role
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list", "delete", "create"]
- apiGroups: ["secrets-store.csi.x-k8s.io"]
  resources: ["secretstorageclass"]
  verbs: ["get", "list", "delete", "create"]

---

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: spc-role-binding
subjects:
- kind: ServiceAccount
  name: spc-manager
roleRef:
  kind: Role
  name: spc-role
  apiGroup: rbac.authorization.k8s.io
```

### Pre-Upgrade/Install Hook: Deleting the Secret and SecretStorageClass

We're using pre-upgrade and pre-install hooks because typically we're using `helm upgrade --install`
in our CIs to deploy applications, and we want to cover both cases.

This hook triggers before Helm upgrades or installs, ensuring old secrets are cleared before new 
configurations apply:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ .Release.Name }}-cleanup-resources"
  annotations:
    "helm.sh/hook": "pre-install,pre-upgrade"
    "helm.sh/hook-weight": "-10"
    "helm.sh/hook-delete-policy": "before-hook-creation,hook-succeeded"
spec:
  template:
    spec:
      serviceAccountName: spc-manager
      restartPolicy: OnFailure
      containers:
      - name: kubectl
        image: bitnami/kubectl
        command: ["/bin/sh", "-c", "kubectl delete secretstorageclass example-name --ignore-not-found; kubectl delete secret example-secret-name --ignore-not-found"]
```

## Strategic Solutions and Best Practices

While the immediate solution involves using Helm hooks, the recommended long-term approach is to 
integrate secrets directly into pods using the `secretStorageClass`. This method enhances security 
by avoiding intermediate storage of sensitive data in Kubernetes `Secret` objects. For a deeper dive 
into setting this up, refer to HashiCorp's [guide on injecting secrets into Kubernetes pods](https://developer.hashicorp.com/vault/tutorials/vault-agent/agent-env-vars).

### Understanding SecretStorageClass Behavior

* Secret Creation: The SecretStorageClass in conjunction with a CSI driver like the HashiCorp Vault 
CSI driver should dynamically inject secrets into the pods based on the configuration specified. 
However, this typically involves mounting secrets as volumes rather than creating standalone 
Kubernetes Secret objects.
* Secret Regeneration: When using a SecretStorageClass that relies on a CSI driver, the actual 
Kubernetes Secret object isn't automatically recreated when deleted unless there's specific logic to
do so. The CSI driver mainly manages the lifecycle of secrets inside pod volumes directly.

If possible the application should be refactored to use the secrets mounted directly into the pod.
This way we don't have to restart the containers/apps when the secrets are updated, and it's safer.

There's also the alternative to using [Vault Operator](https://developer.hashicorp.com/vault/tutorials/kubernetes/vault-secrets-operator), 
which facilitates integration with Kubernetes Authentication.

## Conclusion: Securing the Migration Path

By implementing these solutions, we ensure not only that our application functions seamlessly in 
its new cloud environment but also pave the way for adopting more secure secret management practices. 
The strategic use of Helm hooks here illustrates a powerful method to bridge temporary gaps during
migration, setting the stage for more advanced configurations in the future.