**`sed` is a stream editor for filtering and transforming text (search/replace, delete, insert lines).**  
Processes input line-by-line; core syntax: `sed 'command' file`. Most common: substitution `s/old/new/g`.[](https://www.geeksforgeeks.org/linux-unix/sed-command-in-linux-unix-with-examples/)

## Common Options

|Option|Description|Example|
|---|---|---|
|`-i`|Edit file **in-place** (save changes)|`sed -i 's/old/new/g' file.txt`|
|`-n`|Suppress automatic printing|`sed -n 'p' file.txt`|
|`-r`/`-E`|Extended regex (no `\` escaping)|`sed -r 's/\d+/NUMBER/g'`|
|`-e`|Multiple commands|`sed -e 's/a/A/' -e 's/b/B/'`|

## Substitution Examples
``` bash
# Replace first occurrence
echo "hello world" | sed 's/hello/HI/'
# Output: HI world

# Global replace (all occurrences)
echo "cat cat dog" | sed 's/cat/dog/g'
# Output: dog dog dog

# Delete matching lines
sed '/ERROR/d' app.log
```

## DevOps Use Cases

``` bash
# Fix YAML indentation
sed -i 's/^  /    /g' config.yaml

# Comment out lines
sed -i '/port: 80/s/^/#/' nginx.conf

# Extract version from pom.xml
grep -o 'version>.*<' pom.xml | sed 's/version>\(.*\)<\/version>/\1/'
```


## Line Addressing

``` bash
# Lines 5-10 only
sed '5,10s/old/new/g' file.txt

# Every 3rd line
sed '0,3s/old/new/g' file.txt
```