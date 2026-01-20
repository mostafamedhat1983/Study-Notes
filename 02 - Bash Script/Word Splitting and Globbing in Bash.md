---
tags:
  - Linux
  - Bash_Script
---


Word splitting and globbing (also called pathname expansion) are two distinct expansion mechanisms that Bash performs on unquoted strings.​

## Word Splitting

Word splitting occurs when Bash divides the results of parameter expansion, command substitution, and arithmetic expansion into separate words based on delimiter characters. This process only affects expansions that are **not enclosed in double quotes**.​

**How it works**: Bash uses the Internal Field Separator (IFS) variable to determine word boundaries. The default IFS contains three characters: space, tab, and newline. When an unquoted expansion occurs, Bash splits the result at each IFS character into multiple words.​

**Example**:

``` bash
files="file1.txt file2.txt file3.txt"
echo $files          # Outputs: file1.txt file2.txt file3.txt (3 separate arguments)
echo "$files"        # Outputs: file1.txt file2.txt file3.txt (1 single argument)
```

Without quotes, `$files` undergoes word splitting and becomes three separate words. With quotes, it remains a single argument.​

**Common pitfall**: If a variable contains spaces and you don't quote it, word splitting will break it apart:

``` bash
filename="my document.txt"
rm $filename        # Tries to delete "my" and "document.txt" separately
rm "$filename"      # Correctly deletes "my document.txt"
```
## Globbing (Pathname Expansion)

Globbing is the process where Bash expands wildcard patterns into matching filenames. This happens after word splitting in the expansion order. Common glob patterns include `*` (matches any characters), `?` (matches single character), and `[...]` (matches character ranges).[](https://stackoverflow.com/questions/18498218/bash-word-splitting-mechanism)​

**Example**:

``` bash
echo *.txt          # Expands to all .txt files in current directory
echo "*.txt"        # Outputs literal string: *.txt (no expansion)
```
Like word splitting, globbing only occurs on unquoted strings. When enclosed in quotes, the pattern remains literal and no filename expansion happens.​

## Prevention

To prevent both word splitting and globbing, always quote your variable expansions: `"$variable"`. If IFS is set to null (empty string), word splitting is disabled entirely, though this isn't commonly recommended