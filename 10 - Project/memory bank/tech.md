# Technology Stack

## Programming Languages
- **HCL (Terraform)**: Infrastructure as Code for AWS resource provisioning
- **Python 3.11+**: Backend API development with FastAPI and Streamlit
- **YAML**: Kubernetes manifests, Helm charts, and Ansible playbooks
- **Bash/Shell**: Automation scripts and CI/CD pipelines

## Infrastructure Technologies

### Core AWS Services
- **Amazon EKS 1.34**: Managed Kubernetes with Pod Identity and modern access patterns
- **Amazon RDS MySQL 8.0**: Managed database with Multi-AZ, encryption, and automated backups
- **AWS Bedrock DeepSeek V3.1**: Serverless AI model with 64K context window ($0.27/1M tokens)
- **AWS Secrets Manager**: Encrypted credential storage with KMS and automatic rotation
- **Amazon ECR**: Private container registry with IAM authentication

### Infrastructure as Code
- **Terraform >= 1.0**: Resource provisioning with S3 native state locking
- **Packer >= 1.8**: Immutable AMI builds with vulnerability scanning
- **Ansible >= 2.9**: Configuration management and software provisioning
- **Trivy v0.67.2**: Container and filesystem vulnerability scanning

### Container & Orchestration
- **Docker**: Container runtime and image building
- **Kubernetes**: Container orchestration with health probes and rolling updates
- **Helm 3.x**: Package manager for Kubernetes with multi-environment support

## Application Technologies

### Backend Stack
- **FastAPI**: Async Python web framework with automatic OpenAPI documentation
- **Pydantic**: Data validation and serialization with type hints
- **aiomysql**: Async MySQL driver with connection pooling
- **boto3**: AWS SDK for Python with Bedrock and Secrets Manager integration

### Frontend Stack
- **Streamlit**: Python web framework for data applications and dashboards
- **Session State Management**: Persistent user sessions across page reloads

### Development Dependencies
```python
# Backend (backend/requirements.txt)
fastapi==0.104.1
uvicorn==0.24.0
aiomysql==0.2.0
boto3==1.34.0
pydantic==2.5.0

# Frontend (frontend/requirements.txt)
streamlit==1.28.1
requests==2.31.0
```

## Build Systems & Tools

### Infrastructure Build
```bash
# Terraform workflow
terraform init
terraform plan
terraform apply

# Packer AMI build
packer init jenkins.pkr.hcl
packer build jenkins.pkr.hcl

# Ansible provisioning (called by Packer)
ansible-playbook jenkins-playbook.yml
```

### Application Build
```bash
# Container builds
docker build -t platform-app:backend .
docker build -t platform-app:frontend .

# Kubernetes deployment
helm upgrade --install chatbot ./k8s -f values-dev.yaml
kubectl get pods -l app=chatbot-backend
```

## Development Environment

### Prerequisites
- **AWS CLI**: Configured with appropriate IAM permissions for us-east-2 region
- **kubectl**: Kubernetes command-line tool configured for EKS access
- **Docker**: Container runtime for local builds and testing
- **Git**: Version control with SSH key authentication

### Local Development Commands
```bash
# Infrastructure deployment
cd terraform/dev && terraform apply

# Application deployment  
cd k8s && helm upgrade --install chatbot . -f values-dev.yaml

# Local testing
kubectl port-forward service/chatbot-frontend 8501:8501
curl http://localhost:8000/health  # Backend health check
```

## Security & Compliance Tools
- **AWS KMS**: Encryption key management for data at rest
- **AWS IAM**: Identity and access management with least privilege policies
- **Trivy**: Vulnerability scanning for containers and filesystems
- **SSL/TLS**: End-to-end encryption for all network communications

## Monitoring & Observability
- **Kubernetes Health Probes**: Liveness and readiness checks for automatic recovery
- **AWS CloudTrail**: API call logging and audit trails
- **Container Logs**: Structured logging with kubectl and CloudWatch integration
- **Resource Monitoring**: CPU/memory usage tracking with Kubernetes metrics