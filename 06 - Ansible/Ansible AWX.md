---
tags:
  - Ansible
---
Ansible AWX is the open-source, upstream project that provides a web UI, REST API, and task engine on top of Ansible so you can centrally manage playbooks, inventories, credentials, job templates, and automation workflows instead of running Ansible only from the CLI. It is effectively the community edition of what Red Hat ships commercially as Ansible Automation Platform (formerly Ansible Tower), aimed at multi-user, team, and enterprise-style automation scenarios.

## What AWX actually is

- AWX is a control plane for Ansible: it stores projects (playbooks from Git or other SCM), inventories, credentials, job templates, and workflows, and then executes them via a web UI or API.
- It runs as a web application plus task engine (typically containerized on Kubernetes or Docker), exposing a REST API that CI/CD systems and other tools can call to trigger automation jobs.

## Key capabilities

- Role-Based Access Control (RBAC): define organizations, teams, and users, and grant fine-grained permissions on inventories, credentials, job templates, etc., which is essential once more than one engineer is running Ansible.
- Centralized credential management: securely store SSH keys, API tokens, and cloud provider credentials, often with integration hooks to external secret backends; AWX encrypts stored credentials and only exposes them to jobs at runtime.
- Job templates, workflows, and scheduling: standardize how playbooks are run (extra vars, limits, tags), chain multiple job templates into workflows with approval gates, and schedule them for periodic runs.
- Logging, auditing, and notifications: keep detailed job logs and audit history, and send notifications (email, Slack, webhooks, etc.) on job success/failure, which is important for compliance and operational visibility.

## Where AWX fits in a DevOps architecture

- Use AWX when you need a multi-user, auditable Ansible control plane that integrates with Git, CI/CD, and enterprise auth (LDAP/SSO), rather than running ad-hoc playbooks from a developer laptop.
- AWX is appropriate when you need audit trails (who ran what playbook, against which hosts, when), approval gates in workflows, or centralized credential storage that doesn't expose secrets to individual engineers.
- It is not necessary for small teams or single-engineer setups where plain Ansible CLI plus a shared Git repo is sufficient.

## AWX vs Ansible Tower vs Ansible Automation Platform

| Product | Vendor | Cost | Notes |
|---|---|---|---|
| AWX | Community (upstream) | Free | No SLA, rolling releases, may be unstable |
| Ansible Tower | Red Hat (legacy) | Commercial | Stable, supported, now superseded |
| Ansible Automation Platform (AAP) | Red Hat | Commercial | Current product; includes AWX core + extra tooling |

AWX is the upstream where new features land first; Red Hat stabilizes and tests them before shipping in AAP.

## Deployment

AWX is deployed on Kubernetes using the AWX Operator:

```bash
# Install the AWX Operator
kubectl apply -k github.com/ansible/awx-operator/config/default?ref=<version>

# Create an AWX custom resource
cat <<EOF | kubectl apply -f -
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  service_type: nodeport
EOF

# Watch deployment
kubectl get pods -l app.kubernetes.io/managed-by=awx-operator -w
```

## Key concepts

| Concept | Description |
|---|---|
| **Organization** | Top-level grouping for teams, users, inventories, projects |
| **Inventory** | List of hosts/groups that playbooks run against |
| **Project** | A Git repo (or other SCM) containing playbooks |
| **Credential** | Stored secret (SSH key, API token, cloud creds) |
| **Job Template** | Saved combination of playbook + inventory + credentials + extra vars |
| **Workflow** | Directed graph of job templates with conditional branching |
| **Schedule** | Cron-like trigger to run a job template automatically |

## Official docs

- AWX GitHub: https://github.com/ansible/awx
- AWX Operator: https://github.com/ansible/awx-operator
- Ansible Automation Platform: https://www.redhat.com/en/technologies/management/ansible
