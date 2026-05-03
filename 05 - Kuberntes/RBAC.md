# RBAC — Role-Based Access Control

Kubernetes RBAC controls **who can do what** on which resources inside the cluster. It is enabled by default since Kubernetes 1.8.

---

## Core Objects

| Object | Scope | Purpose |
|---|---|---|
| `Role` | Namespace | Grants permissions within a single namespace |
| `ClusterRole` | Cluster-wide | Grants permissions across all namespaces or non-namespaced resources |
| `RoleBinding` | Namespace | Binds a Role or ClusterRole to a subject within a namespace |
| `ClusterRoleBinding` | Cluster-wide | Binds a ClusterRole to a subject across the entire cluster |
| `ServiceAccount` | Namespace | Identity for Pods to authenticate against the API server |

---

## Role

Defines a set of allowed actions (`verbs`) on specific resources within a namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
  - apiGroups: [""]          # "" = core API group
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
```

### Common Verbs
`get`, `list`, `watch`, `create`, `update`, `patch`, `delete`, `deletecollection`

---

## ClusterRole

Same structure as Role but cluster-scoped. Use for:
- Non-namespaced resources (nodes, PersistentVolumes, namespaces)
- Granting the same role across all namespaces via ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
```

---

## RoleBinding

Binds a Role (or ClusterRole) to a subject within a specific namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: production
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
  - kind: ServiceAccount
    name: my-app
    namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### Subject kinds
- `User` — external user (e.g., from kubeconfig)
- `Group` — group of users
- `ServiceAccount` — Pod identity within Kubernetes

---

## ClusterRoleBinding

Binds a ClusterRole to a subject cluster-wide.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-binding
subjects:
  - kind: User
    name: admin-user
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

---

## ServiceAccount

A ServiceAccount provides an identity for processes running in Pods to authenticate with the Kubernetes API.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: production
```

```yaml
# Reference in a Pod
spec:
  serviceAccountName: my-app
```

- Every namespace has a `default` ServiceAccount — avoid using it for production workloads
- With EKS: use **IRSA** or **EKS Pod Identity** to attach IAM roles to ServiceAccounts
- Disable token auto-mounting if the Pod doesn't need API access:

```yaml
spec:
  automountServiceAccountToken: false
```

---

## Useful Commands

```bash
# Check what a user can do
kubectl auth can-i list pods --as alice -n production
kubectl auth can-i "*" "*" --as admin-user

# List roles and bindings
kubectl get roles,rolebindings -n production
kubectl get clusterroles,clusterrolebindings

# Describe a role
kubectl describe role pod-reader -n production

# Check service account permissions
kubectl auth can-i list pods --as system:serviceaccount:production:my-app
```

---

## Best Practices

- Follow **least privilege** — grant only the verbs and resources needed
- Avoid `cluster-admin` bindings for application service accounts
- Use namespaced Roles + RoleBindings over ClusterRole + ClusterRoleBinding where possible
- Create dedicated ServiceAccounts per application — never share the `default`
- Use `kubectl auth can-i` to verify permissions before deploying
- Audit RBAC regularly — remove stale bindings
