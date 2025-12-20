---
tags:
  - memory-bank
last_updated: 2025-12-13
---
# Project Structure

## Multi-Repository Architecture
Two separate repositories following platform engineering best practices with clear separation of concerns:

### terraform-aws-eks (Infrastructure)
```
terraform-aws-eks/
├── terraform/
│   ├── dev/                    # Development environment (~$207/mo)
│   │   ├── main.tf             # Resource definitions
│   │   ├── variables.tf        # Input variables
│   │   ├── outputs.tf          # Output values
│   │   ├── provider.tf         # AWS provider config
│   │   └── backend.tf          # S3 state backend with native locking
│   ├── prod/                   # Production environment (~$289/mo)
│   │   └── (same structure as dev)
│   └── modules/                # Reusable Terraform modules
│       ├── network/            # VPC, subnets, Regional NAT Gateway, security groups
│       ├── ec2/                # EC2 instances with encrypted EBS volumes
│       ├── rds/                # RDS MySQL with Secrets Manager integration
│       ├── role/               # Flexible IAM role module (EC2, EKS, Pod Identity)
│       └── eks/                # Complete EKS 1.34 cluster with Pod Identity, EBS CSI
├── packer/
│   ├── jenkins.pkr.hcl         # Packer template for Jenkins AMI
│   ├── manifest.json           # AMI build output (gitignored)
│   └── ansible/
│       └── jenkins-playbook.yml # Ansible provisioning (Docker, Java, Jenkins, kubectl)
├── docs/                       # Architecture decisions, security docs
└── README.md                   # Comprehensive infrastructure documentation
```

### platform-ai-chatbot (Application)
```
platform-ai-chatbot/
├── backend/
│   ├── main.py                 # FastAPI app with Bedrock DeepSeek V3.1 & MySQL
│   ├── Dockerfile              # Backend container image
│   └── requirements.txt        # Python dependencies (FastAPI, aiomysql, boto3, slowapi)
├── frontend/
│   ├── app.py                  # Streamlit UI with session management & error handling
│   ├── Dockerfile              # Frontend container image
│   └── requirements.txt        # Python dependencies (Streamlit, requests)
├── jenkins/
│   ├── Jenkinsfile                  # Application CI/CD pipeline (build, scan, deploy)
│   ├── Jenkinsfile-setup            # kubectl context configuration (runs on Jenkins EC2)
│   ├── Jenkinsfile-alb-controller   # ALB controller deployment (Kubernetes pod agent)
│   └── Jenkinsfile-monitoring       # Monitoring stack deployment (Prometheus, Grafana, Falco)
├── k8s/
│   ├── Chart.yaml              # Helm chart metadata v0.1.0
│   ├── values.yaml             # Default Helm values (shared base config)
│   ├── values-dev.yaml         # Development overrides (2 replicas, cost-optimized resources)
│   ├── values-prod.yaml        # Production overrides (3 replicas, Multi-AZ RDS)
│   ├── grafana-ingress.yaml    # Grafana ALB ingress (used by monitoring pipeline)
│   └── templates/              # Kubernetes manifests
│       ├── chatbot-backend-deployment.yaml      # Backend pods with init container secrets
│       ├── chatbot-backend-service.yaml         # Backend ClusterIP service (port 8000)
│       ├── chatbot-backend-service-account.yaml # Pod Identity for AWS Bedrock + Secrets Manager
│       ├── chatbot-backend-rbac.yaml            # RBAC Role + RoleBinding (minimal permissions)
│       ├── chatbot-backend-pdb.yaml             # Pod Disruption Budget (minAvailable: 1)
│       ├── chatbot-backend-hpa.yaml             # Horizontal Pod Autoscaler (75% CPU target)
│       ├── chatbot-frontend-deployment.yaml     # Frontend pods
│       ├── chatbot-frontend-service.yaml        # Frontend ClusterIP service (port 8501)
│       ├── chatbot-frontend-pdb.yaml            # Frontend disruption budget
│       ├── ingress.yaml                         # Chatbot ALB ingress with TLS
│       ├── network-policy-default-deny.yaml     # Default deny all traffic (zero-trust)
│       ├── network-policy-backend.yaml          # Backend ingress (frontend only) + egress (RDS, AWS APIs)
│       ├── network-policy-frontend.yaml         # Frontend ingress (ALB) + egress (backend)
│       └── storage-class.yaml                   # EBS gp3 StorageClass with encryption
├── docs/
│   └── images/                 # Screenshots and architecture diagrams
└── README.md                   # Application deployment guide with SSL setup

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
- **Security Layer**: Pod Identity, init container secrets, RBAC, network policies, Falco runtime monitoring
- **CI/CD Layer**: Jenkins pipelines with Docker-in-Docker builds and Trivy vulnerability scanning

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

## Capacity Planning

### Dev Environment (5 t3.small nodes = 55 pod capacity)
- **Steady State**: ~31-32 pods (56-58% utilization)
  - System (kube-system): 11 pods
  - ALB Controller: 2 pods
  - Monitoring: 6-7 pods (Prometheus, Grafana, Metrics Server)
  - Falco: 8 pods (5 daemonset + 3 supporting)
  - Application: 4 pods (2 backend + 2 frontend)
- **With HPA Max Scaling**: ~36-37 pods (65-67% utilization)
  - Application scales to 9 pods (5 backend + 4 frontend)

### Prod Environment (4 t3.medium nodes = 68 pod capacity)
- **Steady State**: ~32-33 pods (47-49% utilization)
  - System (kube-system): 11 pods
  - ALB Controller: 2 pods
  - Monitoring: 6-7 pods
  - Falco: 7 pods (4 daemonset + 3 supporting)
  - Application: 6 pods (3 backend + 3 frontend)
- **With HPA Max Scaling**: ~41-42 pods (60-62% utilization)
  - Application scales to 13 pods (7 backend + 6 frontend)

### Jenkins Pipeline Pods (Temporary)
- Application Pipeline: 1 pod (docker-dind + kubectl containers)
- Max concurrent: 1-2 pipeline pods during builds
- Adds 2-4 pods to steady state during CI/CD operations

## Component Relationships
1. **Infrastructure Foundation**: Terraform provisions AWS resources (VPC, EKS, RDS)
2. **AMI Pipeline**: Packer + Ansible creates immutable Jenkins images
3. **Application Deployment**: Helm charts deploy containerized apps to EKS
4. **Service Integration**: Applications use Pod Identity to access AWS services
5. **Data Flow**: Frontend → Backend → Bedrock AI + MySQL persistence