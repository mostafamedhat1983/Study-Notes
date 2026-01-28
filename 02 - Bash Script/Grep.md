---
tags:
  - Linux
  - Bash_Script
---
**`grep` searches files or input for lines matching a pattern (string or regex).**  
Basic syntax: `grep [options] "pattern" [file]`. Without a file, it reads from stdin (pipes).[](https://www.geeksforgeeks.org/linux-unix/grep-command-in-unixlinux/)

## Common Options

- `-i`: Case-insensitive search ( ignore upper and lowercase)
    
- `-v`: Show non-matching lines  ex : grep -v "#"    to ignore lines with comments
    
- `-r`: Recursive directory search , search inside all files in directory
    
- `-n`: Show line numbers
    
- `-l`: List matching files only
    
- `-c`: Count matches
    
- `-E`: Extended regex (no escaping needed for `|`, `+`, etc.) [](https://phoenixnap.com/kb/grep-command-linux-unix-examples)  
## Grep Context Options

|Option|Description|Example|
|---|---|---|
|`-o`|Print **only** matching part|`grep -o "id=\d+" log` → `id=123`|
|`-A N`|**N lines After** match|`grep -A 2 "ERROR" log`|
|`-B N`|**N lines Before** match|`grep -B 3 "CRITICAL" log`|
|`-C N`|**N lines Before+After**|`grep -C 1 "failed" log`|

## Examples

``` bash
# Search file for "error"
grep "error" /var/log/app.log

# Case-insensitive, show line numbers
grep -in "error" app.log

# Pipe example (ls output → filter .txt files)
ls *.txt | grep "report"

# Recursive search in directory
grep -r "TODO" src/

# Count occurrences
grep -c "failed" *.log
```

Outputs line count per file.[](https://phoenixnap.com/kb/grep-command-linux-unix-examples)

## DevOps Patterns

``` bash
# Find failed pods
kubectl get pods | grep -E "0/1|CrashLoopBackOff"

# Parse AWS logs
grep -E "\[ERROR\]" cloudwatch.log | grep "AccessDenied"
```

Combine with pipes/regex for log analysis, CI/CD filtering.