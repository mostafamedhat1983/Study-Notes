---
tags:
  - Bash_Script
  - Linux
---
AWK is a powerful text-processing language built into bash that excels at extracting fields, performing calculations, and manipulating data in structured text files. It processes input line-by-line, applying patterns and actions to filter and transform data efficiently.[](https://www.geeksforgeeks.org/linux-unix/awk-command-unixlinux-examples/)

## Basic Syntax

The fundamental AWK syntax follows this structure:[](https://www.geeksforgeeks.org/linux-unix/awk-command-unixlinux-examples/)​

``` bash 
awk [options] 'pattern {action}' input-file
```

You can use AWK in three ways: with only a pattern (prints matching lines), with only an action (applies to all lines), or combining both (action applies only to pattern-matched lines).[](https://www.sampleqa.in/bash/AWK-Programming.html)​

## Essential Commands and Features

## Field Extraction

AWK automatically splits each line into fields using whitespace as the default delimiter. Fields are accessed using `$1`, `$2`, `$3`, etc., while `$0` represents the entire line:[](https://www.w3schools.com/bash/bash_awk.php)

``` bash 
# Print first column from CSV
awk -F"," '{print $1}' data.csv

# Print multiple fields from /etc/passwd
awk -F":" '{print $1, $7}' /etc/passwd
```
## Special Variables

AWK provides built-in variables including `NR` (line number), `NF` (number of fields), `FS` (field separator), and `OFS` (output field separator).​[](https://www.sampleqa.in/bash/AWK-Programming.html)​

## Mathematical Operations

You can perform calculations directly within AWK scripts:[](https://funprojects.blog/2020/11/04/using-awk-in-bash-scripts/)​​

``` bash 
# Calculate division with formatting
echo "3 4" | awk '{c=$2/$1; printf "%0.4f\n", c}'

# Sum values from a column
ls -l | grep '^-' | awk '{sum += $5} END {print sum, "bytes"}'
```

## Pattern Matching with BEGIN and END

The `BEGIN` block executes before processing any input, while `END` executes after all lines are processed:[](https://www.sampleqa.in/bash/AWK-Programming.html)

``` bash 
awk 'BEGIN {print "Processing started..."} {sum += $3} END {print "Total:", sum}' file.txt
```

## Common Options

- **`-F`**: Sets custom field delimiter[](https://www.w3schools.com/bash/bash_awk.php)
    
- **`-v`**: Assigns variables for use within the script[](https://www.w3schools.com/bash/bash_awk.php)
    
- **`-f`**: Executes AWK program from an external file[](https://www.pluralsight.com/resources/blog/cloud/the-awk-command-bash-basics)
    

## Practical DevOps Use Cases

## Process Management

Find and kill processes by name:[](https://funprojects.blog/2020/11/04/using-awk-in-bash-scripts/)​

``` bash 
ps -e | grep "process_name" | awk '{print $1}' | xargs kill -9
```

## Log Analysis

Filter processes with non-zero CPU time:[](https://funprojects.blog/2020/11/04/using-awk-in-bash-scripts/)​

``` bash 
ps -e | awk '{if ($3 != "00:00:00") print $0}'
```

## Data Transformation

Convert CSV to formatted output or add timestamps:[](https://funprojects.blog/2020/11/04/using-awk-in-bash-scripts/)​

``` bash 
cat data.txt | awk '{print strftime("%H:%M:%S ",systime()) $0}'
```

## Conditional Logic

Apply different actions based on field values:[](https://funprojects.blog/2020/11/04/using-awk-in-bash-scripts/)​

``` bash 
awk '{if ($3 < 4) {print $1 "\t small"} else {print $1 "\t medium"}}' data.txt

```

AWK's combination of pattern matching, field processing, and built-in functions makes it indispensable for DevOps automation tasks involving log parsing, system monitoring, and data pipeline transformations.