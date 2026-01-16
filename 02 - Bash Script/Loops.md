---
tags:
  - Linux
  - Bash_Script
---
Bash provides three types of loops for automating repetitive tasks: `for`, `while`, and `until` loops. Each loop type serves different use cases depending on how you want to control iteration.[](https://itsfoss.com/bash-loops/)​

## For Loop

The `for` loop iterates over a list of items or a range of values.​

**Basic syntax:**


``` bash
for variable in list; do
    commands
done
```



**Examples:**

Iterate over a range:[](https://www.w3schools.com/bash/bash_loops.php)​
``` bash
for i in {1..5}; do
    echo "Iteration $i"
done
```


Iterate over a list of strings:[](https://phoenixnap.com/kb/bash-for-loop)​


``` bash
for fruit in apple banana cherry; do
    echo "Fruit: $fruit"
done
```

C-style for loop:[](https://earthly.dev/blog/loops-in-bash/)​


``` bash
for (( i=1; i<=10; i++ )); do
    echo $i
done
```


## While Loop

The `while` loop executes commands as long as the condition is **true**.​

**Syntax:**


``` bash
while [ condition ]; do
    commands
done
```


**Example**:[](https://earthly.dev/blog/loops-in-bash/)​

bash
``` bash
i=1
while [ $i -le 10 ]; do
    echo $i
    ((i++))
done
```



## Until Loop

The `until` loop executes commands as long as the condition is **false** and stops when it becomes true. It's the inverse of a `while` loop.​

**Syntax:**

bash

``` bash
until [ condition ]; do
    commands
done
```

**Example**:[](https://www.w3schools.com/bash/bash_loops.php)​

```bash 
count=1
until [ $count -gt 5 ]; do
    echo "Count is $count"
    ((count++))
done
```



## Loop Control Statements

You can control loop execution with:[](https://earthly.dev/blog/loops-in-bash/)​

- **`break`**: Exit the loop immediately
    
- **`continue`**: Skip the current iteration and move to the next one
    

**Example with break**:[](https://earthly.dev/blog/loops-in-bash/)​

``` bash
for (( i=1; i<=10; i++ )); do
    if [ $i -eq 6 ]; then
        break
    fi
    echo $i
done
```