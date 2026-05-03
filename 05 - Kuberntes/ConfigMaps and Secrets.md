---
tags:
  - Kubernetes
---
ConfigMaps and Secrets are the standard Kubernetes way to inject configuration and sensitive data into Pods without hardcoding values in container images.

---

## Main idea

- ConfigMap = non-sensitive configuration (URLs, feature flags, config files)
- Secret = sensitive data (passwords, API keys, TLS certs, tokens)
- both decouple config from the container image
- both can be injected as environment variables or mounted as files

---

## ConfigMap

### Creating a ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  DB_HOST: "db.production.svc.cluster.local"
```

```bash
# From literal values
kubectl create configmap app-config --from-literal=APP_ENV=production

# From a file
kubectl create configmap app-config --from-file=config.properties
```

### Using ConfigMap as environment variables

```yaml
spec:
  containers:
    - name: app
      image: my-app:1.0
      envFrom:
        - configMapRef:
            name: app-config
```

### Using a specific key from ConfigMap

```yaml
env:
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: LOG_LEVEL
```

### Mounting ConfigMap as a file

```yaml
spec:
  volumes:
    - name: config-volume
      configMap:
        name: app-config
  containers:
    - name: app
      image: my-app:1.0
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
```

Each key in the ConfigMap becomes a file under `/etc/config/`.

---

## Secret

### Secret types

| Type | Use |
|---|---|
| `Opaque` | default — any key-value data |
| `kubernetes.io/tls` | TLS certificate and key |
| `kubernetes.io/dockerconfigjson` | Docker registry credentials |
| `kubernetes.io/service-account-token` | service account token |

### Creating a Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQxMjM=   # base64 encoded
  DB_USER: YWRtaW4=
```

```bash
# From literal (kubectl handles base64 encoding)
kubectl create secret generic db-secret \
  --from-literal=DB_PASSWORD=password123 \
  --from-literal=DB_USER=admin

# TLS secret
kubectl create secret tls my-tls \
  --cert=tls.crt \
  --key=tls.key
```

### Using Secret as environment variables

```yaml
spec:
  containers:
    - name: app
      image: my-app:1.0
      envFrom:
        - secretRef:
            name: db-secret
```

### Using a specific key from Secret

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: DB_PASSWORD
```

### Mounting Secret as a file

```yaml
spec:
  volumes:
    - name: secret-volume
      secret:
        secretName: db-secret
  containers:
    - name: app
      image: my-app:1.0
      volumeMounts:
        - name: secret-volume
          mountPath: /etc/secrets
          readOnly: true
```

---

## Important: Secrets are base64 encoded, not encrypted

- base64 is encoding, not encryption
- anyone with access to the Secret object can decode it
- for real encryption at rest, enable etcd encryption or use external secret managers
- for production: use AWS Secrets Manager, HashiCorp Vault, or Bitnami Sealed Secrets

---

## Common mistakes

- hardcoding secrets in container images or environment variables in manifests
- storing Secret YAML files in Git without encryption (use Sealed Secrets or SOPS)
- confusing base64 encoding with encryption
- not setting RBAC to restrict who can read Secrets
- mounting Secrets as env vars instead of files (env vars are easier to leak via logs)
- not rotating secrets when they change

---

## Good practices

- store ConfigMaps in Git (they are not sensitive)
- never store raw Secret YAML in Git — use Sealed Secrets, SOPS, or external secret managers
- use volume mounts for secrets rather than env vars where possible
- set RBAC so only the Pods that need a Secret can read it
- use `immutable: true` for ConfigMaps/Secrets that should not change at runtime

---

## Must memorize

```bash
kubectl create configmap <name> --from-literal=KEY=VALUE
kubectl create secret generic <name> --from-literal=KEY=VALUE
kubectl get configmap <name> -o yaml
kubectl get secret <name> -o yaml
echo <base64string> | base64 --decode
```

---

## Key ideas

- ConfigMaps store non-sensitive config; Secrets store sensitive data.
- Both can be injected as environment variables or mounted as files.
- Secrets are base64 encoded, not encrypted — protect them with RBAC and etcd encryption.
- Never commit raw Secret YAML to Git.
- Use Bitnami Sealed Secrets or an external secret manager for GitOps workflows.
