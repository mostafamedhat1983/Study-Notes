---
tags:
  - AWS
  - SSM
---
aws ssm start-session  --target i-05a9a97c63053ff4d --document-name AWS-StartPortForwardingSession  --parameters "portNumber=8501,localPortNumber=8501" --region=us-east-2