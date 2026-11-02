---
tags:
  - AWS
  - EKS
  - Kubernetes
---
# Amazon EKS Pod Identity

Amazon EKS Pod Identity lets workloads in an EKS cluster obtain temporary AWS credentials for an IAM role associated with their Kubernetes ServiceAccount. It avoids storing long-term AWS access keys in the Pod.

## Components

- **Pod Identity association:** maps a cluster, namespace, ServiceAccount, and IAM role.
- **Pod Identity Agent:** a DaemonSet on cluster nodes that provides a local credential endpoint to Pods.
- **EKS Auth API:** the agent calls `AssumeRoleForPodIdentity` to obtain credentials for the role association.
- **AWS SDK/CLI credential provider:** supported SDKs discover the container credential endpoint and request credentials from the agent.

The agent does not intercept arbitrary AWS API calls. The application or AWS SDK first obtains credentials from the container credential endpoint, then uses those credentials to sign its normal AWS service requests.

## Credential flow

1. Create an IAM role with the permissions the workload needs.
2. Configure the role trust policy for EKS Pod Identity, including the `pods.eks.amazonaws.com` service principal and the required `sts:AssumeRole` and `sts:TagSession` permissions.
3. Create a Pod Identity association for the EKS cluster, namespace, ServiceAccount, and role.
4. Install/enable the EKS Pod Identity Agent add-on, unless it is provided/managed by the EKS environment in use.
5. EKS injects credential-provider environment configuration into Pods using the associated ServiceAccount. The variables identify the local container credential endpoint and authorization token file; they are not the AWS access key/secret themselves.
6. A supported AWS SDK/CLI uses the container credential provider to contact the local agent.
7. The agent calls the EKS Auth API and returns temporary role credentials to the SDK.
8. The SDK uses those temporary credentials to call AWS services such as Secrets Manager or EC2.

A conceptual view:

```text
Application in Pod
    |
    | AWS SDK credential provider
    v
Local Pod Identity Agent endpoint
    |
    | EKS Auth: AssumeRoleForPodIdentity
    v
EKS Auth service
    |
    v
Temporary credentials returned to SDK
    |
    v
AWS service API (for example, Secrets Manager)
```

## Important security details

- Credentials are temporary, but they are available to processes in the Pod that can access the credential provider. Do not claim that no credentials exist in the Pod.
- The agent's endpoint and authorization token file are not themselves the AWS role credentials.
- Containers in the same Pod share the Pod's ServiceAccount identity; Pod Identity is not a per-container IAM boundary.
- Use a dedicated ServiceAccount and least-privilege IAM policy for each workload where practical.
- Restrict access to node credentials and the Pod Identity Agent endpoint according to AWS guidance.
- Use supported AWS SDK versions; older SDKs may not support the container credential provider.
- Credential refresh is handled through the SDK/provider flow. Do not hard-code a fixed credential lifetime in notes; the returned expiration is part of the credential response.
- CloudTrail can record role activity. Use the available session context and CloudTrail fields rather than assuming every event will identify a Kubernetes Pod by a human-readable Pod name.

## Example: EKS Pod Identity association

```bash
aws eks create-pod-identity-association \
  --cluster-name <cluster> \
  --namespace <namespace> \
  --service-account <service-account> \
  --role-arn arn:aws:iam::<account-id>:role/<role-name>
```

The role trust policy must also permit the EKS Pod Identity service principal to assume the role and tag the session. Follow the current AWS documentation for the exact trust policy and condition keys.

## Example: EBS CSI controller

The EBS CSI **controller** component calls AWS APIs to provision and manage EBS volumes. Its ServiceAccount should be associated with an IAM role that has the required EBS CSI permissions (for example, the AWS-managed `AmazonEBSCSIDriverPolicy`, or a carefully scoped policy).

The node component performs node-side operations such as staging, publishing, mounting, and unmounting volumes. Do not assume that the node DaemonSet needs the same EC2 provisioning permissions as the controller; follow the driver version's AWS installation guidance.

## Troubleshooting

```bash
# Agent DaemonSet / Pods
kubectl get daemonset -n kube-system
kubectl get pods -n kube-system | grep -i pod-identity

# Confirm ServiceAccount on the Pod
kubectl get pod <pod> -n <namespace> -o jsonpath='{.spec.serviceAccountName}'

# Inspect Pod Identity environment configuration (not the credential values)
kubectl exec -n <namespace> <pod> -- env | grep AWS_CONTAINER_CREDENTIALS

# Confirm associations
aws eks list-pod-identity-associations --cluster-name <cluster>
```

If credential retrieval fails, verify the association, namespace and ServiceAccount names, role trust policy, agent health, network access to the EKS Auth service, and AWS SDK support.

## Pod Identity compared with IRSA

Both provide IAM roles to Kubernetes workloads without long-term static access keys. IRSA uses an OIDC identity provider and web identity token flow. EKS Pod Identity uses an EKS-managed association and the node agent/EKS Auth flow. Which is appropriate depends on cluster version, AWS environment, SDK compatibility, and operational requirements.

## Interview summary

EKS Pod Identity maps a ServiceAccount to an IAM role. EKS configures the Pod to use a local container credential endpoint; the Pod Identity Agent calls the EKS Auth API and supplies temporary credentials to supported SDKs. The SDK then signs requests to AWS services. The agent does not proxy or intercept the application's AWS service API calls.
