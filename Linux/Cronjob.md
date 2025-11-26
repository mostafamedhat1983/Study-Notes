---
tags:
  - Linux
---
## crontab

## install cronie

sudo yum install cronie -y

## start and check the service

sudo systemctl start crond

sudo systemctl status crond

## add cronjob for root

sudo crontab -e

## add cronjob for another user(you must have `sudo` or root access)

 sudo crontab -u otheruser -e