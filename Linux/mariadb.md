---
tags:
  - Linux
  - database
---
# mariadb service wont start beacaue mariadb folder was not owned by mysql
log viewed from location var/log/mariadb

## change owner using chown

chown user:group file or folder

sudo systemctl restart mariadb 

sudo systemctl status mariadb