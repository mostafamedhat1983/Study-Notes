---
tags:
  - Ansible
---
## 1. Security

- `command` does **not** run through a shell (`/bin/sh`), so:
    
    - No shell operators: `|`, `>`, `<`, `&&`, `||`, `;`, etc.
        
    - Much harder to get **command injection** if any part of the command ever came from variables or user input.[](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/shell_module.html)​
        
- `shell` runs via a shell, so any unquoted variable can become an injection point.[](https://stackoverflow.com/questions/56663332/what-is-the-difference-between-shell-and-command-in-ansible)​
    

For AMI builds and infra automation, picking the safer primitive (`command`) is exactly what most security guides recommend

## 2. Predictability

`command` is very literal: it just runs the binary and arguments you give it. No surprises from the remote user’s shell environment, aliases, or functions.[](https://blog.krybot.com/t/the-shell-and-command-dilemma-in-ansible-modules/25175)​

That means:

- `docker --version` runs the same way everywhere
    
- No dependency on `.bashrc`, aliases, or weird PATH tricks
    

For golden images, you want this “boring and predictable” behavior.