---
tags:
  - AWS
  - Trivy
---
aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin 1234567890.dkr.ecr.us-east-2.amazonaws.com
docker pull 1234567890.dkr.ecr.us-east-2.amazonaws.com/platform-app:21-frontend
sudo trivy image 1234567890.dkr.ecr.us-east-2.amazonaws.com/platform-app:21-frontend