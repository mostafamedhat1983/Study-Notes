---
tags:
  - Jenkins
  - AWS
  - Linux
---

# 1) Update system packages

sudo yum update -y

# 2) Add Jenkins repo

sudo wget -O /etc/yum.repos.d/jenkins.repo [https://pkg.jenkins.io/redhat-stable/jenkins.repo](https://pkg.jenkins.io/redhat-stable/jenkins.repo)

# 3) Import Jenkins GPG key

sudo rpm --import [https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key](https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key)

# 4) Refresh metadata

sudo yum makecache -y

# 5) Install Java (Amazon Corretto 21 for example)

sudo yum install -y java-21-amazon-corretto

# 6) Install Jenkins

sudo yum install -y jenkins

# 7) Enable and start Jenkins service

sudo systemctl enable jenkins  
sudo systemctl start jenkins

# 8) Check Jenkins service status

sudo systemctl status jenkins