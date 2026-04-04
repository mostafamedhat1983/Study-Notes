---
tags:
  - Linux
---
In Linux, each user has a numeric **user ID** (UID) that the kernel uses to identify them and decide permissions on files and processes. The username is just a label; the UID is what actually matters for access control.

## What is a UID?

- A UID is a unique integer assigned to each user account on the system.
    
- It is stored in `/etc/passwd` (or a central directory like LDAP) along with the username, home directory, shell, etc.
    
- Special ranges have meaning, for example:
    
    - UID 0 is `root` (full privileges).
        
    - Low UIDs (like 1–999 on many distros) are often reserved for system and service accounts.
        
    - Regular human users typically start from 1000 upward on many modern distributions.
        

## How to see your UID

Run:

``` bash
id
```

Typical output:

``` bash
uid=1000(username) gid=1000(username) groups=1000(username),27(sudo),...
```

- `uid=1000` here is your user ID.
    
- You can get just the UID with:
    
``` bsah
id -u
```

And for another user:

``` bash
id -u someuser
```



| User type           | Description                                                                      | Typical UID range  | Can log in interactively?   | Default shell (common)              | Main purpose                                                     |
| ------------------- | -------------------------------------------------------------------------------- | ------------------ | --------------------------- | ----------------------------------- | ---------------------------------------------------------------- |
| Root (superuser)    | The built‑in administrator account with full system control.                     | `0`                | Yes (often discouraged)     | `/bin/bash` or `/bin/sh`            | Install software, manage users, change configs.                  |
| Normal user         | Regular human account for daily work; limited to own files and directories.      | ≥1000              | Yes                         | `/bin/bash` or `/bin/sh`            | Coding, browsing, running apps in home directory.                |
| Service/system user | Non‑human account used by daemons/services (e.g., `nginx`, `mysql`, `www‑data`). | 1–999 (often 1–99) | Usually no (no login shell) | `/usr/sbin/nologin` or `/bin/false` | Run background services with restricted privileges for security. |
## Where user info is stored

- `/etc/passwd` – Basic user data: username, UID, GID, home directory, shell.
    
- `/etc/shadow` – Encrypted passwords and password‑policy settings (readable only by root).
    
- `/etc/group` – Group definitions that users can belong to (for shared permissions).

## Create / modify / delete users

- `useradd` – create a new user account.  
    Example: `sudo useradd -m -s /bin/bash james`
    `-m` – create the user’s home directory under `/home` (e.g., `/home/username`)
    `-s` – specify the **login shell** for this user (e.g., `/bin/bash`, `/bin/zsh`, `/bin/false`, etc.).
    
- `adduser` – higher‑level wrapper on some distros (e.g., Debian/Ubuntu) that prompts for password and home directory.  
    Example: `sudo adduser mostafa`
    - Automatically creates a home directory, often sets a default shell, and **prompts** you for password and extra info (full name, phone, etc.).
    
- `sudo useradd -s /sbin/nologin username`  create user with a non-interactive shell

- `sudo useradd -e YYYY-MM-DD username`  create user with expiry date 
    
- `passwd` – set or change a user’s password.  
    Example: `sudo passwd james`
    
- `usermod` – modify an existing user (shell, home, groups, UID, etc.).  
    Example: `sudo usermod -s /bin/zsh james`
    
- `userdel` – delete a user account.  
    Example: `sudo userdel -r james` (with home + mail)
    

## Groups and permissions

- `groupadd` – create a new group.  
    Example: `sudo groupadd developers`
    
- `usermod -a -G` – add a user to a supplementary group.  
    Example: `sudo usermod -a -G developers james`
    - `-a` – **append** (add to existing groups; do not remove any).
     `-G` – specify the **supplementary(secondary) groups** to add the user to.
     
- `groups` – list groups a user belongs to.  
    Example: `groups james`
    
- `id` – show UID, GID, and groups for a user.  
    Example: `id james`
    

## List users and inspect accounts

- `cat /etc/passwd` – view all user accounts (name, UID, GID, home, shell).
    
- `cut -d: -f1 /etc/passwd` – list just usernames.
    
- `who` – show who is currently logged in.
    
- `w` – show who is logged in plus their processes.
    
- `last` – show recent login history.
    
- `lsof -u username` – List all files opened by a specific user.

