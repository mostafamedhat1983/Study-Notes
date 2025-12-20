---
tags:
  - memory-bank
last_updated: 2025-12-13
---
# Product Overview

## Project Purpose
Multi-repository AWS cloud platform demonstrating enterprise-grade DevOps practices through infrastructure-as-code and cloud-native application deployment. Built from scratch over 5 months to showcase modern AWS services (2023-2024 features), Kubernetes orchestration, and production-ready security patterns with hands-on implementation and debugging.

## Value Proposition
- **Production-Ready Infrastructure**: 85+ AWS resources across 2 environments with comprehensive security, encryption, and high availability
- **Modern Cloud Architecture**: EKS 1.34, RDS MySQL 8.0, AWS Bedrock DeepSeek V3.1 AI integration with Pod Identity, init container secrets management, and zero-credential exposure
- **DevOps Excellence**: Immutable infrastructure with Packer/Ansible, Terraform modules, Helm multi-environment deployments, Jenkins CI/CD automation with Docker-in-Docker builds
- **Security Focus**: Defense-in-depth with encryption at rest/transit/in-use, network policies, RBAC, Falco runtime monitoring, Trivy vulnerability scanning, SSM Session Manager access
- **Learning Platform**: Educational project demonstrating real-world cloud engineering skills, architecture decisions, and production deployment patterns

## Key Features

### Infrastructure (terraform-aws-eks)
- **Multi-Environment Setup**: Development (~$207/mo) and Production (~$289/mo) with strategic cost optimization
- **Regional NAT Gateway (Dec 2024)**: Single NAT Gateway with built-in HA across all AZs ($35/mo), replacing zonal NAT architecture
- **S3 Native State Locking (2024)**: `use_lockfile = true` instead of legacy DynamoDB approach
- **EKS Access Entry API (2023)**: Modern user/role access management, replacing deprecated `aws-auth` ConfigMap
- **AWS EKS 1.34 Cluster**: Pod Identity, EBS CSI Driver with Pod Identity authentication, modern access management
  - Dev: 5x t3.small nodes (55 pod capacity, 56-58% steady state utilization with ~31-32 pods)
  - Prod: 4x t3.medium nodes (68 pod capacity, 47-49% steady state utilization with ~32-33 pods)
- **RDS MySQL 8.0**: Dev single-AZ (db.t3.micro, 20GB), Prod Multi-AZ (db.t3.small, 50GB, 7-day backups), both encrypted
- **Immutable AMIs**: Packer + Ansible builds (not user data scripts) with Trivy vulnerability scanning (fails on CRITICAL)
- **Security Hardening**: KMS encryption everywhere, IAM least privilege, SSM Session Manager access (no bastion, no SSH keys)
- **Modular Terraform**: Reusable modules (network, ec2, rds, role, eks) with flexible IAM role module reused for EC2, EKS, Pod Identity
- **Jenkins Infrastructure**: Controller on EC2 t3.medium, ephemeral agents as EKS pods (cost-effective, scalable)

### Application (platform-ai-chatbot)
- **AI Integration**: AWS Bedrock DeepSeek V3.1 serverless model ($0.27/1M tokens, 64K context window, pay-per-use, no GPU management)
- **Full-Stack Architecture**: FastAPI async backend + Streamlit frontend with session management
- **Database Persistence**: RDS MySQL with aiomysql connection pooling, conversation history, SSL/TLS connections
- **Cloud-Native Deployment**: Kubernetes with Helm charts, multi-environment values files (values-dev.yaml, values-prod.yaml)
- **Enterprise Security**: 
  - Pod Identity for AWS authentication (no static credentials, OIDC tokens, or service account files)
  - Init container secrets from AWS Secrets Manager (better debugging visibility than CSI driver)
  - Network policies (zero-trust: default deny all, frontend → backend only, backend accepts frontend traffic only)
  - RBAC with service accounts and minimal permissions
  - Falco runtime monitoring with Falcosidekick UI
  - Security context: non-root containers (UID 1000), all capabilities dropped, seccomp profile
- **Vulnerability Scanning**: Trivy in Jenkins CI/CD pipeline (fails on CRITICAL, archives HIGH/MEDIUM/LOW reports)
- **High Availability**: 
  - Pod disruption budgets (minAvailable: 1 for backend/frontend)
  - PodAntiAffinity spreads replicas across nodes
  - Rolling updates (MaxUnavailable=1, MaxSurge=1) for zero-downtime deployments
  - Horizontal Pod Autoscaler (dev: 2-5 backend replicas, prod: 3-7 backend replicas, 75% CPU target)
  - Health probes: liveness (restart unhealthy pods after 30s), readiness (traffic routing control after 5s)
- **Monitoring Stack**: Prometheus metrics collection, Grafana visualization dashboards, Metrics Server for HPA, Falco security events
- **Build System**: Jenkins with Docker-in-Docker (privileged containers) for container image builds, kubectl for deployments

### Capacity Planning Details
**Dev Environment (5 t3.small nodes = 55 pod capacity)**
- Steady State: ~31-32 pods (56-58% utilization): System 11, ALB 2, Monitoring 6-7, Falco 8, App 4
- With HPA Max: ~36-37 pods (65-67% utilization): App scales to 9 pods
- Jenkins Pipeline: +1-2 temporary pods during builds

**Prod Environment (4 t3.medium nodes = 68 pod capacity)**
- Steady State: ~32-33 pods (47-49% utilization): System 11, ALB 2, Monitoring 6-7, Falco 7, App 6
- With HPA Max: ~41-42 pods (60-62% utilization): App scales to 13 pods
- Jenkins Pipeline: +1-2 temporary pods during builds

## Target Users
- **DevOps Engineers**: Learning modern AWS (2023-2024 features) and Kubernetes patterns with hands-on examples
- **Cloud Architects**: Evaluating infrastructure design decisions, architecture tradeoffs, and security practices
- **Platform Engineers**: Understanding multi-environment deployment strategies, CI/CD pipelines, GitOps workflows
- **Hiring Managers**: Assessing hands-on cloud engineering capabilities with production-ready implementations
- **Students/Learners**: Educational resource for AWS, Terraform, Kubernetes best practices with real-world complexity

## Use Cases
- **Portfolio Demonstration**: Showcasing production-ready cloud engineering skills with 85+ AWS resources and modern features
- **Learning Reference**: Educational resource with architecture decisions documented, tradeoffs explained, debugging examples
- **Architecture Template**: Foundation for similar cloud-native platforms with reusable Terraform modules and Helm charts
- **Interview Preparation**: Technical discussion points covering AWS, Kubernetes, security, CI/CD, monitoring, cost optimization
- **Hands-On Practice**: Working environment for testing AWS services, Kubernetes features, deployment strategies