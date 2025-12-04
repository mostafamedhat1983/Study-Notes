---
tags:
  - AWS
  - SSM
---
aws ssm start-session  --target i-09423a1abf8dfac61 --document-name AWS-StartPortForwardingSession  --parameters "portNumber=8080,localPortNumber=8080" --region=us-east-2