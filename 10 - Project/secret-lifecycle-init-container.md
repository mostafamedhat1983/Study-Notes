---
tags:
  - My_CV_Project
  - AWS
  - Kubernetes
  - EKS
  - Security
---
# Database Secret Lifecycle: EKS Init Container

## Purpose

This note documents what the project code actually implements for database credentials. It separates verified implementation details from security recommendations. Do not describe recommendations as features that were deployed.

## Verified architecture

1. The database credentials originate in AWS Secrets Manager.
2. The backend Pod uses the `chatbot-backend-service-account` service account.
3. EKS Pod Identity associates that service account with an IAM role. The role policy in Terraform allows Secrets Manager read operations and Bedrock model invocation.
4. The `fetch-secrets` init container uses the AWS CLI to retrieve the JSON secret.
5. It parses the values and creates or updates a Kubernetes Secret named `chatbot-backend-db-credentials`.
6. The backend container imports that Kubernetes Secret through `envFrom.secretRef`.
7. The backend reads the resulting environment variables to configure its MySQL connection pool.

The init container completes before the backend container starts. Its process environment is **not** automatically inherited by the backend container. The Kubernetes Secret is the hand-off mechanism.

## Lifecycle, storage, and encryption

| Stage | What the repository shows | Accurate security description |
|---|---|---|
| AWS Secrets Manager | The init container calls `aws secretsmanager get-secret-value`. | Secrets Manager encrypts secret values at rest using AWS KMS. The AWS CLI communicates with the AWS API over HTTPS/TLS. |
| Init container | The JSON value is held temporarily in shell variables and parsed into individual values. | Plaintext exists in the init container's process environment / shell memory. Do not claim it is encrypted in process memory. |
| Kubernetes API | The init container creates/updates a Kubernetes Secret using `kubectl`. | The Secret is a Kubernetes API object persisted in the cluster datastore; it is not memory-only and is not zero-persistence. Base64 encoding is not encryption. |
| EKS control plane datastore | Terraform configures EKS encryption with a KMS key for Kubernetes Secrets. | The repository configures KMS envelope encryption for Secrets. This is at-rest protection, not a claim that the Secret never persists. |
| Backend container | `envFrom.secretRef` exposes the Secret's keys as environment variables. | The backend process receives plaintext credentials in its environment. Environment variables are not an encrypted secret store. |
| RDS | Terraform sets `storage_encrypted = true`. | RDS storage is encrypted at rest. This does not encrypt the database network connection. |
| Backend-to-RDS connection | The actual `aiomysql.create_pool()` call supplies host, port, username, password, database, and pool settings, but no SSL/TLS context. | The checked application code does not configure MySQL TLS. Do not claim that application-to-RDS traffic is encrypted or that RDS enforces TLS. |

### Important distinction: RDS TLS

The RDS module accepts a parameter-group name but the checked module does not itself define a parameter group setting `require_secure_transport = ON`. The checked backend connection code also does not pass an SSL context to `aiomysql.create_pool()`.

Therefore, based on the repositories inspected, TLS for the backend-to-RDS connection is **not demonstrated**. AWS documents that RDS for MySQL defaults `require_secure_transport` to `OFF`; enforcing TLS requires setting it to `ON` in the DB parameter group and configuring the client to use TLS.

Do not state that all network connections in this project are encrypted. The AWS API call uses HTTPS, but the MySQL connection needs separate configuration and verification.

## IAM and Kubernetes RBAC

### EKS Pod Identity

The Deployment uses:

```yaml
serviceAccountName: chatbot-backend-service-account
```

The Terraform Pod Identity association uses the same service-account name and the configured `var.chatbot_namespace`. The IAM role policy includes:
- Bedrock model invocation permissions.
- Secrets Manager `GetSecretValue` and `DescribeSecret` permissions scoped by the policy's secret ARN pattern.

**Security implication:** Pod Identity is associated with the Pod's service account, not with one individual container. Both the init container and backend container run in the same Pod and use that service-account identity. The backend container therefore has the IAM identity too, even though its current database-credential flow reads a Kubernetes Secret.

### Kubernetes RBAC

The checked RBAC manifest creates a namespaced Role and RoleBinding in the `default` namespace. The Role grants `get`, `list`, `create`, `update`, and `patch` on Kubernetes Secrets.

Important limitations:
- The permissions apply to Secrets across the namespace, not only to `chatbot-backend-db-credentials`.
- `list` is especially broad because it can expose the values of multiple Secrets in the namespace.
- The Role and RoleBinding are hard-coded to `default`, while the ServiceAccount manifest uses `{{ .Release.Namespace }}` and the Terraform association uses `var.chatbot_namespace`. These must resolve to the same namespace or the RBAC binding / Pod Identity association will not line up.

Do not describe this as per-secret least-privilege RBAC.

## Terraform state: an additional secret exposure point

The RDS module reads the secret version using the Terraform data source `aws_secretsmanager_secret_version.db_credentials`, decodes `secret_string`, and uses the resulting username and password in `aws_db_instance`.

That means the secret value can be recorded in Terraform state. Marking an output or variable as `sensitive` only redacts normal CLI display; it does not by itself remove the value from state.

For an interview, say that the module consumes credentials from Secrets Manager, but acknowledge that Terraform state must be protected as sensitive data. Use an encrypted remote backend, tightly restrict state access, enable appropriate state versioning / access auditing, and avoid exposing state in logs or artifacts. The checked source files do not establish the backend's actual storage and access controls, so do not claim those protections were configured unless you can verify them separately.

## Rotation and refresh behavior

The init container fetches credentials when a Pod is initialized. It then creates or updates the Kubernetes Secret. The backend receives Secret values as environment variables through `envFrom`.

Changing the Kubernetes Secret does not update environment variables inside an already-running container. A new Pod is required for the backend to receive the new values. A rollout restart is one way to trigger new Pods:

```bash
kubectl rollout restart deployment chatbot-backend-deployment
```

The init container runs again for each newly created Pod and fetches the then-current Secrets Manager value.

The repository demonstrates a startup fetch and manual restart workflow. It does **not** demonstrate automatic secret rotation end-to-end, an automatic restart controller, or live credential refresh inside the application.

## Issues and recommended improvements

These are recommendations, not claims about what the project already implemented:

1. **Pin the init-container image.** The Deployment uses `amazon/aws-cli:latest`. Pin a tested version or immutable digest to make deployments reproducible.
2. **Avoid downloading kubectl at container startup.** Include a compatible, pinned client in a purpose-built init image. This removes a runtime download dependency and makes the toolchain predictable.
3. **Reduce command-line secret exposure.** The current `kubectl create secret --from-literal=DB_PASSWORD=...` invocation places the password in a child process argument. Prefer a carefully constructed manifest streamed through standard input, with shell tracing disabled and no secret output.
4. **Narrow RBAC.** Avoid namespace-wide `list` access if it is not required. Restrict permissions to the smallest practical scope, and ensure the RoleBinding namespace matches the Helm release and Pod Identity association namespace.
5. **Separate AWS permissions where practical.** The current Pod Identity role is shared by every container in the Pod. Consider whether the application needs Secrets Manager read access, or whether the init-container pattern can be redesigned to reduce the backend's access.
6. **Enforce database TLS.** Configure the RDS MySQL parameter group with `require_secure_transport = ON`, configure `aiomysql` with certificate validation, and test the connection. Do not merely set an SSL flag without validating the server certificate.
7. **Protect Terraform state.** Treat state as secret-bearing because the RDS module reads the Secrets Manager secret value into Terraform. Verify backend encryption, access controls, state history, and CI artifact/log handling.
8. **Review Kubernetes Secret use.** A Secrets Store CSI integration or a direct application-side Secrets Manager retrieval design may reduce the need to persist credentials as a Kubernetes Secret. Evaluate the operational and IAM trade-offs before changing the architecture.

## Safe troubleshooting

Do not print the environment or the Secret contents when checking whether credentials are present.

Check only whether required variables are set:

```bash
kubectl exec <pod-name> -c backend-container -- sh -c '
  for name in DB_HOST DB_USER DB_PASSWORD DB_NAME DB_PORT; do
    if printenv "$name" >/dev/null 2>&1; then
      echo "$name: set"
    else
      echo "$name: MISSING"
    fi
  done
'
```

Check the init-container logs (the current script prints status messages):

```bash
kubectl logs <pod-name> -c fetch-secrets
```

Avoid commands such as `env`, `printenv` without a variable name, `kubectl get secret -o yaml`, or shell tracing (`set -x`) in shared terminals or captured sessions; they can reveal credentials or their encoded values.

## Interview-ready explanation

"The backend Pod uses EKS Pod Identity through its Kubernetes service account. An init container retrieves the database credential JSON from AWS Secrets Manager, parses it, and creates or updates a Kubernetes Secret through the Kubernetes API. The backend container consumes that Secret with `envFrom`, so the init container's environment is not being shared directly with the application. EKS is configured for KMS encryption of Kubernetes Secrets at rest, and the RDS storage is encrypted at rest. The current implementation fetches credentials at Pod startup, so a new Pod is needed to consume rotated credentials. I would not claim that the database connection uses TLS based on the current code: the `aiomysql` connection does not configure SSL, and the checked RDS configuration does not show `require_secure_transport` being enabled. I would also protect Terraform state because the RDS module reads the secret value into Terraform."

## Source files checked

- `platform-ai-chatbot/k8s/templates/chatbot-backend-deployment.yaml`
- `platform-ai-chatbot/k8s/templates/chatbot-backend-service-account.yaml`
- `platform-ai-chatbot/k8s/templates/chatbot-backend-rbac.yaml`
- `platform-ai-chatbot/backend/main.py`
- `terraform-aws-eks/terraform/modules/eks/main.tf`
- `terraform-aws-eks/terraform/modules/rds/main.tf`
- `terraform-aws-eks/terraform/modules/rds/variables.tf`

## Official references

- Kubernetes init containers: https://kubernetes.io/docs/concepts/workloads/pods/init-containers/
- Kubernetes Secrets: https://kubernetes.io/docs/concepts/configuration/secret/
- EKS envelope encryption: https://docs.aws.amazon.com/eks/latest/userguide/envelope-encryption.html
- RDS MySQL TLS enforcement: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/mysql-ssl-connections.require-ssl.html
- Terraform sensitive-data guidance: https://docs.aws.amazon.com/prescriptive-guidance/latest/secure-sensitive-data-secrets-manager-terraform/sensitive-data-terraform.html

---

**Last reviewed:** 2026-11-02  
**Project:** AI Chatbot Platform with AWS Bedrock
