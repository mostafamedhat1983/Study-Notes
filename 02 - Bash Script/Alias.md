---
tags:
  - Bash_Script
  - Linux
---
An alias in bash is a user-defined shortcut that maps a simple command name to a longer or more complex command sequence. When you execute an alias, the bash interpreter replaces the alias name with the full command string it represents, saving you from repetitive typing.​

## Syntax and Creation

The basic syntax for creating an alias is:​

text
``` bash
alias shortname='longer command 
```


For example, `alias ll='ls -la'` creates a shortcut `ll` that executes `ls -la` to list all files in long format. The command string must be enclosed in quotes, with no spaces around the equal sign.​

## Where Aliases Are Defined

Aliases are typically defined in shell configuration files like `~/.bashrc` or `~/.bash_profile`. These files load when you start a new shell session, making your aliases available automatically. You can also create temporary aliases directly in the terminal that only last for the current session.​

## Limitations in Scripts

Bash does not expand aliases in non-interactive shells by default, so aliases defined in ~/.bashrc normally are not available to a script executed as bash script.sh. Enabling shopt -s expand_aliases changes this behavior, but shell functions are generally clearer for reusable script logic. Alias expansion occurs when Bash reads commands, not when it executes them. Sourcing a file makes its definitions available in the current shell; executing a file runs it in a separate process.[](https://stackoverflow.com/questions/24054154/how-do-create-an-alias-in-shell-scripts)​

## Common Use Cases

Aliases are particularly useful for commands you run frequently, such as adding default options to common commands, creating shortcuts for long command sequences, or simplifying navigation and file operations.​