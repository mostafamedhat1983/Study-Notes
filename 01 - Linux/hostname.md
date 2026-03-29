---
tags:
  - Linux
---
``` bash 
mosta@Mostafa-LT:~$ hostname
Mostafa-LT

```

```bash
mosta@Mostafa-LT:~$ hostnamectl
 Static hostname: Mostafa-LT
       Icon name: computer-container
         Chassis: container ☐
      Machine ID: 44909182e51341de8e5ed2d903235c1e
         Boot ID: 41ae94c8c7a54a6a85d63dd4bd374ce5
  Virtualization: wsl
Operating System: Ubuntu 24.04.2 LTS
          Kernel: Linux 6.6.87.2-microsoft-standard-WSL2
    Architecture: x86-64

```

# change hostname

## Change hostname permanently


``
```bash
sudo hostnamectl set-hostname new-hostname
```


This is the recommended way on systemd-based distributions such as Ubuntu, Debian, Fedora, and CentOS 7+. The change is meant to persist after reboot.

## Change hostname temporarily


```bash
sudo hostname new-hostname
```

This changes the hostname for the current session and usually reverts after reboot unless the persistent configuration is also updated

## Or edit /etc/hostname