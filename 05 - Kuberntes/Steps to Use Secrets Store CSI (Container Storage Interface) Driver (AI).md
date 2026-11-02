---
tags:
  - AWS
  - Kubernetes
  - EKS
---
# Using AWS Secrets Manager with the Secrets Store CSI Driver on EKS

This is a conceptual checklist. Exact add-on names, CLI flags, and supported identity options must be checked against the EKS/Kubernetes version and AWS add-on documentation used by the cluster.

## Prerequisites

- An EKS cluster and a compatible Secrets Store CSI Driver installation.
- The AWS provider for the Secrets Store CSI Driver (ASCP) installed using the supported EKS add-on or the provider's official installation method.
- An IAM role with least-privilege access to the required Secrets Manager secret ARNs.
- An identity method supported by the provider configuration: EKS Pod Identity or IRSA, as appropriate.
- A Kubernetes ServiceAccount associated with the IAM role.

## 1. Create and authorize the workload identity

Create an IAM policy that grants only the actions and secret resources required by the workload. For a simple read, this commonly includes `secretsmanager:GetSecretValue`; if the secret uses a customer-managed KMS key, the role may also need the appropriate `kms:Decrypt` permission and the key policy must allow the use.

Associate the role with the ServiceAccount using EKS Pod Identity or configure IRSA. Do not assume Pod Identity replaces IRSA in all environments; both are supported identity mechanisms with different prerequisites.

## 2. Create a SecretProviderClass

Example for the AWS provider using Pod Identity (check the provider's current parameter schema for the installed release):

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: app-secrets
  namespace: default
spec:
  provider: aws
  parameters:
    usePodIdentity: "true"
    objects: |
      - objectName: "my-app/database"
        objectType: "secretsmanager"
```

The `SecretProviderClass` is namespaced. Its name must match the `secretProviderClass` volume attribute used by the Pod.

## 3. Mount the secret in a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
  namespace: default
spec:
  serviceAccountName: app-secrets-sa
  containers:
    - name: app
      image: nginx:stable
      volumeMounts:
        - name: secrets-store
          mountPath: /mnt/secrets-store
          readOnly: true
  volumes:
    - name: secrets-store
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: app-secrets
```

The provider retrieves the external secret and mounts the selected content as files. The application must read the mounted file path; the CSI driver does not automatically turn the value into an environment variable.

## 4. Optional synchronization to a Kubernetes Secret

The CSI driver can optionally synchronize mounted content into a Kubernetes Secret when the relevant sync feature is configured. This requires a Pod mounting the SecretProviderClass; the Kubernetes Secret lifecycle is tied to the mounted Pod usage as described by the driver documentation.

If the application consumes a native Kubernetes Secret through `env` or `envFrom`, remember:
- The value is now also stored as a Kubernetes Secret (and therefore in the cluster's data store).
- Enable and verify encryption at rest for Kubernetes Secrets in etcd according to the cluster's security requirements.
- Environment variables in an already-running container do not update when the Secret changes; restart/recreate Pods to consume updated environment values.
- File-mounted secret rotation behavior depends on the driver/provider rotation configuration and application reload behavior.

## 5. Verify

```bash
kubectl get secretproviderclass -n default
kubectl describe pod app -n default
kubectl exec -n default app -- ls -l /mnt/secrets-store
kubectl exec -n default app -- cat /mnt/secrets-store/<mounted-file>
```

Do not paste real secret values into terminal transcripts, tickets, or shared logs.

## Security notes

- Scope IAM access to specific secret ARNs; avoid broad account-wide read policies for production workloads.
- Protect the IAM role trust policy and ServiceAccount association.
- A mounted secret is still accessible to processes in the container that can read the mounted file.
- CSI volume mounts are not the same as fetching a secret directly into application memory.
- Syncing to a Kubernetes Secret creates an additional copy inside the cluster; consider whether the application can consume mounted files instead.
- Secret rotation is not automatically equivalent to application reload. Confirm the provider rotation feature and how the application refreshes credentials.

## Troubleshooting

Check Pod Events, CSI driver/provider Pods and logs, the SecretProviderClass name/namespace, IAM association/trust, IAM policy, secret ARN/region, KMS permissions, and the provider's supported identity configuration.

Official references:
- AWS Secrets and Configuration Provider for Secrets Store CSI Driver: https://github.com/aws/secrets-store-csi-driver-provider-aws
- Kubernetes Secrets Store CSI Driver documentation: https://secrets-store-csi-driver.sigs.k8s.io/
- Amazon EKS Pod Identity documentation: https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html
