you can set a variable in Linux by writing `variable=value` 
```bash
myVar="Hello World"
```
but this variable will be a shell variable that is only available in the current shell session
to reference the variable value use $
```bash
 echo $PATH
```
The command `echo $PATH` displays the current value of the **PATH environment variable**, which is a list of directories your system searches to find the commands you type.​

* Why is it important?

When you type a command (like `ls` or `python`) without specifying its full location (like `/bin/ls`), Linux doesn't check every folder on your hard drive. Instead, it looks through the directories listed in `$PATH`, in order, from left to right.
**Order matters**: If you have two versions of a program, the system runs the one found in the _first_ directory listed
## Environment Variables

To make a variable available to child processes and other programs, you need to export it using the `export`
```bash
export VARIABLE_NAME=value
```
You can also first set a variable and then export it separately:
```bash
JAVA_HOME=/usr/bin/java
export JAVA_HOME
```
## Important Notes

Variables with multiple values should be separated by colons, and values containing spaces must be enclosed in quotes:
```bash
PATH=/usr/bin:/usr/local/bin
DESCRIPTION="Value with spaces"
```

---
