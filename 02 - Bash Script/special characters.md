````**Single quotes (`' '`)** preserve the **literal value** of every character—no variable expansion, command substitution, or escape sequences are interpreted [web:37][web:38]. **Double quotes (`" "`)** allow **variable expansion** (`$var`), **command substitution** (`$(cmd)` or `` `cmd` ``), and **escape sequences** (`\n`, `\t`), but preserve literal values of most other characters [web:37][web:41]. --- ## WHY THIS DISTINCTION EXISTS Bash needs two quoting mechanisms to handle different use cases: 1. **Literal strings** (single quotes): When you want exactly what you type, no interpretation 2. **Dynamic strings** (double quotes): When you need to embed variable values or command output This design allows precision: use single quotes for **data** (SQL queries, regex patterns, literal text), double quotes for **interpolation** (user messages, log entries with variables) [web:41][web:42]. --- ## CORE BEHAVIOR COMPARISON | Feature | Single Quotes `' '` | Double Quotes `" "` | |---------|---------------------|---------------------| | **Variable expansion** | ✗ Disabled (`$var` → literal `$var`) | ✓ Enabled (`$var` → value) [web:37][web:38] | | **Command substitution** | ✗ Disabled (`$(cmd)` → literal text) | ✓ Enabled (`$(cmd)` → output) [web:42][web:44] | | **Escape sequences** | ✗ Literal (`\n` → backslash + n) | ✓ Interpreted (`\n` → newline) [web:38][web:41] | | **Backslash escaping** | ✗ No effect (except can't escape `'`) | ✓ Works (`\"`, `\\`, `\$`) [web:37] | | **Whitespace preservation** | ✓ Preserved | ✓ Preserved [web:42] | | **Glob expansion** | ✗ Disabled (`*` → literal `*`) | ✗ Disabled (`*` → literal `*`) [web:38] | | **Word splitting** | ✗ Prevented | ✗ Prevented | | **Nesting same quote** | ✗ Impossible | ✓ Can escape: `\"` [web:37][web:42] | --- ## PRACTICAL EXAMPLES ### Variable Expansion ```bash name="Alice" echo 'Hello $name'    # Output: Hello $name echo "Hello $name"    # Output: Hello Alice````

[web:37][web:38][web:43]

## Command Substitution

bash

`echo 'Current directory: $(pwd)'    # Output: Current directory: $(pwd) echo "Current directory: $(pwd)"    # Output: Current directory: /home/user`

[web:42][web:44]

## Escape Sequences

bash

`echo 'Line 1\nLine 2'    # Output: Line 1\nLine 2 (literal) echo "Line 1\nLine 2"    # Output: Line 1\nLine 2 (literal in bash, but newline in echo -e) echo -e "Line 1\nLine 2" # Output: (two lines)`

[web:38][web:41]

## Special Characters

bash

`echo 'Price: $100'       # Output: Price: $100 (literal) echo "Price: $100"       # Output: Price: 00 (tries to expand $1, then 00) echo "Price: \$100"      # Output: Price: $100 (escaped)`

[web:37][web:42]

## Backlash Handling

bash

`path='C:\Windows\System32'    # Output: C:\Windows\System32 (literal) path="C:\Windows\System32"    # Output: C:WindowsSystem32 (backslashes interpreted) path="C:\\Windows\\System32"  # Output: C:\Windows\System32 (escaped)`

[web:41]

---

## WHEN TO USE EACH

## Use Single Quotes (`' '`) When:

1. **Literal strings with special characters**
    

bash

`regex='[a-zA-Z0-9]+@[a-z]+\.[a-z]{2,}' sql="SELECT * FROM users WHERE name='$user'"  # Outer double, inner single`

2. **Preventing variable expansion**
    

bash

`echo 'Variables like $HOME and $USER will not expand'`

3. **Multi-line strings (heredoc alternative)**
    

bash

`message='Line 1 Line 2 Line 3'`

4. **File paths with backslashes (Windows-style)**
    

bash

`path='C:\Users\Admin\file.txt'`

[web:38][web:41][web:42]

## Use Double Quotes (`" "`) When:

1. **Variable interpolation required**
    

bash

`echo "Hello, $USER! Your home is $HOME" log_msg="Error occurred at $(date)"`

2. **Preserving whitespace with variables**
    

bash

`filename="my file.txt" cat "$filename"  # Correct: treats as single argument cat $filename    # WRONG: treats as two arguments (my, file.txt)`

3. **Command substitution**
    

bash

`echo "Files in directory: $(ls | wc -l)" echo "Current user: $(whoami)"`

4. **Escaping specific characters**
    

bash

`echo "He said, \"Hello!\"" echo "Price: \$50"`

[web:37][web:38][web:41]

---

## ADVANCED PATTERNS

## Nesting Quotes

bash

`# Single inside double echo "He said, 'Hello World!'" # Double inside single (use escaping outside) echo 'He said, "Hello World!"' # Concatenating different quote types echo 'Single part'"Double part"'Single again'`

[web:42]

## Escaping Single Quotes Within Single Quotes

**PROBLEM:** Cannot escape `'` inside `' '` [web:37]

**WRONG:**

bash

`echo 'It\'s wrong'  # Syntax error`

**CORRECT Solutions:**

bash

`# Method 1: End quote, escape, start quote echo 'It'\''s correct' # Method 2: Use double quotes echo "It's correct" # Method 3: Mix quote types echo 'It'"'"'s correct'`

[web:37][web:41]

## Array with Spaces

bash

`# WRONG: word splitting breaks array files=$(ls *.txt) for f in $files; do echo "$f"; done  # Breaks on spaces # CORRECT: preserve with double quotes files=$(ls *.txt) for f in "$files"; do echo "$f"; done`

## Preserving Command Output

bash

`# Without quotes: newlines become spaces output=$(cat file.txt) echo $output  # One line # With quotes: preserves newlines output=$(cat file.txt) echo "$output"  # Multiple lines preserved`

[web:42]

---

## SPECIAL CHARACTERS BEHAVIOR

|Character|Single Quotes|Double Quotes|No Quotes|
|---|---|---|---|
|`$var`|Literal `$var`|Expands to value|Expands + word split|
|`$(cmd)`|Literal text|Command output|Command output + word split|
|`\n`|Literal `\n`|Literal `\n` (unless `echo -e`)|Literal `\n`|
|`*`|Literal `*`|Literal `*`|Glob expansion|
|`!`|Literal `!`|History expansion (interactive)|History expansion|
|`\`|Literal `\`|Escape next char|Escape next char|
|`"`|Literal `"`|Ends quote (unless `\"`)|No special meaning|
|`'`|Cannot include|Literal `'`|No special meaning|

[web:37][web:38]

---

## SECURITY IMPLICATIONS

## Code Injection Risk

bash

`# DANGEROUS: user input with double quotes user_input="'; rm -rf /; echo '" eval "echo $user_input"  # Executes rm command! # SAFE: single quotes prevent expansion eval 'echo $user_input'  # Prints literal string`

## SQL Injection Prevention

bash

`# WRONG: vulnerable to SQL injection query="SELECT * FROM users WHERE name='$username'" # BETTER: use single quotes for SQL, parameterized queries query="SELECT * FROM users WHERE name=?"  # Prepared statement`

## Always Quote Variables in File Operations

bash

`# VULNERABLE: filename="test; rm -rf /" rm $filename  # Executes rm -rf / # SAFE: rm "$filename"  # Treats as single argument`

[web:37][web:41]

---

## BEST PRACTICES

1. **Default to double quotes** for variables: `"$var"` prevents word splitting [web:42]
    
2. **Use single quotes** for literal strings with special characters [web:41]
    
3. **Always quote variable expansions**: `"$1"`, `"$@"`, `"${array[@]}"` [web:37]
    
4. **Use `$'...'` for ANSI-C quoting** (special case):
    

bash

`echo $'Line 1\nLine 2'  # Interprets \n as newline echo $'Tab:\tSeparated'  # Interprets \t as tab`

5. **Avoid unquoted variables** except in specific controlled cases [web:42]
    

---

## DECISION FLOWCHART

text

`Need to include variable values or command output? ├─ YES → Use double quotes: "..." │   └─ Need literal $, ", or \? → Escape: \$, \", \\ │ └─ NO → Use single quotes: '...'     └─ Need to include single quote? → Use one of:        - 'text'\''more'        - "It's text"        - $'It\'s text'`