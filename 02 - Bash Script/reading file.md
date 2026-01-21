---
tags:
  - Bash_Script
  - Linux
---
## The Canonical `read` Pattern for Files

**The gold standard for line-by-line file reading:**

``` bash
while IFS= read -r line; do
    # Process "$line"
done < file.txt
```

This is the **safest, most memory-efficient** method for files of any size.​

---

## Why This Pattern Wins

|Pattern|Memory|Preserves Whitespace|Large Files|Variable Scope|
|---|---|---|---|---|
|`while IFS= read -r < file`|Stream|✓|✓|Main shell|
|`cat file \| while read`|Stream|✓|✓|**Lost** (subshell) [](https://www.warp.dev/terminus/bash-if-statement)​|
|`for line in $(cat file)`|**Full load**|✗|✗|Main shell|
|`mapfile -t array < file`|**Full load**|✓|✗|Main shell|

**Why `while IFS= read -r` is perfect:**

- **Streams** 1 line at a time (no memory limit)
    
- **IFS=** preserves leading/trailing whitespace
    
- **`-r`** prevents backslash interpretation
    
- **`< file`** avoids subshell (variables persist)​
    

---

## Essential Patterns

## 1. Basic Line Reading (Most Common)

``` bash
while IFS= read -r line; do
    echo "$line"
done < file.txt
```
## 2. Free stdin for Interactive Prompts

``` bash
while IFS= read -r -u9 line; do
    read -rp "Process '$line'? (y/n): " answer
    [[ $answer == y ]] && process "$line"
done 9< file.txt
```

## 3. Skip Empty Lines or Comments

``` bash
while IFS= read -r line; do
    [[ -z $line || $line =~ ^[[:space:]]*# ]] && continue
    echo "$line"
done < config.txt
```
## 4. CSV Parsing

``` bash
while IFS=',' read -r col1 col2 col3; do
    echo "Name: $col1, Age: $col2, City: $col3"
done < data.csv
```

## 5. `/etc/passwd` Style (Colon-Delimited)

``` bash
while IFS=':' read -r user pass uid gid gecos home shell; do
    echo "User: $user (UID: $uid)"
done < /etc/passwd
```

## 6. Process Substitution (Command Output)

``` bash
while IFS= read -r line; do
    echo "$line"
done < <(grep "ERROR" /var/log/app.log)
```
## 7. Read Key-Value Config

``` bash
declare -A config
while IFS='=' read -r key value; do
    config[$key]=$value
done < config.ini
echo "Host: ${config[host]}"
```

## 8. Handle Last Line Without Newline

``` bash
while IFS= read -r line || [[ -n $line ]]; do
    echo "$line"
done < file.txt
```

---

## IFS Patterns by File Format

|Format|IFS Setting|Example|
|---|---|---|
|CSV|`IFS=','`|`while IFS=',' read -r f1 f2`|
|TSV|`IFS=$'\t'`|`while IFS=$'\t' read -r f1 f2`|
|Colon|`IFS=':'`|`/etc/passwd` parsing|
|Whitespace|`IFS=`|Preserve all whitespace|
|Preserve Everything|`IFS=`|Line-by-line with spaces|

---

## Anti-Patterns (Avoid These!)

## ❌ **Pipe Creates Subshell (Variables Lost)**

``` bash
count=0
cat file.txt | while IFS= read -r line; do
    count=$((count + 1))
done
echo "$count"  # Prints 0! Variable lost [web:6]
```

**✓ Fix:** Use input redirection

``` bash
count=0
while IFS= read -r line; do
    count=$((count + 1))
done < file.txt
echo "$count"  # Works!
```

## ❌ **For Loop with `cat` (Memory + Word Splitting)**

``` bash
for line in $(cat file.txt); do  # Breaks on spaces!
    echo "$line"
done
```
## ❌ **No `-r` Flag (Backslash Interpretation)**

``` bash
while read line; do  # \n → actual newline!
    echo "$line"
done < file.txt
```

## ❌ **No `IFS=` (Whitespace Stripped)**

``` bash
while read -r line; do  # Trims leading/trailing spaces
    echo "$line"
done < file.txt
```

---

## Quick Reference

``` bash
while IFS= read -r line; do
    # Always quote "$line"
    # Variables persist
    # Streams line-by-line
done < file.txt

```

**Always quote `"$line"`**, always use `IFS= read -r`, always redirect `< file`. This pattern handles 99% of file reading needs safely and efficiently.