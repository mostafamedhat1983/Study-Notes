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