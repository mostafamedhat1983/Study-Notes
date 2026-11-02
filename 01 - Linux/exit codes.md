---
tags:
  - Linux
---
An exit status is an integer returned by a process when it terminates. In a shell, `$?` contains the status of the most recently completed foreground pipeline or command. By convention, zero means success and non-zero means some form of failure, but the exact meaning of non-zero values is command-specific.

## Common conventions

- `0`: success.
- `1`: commonly used for a general error; exact meaning depends on the command.
- `2`: often used for incorrect command usage or invalid arguments by shell utilities; not a universal rule.
- `126`: shell convention meaning a command was found but could not be executed.
- `127`: shell convention meaning a command could not be found.
- `128 + N`: Bash convention commonly used when a command terminates because of fatal signal N. For example, 130 commonly indicates SIGINT and 143 commonly indicates SIGTERM.
- `255`: a valid shell-visible status value; some programs use it for errors. It is not a universal “out of range” status.

## Important distinction: 128 and signal statuses

Do not memorize `128` as “invalid argument to exit.” Bash reports statuses in the range 0–255. A status of 128 alone is not a universal error meaning. The `128 + N` convention applies when a process is terminated by signal N, within the shell's status conventions.

When a program passes an exit value outside the supported 0–255 range, the shell or operating system interface may expose a truncated status. Avoid relying on out-of-range values.

## Bash usage

```bash
some_command
status=$?

if [ "$status" -eq 0 ]; then
  echo "Success"
else
  echo "Failed with status $status"
fi
```

Prefer checking a command directly when possible:

```bash
if some_command; then
  echo "Success"
else
  echo "Failed with status $?"
fi
```

Use meaningful exit statuses in scripts and document their meaning. Do not assume that every program follows the same mapping for statuses 1 and 2.

## Quick reference

| Status | Common interpretation |
|---|---|
| 0 | Success |
| 1 | Common general-error convention; command-specific |
| 2 | Common usage-error convention for some utilities; command-specific |
| 126 | Command found but not executable (shell convention) |
| 127 | Command not found (shell convention) |
| 130 | Common result for termination by SIGINT |
| 143 | Common result for termination by SIGTERM |

## DevOps troubleshooting

In CI/CD, inspect the failing command's own documentation and logs before interpreting a non-zero status. A failed pipeline step may be caused by a command returning non-zero, a shell option such as `set -e`, or pipeline status handling. A trap such as `trap '...' ERR` does not replace explicit error handling and has Bash-specific semantics.
