---
tags:
  - Jenkins
---
sudo vim /lib/systemd/system/jenkins.service

change Environment="JENKINS_PORT=8080"

then restart jenkins

sudo systemctl daemon-reload

 sudo service jenkins restart