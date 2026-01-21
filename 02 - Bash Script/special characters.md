---
tags:
  - Linux
  - Bash_Script
---
## Single vs Double Quotes in Bash

**Single quotes** `' '` preserve the **literal value** of every character inside them: no variable expansion, command substitution, or escape sequences are interpreted.  
**Double quotes** `" "` allow **variable expansion** (`$var`), **command substitution** (`$(cmd)` or `` `cmd` ``), and certain escape sequences, but they still protect spaces and most other characters.

---

## Why Both Exist

Bash has two quoting mechanisms to cover two main use cases:

1. **Literal strings (single quotes)**  
    When you want exactly what you type, with no interpretation at all.
    
2. **Dynamic strings (double quotes)**  
    When you need variable values or command output inside the string.
    

Use single quotes for **data** (SQL snippets, regex patterns, literal text), and double quotes for **interpolation** (log messages, prompts, paths with variables).

---

## Core Behavior Comparison

|Feature|Single Quotes `' '`|Double Quotes `" "`|
|---|---|---|
|Variable expansion|Disabled: `$var` stays `$var`|Enabled: `$var` becomes its value|
|Command substitution|Disabled: `$(cmd)` stays literal|Enabled: `$(cmd)` becomes command output|
|Escape sequences (`\n`)|Literal `\` and `n`|`\` can escape some chars (depends on context)|
|Backslash escaping|No effect (except cannot escape `'`)|Works: `\"`, `\\`, `\$`|
|Whitespace preservation|Preserved|Preserved|
|Glob expansion (`*`, `?`)|Disabled|Disabled|
|Word splitting|Prevented|Prevented|
|Nest same quote type|Impossible|Possible via escaping: `\"`|

---

## Practical Examples

## Variable Expansion

``` bash
name="Alice"

echo 'Hello $name'    # Output: Hello $name
echo "Hello $name"    # Output: Hello Alice
```

## Command Substitution

``` bash
echo 'Current directory: $(pwd)'   # Output: Current directory: $(pwd)
echo "Current directory: $(pwd)"   # Output: Current directory: /home/user
```

## Escape Sequences

``` bash
echo 'Line 1\nLine 2'
# Output: Line 1\nLine 2 (literal backslash+n)

echo "Line 1\nLine 2"
# Output: Line 1\nLine 2 (literal in plain echo)

echo -e "Line 1\nLine 2"
# Output:
# Line 1
# Line 2
```
## Special Characters and `$`

``` bash
echo 'Price: $100'
# Output: Price: $100  (literal)

echo "Price: $100"
# Output usually: Price: 00  (tries to expand $1, then leaves 00)

echo "Price: \$100"
# Output: Price: $100  (escaped)
```
## Backslash Handling (Windows-like Paths)

``` bash
path='C:\Windows\System32'
echo "$path"
# Output: C:\Windows\System32 (literal inside single quotes at assignment)

path="C:\Windows\System32"
echo "$path"
# Often Output: C:WindowsSystem32 (backslashes may be interpreted or removed)

path="C:\\Windows\\System32"
echo "$path"
# Output: C:\Windows\System32
```

---

## When to Use Each

## Use Single Quotes `' '` When

1. **Literal strings with special characters**


``` bash
regex='[a-zA-Z0-9]+@[a-z]+\.[a-z]{2,}'
sql="SELECT * FROM users WHERE name='$user'"  # Outer double, inner single
```

2. **Prevent variable expansion**


``` bash
echo 'Variables like $HOME and $USER will not expand'
```

3. **Simple multi-line or block literal text**


``` bash
message='Line 1
Line 2
Line 3'
```
3. **File paths with many backslashes**


``` bash
path='C:\Users\Admin\file.txt'
```

## Use Double Quotes `" "` When

1. **You need variable interpolation**


``` bash
echo "Hello, $USER! Your home is $HOME"
log_msg="Error occurred at $(date)"
```

2. **Preserving whitespace in variables**


``` bash
filename="my file.txt"

cat "$filename"   # Correct: treated as single argument
cat $filename     # Wrong: splits into "my" and "file.txt"
```

3. **Command substitution**


``` bash
echo "Files in directory: $(ls | wc -l)"
echo "Current user: $(whoami)"
```

4. **Escaping specific characters while still expanding variables**

``` bash
echo "He said, \"Hello!\""
echo "Price: \$50"
```

---

## Advanced Patterns

## Nesting Quotes

``` bash
# Single inside double
echo "He said, 'Hello World!'"

# Double inside single
echo 'He said, "Hello World!"'

# Concatenate different quoted parts
echo 'Single part '"Double part"' single again'
```

## Single Quotes Inside Single Quotes

You cannot write a literal `'` directly inside `'...'`.
``` bash
# WRONG
echo 'It\'s wrong'   # Syntax error
```

Correct approaches:

``` bash
# Method 1: Close, escape, reopen
echo 'It'\''s correct'

# Method 2: Use double quotes
echo "It's correct"

# Method 3: Mix types
echo 'It'"'"'s correct'
```

---

## Best Practices

1. **Default to quoting variables**:  
    Use `"${var}"`, `"${1}"`, `"${@}"`, `"${array[@]}"` to avoid word splitting and globbing.
    
2. **Use single quotes** for strings that must be completely literal.
    
3. **Use double quotes** when you need expansions but want to preserve spaces and prevent globbing.
    
4. **Use `$'...'` (ANSI C quoting)** for explicit escape sequences:
    

``` bash
echo $'Line 1\nLine 2'
echo $'Tab:\tSeparated'
```

5. Avoid unquoted variables unless you really need splitting/globbing and fully understand the consequences.
    

---

## Quick Decision Guide

- Need variables / `$(cmd)` inside? → **Use `"..."`**
    
- No expansions needed, want purely literal text? → **Use `'...'`**
    
- Need a single quote inside a mostly literal string?
    
    - Use one of:
        
        - `'It'\''s ok'`
            
        - `"It's ok"`
            
        - `$'It\'s ok'`