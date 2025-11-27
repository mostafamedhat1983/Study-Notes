---
tags:
  - Jenkins
  - Docker
---
sudo usermod -aG docker jenkins

 sudo systemctl restart docker

newgrp docker

 sudo systemctl restart jenkins