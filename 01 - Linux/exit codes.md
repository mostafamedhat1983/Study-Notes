---
tags:
  - Linux
---
Linux exit codes (also called exit status) are 8-bit integers (0-255) returned by commands or scripts to indicate success or failure, accessible via `$?`. Zero means success; non-zero values signal errors, with conventions for common issues like command not found.[](https://pressbooks.senecapolytechnic.ca/uli101/chapter/exit-status-in-shell-scripting/)

## Common Exit Codes

- `0`: Success – command completed without errors.[](https://pressbooks.senecapolytechnic.ca/uli101/chapter/exit-status-in-shell-scripting/)​
    
- `1`: General error – catch-all for failures like syntax issues or missing files.[](https://www.networkworld.com/article/3546937/understanding-exit-codes-on-linux-2.html)​
    
- `2`: Misuse of shell builtins or invalid arguments (e.g., wrong `ls` options).[](https://pressbooks.senecapolytechnic.ca/uli101/chapter/exit-status-in-shell-scripting/)​
    
- `126`: Command found but not executable (permission denied).[](https://itsfoss.com/linux-exit-codes/)​
    
- `127`: Command not found (PATH issue or typo).[](https://itsfoss.com/linux-exit-codes/)​
    
- `128`: Invalid argument to `exit`.[](https://stackoverflow.com/questions/1101957/are-there-any-standard-exit-status-codes-in-linux)​
    
- `128+N`: Fatal error from signal N (e.g., `130` for SIGINT/Ctrl+C, `143` for SIGTERM).[](https://itsfoss.com/linux-exit-codes/)​
    
- `255`: Exit value out of range (modulo 256 wraps higher values).[](https://www.baeldung.com/linux/status-codes)​
    

## Bash Scripting Usage

Check codes in scripts: `if [ $? -ne 0 ]; then echo "Failed"; fi`. Use `exit 1` to propagate errors in functions. For DevOps (your bash/K8s workflows), trap them in CI/CD: `trap 'exit 1' ERR`.[](https://pressbooks.senecapolytechnic.ca/uli101/chapter/exit-status-in-shell-scripting/)​

## Quick Reference Table

| Code | Meaning           | Example Cause                                                                                                     |
| ---- | ----------------- | ----------------------------------------------------------------------------------------------------------------- |
| 0    | Success           | `ls /tmp`[](https://pressbooks.senecapolytechnic.ca/uli101/chapter/exit-status-in-shell-scripting/)​              |
| 1    | General error     | `grep nonexistent file`[](https://www.networkworld.com/article/3546937/understanding-exit-codes-on-linux-2.html)​ |
| 127  | Command not found | `nonexistentcmd`[](https://itsfoss.com/linux-exit-codes/)​                                                        |
| 130  | Ctrl+C interrupt  | User stops long process                                                                                           |