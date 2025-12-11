---
tags:
  - AWS
  - SSM
---
aws ssm start-session  --target i-0cca7e05036eb7746 --document-name AWS-StartPortForwardingSession  --parameters "portNumber=8080,localPortNumber=8080" --region=us-east-2