
Bitnami Sealed Secrets is a Kubernetes add-on that lets you encrypt Kubernetes Secret manifests into a custom resource called `SealedSecret`, which is safe to commit to Git (even in public repos) and can only be decrypted by a controller running inside your cluster. [web:2][web:3] The workflow is: you create a normal Secret manifest locally, encrypt it with the `kubeseal` CLI using the controller’s public key, store the resulting `SealedSecret` in Git, and the in-cluster controller automatically decrypts it back into a normal `Secret` at runtime. [web:1][web:2][web:5]

In practice, Sealed Secrets consists of:
- A Kubernetes controller that watches `SealedSecret` custom resources and creates/updates the corresponding `Secret` objects in the cluster.
- A `kubeseal` CLI that uses asymmetric cryptography and the controller’s public key to encrypt the Secret manifest into a `SealedSecret` object. [web:2][web:3]

---

## Source of Truth (Official Doc Links)

- Bitnami Sealed Secrets GitHub repository (primary spec, design, and usage examples):  
  https://github.com/bitnami-labs/sealed-secrets [web:2]  
- Bitnami package page for Sealed Secrets (product description and container images):  
  https://bitnami.com/stacks/sealed-secrets [web:3]  
- Helm chart on Artifact Hub (installation and configuration via Helm):  
  https://artifacthub.io/packages/helm/bitnami/sealed-secrets [web:4]  
- FluxCD documentation on using Sealed Secrets in GitOps workflows:  
  https://fluxcd.io/flux/guides/sealed-secrets/ [web:5]  

These are the sources that should be treated as the canonical references for architecture, deployment, and lifecycle behavior.

---

## Architectural Reasoning

Sealed Secrets exists to solve the GitOps problem: “everything is declarative in Git except Secrets.” It allows you to keep Secrets in the same Git repo as the rest of your manifests while ensuring that the value is encrypted with a key that only exists inside the cluster, not in your CI system or developer machines. [web:2][web:5] The encryption is “one-way” from the user’s point of view: you cannot retrieve the original Secret value from the `SealedSecret` without access to the controller’s private key in the cluster, which makes the stored manifests safe to inspect or leak from Git alone. [web:2][web:3][web:4]

From an architectural perspective:
- **Why Sealed Secrets instead of SOPS + KMS/age?**  
  Sealed Secrets keeps key material attached to the cluster (controller key pair) rather than an external KMS, which can simplify multi-tenant or on-prem setups and avoid cloud lock-in. [web:2] In contrast, SOPS shifts trust to cloud KMS or external key management; that can be preferable when you already have strong KMS governance but adds extra dependencies and IAM complexity.
- **Why Sealed Secrets instead of Vault + External Secrets Operator?**  
  Vault + ESO gives a full-featured secrets manager (dynamic credentials, audit, revocation), but it is significantly more complex to deploy, operate, and secure. Sealed Secrets focuses specifically on Git-safe encryption of static secrets and leverages native Kubernetes `Secret` objects at runtime, so the operational model is much lighter. [web:1][web:5]
- **Scoping behavior:**  
  Sealed Secrets supports different “scopes” (strict, namespace-wide, cluster-wide) that control where a given `SealedSecret` can be decrypted, helping prevent replaying a sealed blob into another namespace or with another name unless that is explicitly allowed. [web:2] This is an architectural control to limit blast radius if manifests are copied around.

The trade-off is that Sealed Secrets is tightly coupled to the cluster where the controller key pair lives; restoring or migrating `SealedSecret` objects to a different cluster requires key management (e.g., moving or reusing the sealing key), whereas SOPS or Vault are naturally more cluster-agnostic.

---

## Security & Best Practice Check

Security-wise, Sealed Secrets is a hardening layer for **storage and transport of manifests**, not a complete secret protection story by itself. Once the controller decrypts a `SealedSecret`, the resulting Kubernetes `Secret` is just a normal secret object and is only base64-encoded; you still need Kubernetes etcd encryption at rest, strong RBAC, and network policies to prevent lateral movement or secret exfiltration. [web:2][web:5] Also, because `SealedSecret` objects are effectively “write-only,” you should treat rotation as creating a new Secret + resealing, and you must manage the controller’s sealing keys carefully (they’re rotated periodically by the controller, but compromise of those keys means any stored `SealedSecret` could be decrypted). [web:2]

Best-practice alignment:
- Use Sealed Secrets when:
  - You want GitOps-friendly encryption of static secrets and minimal operational overhead.
  - You are okay with secrets living as normal Kubernetes `Secret` objects at runtime.
- Prefer a full secrets manager (Vault, cloud KMS + ESO, or SOPS + KMS) when:
  - You need dynamic/short-lived credentials, centralized auditing, or cross-cluster secret reuse.
  - You already have a strong KMS / HSM governance model and want key management decoupled from clusters.

In a secure-by-default design, Sealed Secrets is acceptable as the Git-facing encryption layer, but it should be combined with: Kubernetes encryption at rest for secrets, minimal RBAC for reading secrets, cluster-wide logging and monitoring for secret access, and clear operational procedures for sealing key rotation and disaster recovery. [web:2][web:5]
