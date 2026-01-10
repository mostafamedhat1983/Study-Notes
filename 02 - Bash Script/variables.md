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
## export vs declare
Both `export` and `declare` are shell built-in commands used to manage variables, but they serve different primary purposes.

## Quick Comparison

|Feature|`export`|`declare`|
|---|---|---|
|**Primary Purpose**|Makes variables available to **child processes** (subshells) ​.|Defines variable **attributes** (type, scope, read-only status) [](https://stackoverflow.com/questions/56627534/in-bash-should-i-use-declare-instead-of-local-and-export)​.|
|**Inheritance**|Variables are inherited by child processes [](https://www.digitalocean.com/community/tutorials/export-command-linux)​.|Variables are **local** to the current shell unless combined with `-x` [](https://stackoverflow.com/questions/56627534/in-bash-should-i-use-declare-instead-of-local-and-export)​.|
|**Portability**|Standard POSIX command; works in almost all shells (`sh`, `bash`, `zsh`) [](https://stackoverflow.com/questions/56627534/in-bash-should-i-use-declare-instead-of-local-and-export)​.|Bash-specific builtin; may not work in basic `sh` or other shells ​.|
|**Data Types**|Treats everything as a string.|Can enforce types like **integers** (`-i`) or **arrays** (`-a`) [](https://stackoverflow.com/questions/56627534/in-bash-should-i-use-declare-instead-of-local-and-export)​.|

## When to Use Which?

- **Use `export`** when you simply need to make a variable (like `PATH` or `JAVA_HOME`) available to other programs or scripts you run from your terminal. It is the standard way to set environment variables.[](https://www.digitalocean.com/community/tutorials/export-command-linux)​
    
    bash
    
    `export MY_VAR="available to child processes"`
    
- **Use `declare`** when writing complex Bash scripts where you need strict control over variable behavior.[](https://stackoverflow.com/questions/56627534/in-bash-should-i-use-declare-instead-of-local-and-export)​
    
    - **Read-only variables:** `declare -r constant="cannot change"`
        
    - **Integers:** `declare -i count=10` (performs math automatically, e.g., `count=10+5` becomes 15)
        
    - **Arrays:** `declare -a my_list`
        

## Can they work together?

Yes. `declare -x variable=value` is functionally equivalent to `export variable=value` in Bash. However, `export` is preferred for this specific task because it is more readable and universally understood as "making this variable global".[](https://stackoverflow.com/questions/56627534/in-bash-should-i-use-declare-instead-of-local-and-export)​