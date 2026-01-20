---
tags:
  - Linux
  - Bash_Script
---
## If Statements in Bash

Bash if statements allow you to execute code conditionally based on whether an expression evaluates to true or false. The basic syntax uses the `if` keyword, a condition in square brackets, the `then` keyword, and closes with `fi` (if spelled backwards).​

## Basic Syntax

``` bash
if [ condition ]; then
    # code to execute if condition is true
fi

```

## If-Else Statements

You can add an `else` clause to execute code when the condition is false:[](https://ioflood.com/blog/bash-if-statement/)​

``` bash
num=15
if [ "$num" -lt 10 ]; then
    echo "Number is less than 10"
else
    echo "Number is greater than or equal to 10"
fi

```

## If-Elif-Else Statements

Use `elif` to check multiple conditions sequentially:[](https://www.geeksforgeeks.org/linux-unix/bash-scripting-if-statement/)​

``` bash
num=15
if [ "$num" -lt 10 ]; then
    echo "Number is less than 10"
elif [ "$num" -lt 20 ]; then
    echo "Number is less than 20"
else
    echo "Number is greater than or equal to 20"
fi

```

## Common Test Conditions

- **Numeric comparisons**: `-eq` (equal), `-ne` (not equal), `-lt` (less than), `-gt` (greater than), `-le` (less than or equal), `-ge` (greater than or equal)​
    
- **String comparisons**: `==` (equal), `!=` (not equal), `-z` (is null), `-n` (is not null)[](https://www.warp.dev/terminus/bash-if-statement)​
    
- **File tests**: `-f` (file exists), `-r` (file is readable), `-d` (directory exists)


## Single vs Double Square Brackets

The double square brackets `[[ ]]` are a Bash-specific extended command that provides more functionality and safety compared to the POSIX-compliant single brackets `[ ]`.​

## Key Differences

**Variable quoting and word splitting**: With `[[ ]]`, you don't need to quote variables to prevent word splitting, while `[ ]` requires quotes. For example, `x='a b'; [[ $x = 'a b' ]]` works correctly, but `[ $x = 'a b' ]` causes a syntax error without quotes.[](https://stackoverflow.com/questions/669452/are-double-square-brackets-preferable-over-single-square-brackets-in-b)​

**Logical operators**: Double brackets support `&&` and `||` for logical AND/OR operations directly within the test, while single brackets treat these as command separators [](https://stackoverflow.com/questions/669452/are-double-square-brackets-preferable-over-single-square-brackets-in-b)​. You can write `[[ $a = 1 && $b = 2 ]]`, but with single brackets you need `[ $a = 1 ] && [ $b = 2 ]` [](https://stackoverflow.com/questions/669452/are-double-square-brackets-preferable-over-single-square-brackets-in-b)​.

**Pattern matching**: Double brackets support glob pattern matching with `==`, allowing tests like `[[ $VAR == a* ]]` to check if a variable starts with 'a'. Single brackets don't support this feature.​

**Regular expressions**: Only `[[ ]]` supports regex matching with `=~`, such as `[[ ab =~ ab? ]]` for POSIX extended regular expressions. Single brackets have no equivalent.​

**Grouping with parentheses**: Double brackets allow grouping conditions with `()` for precedence control, like `[[ (a = a || a = b) && a = b ]]` [](https://stackoverflow.com/questions/669452/are-double-square-brackets-preferable-over-single-square-brackets-in-b)​. Single brackets require external command grouping with braces [](https://stackoverflow.com/questions/669452/are-double-square-brackets-preferable-over-single-square-brackets-in-b)​.

## Performance and Portability

Double brackets are approximately 1.42x faster than single brackets in performance tests. However, `[[ ]]` is Bash-specific and not POSIX-compliant, while `[ ]` works in all POSIX shells, making it more portable across different Unix systems.​



## When to Use Double Brackets

Double brackets `[[ ]]` are generally preferable in Bash scripts for several reasons, but the decision depends on your specific requirements.[](https://stackoverflow.com/questions/669452/are-double-square-brackets-preferable-over-single-square-brackets-in-b)​

## Advantages of Using `[[ ]]`

**Safety and robustness**: Double brackets prevent word splitting and pathname expansion, eliminating the need to quote variables in most cases. This reduces syntax errors—for example, `x='a b'; [[ $x = 'a b' ]]` works without quotes, while `[ $x = 'a b' ]` causes an error.​

**Better performance**: Double brackets are approximately 1.42x faster than single brackets in tests involving simple string comparisons. The performance difference is most pronounced with shorter string comparisons but remains measurable even with longer strings.[](https://stackoverflow.com/questions/669452/are-double-square-brackets-preferable-over-single-square-brackets-in-b)​

**Enhanced functionality**: `[[ ]]` supports regex matching with `=~`, glob pattern matching with `==`, and allows logical operators `&&` and `||` directly within the test ​. You can also use parentheses for grouping conditions, which isn't possible with single brackets [](https://stackoverflow.com/questions/669452/are-double-square-brackets-preferable-over-single-square-brackets-in-b)​.

**Improved readability**: The modern C-like syntax of double brackets makes code more readable and easier to maintain, especially for complex conditional logic.​

## When to Avoid `[[ ]]`

**Portability concerns**: If your script needs to run on non-Bash shells or strict POSIX environments like `dash`, you must use single brackets `[ ]` instead. Double brackets are Bash-specific and not POSIX-compliant.​

**Legacy system compatibility**: Older or limited versions of Bash may not support double brackets. For maximum compatibility across diverse Unix systems, single brackets remain the safer choice.​

## Recommendation

For pure Bash scripting on modern systems, using `[[ ]]` consistently is recommended due to its safety, performance, and feature advantages. However, if portability to other shells is a requirement, stick with single brackets `[ ]`.​