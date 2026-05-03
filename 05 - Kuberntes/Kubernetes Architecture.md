---
tags:
  - Kubernetes
---
Kubernetes is a container orchestration platform that automates deployment, scaling, and management of containerized applications.

---

## Main idea

Kubernetes follows a **control plane / worker node** model.

- the control plane makes decisions about the cluster
- worker nodes run the actual workloads (Pods)
- all components communicate through the API server
- everything is stored in etcd

---

## Control plane components

### API Server (`kube-apiserver`)

- the front door to Kubernetes
- all kubectl commands, controllers, and components talk to it
- validates and processes REST requests
- the only component that reads and writes to etcd

### etcd

- distributed key-value store
- stores the entire cluster state: nodes, pods, configs, secrets
- if etcd is lost, the cluster state is lost
- always back it up in production

### Scheduler (`kube-scheduler`)

- watches for new Pods with no assigned node
- picks the best node based on resource requests, taints, affinity rules
- does not run the Pod — just assigns it to a node

### Controller Manager (`kube-controller-manager`)

- runs all built-in controllers in a single process
- each controller watches the API server and reconciles actual vs desired state
- examples: Deployment controller, ReplicaSet controller, Node controller, Job controller

### Cloud Controller Manager

- connects Kubernetes to cloud provider APIs
- manages cloud load balancers, node lifecycle, storage volumes
- only present in cloud-managed clusters (EKS, GKE, AKS)

---

## Worker node components

### kubelet

- agent that runs on every worker node
- receives Pod specs from the API server
- ensures containers are running and healthy
- reports node and Pod status back to the control plane

### kube-proxy

- runs on every node
- maintains network rules (iptables or IPVS)
- handles routing for Service IPs to Pod IPs
- does not proxy traffic itself — it sets up kernel-level rules

### Container Runtime

- the software that actually runs containers
- common runtimes: containerd, CRI-O
- Docker was used historically but is no longer directly supported
- Kubernetes talks to the runtime via the CRI (Container Runtime Interface)

---

## How a Pod gets created (full flow)

1. you run `kubectl apply -f pod.yaml`
2. kubectl sends the request to the API server
3. API server validates and stores the Pod object in etcd
4. the scheduler sees the unscheduled Pod and picks a node
5. the API server updates the Pod object with the node assignment
6. the kubelet on that node sees the Pod assigned to it
7. kubelet tells the container runtime to pull the image and start the container
8. kubelet reports status back to the API server

---

## Declarative model

Kubernetes is declarative:
- you describe the desired state in a YAML manifest
- Kubernetes continuously works to match actual state to desired state
- controllers run reconciliation loops constantly

Simple idea:
- you say: "I want 3 replicas of this app"
- Kubernetes figures out how to make that happen and keeps it that way

---

## Common mistakes

- confusing the scheduler (assigns nodes) with the kubelet (runs pods)
- not backing up etcd in production
- running workloads on control plane nodes in production
- ignoring cloud controller manager when debugging cloud resource issues
- assuming Docker is still the container runtime (it was removed in K8s 1.24+)

---

## Good practices

- never run user workloads on control plane nodes in production
- use odd numbers for etcd nodes (3 or 5) for quorum
- back up etcd regularly
- use managed control planes (EKS, GKE, AKS) in production to reduce operational burden
- understand the reconciliation loop — it is the core of how Kubernetes works

---

## Must memorize

```
Control plane:
  kube-apiserver     → single entry point for all operations
  etcd               → cluster state store (back this up)
  kube-scheduler     → assigns pods to nodes
  kube-controller-manager → runs all controllers

Worker node:
  kubelet            → runs pods on the node
  kube-proxy         → handles service networking rules
  container runtime  → runs the containers (containerd, CRI-O)
```

---

## Key ideas

- Kubernetes separates decision-making (control plane) from execution (worker nodes).
- All state lives in etcd — it is the single source of truth.
- The API server is the only component that touches etcd directly.
- kubelet runs on every node and ensures Pods are healthy.
- Kubernetes is declarative — you describe what you want, not how to achieve it.
- Controllers run continuous reconciliation loops to maintain desired state.
