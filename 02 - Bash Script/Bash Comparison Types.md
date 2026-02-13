---
tags:
  - Bash_Script
  - Linux
---
Bash provides different comparison operators for integers, strings, and files, each with specific syntax requirements.​

## Integer Comparison Operators

Integer comparisons use hyphenated operators within test brackets `[ ]` or `[[ ]]`:​

- `-eq`: Equal to​
    
- `-ne`: Not equal to​
    
- `-lt`: Less than​
    
- `-le`: Less than or equal to​
    
- `-gt`: Greater than​
    
- `-ge`: Greater than or equal to​
    

**Example**:

``` bash
num1=10
num2=5
if [ "$num1" -gt "$num2" ]; then
    echo "num1 is greater than num2"
fi
```
You can also use C-style operators (`<`, `>`, `<=`, `>=`) inside double parentheses `(( ))` for arithmetic comparisons:​

``` bash
if ((num1 > num2)); then
    echo "num1 is greater"
fi
```
## String Comparison Operators

String comparisons use different operators:​

- `=` or `==`: Strings are equal​
    
- `!=`: Strings are not equal​
    
- `<`: Less than in ASCII alphabetical order (requires `[[ ]]`)​
    
- `>`: Greater than in ASCII alphabetical order (requires `[[ ]]`)​
    
- `-z`: String is empty (null)​
    
- `-n`: String is not empty​
    

**Example**:

``` bash
str1="hello"
str2="world"
if [ "$str1" == "$str2" ]; then
    echo "Strings are equal"
fi

if [ -z "$str1" ]; then
    echo "String is empty"
fi
```

## File Test Operators

File test operators check file properties and existence:​

- `-e`: File exists[](https://tldp.org/LDP/abs/html/comparison-ops.html)​
    
- `-f`: Regular file exists[](https://tldp.org/LDP/abs/html/comparison-ops.html)​
    
- `-d`: Directory exists[](https://tldp.org/LDP/abs/html/comparison-ops.html)​
    
- `-r`: File is readable[](https://tldp.org/LDP/abs/html/comparison-ops.html)​
    
- `-w`: File is writable[](https://tldp.org/LDP/abs/html/comparison-ops.html)​
    
- `-x`: File is executable[](https://tldp.org/LDP/abs/html/comparison-ops.html)​
    
- `-s`: File exists and is not empty[](https://tldp.org/LDP/abs/html/comparison-ops.html)​
    

**Example**:

``` bash
if [ -f "myfile.txt" ]; then
    echo "File exists"
fi
```
## Important Notes

Always quote variables in comparisons (`"$var"`) to avoid errors when variables are empty or contain spaces. Use integer operators for numeric comparisons and string operators for text comparisons—mixing them can produce unexpected results.

---

**`[[ ]]` vs `(( ))` - Complete Comparison**

## Overview

- `[[ ]]` = **String/pattern testing** (Bash keyword)
    
- `(( ))` = **Arithmetic evaluation** (compound command)
    

## Key Differences

|Feature|`[[ ]]`|`(( ))`|
|---|---|---|
|**Purpose**|String comparison, file tests, regex|**Math operations, numeric tests**|
|**Variables**|`$var` (strings)|`var` (no `$`, numeric)|
|**Equality**|`==` (string), `=~` (regex)|`==` (**numeric**)|
|**Operators**|`-lt`, `-gt`, `&&`, `\|`|`<`, `>`, `<=`, `|
|**Empty vars**|Handles safely|Assumes 0|
|**Context**|`if [[ $x == "yes" ]]`|`if (( x > 5 ))`|

