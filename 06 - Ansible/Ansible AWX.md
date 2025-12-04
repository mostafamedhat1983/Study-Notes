---
tags:
  - Ansible
---
Ansible AWX is the open‑source, upstream project that provides a web UI, REST API, and task engine on top of Ansible so you can centrally manage playbooks, inventories, credentials, job templates, and automation workflows instead of running Ansible only from the CLI[web:48][web:56]. It is effectively the community edition of what Red Hat ships commercially as Ansible Automation Platform (formerly Ansible Tower), aimed at multi‑user, team, and enterprise-style automation scenarios[web:50][web:51].

## What AWX actually is

- AWX is a control plane for Ansible: it stores projects (playbooks from Git or other SCM), inventories, credentials, job templates, and workflows, and then executes them via a web UI or API[web:48][web:49][web:56].
- It runs as a web application plus task engine (typically containerized on Kubernetes or Docker), exposing a REST API that CI/CD systems and other tools can call to trigger automation jobs[web:48][web:49].

## Key capabilities

- Role-Based Access Control (RBAC): define organizations, teams, and users, and grant fine-grained permissions on inventories, credentials, job templates, etc., which is essential once more than one engineer is running Ansible[web:49][web:51][web:53].
- Centralized credential management: securely store SSH keys, API tokens, and cloud provider credentials, often with integration hooks to external secret backends; AWX encrypts stored credentials and only exposes them to jobs at runtime[web:51][web:53][web:54].
- Job templates, workflows, and scheduling: standardize how playbooks are run (extra vars, limits, tags), chain multiple job templates into workflows with approval gates, and schedule them for periodic runs[web:48][web:49][web:51].
- Logging, auditing, and notifications: keep detailed job logs and audit history, and send notifications (email, Slack, webhooks, etc.) on job success/failure, which is important for compliance and operational visibility[web:48][web:49][web:54].

## Where AWX fits in a DevOps architecture

- Use AWX when you need a multi‑user, auditable Ansible control plane that integrates with Git, CI/CD, and enterprise auth (LDAP/SSO), rather than
