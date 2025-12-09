---
tags:
  - memory-bank
---
# Product Overview

## Project Purpose
Multi-repository AWS cloud platform demonstrating enterprise-grade DevOps practices through infrastructure-as-code and cloud-native application deployment. Built from scratch to showcase modern AWS services, Kubernetes orchestration, and production-ready security patterns.

## Value Proposition
- **Production-Ready Infrastructure**: 75+ AWS resources across 2 environments with comprehensive security, encryption, and high availability
- **Modern Cloud Architecture**: EKS, RDS, Bedrock AI integration with Pod Identity, secrets management, and zero-credential exposure
- **DevOps Excellence**: Immutable infrastructure with Packer/Ansible, Terraform modules, Helm deployments, and CI/CD automation
- **Learning Platform**: Educational project demonstrating real-world cloud engineering skills and best practices

## Key Features

### Infrastructure (terraform-aws-eks)
- **Multi-Environment Setup**: Development ($192/mo) and Production ($259/mo) with strategic cost optimization
- **Regional NAT Gateway**: December 2024 AWS feature providing HA across all AZs at single NAT cost
- **AWS EKS Cluster**: Kubernetes 1.34 with Pod Identity, EBS CSI Driver, and modern access management
- **Immutable AMIs**: Packer + Ansible builds with Trivy vulnerability scanning
- **Security Hardening**: Encryption at rest/transit, IAM least privilege, SSM Session Manager access
- **Modular Terraform**: Reusable modules for network, compute, database, and IAM components

### Application (platform-ai-chatbot)
- **AI Integration**: AWS Bedrock DeepSeek V3.1 with serverless pay-per-use model
- **Full-Stack Architecture**: FastAPI backend + Streamlit frontend with MySQL persistence
- **Cloud-Native Deployment**: Kubernetes with Helm charts, health probes, and zero-downtime updates
- **Enterprise Security**: Pod Identity, init container secrets, network policies, Pod Security Standards, Falco runtime monitoring
- **Vulnerability Scanning**: Trivy in CI/CD pipeline (fails on CRITICAL, archives reports)
- **High Availability**: Pod disruption budgets, anti-affinity rules, rolling deployments, HPA

## Target Users
- **DevOps Engineers**: Learning modern AWS and Kubernetes patterns
- **Cloud Architects**: Evaluating infrastructure design decisions and security practices
- **Platform Engineers**: Understanding multi-environment deployment strategies
- **Hiring Managers**: Assessing hands-on cloud engineering capabilities

## Use Cases
- **Portfolio Demonstration**: Showcasing production-ready cloud engineering skills
- **Learning Reference**: Educational resource for AWS, Terraform, and Kubernetes best practices
- **Architecture Template**: Foundation for similar cloud-native platforms
- **Interview Preparation**: Technical discussion points for DevOps/cloud engineering roles