---
tags:
  - Jenkins
  - AWS
  - SSM
---
 `aws ssm start-session \`

    --target i-0d39a2a638b6901f0 \

     --document-name AWS-StartPortForwardingSessionToRemoteHost \

     --parameters '{"portNumber":["8080"], "localPortNumber":["8080"]}' --region=us-east-2