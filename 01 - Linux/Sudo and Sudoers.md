---
tags:
  - Linux
---
## What `sudo` does

`sudo` lets **certain users run commands as another user** (usually `root`) based on rules in `/etc/sudoers`.

- Typical use:
    
    
    `sudo systemctl restart nginx`
    
- You keep your own account, but the command runs with elevated privileges.


## `sudo` vs `sudo -i`

- `sudo command`  
    Runs a **single command** as the target user (usually root) and returns to your normal shell.  
    Example:
    
    `sudo yum install git`
    
- `sudo -i`  
    Opens a **login shell as root**:
    
    - Changes your working directory to `/root`.
        
    - Loads root’s environment (`.profile`, `.bashrc`, etc.).  
        Example:
        
        `sudo -i # now you’re in a full root session`
        

## `/etc/sudoers` – the core config

- Location:
    `/etc/sudoers`
    
- This file defines:
    
    - Which users (or groups) can use `sudo`.
        
    - Which commands they can run and as which user.
        

## Editing securely with `visudo`

You **must not edit `/etc/sudoers` directly** with plain `vim` or `nano` because a syntax error can break `sudo`.

- Use:
    
    `sudo visudo`
    
- `visudo`:
    
    - Opens the file in an editor.
        
    - Checks syntax before saving; if there’s an error, it won’t let you write it until you fix it.
        

Example of a user line:

text

`root    ALL=(ALL) ALL 
`ansible ALL=(ALL) ALL`

This means `ansible` can run **any command** as **any user** on **any host** (in a single‑host setup, the host part is ignored).

---

## Using groups via `/etc/sudoers.d/`

Instead of editing `/etc/sudoers` directly, it’s safer and cleaner to use `/etc/sudoers.d/`.

- Directory:
    
    `/etc/sudoers.d/`
    
- Example: create a file `/etc/sudoers.d/devops`:
    
    `%devops ALL=(ALL) ALL`
    
    - `%devops` = the group `devops`.
        
    - Any user in that group can use `sudo` as defined.
        

You can also give a user passwordless sudo:

`ansible ALL=(ALL) NOPASSWD: ALL`

This means `ansible` can run `sudo` without being prompted for its own password, which is useful for scripts and automation.

---

## Common sudo patterns and commands

## Basic sudo usage

- Run a command as root:
    
    `sudo systemctl restart nginx`
    
- Switch to a full root login shell:
    
    `sudo -i`
    
- Run a command as a different user:
    
    `sudo -u www-data whoami`
    
- List your sudo privileges:
    
    `sudo -l`
    

## Config‑related commands

- Edit main sudoers safely:
    
    `sudo visudo`
    
- Edit a file in the sudoers.d directory:
    
    `sudo visudo -f /etc/sudoers.d/devops`
    

## Restricted sudo (for specific commands)

You can restrict a user to only certain commands:

`ansible ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx, /usr/bin/haproxyctl reload`

This means `ansible` can run only those commands with `sudo`, and without a password prompt.

---

## Key points to remember for your notes

- `sudo` = controlled way for normal users to run commands with elevated privileges.
    
- `/etc/sudoers` = the main config; edit only via `visudo`.
    
- `/etc/sudoers.d/` = preferred place for custom rules (per user, per group).
    
- `sudo -i` = full login shell as root; `sudo command` = single command as root.
    
- `NOPASSWD: ALL` lets a user `sudo` without password; use cautiously.