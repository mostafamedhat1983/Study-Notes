---
tags:
  - Linux
  - Bash_Script
---
``The `read` builtin command reads one line of data from standard input (or a file descriptor) and assigns it to variables [web:14]. Basic syntax: ```bash read [options] [variable_names]``

If no variable is specified, input is stored in the `$REPLY` variable 

---

## SYNTAX & BEHAVIOR

bash

`# Basic usage read var1 var2 var3 # User types: "hello world bash" # Result: var1="hello", var2="world", var3="bash"`

**Key behaviors:**

- Splits input into words using `IFS` (Internal Field Separator, default: space/tab/newline) 
    
- If more words than variables exist, last variable gets remaining words [web:14]
    
- If no variable specified, entire line goes to `$REPLY` [web:14]
    

---

## ESSENTIAL OPTIONS

|Option|Description|Use Case|
|---|---|---|
|`-r`|Disable backslash escapes|**Always use this** - prevents `\n` becoming newline [web:13][web:14][web:17]|
|`-p "prompt"`|Display prompt before reading|Interactive input: `read -p "Enter name: " name` [web:13][web:16]|
|`-s`|Silent mode (no echo)|Password input [web:13][web:14]|
|`-t <seconds>`|Timeout after N seconds|Prevent infinite wait [web:13][web:16]|
|`-n <N>`|Read exactly N characters|Single keypress: `read -n 1` [web:13][web:14]|
|`-a <array>`|Read into array (word-by-word)|`read -a myarray` [web:14][web:19]|
|`-d <delim>`|Use custom delimiter instead of newline|`read -d ":"` stops at colon [web:14][web:16]|
|`-u <fd>`|Read from file descriptor|`read -u 9 var` reads from FD 9 [web:13][web:14]|
|`-e`|Use readline (with history/editing)|Interactive shells only [web:14][web:16]|

---

## IMPLEMENTATION PATTERNS

## 1. User Input with Prompt

bash

`read -rp "Enter your name: " username echo "Hello, $username"`

## 2. Password Input (Hidden)

bash

`read -rsp "Enter password: " password echo  # newline after hidden input`

## 3. Timeout (Prevent Hanging)

bash

`if read -rt 5 -p "Press enter within 5 seconds: "; then     echo "You made it!" else     echo "Timeout!" fi`

## 4. Read Multiple Variables

bash

`read -rp "First and Last name: " first last echo "First: $first, Last: $last"`

## 5. Read into Array

bash

`read -ra words <<< "apple banana cherry" echo "${words}"  # apple echo "${words[@]}"  # all elements`

## 6. Read from File Descriptor

bash

`exec 9< /etc/passwd while read -ru 9 line; do     echo "$line" done exec 9<&-  # close FD`

## 7. Custom Delimiter (CSV parsing)

bash

`IFS=',' read -r col1 col2 col3 <<< "data1,data2,data3"`

## 8. Single Character Input

bash

`read -n 1 -rp "Continue? (y/n): " choice echo [[ $choice == "y" ]] && echo "Continuing..."`

---

## WHY `-r` IS CRITICAL

**Without `-r` (DANGEROUS):**

bash

`read input <<< "hello\tworld" echo "$input"  # Output: hello    world (tab interpreted)`

**With `-r` (SAFE):**

bash

`read -r input <<< "hello\tworld" echo "$input"  # Output: hello\tworld (literal backslash-t)`

The `-r` flag treats backslashes literally, preventing:

- Code injection via escape sequences [web:14][web:17]
    
- Unintended line continuation with `\` at end of line [web:16]
    

---

## IFS (INTERNAL FIELD SEPARATOR)

Controls how `read` splits input into variables [web:14][web:16]:

**Default IFS (space/tab/newline):**

bash

`read -r a b c <<< "one   two   three" # a="one", b="two", c="three" (multiple spaces collapsed)`

**Custom IFS:**

bash

`IFS=':' read -r user pass uid gid <<< "root:x:0:0" # user="root", pass="x", uid="0", gid="0"`

**Important:** When `IFS` is non-whitespace, fields are separated by **exactly one character** [web:14]:

bash

`IFS=':' read -r a b c <<< "hello::world" # a="hello", b="" (empty), c="world"`

---

## RETURN STATUS

|Exit Code|Meaning|
|---|---|
|0|Success|
|1|EOF reached or invalid file descriptor [web:14]|
|2|Invalid option [web:14]|
|>128|Timeout occurred [web:14]|

---

## SECURITY & BEST PRACTICES

1. **Always use `-r`**: Prevents escape sequence interpretation [web:14][web:17]
    
2. **Quote variables**: Use `"$var"` not `$var` to preserve whitespace
    
3. **Use `-p` instead of `echo` + `read`**: Atomic operation, cleaner code
    
4. **Set timeout for user input**: Prevent script hanging with `-t`
    
5. **Use `-s` for sensitive data**: Passwords should never echo to terminal [web:13]
    
6. **Preserve `IFS` if modifying**:
    

bash

`OLD_IFS=$IFS IFS=':' read -r fields IFS=$OLD_IFS`

---

## COMMON PITFALLS

**WRONG - Piping loses variable scope:**

bash

`echo "hello" | read var echo "$var"  # Empty! (ran in subshell)`

**CORRECT - Use here-string or file descriptor:**

bash

`read -r var <<< "hello" echo "$var"  # Works!`

**WRONG - No `-r` with file paths:**

bash

`read path <<< "C:\tmp\new" # Interprets \t as tab, \n as newline`

**CORRECT:**

bash

`read -r path <<< "C:\tmp\new" # Preserves literal backslashes`