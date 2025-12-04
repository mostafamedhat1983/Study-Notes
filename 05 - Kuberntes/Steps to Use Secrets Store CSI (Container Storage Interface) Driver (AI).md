---
tags:
  - AWS
  - Kubernetes
  - EKS
---


   Step 1: Create Secret in AWS Secrets Manager
   •  Store database credentials in AWS Secrets Manager (JSON format with username, password, dbname)
   •  Use naming convention that matches your infrastructure's IAM policy (e.g., platform-*-dev-*)

   Step 2: Create IAM Role for Backend Pods
   •  Create IAM role with permissions to read specific secrets from Secrets Manager
   •  Configure trust policy for EKS Pod Identity (pods.eks.amazonaws.com)

   Step 3: Create Pod Identity Association
   •  Link the IAM role to a Kubernetes service account in your namespace
   •  This allows pods using that service account to assume the IAM role

   Step 4: Create Service Account in Kubernetes
   •  Define a service account that pods will use
   •  This gets associated with the IAM role via Pod Identity

   Step 5: Create SecretProviderClass Manifest
   •  Define which secrets to fetch from AWS Secrets Manager
   •  Specify how to map secret keys to files
   •  Optionally sync to Kubernetes Secret for environment variables

   Step 6: Update Deployment to Use Service Account
   •  Add serviceAccountName to pod spec

   Step 7: Add Volume and Volume Mount to Deployment
   •  Add CSI volume referencing the SecretProviderClass
   •  Mount the volume to a path in the container (e.g., /mnt/secrets)

   Step 8: Update Application Code (Optional)
   •  Modify backend to read credentials from mounted files instead of environment variables
   •  Or use the synced Kubernetes Secret as environment variables