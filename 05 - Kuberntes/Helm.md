# Helm

Helm is the package manager for Kubernetes. It packages Kubernetes manifests into **Charts** and manages their lifecycle (install, upgrade, rollback, uninstall).

---

## Core Concepts

| Term | Description |
|---|---|
| **Chart** | A packaged collection of Kubernetes manifests + templates |
| **Release** | A deployed instance of a chart in a cluster |
| **Repository** | A collection of charts hosted remotely |
| **Values** | Configuration inputs that customize a chart |
| **Template** | A Kubernetes manifest with Go templating syntax |

---

## Installation

```bash
# Install Helm (Linux)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify
helm version
```

---

## Repositories

```bash
# Add a repo
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add stable https://charts.helm.sh/stable

# Update repo index
helm repo update

# List repos
helm repo list

# Search for a chart
helm search repo nginx
helm search hub wordpress          # search Artifact Hub
```

---

## Installing and Managing Releases

```bash
# Install a chart
helm install <release-name> <chart>
helm install my-nginx bitnami/nginx

# Install into a specific namespace (create if needed)
helm install my-app bitnami/nginx -n production --create-namespace

# Install with custom values
helm install my-app bitnami/nginx --set service.type=ClusterIP
helm install my-app bitnami/nginx -f custom-values.yaml

# Dry run (preview manifests without installing)
helm install my-app bitnami/nginx --dry-run --debug

# List all releases
helm list
helm list -A                        # all namespaces
helm list -n production

# Get release status
helm status my-app -n production

# Uninstall a release
helm uninstall my-app -n production
```

---

## Upgrading and Rolling Back

```bash
# Upgrade a release
helm upgrade my-app bitnami/nginx -f values.yaml

# Install if not exists, upgrade if it does
helm upgrade --install my-app bitnami/nginx -f values.yaml -n production --create-namespace

# View release history
helm history my-app -n production

# Rollback to previous revision
helm rollback my-app -n production

# Rollback to specific revision
helm rollback my-app 2 -n production
```

---

## Working with Values

```bash
# Show default values of a chart
helm show values bitnami/nginx

# Show all chart info
helm show all bitnami/nginx

# Get values of a deployed release
helm get values my-app -n production

# Get all values (including defaults)
helm get values my-app -n production --all
```

### values.yaml example
```yaml
replicaCount: 3

image:
  repository: nginx
  tag: "1.25"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```

---

## Chart Structure

```
my-chart/
├── Chart.yaml          # Chart metadata (name, version, description)
├── values.yaml         # Default configuration values
├── templates/          # Kubernetes manifest templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl    # Named template helpers (not rendered directly)
│   └── NOTES.txt       # Post-install instructions shown to user
└── charts/             # Dependency charts
```

### Chart.yaml
```yaml
apiVersion: v2
name: my-app
description: A Helm chart for my application
type: application
version: 0.1.0           # Chart version
appVersion: "1.2.3"      # App version being deployed
```

---

## Templating Basics

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-app
  labels:
    app: {{ .Chart.Name }}
    version: {{ .Chart.AppVersion }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.service.port }}
```

### Common Template Objects
| Object | Description |
|---|---|
| `.Release.Name` | Name of the release |
| `.Release.Namespace` | Namespace of the release |
| `.Chart.Name` | Chart name |
| `.Chart.Version` | Chart version |
| `.Values.*` | Values from values.yaml or --set |
| `.Files.Get` | Read a file from the chart |

### Template Functions
```yaml
# Default value
{{ .Values.replicas | default 1 }}

# Quote a string
{{ .Values.name | quote }}

# Upper case
{{ .Values.env | upper }}

# Conditional
{{- if .Values.ingress.enabled }}
  # ingress manifest
{{- end }}

# Loop
{{- range .Values.envVars }}
  - name: {{ .name }}
    value: {{ .value }}
{{- end }}
```

---

## Creating and Packaging Charts

```bash
# Create a new chart scaffold
helm create my-chart

# Lint a chart for errors
helm lint my-chart/

# Render templates locally (debug)
helm template my-release my-chart/ -f values.yaml

# Package chart into a .tgz
helm package my-chart/

# Push to OCI registry (Helm 3.8+)
helm push my-chart-0.1.0.tgz oci://registry.example.com/charts
```

---

## Helm with AWS ECR (OCI)

```bash
# Login to ECR for Helm
aws ecr get-login-password --region us-east-1 | \
  helm registry login --username AWS --password-stdin \
  <account-id>.dkr.ecr.us-east-1.amazonaws.com

# Push chart to ECR
helm push my-chart-0.1.0.tgz oci://<account-id>.dkr.ecr.us-east-1.amazonaws.com/charts

# Install from ECR
helm install my-app oci://<account-id>.dkr.ecr.us-east-1.amazonaws.com/charts/my-chart --version 0.1.0
```

---

## Best Practices

- Always use `helm upgrade --install` in CI/CD pipelines (idempotent)
- Pin chart versions explicitly — never rely on `latest`
- Keep environment-specific overrides in separate `values-prod.yaml`, `values-staging.yaml` files
- Use `helm lint` and `helm template` in CI before deploying
- Store charts in version control
- Use `--atomic` flag in CI to auto-rollback on failed upgrade
- Use `--wait` to block until all resources are ready

```bash
# Production-safe upgrade
helm upgrade --install my-app ./chart \
  -f values-prod.yaml \
  -n production \
  --create-namespace \
  --atomic \
  --wait \
  --timeout 5m
```
