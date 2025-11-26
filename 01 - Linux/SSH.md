---
tags:
  - Linux
  - SSH
---
# ssh config file
/etc/ssh/sshd_config  

---
# password-less authentication linux using ssh key

## generate ssh key

ssh-keygen -t rsa -b 4096
## copy key to remote machine

ssh-copy-id user@remote_server_ip