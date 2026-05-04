---
tags:
  - Linux
---
Linux exit codes (also called exit status) are 8-bit integers (0-255) returned by commands or scripts to indicate success or failure, accessible via `$?`. Zero means success; non-zero values signal errors, with conventions for common issues like command not found.

## Common Exit Codes

- `0`: Success – command completed without errors.
    
- `1`: General error – catch-all for failures like syntax issues or missing files.
    
- `2`: Misuse of shell builtins or invalid arguments (e.g., wrong `ls` options).
    
- `126`: Command found but not executable (permission denied).
    
- `127`: Command not found (PATH issue or typo).
    
- `128`: Invalid argument to `exit`.
    
- `128+N`: Fatal error from signal N (e.g., `130` for SIGINT/Ctrl+C, `143` for SIGTERM).
    
- `255`: Exit value out of range (modulo 256 wraps higher values).
    

## Bash Scripting Usage

Check codes in scripts: `if [ $? -ne 0 ]; then echo "Failed"; fi`. Use `exit 1` to propagate errors in functions. For DevOps (your bash/K8s workflows), trap them in CI/CD: `trap 'exit 1' ERR`.

## Quick Reference Table

| Code | Meaning           | Example Cause                        |
| ---- | ----------------- | ------------------------------------ |
| 0    | Success           | `ls /tmp`                            |
| 1    | General error     | `grep nonexistent file`              |
| 127  | Command not found | `nonexistentcmd`                     |
| 130  | Ctrl+C interrupt  | User stops long process              |
