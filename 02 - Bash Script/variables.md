---
tags:
  - Linux
  - Bash_Script
---
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

___

### Declare command
The `declare` command in Bash is a built-in shell command used to declare variables and set their attributes. It provides fine-grained control over variable behavior, making it particularly useful for scripts requiring typed variables, arrays, or read-only values.​

## Syntax

The basic syntax is:[](https://phoenixnap.com/kb/bash-declare)​

bash

`declare [options] [variable-name]="[value]"`

## Common Options

The most frequently used options include:​

- **-a**: Declare an indexed array
    
- **-A**: Declare an associative array
    
- **-i**: Declare an integer variable
    
- **-r**: Declare a read-only variable (cannot be changed)
    
- **-x**: Export the variable (make it available to child processes)
    
- **-f**: Declare or display functions
    
- **-p**: Display variable value and attributes
    
- **-g**: Declare a global variable (useful inside functions)
    

## Examples

**Declaring an integer variable**:[](https://www.tutorialspoint.com/bash-declare-statement-syntax-and-examples)​

bash

`declare -i my_num=5`

**Declaring an indexed array**:[](https://www.tutorialspoint.com/bash-declare-statement-syntax-and-examples)​

bash

`declare -a my_array=("apple" "banana" "cherry")`

**Declaring a read-only variable**:[](https://www.tutorialspoint.com/bash-declare-statement-syntax-and-examples)​

bash

`declare -r my_name="John"`

**Declaring multiple variables at once**:[](https://www.tutorialspoint.com/bash-declare-statement-syntax-and-examples)​

bash

`declare var1=10 var2="text" var3=("one" "two" "three")`

**Exporting a variable**:[](https://www.tutorialspoint.com/unix_commands/declare.htm)​

bash

`declare -x PATH_VAR="/usr/local/bin"`

---
### Typeset command

The `typeset` command in Bash is a built-in command that declares and modifies variables with specific attributes. It is functionally identical to the `declare` command—they are exact synonyms in Bash.​

## Syntax

bash

`typeset [options] name[=value]`

## Relationship to declare

`typeset` was provided primarily for compatibility with the Korn shell (ksh). However, **`typeset` has been deprecated since Bash version 4.0**, and the `declare` command should be used instead for better forward compatibility. Modern Bash versions may display deprecation warnings when using `typeset`.​

## Common Options

The main options supported in Bash include:[](https://www.tutorialspoint.com/unix_commands/typeset.htm)​

- **-i**: Declare an integer variable
    
- **-r**: Declare a read-only variable
    
- **-x**: Export variable for access in sub-processes
    
- **-u**: Convert lowercase characters to uppercase
    
- **-l**: Convert uppercase characters to lowercase
    
- **-f**: Display or work with functions
    

## Examples

**Declaring an integer**:[](https://www.tutorialspoint.com/unix_commands/typeset.htm)​

bash

`typeset -i num=100 num+=1`

**Declaring a read-only variable**:[](https://www.tutorialspoint.com/unix_commands/typeset.htm)​

bash

`typeset -r name="tutorialspoint"`

**Creating a local variable in a function**:[](https://techalmirah.com/typeset-command-in-bash/)​

bash

`typeset x=5  # Local to the function scope`

**Converting to uppercase**:[](https://www.tutorialspoint.com/unix_commands/typeset.htm)​

bash

`typeset -u name="hello"  # Automatically converts to HELLO`

Since `typeset` is deprecated, use `declare` instead for all new Bash scripts to ensure compatibility with current and future Bash versions.[](https://www.baeldung.com/linux/declare-vs-typeset)

---

Here's a comparison of variable scope behavior when using `typeset`, `declare`, and `export` in Bash:

## Scope Comparison Table

|Command|Script-Level Scope|Function-Level Scope|Exported to Child Processes|Notes|
|---|---|---|---|---|
|`typeset VAR=value`|Global [](https://stackoverflow.com/questions/4419704/differences-between-declare-typeset-and-local-variable-in-bash)​|Local [](https://stackoverflow.com/questions/4419704/differences-between-declare-typeset-and-local-variable-in-bash)​|No|Deprecated; identical to `declare` [](https://stackoverflow.com/questions/4419704/differences-between-declare-typeset-and-local-variable-in-bash)​|
|`declare VAR=value`|Global [](https://stackoverflow.com/questions/4419704/differences-between-declare-typeset-and-local-variable-in-bash)​|Local [](https://stackoverflow.com/questions/4419704/differences-between-declare-typeset-and-local-variable-in-bash)​|No|Preferred modern approach [](https://www.baeldung.com/linux/declare-vs-typeset)​|
|`declare -x VAR=value`|Global [](https://www.baeldung.com/linux/declare-vs-typeset)​|Local [](https://delightlylinux.wordpress.com/2024/03/23/bash-typed-variables-with-declare/)​|Yes ​|Same as `export`|
|`export VAR=value`|Global [](https://delightlylinux.wordpress.com/2024/03/23/bash-typed-variables-with-declare/)​|Global [](https://delightlylinux.wordpress.com/2024/03/23/bash-typed-variables-with-declare/)​|Yes [](https://delightlylinux.wordpress.com/2024/03/23/bash-typed-variables-with-declare/)​|Always creates global variables|
|`VAR=value` (simple assignment)|Global|Global [](https://stackoverflow.com/questions/4419704/differences-between-declare-typeset-and-local-variable-in-bash)​|No|No export unless preceded by `export`|

## Key Differences

## At Script Level (Outside Functions)

All three commands create global variables accessible throughout the script. However, only `declare -x` and `export` make variables available to child processes (subshells or external scripts).​

## Inside Functions

- **`declare`/`typeset`**: Create local variables by default​
    
- **`declare -g`**: Explicitly creates global variables from within functions​
    
- **`export`**: Always creates global variables regardless of where used[](https://delightlylinux.wordpress.com/2024/03/23/bash-typed-variables-with-declare/)​
    

## Child Process Visibility

Variables must be exported using `export` or `declare -x` to be accessible in subshells or external scripts. Without export, variables remain in the current shell's environment only.​

## Example

bash

`#!/bin/bash # Script level typeset var1="local to script"     # Global, not exported declare var2="also global"         # Global, not exported declare -x var3="exported"         # Global AND exported export var4="also exported"        # Global AND exported # Inside function function my_func {     declare local_var="stays here"      # Local to function    declare -g global_from_func="global"  # Global from function    export always_global="exported"     # Global AND exported }`

Use `declare` over `typeset` for better compatibility with modern Bash versions.[](https://www.baeldung.com/linux/declare-vs-typeset)​​