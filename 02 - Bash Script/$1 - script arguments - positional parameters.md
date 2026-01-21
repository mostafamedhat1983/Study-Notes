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
    

## Core Positional Parameters

|Parameter|Description|
|---|---|
|`$0`|Script name or command being executed [](https://bash-hackers.gabe565.com/scripting/posparams/)​|
|`$1` through `$9`|First through ninth arguments [](https://bash-hackers.gabe565.com/scripting/posparams/)​|
|`${10}` and beyond|Arguments beyond position 9 (require braces) ​|
|`$#`|Total count of arguments [](https://www.spsanderson.com/steveondata/posts/2025-04-18/)​|
|`$@`|All arguments as separate quoted strings [](https://www.spsanderson.com/steveondata/posts/2025-04-18/)​|
|`$*`|All arguments as a single string [](https://www.spsanderson.com/steveondata/posts/2025-04-18/)​|

## Basic Example

``` bash
#!/bin/bash
echo "Script name: $0"
echo "First argument: $1"
echo "Second argument: $2"
echo "Total arguments: $#"

```

Running `./script.sh DevOps Engineer` would output:[](https://linuxcommand.org/lc3_wss0120.php)​

- Script name: ./script.sh
    
- First argument: DevOps
    
- Second argument: Engineer
    
- Total arguments: 2
    

## The shift Command

The `shift` command removes `$1` and moves all parameters down by one position: `$2` becomes `$1`, `$3` becomes `$2`, and so on. This is useful for processing arguments one at a time:​

``` bash
while [ "$1" != "" ]; do
    echo "Parameter: $1"
    shift
done
```

## Iterating Through All Arguments

The most flexible approach uses `$@` in a for loop:[](https://bash-hackers.gabe565.com/scripting/posparams/)​

``` bash
for arg in "$@"; do
    echo "$arg"
done
```

This preserves spaces in arguments and handles any number of parameters​

## Setting Parameters Manually

You can set positional parameters inside a script using the `set` command:​

``` bash
set "This is" my new "set of" positional
# $1 = "This is"
# $2 = "my"
# $3 = "new"
# $4 = "set of"
# $5 = "positional"
```