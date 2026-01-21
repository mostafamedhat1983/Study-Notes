---
tags:
  - Bash_Script
  - Linux
---
Bash functions are reusable blocks of code that help organize scripts, avoid repetition, and improve readability.​

## Function Syntax

There are two main ways to define bash functions:​

**Format 1: Using parentheses (most common)**

``` bash
function_name() {
    commands
}
```

**Format 2: Using the `function` keyword**

``` bash
function function_name {
    commands
}
```

Both formats work identically. The first format is more portable and widely preferred.​

## Basic Example

``` bash
#!/bin/bash

# Define the function
greet() {
    echo "Hello, $1!"
}

# Call the function
greet "DevOps"
```

Output: `Hello, DevOps!`​

## Key Concepts

**Calling functions**: Simply use the function name. Functions must be defined before they're called in the script.​

**Arguments**: Functions accept positional parameters just like scripts:​

- `$1`, `$2`, `$3` etc. - individual arguments passed to the function
    
- `$@` - all arguments
    
- `$#` - count of arguments
    

**Return values**: Functions return exit status codes (0-255) using `return`. To return actual values, use command substitution:[](https://www.shell-tips.com/bash/functions/)​

``` bash
get_sum() {
    echo $(($1 + $2))
}

result=$(get_sum 5 10)
echo $result  # Outputs: 15
```

**Variable scope**: Variables are global by default. Use the `local` keyword to create function-scoped variables.[](https://phoenixnap.com/kb/bash-function)​

## One-line Syntax

For simple functions, use semicolons to separate commands:​

``` bash
my_function() { echo "Hello"; echo "Bye!"; }
```