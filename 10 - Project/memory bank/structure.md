---
tags:
  - memory-bank
---
# Project Structure

## Multi-Repository Architecture
Two separate repositories following platform engineering best practices:

### terraform-aws-eks (Infrastructure)
```
terraform-aws-eks/
├── terraform/
│   ├── dev/                    # Development environment
│   ├── prod/                   # Production environment  
│   └── modules/                # Reusable Terraform modules
│       ├── network/            # VPC, subnets, NAT, security groups
│       ├── ec2/                # EC2 instances with encrypted EBS
│       ├── rds/                # RDS with Secrets Manager integration
│       ├── role/               # Flexible IAM role module
│       └── eks/                # Complete EKS cluster setup
├── packer/
│   ├── jenkins.pkr.hcl         # Packer template for Jenkins AMI
│   └── ansible/
│       └── jenkins-playbook.yml # Ansible provisioning
└── docs/                       # Private documentation (gitignored)
```

### platform-ai-chatbot (Application)
```
platform-ai-chatbot/
├── backend/
│   ├── main.py                 # FastAPI app with Bedrock & MySQL
│   ├── Dockerfile              # Backend container image
│   └── requirements.txt        # Python dependencies
├── frontend/
│   ├── app.py                  # Streamlit UI with session management
│   ├── Dockerfile              # Frontend container image
│   └── requirements.txt        # Python dependencies
└── k8s/
    ├── Chart.yaml              # Helm chart metadata
    ├── values*.yaml            # Environment-specific configurations
    └── templates/              # Kubernetes manifests
```

## Core Components

### Infrastructure Layer (terraform-aws-eks)
- **Network Module**: Multi-AZ VPC with public/private subnets, Regional NAT Gateway, security groups
- **Compute Module**: EC2 instances with encrypted EBS volumes and IAM roles
- **Database Module**: RDS MySQL with Multi-AZ, encryption, and Secrets Manager integration
- **Container Module**: EKS cluster with Pod Identity, node groups, and CSI drivers
- **Security Module**: IAM roles, policies, KMS keys, and access management

### Application Layer (platform-ai-chatbot)
- **Backend Service**: FastAPI REST API with async MySQL and Bedrock integration
- **Frontend Service**: Streamlit web interface with session state management
- **Deployment Layer**: Helm charts with multi-environment support, health monitoring, HPA
- **Security Layer**: Pod Identity, init container secrets, RBAC, network policies, Pod Security Standards, Falco
- **CI/CD Layer**: Jenkins pipelines with Trivy vulnerability scanning

## Architectural Patterns

### Infrastructure as Code
- **Modular Design**: Reusable Terraform modules with clear interfaces
- **Environment Separation**: Dev/prod isolation with shared modules
- **State Management**: S3 backend with native locking (2024 feature)
- **Immutable Infrastructure**: Packer-built AMIs with Ansible provisioning

### Cloud-Native Application
- **Microservices**: Separate backend/frontend with clear API boundaries
- **Container Orchestration**: Kubernetes deployment with Helm packaging
- **Serverless Integration**: AWS Bedrock for AI without infrastructure management
- **Database Persistence**: RDS MySQL with connection pooling and SSL/TLS

### Security Architecture
- **Defense in Depth**: Multiple security layers from network to application
- **Zero Trust**: Pod Identity authentication without static credentials
- **Encryption Everywhere**: At rest (KMS), in transit (TLS), in use (memory-only)
- **Least Privilege**: IAM policies restricted to specific resources and actions

## Component Relationships
1. **Infrastructure Foundation**: Terraform provisions AWS resources (VPC, EKS, RDS)
2. **AMI Pipeline**: Packer + Ansible creates immutable Jenkins images
3. **Application Deployment**: Helm charts deploy containerized apps to EKS
4. **Service Integration**: Applications use Pod Identity to access AWS services
5. **Data Flow**: Frontend → Backend → Bedrock AI + MySQL persistence