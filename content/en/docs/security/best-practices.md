---
title: "Security Best Practices"
linkTitle: "Best Practices"
description: "Best practices for securing Flux deployments."
weight: 140
---

## Introduction

The Flux project strives to keep its components secure by design and by default.
This document aims to list all security-sensitive options or considerations that
must be taken into account when deploying Flux. And also serve as a guide for
security professionals auditing such deployments.

Not all recommendations are required for a secure deployment. Some may impact the
convenience, performance or resources utilisation of Flux. Therefore, use this in
combination with your own Security Posture and Risk Appetite.

Some recommendations may overlap with Kubernetes security recommendations, to keep
this short and more easily maintainable, please refer to [Kubernetes CIS Benchmark] for non
Flux-specific guidance.

## Security Best Practices

The recommendations below are based on Flux latest version.


### Helm Controller

#### Start-up flags

- Ensure controller was not started with `--insecure-kubeconfig-exec=true`.
  <details>
    <summary>Rationale</summary>
    KubeConfigs support the execution of a binary command to return the token required to authenticate against a Kubernetes cluster.

    This is very handy for acquiring contextual tokens that are time-bound (e.g. aws-iam-authenticator).  
    However, this may be open for abuse in multi-tenancy environments and therefore is disabled by default.
  </details>
  <details>
    <summary>Audit Procedure</summary>
    Check Helm Controller's pod YAML for the arguments used at start-up:
    
    `kubectl describe pod -n flux-system -l app=helm-controller | grep -B 5 -A 10 Args`
  </details>

- Ensure controller was not started with `--insecure-kubeconfig-tls=true`.
  <details>
    <summary>Rationale</summary>
    Disable the enforcement of TLS when accessing the API Server of remote clusters.
    
    This flag was created to enable scenarios in which non-production clusters need to be accessed via HTTP. Do not disable TLS in production.
  </details>
  <details>
    <summary>Audit Procedure</summary>
    Check Helm Controller's pod YAML for the arguments used at start-up:
    
    `kubectl describe pod -n flux-system -l app=helm-controller | grep -B 5 -A 10 Args`
  </details>

### Kustomize Controller

#### Start-up flags

- Ensure controller was not started with `--insecure-kubeconfig-exec=true`.
  <details>
    <summary>Rationale</summary>
    KubeConfigs support the execution of a binary command to return the token required to authenticate against a Kubernetes cluster.

    This is very handy for acquiring contextual tokens that are time-bound (e.g. aws-iam-authenticator).  
    However, this may be open for abuse in multi-tenancy environments and therefore is disabled by default.
  </details>
  <details>
    <summary>Audit Procedure</summary>
    Check Kustomize Controller's pod YAML for the arguments used at start-up:
    
    `kubectl describe pod -n flux-system -l app=kustomize-controller | grep -B 5 -A 10 Args`
  </details>

- Ensure controller was not started with `--insecure-kubeconfig-tls=true`.
  <details>
    <summary>Rationale</summary>
    Disable the enforcement of TLS when accessing the API Server of remote clusters.
    
    This flag was created to enable scenarios in which non-production clusters need to be accessed via HTTP. Do not disable TLS in production.
  </details>
  <details>
    <summary>Audit Procedure</summary>
    Check Kustomize Controller's pod YAML for the arguments used at start-up:
    
    `kubectl describe pod -n flux-system -l app=kustomize-controller | grep -B 5 -A 10 Args`
  </details>

- Ensure controller was started with `--no-remote-bases=true`.
  <details>
    <summary>Rationale</summary>
    By default the Kustomize controller allows for kustomize overlays to refer to external bases. 
    This has a performance penalty, as the bases will have to be downloaded on demand during each reconciliation.<br>
    When using external bases, there can't be any assurances that the externally declared state won't change.
    In this case, the source lose its hermetic properties. Changes at the external bases will result on changes in the cluster, regardless of the source being modified since the last reconciliation.
  </details>
  <details>
    <summary>Audit Procedure</summary>
    Check Kustomize Controller's pod YAML for the arguments used at start-up:
    
    `kubectl describe pod -n flux-system -l app=kustomize-controller | grep -B 5 -A 10 Args`
  </details>


#### Secret Decryption

- Ensure Secret Decryption is enabled and secrets are not being held in Flux Sources in plain-text.
  <details>
    <summary>Rationale</summary>
    Helm and Kustomize Controllers have auto decryption mechanisms that can decrypt cipher texts on-demand at reconciliation time. That enables for credentials (e.g. passwords, tokens) and sensitive information to be kept encrypted in the sources.    
  </details>
  <details>
    <summary>Audit Procedure</summary>
    
    - Check for credentials stored in the Git Repository at both HEAD and historical commits. Auto-detection tools can be used for this such as [GitLeaks](https://github.com/zricethezav/gitleaks), [Trufflehog](https://github.com/trufflesecurity/trufflehog) and [Squealer](https://github.com/owenrumney/squealer).
    - Check whether Secret Decryption is properly in each `spec.decryption` field of the cluster's `Kustomization` objects.
  </details>


## Additional Best Practices for Shared Cluster Multi-tenancy

### Multi-tenancy Lock-down

- Ensure `helm-controller`, `kustomize-controller`, `notification-controller`, `image-reflector-controller` and `image-automation-controller` have cross namespace references disabled via `--no-cross-namespace-refs=true`.

  <details>
    <summary>Rationale</summary>
    Block references to Flux objects across namespaces. This assumes that tenants would own one or multiple namespaces, and should not be allowed to consume other tenant's objects, as this could enable them to gain access to sources they do not (or should not) have access to.
  </details>
  <details>
    <summary>Audit Procedure</summary>
    Check the Controller's YAML for the arguments used at start-up:
    
    `kubectl describe pod -n flux-system -l app=helm-controller | grep -B 5 -A 10 Args`
    `kubectl describe pod -n flux-system -l app=kustomize-controller | grep -B 5 -A 10 Args`
    `kubectl describe pod -n flux-system -l app=notification-controller | grep -B 5 -A 10 Args`
    `kubectl describe pod -n flux-system -l app=image-reflector-controller | grep -B 5 -A 10 Args`
    `kubectl describe pod -n flux-system -l app=image-automation-controller | grep -B 5 -A 10 Args`
  </details>

- Ensure `helm-controller` and `kustomize-controller` have a default service account set via `--default-service-account=ServiceAccountName`.

  <details>
    <summary>Rationale</summary>
    Enforces all reconciliations to impersonate a given Service Account, effectively disabling the use of the privileged service account that would otherwise be used by the controller.
    Tenants must set a service account for each object that is responsible for applying changes to the Cluster (i.e. HelmRelease and Kustomization), otherwise Kubernetes' API Server will not authorise the changes. NB: It is recommended that the default service account used has no permissions set to the control plane.
  </details>
  <details>
    <summary>Audit Procedure</summary>
    Check the Controller's YAML for the arguments used at start-up:
    
    `kubectl describe pod -n flux-system -l app=helm-controller | grep -B 5 -A 10 Args`
    `kubectl describe pod -n flux-system -l app=kustomize-controller | grep -B 5 -A 10 Args`
  </details>

### Secret Decryption

- Ensure Secret Decryption is configured correctly, such as each tenant have the correct level of isolation.
  <details>
    <summary>Rationale</summary>
    The secret decryption configuration must be aligned with the level of isolation required across tenants.
    - For higher isolation, each tenant must have their own Key Encryption Key (KEK) configured. Note that the access controls to the aforementioned keys must also be aligned for better isolation.
    - For lower isolation requirements, or for secrets that are shared across multiple tenants, cluster-level keys could be used.
  </details>
  <details>
    <summary>Audit Procedure</summary>
    
    - Check whether the Secret Provider configuration is security hardened. Please seek [SOPS](https://github.com/mozilla/sops) and [SealedSecrets](https://github.com/bitnami-labs/sealed-secrets) documentation for how to best implement each solution.
    - When SealedSecrets are employed, pay special attention to the scopes being used.
  </details>


### Resource Isolation

- Ensure additional Flux instances are deployed when mission critical tenants/workloads must be assured.

  <details>
    <summary>Rationale</summary>
    Sharing the same instances of Flux Components across all tenants and also the Platform Admin, will lead to all reconciliations to compete for the same resources and resource limits. In addition, all Flux objects will be placed on the same queue for reconciliation which is limited by the number of workers set by each controller, which could cause for renconciliation intervals not to be honored.
    For improved reliability, additional instances of Flux Components could be deployed to deal with specific namespaces, effectively creating separate "lanes" that are not disrupted by noisy neighbours. An example of this approach would be having additional instances of both Kustomize and Helm controllers that focuses on applying platform level changes, which do not compete with Tenants changes.    
  </details>
  <details>
    <summary>Audit Procedure</summary>
    Check for the existence of additional Flux controllers instances and their respective scopes. Each "non-global" controller must be started with `--watch-all-namespaces=false` and have the namespace scope set via environment variable `RUNTIME_NAMESPACE`:
    
    `kubectl describe pod -n flux-system -l app=kustomize-controller | grep -B 5 -A 10 Args`
    `kubectl describe pod -n flux-system -l app=helm-controller | grep -B 5 -A 10 Args`
    `kubectl describe pod -n flux-system -l app=source-controller | grep -B 5 -A 10 Args`
  </details>

- Ensure worker nodes are not being shared across tenants and the Flux components.

  <details>
    <summary>Rationale</summary>
    Pods sharing the same worker node may enable threat vectors which might enable a malicious tenant to have a negative impact in the Confidentiality, Integrity or Availability of the co-located pods.
    The Flux components may have Control Plane privileges which some tenants may not. A co-located pod could leverage its privileges in the shared worker node to bypass its own Control Plane access limitations by compromising one of the co-located Flux components. For cases in which cross-tenant isolation requirements must be enforced, the same risks apply.
    Employ techniques to enforce that untrusted workloads are sandboxed. And, ensure that worker nodes are only shared when within the acceptable risks by your security requirements.
  </details>
  <details>
    <summary>Audit Procedure</summary>
    - Check whether there are Admission Controllers/OPA blocking tenants from creating privileged containers.
    - Check whether [RuntimeClass] is being employed to sandbox workloads that may be scheduled in shared worker nodes.
    - Check whether [Taints and Tolerations] are being used to decrease the likelihood of sharing worker nodes across tenants, or with the Flux controllers. Some cloud providers have this encapsulated as Node Pools.
  </details>
  

### Network Isolation

- Ensure the Container Network Interface (CNI) being used in the cluster supports Network Policies.

  <details>
    <summary>Rationale</summary>
    Flux rely on Network Policies to ensure that only Flux components have direct access to the source artefacts kept in the Source Controller.
  </details>
  <details>
    <summary>Audit Procedure</summary>
    - Create a Network Policy and confirm it is being enforced by the CNI.
  </details>

  
## Additional Best Practices for Tenant Dedicated Cluster Multi-tenancy

- Ensure tenants are not able to revoke Platform Admin access to their clusters.

  <details>
    <summary>Rationale</summary>
    In environments in which a management cluster is used to bootstrap and manage other clusters, it is important to ensure that a tenant is not allowed to revoke access from the Platform Admin, effectively denying the Management Cluster the ability to further reconcile changes into the tenant's Cluster.
    The Platform Admin should make sure that at the tenant’s cluster bootstrap process, this is taken into the account and a breakglass procedure is in place to recover access without the need to rebuild the cluster.
  </details>
  <details>
    <summary>Audit Procedure</summary>
    - Check whether alerts are in place in case the Remote Apply operations fails.
    - Check the permission set given to the tenant's users and applications is not overly privileged.
    - Check whether there are Admission Controllers/OPA rules blocking changes in Platform Admin's permissions and overall resources.
  </details>


[Kubernetes CIS Benchmark]: https://www.cisecurity.org/benchmark/kubernetes
[RuntimeClass]: https://kubernetes.io/docs/concepts/containers/runtime-class/
[Taints and Tolerations]: https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/
