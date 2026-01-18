---
tags:
  - Bash_Script
  - Linux
---
``The canonical pattern for reading files with `read` in bash is: ```bash while IFS= read -r line; do     # process "$line" done < file.txt``

This is the **safest, most memory-efficient** method for files of any size [web:3][web:24].

---

## WHY THIS PATTERN IS STANDARD

**Trade-offs: Core patterns**

|Pattern|Memory|Whitespace Preservation|Large File Safe|Subshell Issue|
|---|---|---|---|---|
|`while IFS= read -r < file`|Stream|✓|✓|None|
|`cat file \| while read`|Stream|✓|✓|**Variables lost** [web:6]|
|`for line in $(cat file)`|**Full load**|✗|✗|None|
|`mapfile -t array < file`|**Full load**|✓|✗|None|

The `while IFS= read -r` pattern:

- Streams one line at a time (no memory limit) [web:3][web:24]
    
- Preserves leading/trailing whitespace via `IFS=` [web:2][web:23]
    
- Prevents backslash interpretation via `-r` [web:2][web:3]
    
- Avoids subshell (variables persist) [web:6]
    

---

## CORE PATTERNS

## 1. Basic Line-by-Line Read (MOST COMMON)

bash

`while IFS= read -r line; do     echo "$line" done < file.txt`

**Use case:** Processing any text file line-by-line [web:2][web:24]

## 2. Using File Descriptor (Avoids stdin Conflicts)

bash

`while IFS= read -r -u9 line; do     echo "$line" done 9< file.txt`

**Why:** Frees stdin for user prompts inside the loop [web:2]

bash

`# Example: stdin is available for read inside loop while IFS= read -r -u9 line; do     read -rp "Process $line? (y/n): " answer    [[ $answer == y ]] && process "$line" done 9< file.txt`

## 3. Read with Line Numbers

bash

`line_num=0 while IFS= read -r line; do     ((line_num++))    printf '%d: %s\n' "$line_num" "$line" done < file.txt`

## 4. Skip Empty Lines

bash

`while IFS= read -r line; do     [[ -z $line ]] && continue    echo "$line" done < file.txt`

## 5. Skip Comments (Lines Starting with #)

bash

`while IFS= read -r line; do     [[ $line =~ ^[[:space:]]*# ]] && continue    echo "$line" done < config.txt`

## 6. CSV Parsing (Multi-Field Split)

bash

`while IFS=',' read -r col1 col2 col3; do     echo "Name: $col1, Age: $col2, City: $col3" done < data.csv`

**Note:** Last variable gets remaining fields if more columns exist [web:24]

## 7. Preserve All Fields in Array (CSV)

bash

`while IFS=',' read -ra fields; do     echo "Field count: ${#fields[@]}"    echo "First: ${fields}, Last: ${fields[-1]}" done < data.csv`

## 8. Colon-Delimited Files (/etc/passwd style)

bash

`while IFS=':' read -r user pass uid gid gecos home shell; do     echo "User: $user, UID: $uid, Shell: $shell" done < /etc/passwd`

## 9. Process Every Nth Line

bash

`count=0 while IFS= read -r line; do     ((count++ % 5 == 0)) && echo "$line"  # Every 5th line done < file.txt`

## 10. Read Until Specific Pattern

bash

`while IFS= read -r line; do     [[ $line == "END" ]] && break    echo "$line" done < file.txt`

## 11. Read from Command Output (Process Substitution)

bash

`while IFS= read -r line; do     echo "$line" done < <(grep "ERROR" /var/log/app.log)`

**Why `< <()`?** Avoids subshell created by pipes [web:6]

## 12. Build Array from File (Matching Lines)

bash

`matching_lines=() while IFS= read -r line; do     [[ $line == pattern* ]] && matching_lines+=("$line") done < file.txt echo "Found: ${#matching_lines[@]} matches"`

[web:25]

## 13. Multi-File Processing

bash

`for file in *.log; do     echo "Processing: $file"    while IFS= read -r line; do        [[ $line == *ERROR* ]] && echo "$file: $line"    done < "$file" done`

## 14. Read Key-Value Config Files

bash

`declare -A config while IFS='=' read -r key value; do     config[$key]=$value done < config.ini echo "Host: ${config[host]}" echo "Port: ${config[port]}"`

## 15. Parallel Processing (GNU Parallel)

bash

`while IFS= read -r line; do     echo "$line" done < urls.txt | parallel wget -q {}`

---

## ADVANCED PATTERNS

## Handle Last Line Without Newline

bash

`while IFS= read -r line || [[ -n $line ]]; do     echo "$line" done < file.txt`

**Why:** Files may lack trailing newline; `|| [[ -n $line ]]` catches it [web:24]

## Read Fixed-Width Fields

bash

`while IFS= read -r line; do     field1="${line:0:10}"   # chars 0-9    field2="${line:10:20}"  # chars 10-29    field3="${line:30}"     # chars 30-end    echo "F1: $field1, F2: $field2, F3: $field3" done < fixed_width.txt`

## JSON Line Processing (JSONL/NDJSON)

bash

`while IFS= read -r json_line; do     name=$(echo "$json_line" | jq -r '.name')    echo "Name: $name" done < data.jsonl`

## Read Entire File into Variable

bash

`file_content=$(<file.txt)  # Native bash, no subshell echo "$file_content"`

## Read File into Array (Entire File at Once)

bash

`mapfile -t lines < file.txt # OR: readarray -t lines < file.txt echo "Total lines: ${#lines[@]}" echo "First line: ${lines}"`

**Warning:** Loads entire file into memory [web:3]

---

## IFS PATTERNS FOR DIFFERENT FORMATS

|Format|IFS Setting|Example|
|---|---|---|
|CSV|`IFS=','`|`while IFS=',' read -r f1 f2; do`|
|TSV|`IFS=$'\t'`|`while IFS=$'\t' read -r f1 f2; do`|
|Colon-delimited|`IFS=':'`|`while IFS=':' read -r f1 f2; do`|
|Whitespace-separated|`IFS=` (default)|`read -r f1 f2 f3`|
|Preserve all whitespace|`IFS=` (empty)|`while IFS= read -r line; do`|

---

## ANTI-PATTERNS (AVOID THESE)

## ❌ WRONG: Using Pipe (Variable Loss)

bash

`cat file.txt | while read -r line; do     count=$((count + 1)) done echo "$count"  # Empty! Subshell lost variable`

[web:6]

## ✓ CORRECT: Input Redirection

bash

`count=0 while read -r line; do     count=$((count + 1)) done < file.txt echo "$count"  # Works!`

## ❌ WRONG: For Loop with cat (Memory + Word Splitting)

bash

`for line in $(cat file.txt); do     echo "$line"  # Breaks on whitespace! done`

[web:3]

## ❌ WRONG: No -r Flag (Escape Interpretation)

bash

`while read line; do  # Dangerous!     echo "$line"  # \n becomes newline, \t becomes tab done < file.txt`

## ❌ WRONG: No IFS= (Whitespace Trimming)

bash

`while read -r line; do     echo "$line"  # Leading/trailing spaces stripped done < file.txt`