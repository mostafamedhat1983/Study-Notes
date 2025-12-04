---
tags:
  - My_CV_Project
  - AWS
  - Kubernetes
  - EKS
---
# Secret Lifecycle with Init Container

## Overview

This document explains the complete lifecycle of database credentials using the **init container approach** for the AI chatbot platform. It shows where secrets exist at each step and their encryption status.

---

## Secret Lifecycle Stages

### 1. Storage (AWS Secrets Manager)

**Location:** AWS Secrets Manager  
**Encryption:** ✅ **Encrypted at Rest** (AWS KMS)  
**Format:** JSON object

```json
{
  "username": "admin",
  "password": "YourSecurePassword",
  "dbname": "platformdb",
  "host": "platform-dev-db.xxxxx.us-east-2.rds.amazonaws.com",
  "port": "3306"
}
```

**Security Details:**
- Encrypted with AWS KMS customer-managed key
- Access controlled via IAM policies (Pod Identity)
- Supports automatic rotation
- Audit trail in CloudTrail

---

### 2. Retrieval (Init Container Startup)

**Location:** Network transit (AWS API call)  
**Encryption:** ✅ **Encrypted in Transit** (TLS 1.2+)  
**Process:** Init container → AWS Secrets Manager API

**Init Container Code:**
```python
import boto3
import json
import os

# AWS SDK uses Pod Identity - no credentials needed
client = boto3.client('secretsmanager', region_name='us-east-2')

# Fetch secret over HTTPS
response = client.get_secret_value(SecretId='platform-db-dev-credentials')
secret = json.loads(response['SecretString'])

# Write to environment variables (memory only)
os.environ['DB_HOST'] = secret['host']
os.environ['DB_USER'] = secret['username']
os.environ['DB_PASSWORD'] = secret['password']
os.environ['DB_NAME'] = secret['dbname']
os.environ['DB_PORT'] = secret['port']
```

**Security Details:**
- HTTPS encryption for AWS API calls
- Pod Identity provides temporary AWS credentials
- No static credentials or tokens in pod
- Init container logs show success/failure (not secret values)

**Debugging:**
```bash
# View init container logs (shows fetch status, not secrets)
kubectl logs <pod-name> -c fetch-secrets
```

---

### 3. Environment Variables (Pod Memory)

**Location:** Application container environment  
**Encryption:** ⚠️ **Plaintext in Memory** (encrypted process memory at OS level)  
**Visibility:** Only accessible within pod

**Environment Variables Set:**
```bash
DB_HOST=platform-dev-db.xxxxx.us-east-2.rds.amazonaws.com
DB_USER=admin
DB_PASSWORD=YourSecurePassword
DB_NAME=platformdb
DB_PORT=3306
```

**Security Details:**
- Exists only in container process memory
- **Never written to disk** (no ConfigMaps, Secrets, or files)
- **Never in container images** (not baked into layers)
- **Never in Helm manifests** (only secret name referenced)
- **Never in logs** (application code excludes passwords from logging)
- Cleared when pod terminates

**What's NOT Exposed:**
- ❌ Not in `kubectl describe pod` output
- ❌ Not in `kubectl get pod -o yaml` output
- ❌ Not in container filesystem
- ❌ Not in Docker image layers
- ❌ Not in Helm chart values
- ❌ Not in application logs

---

### 4. Application Usage (Database Connection)

**Location:** MySQL connection pool  
**Encryption:** ✅ **Encrypted in Transit** (SSL/TLS)  
**Process:** Backend pod → RDS MySQL

**Backend Code (main.py):**
```python
import aiomysql
import os

# Read from environment variables (memory)
DB_CONFIG = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'db': os.getenv('DB_NAME'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'ssl': {'ssl': True}  # Force SSL/TLS encryption
}

# Create connection pool (credentials in memory)
db_pool = await aiomysql.create_pool(**DB_CONFIG)
```

**Security Details:**
- SSL/TLS encryption for all MySQL traffic
- Connection pooling reuses credentials without re-fetch
- RDS enforces encryption at rest (KMS)
- Network isolation (RDS in private subnets)

---

### 5. Pod Termination (Cleanup)

**Location:** None (memory cleared)  
**Encryption:** N/A (secrets no longer exist)  
**Process:** Kubernetes terminates pod

**What Happens:**
1. Kubernetes sends SIGTERM to container
2. Application gracefully closes database connections
3. Process memory cleared by OS
4. Environment variables destroyed
5. No secrets persist on disk or in cluster

**Security Details:**
- No cleanup required (secrets never persisted)
- No secret rotation needed in pod (restart fetches fresh secrets)
- No residual data in container filesystem

---

## Secret Lifecycle Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. AWS Secrets Manager                                          │
│    ✅ Encrypted at Rest (KMS)                                   │
│    Location: AWS managed storage                                │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ HTTPS (TLS 1.2+)
                         │ ✅ Encrypted in Transit
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. Init Container (fetch-secrets)                               │
│    ✅ Encrypted in Transit (AWS API over HTTPS)                 │
│    Location: Pod memory (temporary)                             │
│    Action: Fetch secret → Set environment variables             │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ Shared process namespace
                         │ ⚠️ Plaintext in memory
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. Application Container (chatbot-backend)                      │
│    ⚠️ Plaintext in Memory (OS-level encryption)                 │
│    Location: Environment variables (process memory)             │
│    Visibility: Pod only (not in manifests/logs/disk)            │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ MySQL SSL/TLS
                         │ ✅ Encrypted in Transit
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. RDS MySQL Database                                           │
│    ✅ Encrypted at Rest (KMS)                                   │
│    ✅ Encrypted in Transit (SSL/TLS)                            │
│    Location: Private subnet (network isolation)                 │
└─────────────────────────────────────────────────────────────────┘
                         │
                         │ Pod termination
                         │ Memory cleared
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. Cleanup                                                       │
│    ✅ Secrets destroyed (memory cleared)                        │
│    Location: None (no persistence)                              │
└─────────────────────────────────────────────────────────────────┘
```

---

## Encryption Status Summary

| Stage | Location | Encryption Status | Notes |
|-------|----------|-------------------|-------|
| **Storage** | AWS Secrets Manager | ✅ Encrypted at Rest (KMS) | Customer-managed key |
| **Retrieval** | Network (AWS API) | ✅ Encrypted in Transit (TLS 1.2+) | HTTPS only |
| **Pod Memory** | Environment Variables | ⚠️ Plaintext in Memory | OS-level process isolation |
| **Database Connection** | Network (MySQL) | ✅ Encrypted in Transit (SSL/TLS) | Forced SSL |
| **Database Storage** | RDS Volumes | ✅ Encrypted at Rest (KMS) | Managed by RDS |
| **Termination** | None | N/A | Memory cleared, no persistence |

---

## Security Guarantees

### ✅ What's Protected

1. **At Rest:** Secrets Manager (KMS), RDS storage (KMS)
2. **In Transit:** AWS API calls (TLS 1.2+), MySQL connections (SSL/TLS)
3. **Zero Persistence:** Never written to disk, ConfigMaps, Secrets, or logs
4. **Zero Exposure:** Not in images, manifests, or kubectl output
5. **Audit Trail:** CloudTrail logs all Secrets Manager access

### ⚠️ Acceptable Risk

**Plaintext in Memory:** Environment variables exist as plaintext in process memory. This is acceptable because:
- OS-level process isolation prevents cross-pod access
- Memory encrypted at rest by EKS node (encrypted EBS volumes)
- No persistence (cleared on pod termination)
- Industry-standard practice (all applications load secrets to memory)

---

## Init Container vs CSI Driver Comparison

| Aspect | Init Container | CSI Driver |
|--------|----------------|------------|
| **Encryption at Rest** | ✅ Same (Secrets Manager + KMS) | ✅ Same |
| **Encryption in Transit** | ✅ Same (TLS 1.2+) | ✅ Same |
| **Memory Storage** | ⚠️ Environment variables | ⚠️ Volume mount (still memory) |
| **Debugging** | ✅ Logs in app pod | ❌ Logs in kube-system |
| **Error Visibility** | ✅ Direct AWS API errors | ❌ CSI driver abstraction |
| **Architecture** | ✅ Simple (no CSI addon) | ❌ Complex (CSI + SecretProviderClass) |
| **Secret Rotation** | ⚠️ Manual pod restart | ✅ Automatic (with polling) |

**Conclusion:** Both approaches provide equivalent encryption. Init container chosen for operational simplicity and debugging visibility.

---

## Troubleshooting

### View Init Container Logs
```bash
# Check if secrets were fetched successfully
kubectl logs <pod-name> -c fetch-secrets

# Expected output (success):
# Successfully fetched secrets from AWS Secrets Manager
```

### Verify Environment Variables (Without Exposing Secrets)
```bash
# Check if environment variables are set (shows names only)
kubectl exec <pod-name> -c chatbot-backend -- env | grep DB_

# Expected output:
# DB_HOST=platform-dev-db.xxxxx.us-east-2.rds.amazonaws.com
# DB_USER=admin
# DB_PASSWORD=****** (masked by kubectl)
# DB_NAME=platformdb
# DB_PORT=3306
```

### Test Database Connection
```bash
# Backend health check includes DB connectivity
kubectl exec <pod-name> -c chatbot-backend -- curl localhost:8000/health

# Expected output:
# {"status": "healthy", "database": "connected"}
```

---

## Secret Rotation Process

**When RDS password changes:**

1. Update secret in AWS Secrets Manager (manual or automatic rotation)
2. Restart pods to fetch new credentials:
   ```bash
   kubectl rollout restart deployment chatbot-backend
   ```
3. Init container fetches updated secret on pod startup
4. Application connects with new credentials

**No code changes required** - init container always fetches latest secret value.

---

## Best Practices

1. ✅ **Never log secrets:** Application code excludes passwords from logs
2. ✅ **Use SSL/TLS everywhere:** Force encryption for all network connections
3. ✅ **Least privilege IAM:** Pod Identity policy allows only `secretsmanager:GetSecretValue`
4. ✅ **Audit access:** Monitor CloudTrail for Secrets Manager API calls
5. ✅ **Rotate regularly:** Update RDS passwords periodically (manual or automatic)
6. ✅ **Test rotation:** Verify pod restart fetches new credentials successfully

---

## References

- **Infrastructure Repository:** `terraform-aws-eks` (Secrets Manager, RDS, Pod Identity)
- **Application Repository:** `platform-ai-chatbot` (Init container implementation)
- **Kubernetes Manifests:** `k8s/templates/chatbot-backend-deployment.yaml`
- **Backend Code:** `backend/main.py` (Database connection with SSL/TLS)

---

**Last Updated:** 2025-01-XX  
**Author:** Mostafa Medhat  
**Project:** AI Chatbot Platform with AWS Bedrock
