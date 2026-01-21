---
tags:
  - Linux
  - Bash_Script
---
The `break` and `continue` statements control loop execution in bash, allowing you to exit loops early or skip specific iterations.​

## break Statement

The `break` statement terminates the loop immediately and exits to the next command after the loop.​

**Syntax**: `break [n]`

The optional `[n]` argument specifies which enclosing loop to break from (useful in nested loops). By default, `n=1` breaks the innermost loop.​

**Example**:

``` bash
i=0
while [[ $i -lt 5 ]]; do
    echo "Number: $i"
    ((i++))
    if [[ $i -eq 2 ]]; then
        break
    fi
done
echo 'All Done!'

```

Output:​

``` text
Number: 0
Number: 1
All Done!
```

## continue Statement

The `continue` statement skips the remaining commands in the current iteration and jumps to the next iteration of the loop.​

**Syntax**: `continue [n]`

The optional `[n]` specifies which enclosing loop to continue.[](https://www.geeksforgeeks.org/linux-unix/break-and-continue-keywords-in-linux-with-examples/)​

**Example**:

``` bash
i=0
while [[ $i -lt 5 ]]; do
    ((i++))
    if [[ "$i" == '2' ]]; then
        continue
    fi
    echo "Number: $i"
done
echo 'All Done!'

```

Output:​

``` text
Number: 1
Number: 3
Number: 4
Number: 5
All Done!
```

## Practical Use Case

Print only numbers divisible by 9:[](https://linuxize.com/post/bash-break-continue/)​

``` bash
for i in {1..50}; do
    if [[ $(( $i % 9 )) -ne 0 ]]; then
        continue
    fi
    echo "Divisible by 9: $i"
done
```

## Key Differences

- **break**: Exits the entire loop and proceeds to commands after the loop​
    
- **continue**: Skips only the current iteration and proceeds to the next one​
    

Both statements work with `for`, `while`, and `until` loops