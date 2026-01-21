---
tags:
  - Bash_Script
  - Linux
---
The Bash `case` statement matches a single expression against multiple patterns and executes the first matching block, serving as a switch-like alternative to chained `if-elif` statements. It starts with `case`, ends with `esac`, and uses `;;` to terminate each pattern's commands.
## Syntax
``` bash
case EXPRESSION in
  pattern1|pattern2)
    commands
    ;;
  pattern3)
    commands
    ;;
  *)
    default commands
    ;;
esac

```

Patterns support globs (`*`, `?`), alternation (`|`) or , and ranges; the `*` handles unmatched cases

## Examples

**Simple menu:**

``` bash
#!/bin/bash
echo "Choose color (1-5):"
read choice
case $choice in
  1) echo "Blue is primary" ;;
  2) echo "Red is primary" ;;
  3) echo "Yellow is primary" ;;
  4|5) echo "Secondary color" ;;
  *) echo "Invalid" ;;
esac
```

This script prompts input and responds based on the number.​

**Argument parser (DevOps-style):**

``` bash
#!/bin/bash
case "$1" in
  start) echo "Starting service" ;;
  stop) echo "Stopping service" ;;
  restart) echo "Restarting" ;;
  *) echo "Usage: $0 {start|stop|restart}" ;;
esac
```

Ideal for CLI tools in Kubernetes or AWS automation scripts.