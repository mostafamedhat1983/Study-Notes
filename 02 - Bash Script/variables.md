---
tags:
  - Linux
  - Bash_Script
---
you can set a variable in Linux by writing `variable=value` 
```bash
myVar="Hello World"
```
but this variable will be a shell variable that is only available in the current shell session
to reference the variable value use $
```bash
 echo $PATH
```
The command `echo $PATH` displays the current value of the **PATH environment variable**, which is a list of directories your system searches to find the commands you type.​

* Why is it important?

When you type a command (like `ls` or `python`) without specifying its full location (like `/bin/ls`), Linux doesn't check every folder on your hard drive. Instead, it looks through the directories listed in `$PATH`, in order, from left to right.
**Order matters**: If you have two versions of a program, the system runs the one found in the _first_ directory listed
## Environment Variables

To make a variable available to child processes and other programs, you need to export it using the `export`
```bash
export VARIABLE_NAME=value
```
You can also first set a variable and then export it separately:
```bash
JAVA_HOME=/usr/bin/java
export JAVA_HOME
```
## Important Notes

Variables with multiple values should be separated by colons, and values containing spaces must be enclosed in quotes:
```bash
PATH=/usr/bin:/usr/local/bin
DESCRIPTION="Value with spaces"
```

---
## export vs declare
Both `export` and `declare` are shell built-in commands used to manage variables, but they serve different primary purposes.

## Quick Comparison

|Feature|`export`|`declare`|
|---|---|---|
|**Primary Purpose**|Makes variables available to **child processes** (subshells) ​.|Defines variable **attributes** (type, scope, read-only status) [](https://stackoverflow.com/questions/56627534/in-bash-should-i-use-declare-instead-of-local-and-export)​.|
|**Inheritance**|Variables are inherited by child processes [](https://www.digitalocean.com/community/tutorials/export-command-linux)​.|Variables are **local** to the current shell unless combined with `-x` [](https://stackoverflow.com/questions/56627534/in-bash-should-i-use-declare-instead-of-local-and-export)​.|
|**Portability**|Standard POSIX command; works in almost all shells (`sh`, `bash`, `zsh`) [](https://stackoverflow.com/questions/56627534/in-bash-should-i-use-declare-instead-of-local-and-export)​.|Bash-specific builtin; may not work in basic `sh` or other shells ​.|
|**Data Types**|Treats everything as a string.|Can enforce types like **integers** (`-i`) or **arrays** (`-a`) [](https://stackoverflow.com/questions/56627534/in-bash-should-i-use-declare-instead-of-local-and-export)​.|

## When to Use Which?

- **Use `export`** when you simply need to make a variable (like `PATH` or `JAVA_HOME`) available to other programs or scripts you run from your terminal. It is the standard way to set environment variables.[](https://www.digitalocean.com/community/tutorials/export-command-linux)​
    
    bash
    
    `export MY_VAR="available to child processes"`
    
- **Use `declare`** when writing complex Bash scripts where you need strict control over variable behavior.[](https://stackoverflow.com/questions/56627534/in-bash-should-i-use-declare-instead-of-local-and-export)​
    
    - **Read-only variables:** `declare -r constant="cannot change"`
        
    - **Integers:** `declare -i count=10` (performs math automatically, e.g., `count=10+5` becomes 15)
        
    - **Arrays:** `declare -a my_list`
        

## Can they work together?

Yes. `declare -x variable=value` is functionally equivalent to `export variable=value` in Bash. However, `export` is preferred for this specific task because it is more readable and universally understood as "making this variable global".[](https://stackoverflow.com/questions/56627534/in-bash-should-i-use-declare-instead-of-local-and-export)​

___

### Declare command
The `declare` command in Bash is a built-in shell command used to declare variables and set their attributes. It provides fine-grained control over variable behavior, making it particularly useful for scripts requiring typed variables, arrays, or read-only values.​

## Syntax

The basic syntax is:[](https://phoenixnap.com/kb/bash-declare)​

bash

`declare [options] [variable-name]="[value]"`

## Common Options

The most frequently used options include:​

- **-a**: Declare an indexed array
    
- **-A**: Declare an associative array
    
- **-i**: Declare an integer variable
    
- **-r**: Declare a read-only variable (cannot be changed)
    
- **-x**: Export the variable (make it available to child processes)
    
- **-f**: Declare or display functions
    
- **-p**: Display variable value and attributes
    
- **-g**: Declare a global variable (useful inside functions)
    

## Examples

**Declaring an integer variable**:[](https://www.tutorialspoint.com/bash-declare-statement-syntax-and-examples)​

bash

`declare -i my_num=5`

**Declaring an indexed array**:[](https://www.tutorialspoint.com/bash-declare-statement-syntax-and-examples)​

bash

`declare -a my_array=("apple" "banana" "cherry")`

**Declaring a read-only variable**:[](https://www.tutorialspoint.com/bash-declare-statement-syntax-and-examples)​

bash

`declare -r my_name="John"`

**Declaring multiple variables at once**:[](https://www.tutorialspoint.com/bash-declare-statement-syntax-and-examples)​

bash

`declare var1=10 var2="text" var3=("one" "two" "three")`

**Exporting a variable**:[](https://www.tutorialspoint.com/unix_commands/declare.htm)​

bash

`declare -x PATH_VAR="/usr/local/bin"`

---
### Typeset command

The `typeset` command in Bash is a built-in command that declares and modifies variables with specific attributes. It is functionally identical to the `declare` command—they are exact synonyms in Bash.​

## Syntax

bash

`typeset [options] name[=value]`

## Relationship to declare

`typeset` was provided primarily for compatibility with the Korn shell (ksh). However, **`typeset` has been deprecated since Bash version 4.0**, and the `declare` command should be used instead for better forward compatibility. Modern Bash versions may display deprecation warnings when using `typeset`.​

## Common Options

The main options supported in Bash include:[](https://www.tutorialspoint.com/unix_commands/typeset.htm)​

- **-i**: Declare an integer variable
    
- **-r**: Declare a read-only variable
    
- **-x**: Export variable for access in sub-processes
    
- **-u**: Convert lowercase characters to uppercase
    
- **-l**: Convert uppercase characters to lowercase
    
- **-f**: Display or work with functions
    

## Examples

**Declaring an integer**:[](https://www.tutorialspoint.com/unix_commands/typeset.htm)​

bash

`typeset -i num=100 num+=1`

**Declaring a read-only variable**:[](https://www.tutorialspoint.com/unix_commands/typeset.htm)​

bash

`typeset -r name="tutorialspoint"`

**Creating a local variable in a function**:[](https://techalmirah.com/typeset-command-in-bash/)​

bash

`typeset x=5  # Local to the function scope`

**Converting to uppercase**:[](https://www.tutorialspoint.com/unix_commands/typeset.htm)​

bash

`typeset -u name="hello"  # Automatically converts to HELLO`

Since `typeset` is deprecated, use `declare` instead for all new Bash scripts to ensure compatibility with current and future Bash versions.[](https://www.baeldung.com/linux/declare-vs-typeset)

---

Here's a comparison of variable scope behavior when using `typeset`, `declare`, and `export` in Bash:

## Scope Comparison Table

|Command|Script-Level Scope|Function-Level Scope|Exported to Child Processes|Notes|
|---|---|---|---|---|
|`typeset VAR=value`|Global [](https://stackoverflow.com/questions/4419704/differences-between-declare-typeset-and-local-variable-in-bash)​|Local [](https://stackoverflow.com/questions/4419704/differences-between-declare-typeset-and-local-variable-in-bash)​|No|Deprecated; identical to `declare` [](https://stackoverflow.com/questions/4419704/differences-between-declare-typeset-and-local-variable-in-bash)​|
|`declare VAR=value`|Global [](https://stackoverflow.com/questions/4419704/differences-between-declare-typeset-and-local-variable-in-bash)​|Local [](https://stackoverflow.com/questions/4419704/differences-between-declare-typeset-and-local-variable-in-bash)​|No|Preferred modern approach [](https://www.baeldung.com/linux/declare-vs-typeset)​|
|`declare -x VAR=value`|Global [](https://www.baeldung.com/linux/declare-vs-typeset)​|Local [](https://delightlylinux.wordpress.com/2024/03/23/bash-typed-variables-with-declare/)​|Yes ​|Same as `export`|
|`export VAR=value`|Global [](https://delightlylinux.wordpress.com/2024/03/23/bash-typed-variables-with-declare/)​|Global [](https://delightlylinux.wordpress.com/2024/03/23/bash-typed-variables-with-declare/)​|Yes [](https://delightlylinux.wordpress.com/2024/03/23/bash-typed-variables-with-declare/)​|Always creates global variables|
|`VAR=value` (simple assignment)|Global|Global [](https://stackoverflow.com/questions/4419704/differences-between-declare-typeset-and-local-variable-in-bash)​|No|No export unless preceded by `export`|

## Key Differences

## At Script Level (Outside Functions)

All three commands create global variables accessible throughout the script. However, only `declare -x` and `export` make variables available to child processes (subshells or external scripts).​

## Inside Functions

- **`declare`/`typeset`**: Create local variables by default​
    
- **`declare -g`**: Explicitly creates global variables from within functions​
    
- **`export`**: Always creates global variables regardless of where used[](https://delightlylinux.wordpress.com/2024/03/23/bash-typed-variables-with-declare/)​
    

## Child Process Visibility

Variables must be exported using `export` or `declare -x` to be accessible in subshells or external scripts. Without export, variables remain in the current shell's environment only.​

## Example

bash

`#!/bin/bash # Script level typeset var1="local to script"     # Global, not exported declare var2="also global"         # Global, not exported declare -x var3="exported"         # Global AND exported export var4="also exported"        # Global AND exported # Inside function function my_func {     declare local_var="stays here"      # Local to function    declare -g global_from_func="global"  # Global from function    export always_global="exported"     # Global AND exported }`

Use `declare` over `typeset` for better compatibility with modern Bash versions.[](https://www.baeldung.com/linux/declare-vs-typeset)​​

---

## DIRECT ANSWER

Bash variable naming conventions differ based on variable scope and purpose:

- **User-defined local variables**: Use lowercase with underscores: `my_variable`, `user_name` [web:47][web:50]
- **Environment variables & constants**: Use UPPERCASE with underscores: `PATH`, `HOME`, `MAX_CONNECTIONS` [web:49][web:52]
- **Function names**: Use lowercase with underscores: `process_file()`, `validate_input()` [web:49]

**Core rule:** Variables can contain letters, numbers, and underscores, but **cannot start with a number** [web:48][web:51].

---

## WHY THESE CONVENTIONS EXIST

The lowercase/UPPERCASE distinction serves a critical purpose:

1. **Visual differentiation**: Instantly identify environment variables vs local variables [web:47][web:49]
2. **Namespace collision avoidance**: Prevents overwriting system variables (e.g., `PATH`, `HOME`, `USER`) [web:49]
3. **POSIX compatibility**: Environment variables are traditionally uppercase across Unix/Linux systems [web:47]

**Trade-off:** Some teams use ALL_CAPS for all variables (older style [web:52]), but modern practice prefers lowercase for locals to avoid confusion with environment variables [web:47][web:49].

---

## OFFICIAL SYNTAX RULES (MANDATORY)

### Rule 1: Allowed Characters
**Valid:** Letters (a-z, A-Z), numbers (0-9), underscores (_)
**Invalid:** Hyphens (-), spaces, special characters (@, #, $, %, !, etc.)

```bash
# CORRECT
my_variable=10
variable123="value"
_private_var="data"
VAR_NAME="config"

# WRONG
my-variable=10        # Hyphen not allowed
my variable=10        # Space not allowed
my@variable=10        # Special char not allowed
my.variable=10        # Period not allowed
```

## Rule 2: Cannot Start with Number

bash

`# CORRECT var1=100 my_var_2="text" # WRONG 1var=100              # Syntax error 123name="John"        # Invalid`

[web:47][web:48][web:51]

## Rule 3: No Whitespace Around `=`

bash

`# CORRECT name="Alice" # WRONG name = "Alice"        # Interpreted as command 'name' with args '=' and 'Alice' name= "Alice"         # Sets name="" then runs "Alice" command name ="Alice"         # Runs command 'name' with arg '=Alice'`

[web:48]

## Rule 4: Case Sensitive

bash

`var="lowercase" VAR="uppercase" Var="mixed" echo $var    # Output: lowercase echo $VAR    # Output: uppercase echo $Var    # Output: mixed`

[web:51]

## Rule 5: Avoid Reserved Keywords

**Don't use:** `if`, `then`, `else`, `elif`, `fi`, `case`, `esac`, `for`, `while`, `until`, `do`, `done`, `function`, `select`, `time` [web:48][web:51]

bash

`# BAD PRACTICE (causes confusion) if="some value"       # Confuses control flow for=10                # Overrides loop keyword context # Though technically allowed, these create unreadable code`

---

## GOOGLE SHELL STYLE GUIDE (INDUSTRY STANDARD)

## Variable Naming

- **Lowercase with underscores** for local variables: `my_variable` [web:49]
    
- **UPPERCASE with underscores** for environment/global constants: `GLOBAL_CONFIG` [web:49]
    
- **No camelCase or PascalCase** in shell scripts [web:49]
    

## Function Naming

- **Lowercase with underscores**: `function process_data() { }` [web:49]
    
- Use `::` for namespacing in large projects: `mylib::get_value()` [web:49]
    

## Constants (readonly variables)

bash

`# Use uppercase with readonly declaration readonly MAX_RETRIES=3 readonly CONFIG_FILE="/etc/app.conf"`

[web:49][web:53]

---

## BEST PRACTICES

## 1. Use Descriptive Names

bash

`# GOOD: Self-explanatory user_name="Alice" max_connection_timeout=30 tmp_file_path="/tmp/process.tmp" # BAD: Unclear purpose x="Alice" n=30 f="/tmp/process.tmp"`

[web:50]

## 2. Multi-Word Variables (Use Underscores)

bash

`# CORRECT user_first_name="John" database_connection_string="mysql://localhost" # WRONG userfirstname="John"          # Hard to read user-first-name="John"        # Hyphen invalid user first name="John"        # Space invalid`

[web:47][web:50][web:51]

## 3. Prefix Conventions

|Prefix|Purpose|Example|
|---|---|---|
|`tmp_`|Temporary variables|`tmp_result`, `tmp_file`|
|`_`|Private/internal variables|`_internal_counter`|
|`is_`, `has_`|Boolean flags|`is_valid`, `has_permission`|
|`num_`, `count_`|Numeric counters|`num_users`, `count_errors`|

[web:50]

## 4. Avoid Cryptic Abbreviations

bash

`# GOOD customer_id=12345 database_connection="active" # BAD cid=12345                     # What is 'c'? dbc="active"                  # Unclear abbreviation`

## 5. Consistency Across Scripts

bash

`# Pick ONE style and stick to it # Option A: Explicit naming max_retry_count=3 connection_timeout_seconds=30 # Option B: Shorter but consistent max_retries=3 timeout_sec=30`

[web:50]

## 6. Environment vs Local Variables

**Environment Variables (UPPERCASE):**

bash

`export PATH="/usr/local/bin:$PATH" export DATABASE_URL="postgres://localhost" export LOG_LEVEL="debug"`

**Local Variables (lowercase):**

bash

`local user_input="$1" local result="" local counter=0`

[web:47][web:49]

---

## NAMING PATTERNS BY SCOPE

## Script-Level Variables

bash

`#!/bin/bash # Constants (readonly, uppercase) readonly SCRIPT_NAME="$(basename "$0")" readonly VERSION="1.0.0" readonly MAX_ATTEMPTS=3 # Script-wide variables (lowercase) log_file="/var/log/app.log" debug_mode=false user_home="$HOME"`

## Function-Local Variables

bash

`process_user() {     local user_name="$1"          # Function parameter    local user_id="$2"    local tmp_result=""           # Temporary variable         # Processing logic    tmp_result=$(validate_user "$user_name")    echo "$tmp_result" }`

[web:49]

## Loop Variables

bash

`# Use simple, short names for loop iterators for file in *.txt; do     echo "$file" done for i in {1..10}; do     echo "$i" done # Descriptive names for complex loops for server_name in "${servers[@]}"; do     ping -c 1 "$server_name" done`

---

## COMPARISON: COMPETING CONVENTIONS

|Convention|Local Variables|Environment/Constants|Pros|Cons|
|---|---|---|---|---|
|**Google Style** [web:49]|lowercase_underscore|UPPERCASE_UNDERSCORE|Clear separation, POSIX-friendly|None|
|**All UPPERCASE** [web:52]|ALL_UPPERCASE|ALL_UPPERCASE|Easy to spot variables|Conflicts with env vars|
|**camelCase**|camelCase|UPPERCASE|Common in other languages|Non-standard for shell|

**Recommendation:** Follow Google Shell Style Guide [web:49]—it's the de facto industry standard and prevents environment variable conflicts.

---

## SPECIAL CASES

## Private/Internal Variables (Leading Underscore)

bash

`# Convention: _ prefix for "private" scope _internal_config="value" _temp_buffer="" function _helper_function() {     # Internal function, not for external use    echo "Internal helper" }`

[web:50]

## Boolean Variables

bash

`# Use is_, has_, should_ prefixes is_valid=true has_permission=false should_retry=true if [[ $is_valid == true ]]; then     echo "Valid" fi`

## Counters and Indices

bash

`# Be explicit about what's being counted file_count=0 error_count=0 line_number=1 array_index=0`

---

## ANTI-PATTERNS (AVOID)

## ❌ WRONG: Reserved Keywords as Variables

bash

`if="value"                # Confusing while="data"              # Bad practice for="loop"                # Don't do this`

[web:48][web:51]

## ❌ WRONG: Overwriting Shell Variables

bash

`# These are important shell variables—don't overwrite! PATH="/my/path"           # Breaks command execution HOME="/tmp"               # Breaks user environment IFS="custom"              # Breaks word splitting PS1=">"                   # Breaks prompt`

## ❌ WRONG: Inconsistent Naming

bash

`userName="Alice"          # camelCase user_age=30               # snake_case UserEmail="a@b.com"       # PascalCase # Pick ONE convention!`

## ❌ WRONG: Single-Letter Variables (Except Loops)

bash

`# BAD: Unclear purpose a="user data" x=100 z="/path/to/file" # EXCEPTION: Loop variables are OK for i in {1..5}; do echo "$i"; done`

---

## VALIDATION CHECKLIST

Before finalizing variable names, verify:

-  Starts with letter or underscore (not number)
    
-  Contains only letters, numbers, underscores
    
-  No spaces, hyphens, or special characters
    
-  Not a reserved keyword (`if`, `for`, `while`, etc.)
    
-  Doesn't conflict with environment variables (`PATH`, `HOME`)
    
-  Uses consistent casing (lowercase for locals, UPPERCASE for env)
    
-  Descriptive and meaningful
    
-  Follows project/team conventions
    

---

## PRACTICAL EXAMPLES

## Good Naming Convention

bash

`#!/bin/bash # Constants readonly MAX_RETRIES=3 readonly CONFIG_DIR="/etc/myapp" # Environment exports export DATABASE_HOST="localhost" export LOG_LEVEL="info" # Local variables script_name="$(basename "$0")" user_input="$1" tmp_file="/tmp/${script_name}.$$" # Function with local scope process_data() {     local input_file="$1"    local output_dir="$2"    local line_count=0         while IFS= read -r line; do        ((line_count++))        echo "$line" >> "${output_dir}/processed.txt"    done < "$input_file"         echo "Processed $line_count lines" }`

## Bad Naming Convention

bash

`#!/bin/bash # Confusing mix of styles MAX_RETRIES=3             # Should be readonly ConfigDir="/etc/myapp"    # Inconsistent casing export db_host="localhost"  # Should be DATABASE_HOST # Poor variable names x="$1"                    # What is x? f="/tmp/temp.$$"          # Unclear purpose n=0                       # What does n represent? # Reserved keyword confusion for="data"                # Bad practice # Function with poor naming pd() {                    # What does 'pd' mean?     a="$1"                # Unclear    b="$2"                # Unclear    c=0 }`