---
tags:
  - Kubernetes
---
RBAC stands for **Role-Based Access Control**. It is an access control model where **permissions are assigned to roles, and users are then assigned to those roles**, instead of giving permissions directly to individual users[web:58][web:62].

## Core idea

- In RBAC, you define roles such as *admin*, *developer*, *viewer*, *operator*, etc., and attach permissions (what actions are allowed on which resources) to those roles[web:58][web:63].
- Users (or service accounts) are then mapped to one or more roles, and they automatically inherit the permissions of those roles; changing a user’s access usually means just adding or removing roles[web:62][web:65].

## Why RBAC is used

- RBAC simplifies administration in large environments: instead of managing hundreds of per‑user permission sets, you manage a smaller set of roles and their memberships[web:60][web:63].
- It supports security principles like **least privilege** and **separation of duties**, because you can design roles that grant only the minimum access needed for a given job function and prevent conflicts (e.g., a user cannot both request and approve their own changes)[web:58][web:61].

## In DevOps / platforms (Kubernetes, AWX, cloud)

- Kubernetes RBAC, AWX RBAC, and cloud RBAC (e.g., Azure RBAC, AWS IAM roles) are all concrete implementations of this same model: roles define allowed actions on resources, and subjects (users, groups, service accounts) are bound to those roles[web:59][web:62][web:65].
- As a DevOps/security pattern, RBAC is the default way to control who can deploy, modify infrastructure, read logs, view secrets, or manage CI/CD pipelines, and it should be designed deliberately alongside your org’s team structure and change‑management process[web:62][web:64].
