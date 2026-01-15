An alias in bash is a user-defined shortcut that maps a simple command name to a longer or more complex command sequence. When you execute an alias, the bash interpreter replaces the alias name with the full command string it represents, saving you from repetitive typing.​

## Syntax and Creation

The basic syntax for creating an alias is:​

text

`alias shortname='longer command'`

For example, `alias ll='ls -la'` creates a shortcut `ll` that executes `ls -la` to list all files in long format. The command string must be enclosed in quotes, with no spaces around the equal sign.​

## Where Aliases Are Defined

Aliases are typically defined in shell configuration files like `~/.bashrc` or `~/.bash_profile`. These files load when you start a new shell session, making your aliases available automatically. You can also create temporary aliases directly in the terminal that only last for the current session.​

## Limitations in Scripts

Aliases have an important limitation: they cannot be used effectively in bash scripts that you execute. When a script finishes execution, any aliases defined within it are lost because they only exist in that shell process. However, if you define aliases in a separate file and **source** it (rather than execute it), the aliases become available in your current shell.[](https://stackoverflow.com/questions/24054154/how-do-create-an-alias-in-shell-scripts)​

## Common Use Cases

Aliases are particularly useful for commands you run frequently, such as adding default options to common commands, creating shortcuts for long command sequences, or simplifying navigation and file operations.​