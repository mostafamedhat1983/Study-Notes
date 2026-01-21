---
tags:
  - Linux
  - Bash_Script
---
In bash, `$1` is a **positional parameter** that represents the **first argument** passed to a script or function.​

## How It Works

When you execute a bash script with arguments, they are automatically assigned to numbered variables. For example, running `./script.sh Hello World` assigns the values as follows:[[stackoverflow](https://stackoverflow.com/questions/29258603/what-do-0-1-2-mean-in-a-shell-script)]​

- `$0` = ./script.sh (the script name itself)
    
- `$1` = Hello (first argument)
    
- `$2` = World (second argument)
    

You can access additional arguments using `$3`, `$4`, `$5`, and so on. For arguments beyond single digits (like `$10`), you must use braces: `${10}`.[[stackoverflow](https://stackoverflow.com/questions/29258603/what-do-0-1-2-mean-in-a-shell-script)]​

## Practical Example

``` bash
#!/bin/bash
echo "Hello, $1!"
```

Running `./greet.sh Zaira` would output: `Hello, Zaira!​

## Context Matters

The meaning of `$1` depends on scope​

- **In a script**: `$1` refers to the first argument passed when invoking the script (global scope)
    
- **Inside a function**: `$1` refers to the first argument passed to that specific function (local scope)
    

## Related Variables

- `$@` - all arguments as a list​
    
- `$#` - count of total arguments passed​
    
- `$*` - all arguments as a single string