---
tags:
  - AWS
  - SSM
---
aws ssm start-session  --target i-0f9018110f25c9885 --document-name AWS-StartPortForwardingSession  --parameters "portNumber=8080,localPortNumber=8080" --region=us-east-2