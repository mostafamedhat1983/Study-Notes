---
tags:
  - Linux
---
**grep** searches files or command output for text patterns and prints matching lines

## Two Main Modes

## 1. File Search

grep "error" /var/log/syslog          # Inside files 
grep -r "TODO" /src/project/          # Recursive directories`

## 2. Pipe/Filter Output

ps aux | grep docker                  # Filter processes 
ip route | grep docker0               # Filter routes 
ss -tulnp | grep :53                  # Filter ports 
docker logs nginx | grep "404"        # Filter container logs

## Must-Know Options

| Option | Purpose                    | Example                   |
| ------ | -------------------------- | ------------------------- |
| **-i** | Case insensitive           | `grep -i error`           |
| **-v** | Invert (show non-matches)  | `grep -v OK`              |
| **-r** | Recursive (subdirectories) | `grep -r "TODO" src/`     |
| **-n** | Show line numbers          | `grep -n "fail" log.txt`  |
| **-l** | Filenames only             | `grep -l "error" *.log`   |
| **-c** | Count matches              | `grep -c "error" log.txt` |
| **-w** | Whole words only           | `grep -w "docker" output` |

---

**find** locates files and directories by name, type, size, time, permissions—recursively searches directory trees. 

## Core Syntax

```bash
find [start_path] [options] [expression] 
find ~ -name "*.yaml"                   # Your home dir, YAML files 
find /etc -type f -name "*.conf"        # Config files only
```

## Essential Options (Your Notes)

| Option              | Purpose                                                                                        | Example                                                       |
| ------------------- | ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| **-name "pattern"** | Filename match (case-sensitive)                                                                | `find . -name "docker-compose.yml"`                           |
| **-iname**          | Case-insensitive name                                                                          | `find ~ -iname "*.conf"`                                      |
| **-type f/d**       | Files only / Dirs only                                                                         | `find /var/log -type f`                                       |
| **-size +10M**      | Size > 10MB                                                                                    | `find / -size +100M 2>/dev/null`                              |
| **-mtime -7**       | Modified <7 days ago                                                                           | `find /tmp -mtime +7 -delete`                                 |
| **-user mostafa**   | Your files only                                                                                | `find /home -user mostafa`                                    |
| **-v**              | inverts the match, prints lines that **DO NOT** contain the pattern instead of matching lines. | `grep -v "error" logfile`        # Show lines WITHOUT "error" |

---

**head** shows the first lines of a file, and **tail** shows the last lines. Both display 10 lines by default. **tail -f** is used to monitor log files in real time.
``` bash 
head -n 20 file.txt | tail -n 10
```
This gets the first 20 lines, then prints the last 10 of that result, effectively showing lines 11 to 20.

---

**cut** is a Linux text-processing command used to extract specific parts of each line from a file or command output. It is commonly used to pull out columns, characters, or fields without modifying the original file.

## What it does

**cut** reads input line by line and prints only the part you ask for. You can select data by **bytes**, **characters**, or **fields separated by a delimiter**.

## Main options

The most important options are `-b` for bytes, `-c` for characters, and `-f` for fields. When using fields, you usually combine `-f` with `-d` to specify the delimiter, such as `,` or `:`.

## Common syntax

```bash
cut OPTION [FILE] cut OPTION < file command | cut OPTION`
```


If no file is given, `cut` reads from standard input, which makes it very useful in pipelines. At least one of `-b`, `-c`, or `-f` is required.

## Examples
```bash
cut -c 1-5 file.txt
```
This prints characters 1 through 5 from each line.


```bash
cut -d ',' -f 2 file.csv
```
This uses `,` as the delimiter and prints the second field from each line, which is a common way to extract a column from CSV-like data.


```bash
cut -d ':' -f 1 /etc/passwd
```


This prints the first field from `/etc/passwd`, which is typically the username column.

## When to use it

Use `cut` when your input is **simple and consistently structured**, such as colon-separated, comma-separated, or fixed-position text. It is fast and script-friendly, but it is less suitable for messy data or complex parsing.

---


- **awk**: text processing command used to scan lines, split them into fields, filter data, and print selected output.
    
- Default behavior:
    
    - Each input line is a record.
        
    - Fields are split by whitespace.
        
    - `$0` = whole line.
        
    - `$1`, `$2`, `$3` = fields/columns.
        
- Common syntax:

```bash
awk 'pattern { action }' file
```


- Special blocks:
    
    - `BEGIN`: runs before reading input.
        
    - `END`: runs after finishing input.
        
- Useful built-ins:
    
    - `NR`: current line number.
        
    - `NF`: number of fields in current line.
        
    - `FS`: input field separator.
        
    - `OFS`: output field separator.
        

## Examples

- Print all lines:
```bash
awk '{print}' file
```

- Print first column:
```bash
awk '{print $1}' file
```

- Print line number with line:
```bash
awk '{print NR, $0}' file
```

- Print lines where column 3 is greater than 100:
```bash
awk '$3 > 100 {print $1, $3}' file
```

- Skip header row:
```bash
awk 'NR > 1 {print}' file
```


- Use `:` as separator:
```bash
awk -F: '{print $1, $3}' /etc/passwd
```


- Set output separator to comma:
```bash
awk 'BEGIN {OFS=","} {print $1, $2}' file
```


- Sum column 2:
```bash
awk '{sum += $2} END {print sum}' file
```


---

