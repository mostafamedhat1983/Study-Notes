---
tags:
  - Linux
  - Bash_Script
---
Script sourcing in bash is a method of executing a script within the **current shell process** rather than spawning a new subshell. When you source a script using the `source` command or the dot (`.`) operator, all commands, variables, and functions defined in that script become part of your current shell environment and persist after the script finishes.​

## How to Source a Script

You can source a script using two equivalent commands:[](https://dev.to/amrhrabdeen/difference-between-executing-a-script-and-sourcing-it-in-linux-570f)​

- `source scriptname.sh` - The bash-specific command
    
- `. scriptname.sh` - The POSIX-standard syntax (dot-space-filename)
    

## Key Difference from Regular Execution

When you execute a script normally (like `./script.sh`), bash creates a new subshell process where the script runs. Any variables, functions, or directory changes made in that subshell are destroyed when the script completes. However, sourcing runs everything in your current shell, making all changes persistent.​

## Common Use Cases

Sourcing is particularly useful for configuration files like `.bashrc` or `.bash_profile`, where you want environment changes to take effect in your current session. It's also valuable for modular programming in bash, allowing you to load reusable functions and variables from external files into your main script. This follows the DRY (Don't Repeat Yourself) principle by breaking code into smaller, reusable modules.​