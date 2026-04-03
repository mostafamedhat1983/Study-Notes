---
tags:
  - Linux
---
**grep** searches files or command output for text patterns and prints matching lines

## Two Main Modes

## 1. File Search

grep "error" /var/log/syslog          # Inside files 
grep -r "TODO" /src/project/          # Recursive directories`

## 2. Pipe/Filter Output

ps aux | grep docker                  # Filter processes 
ip route | grep docker0               # Filter routes 
ss -tulnp | grep :53                  # Filter ports 
docker logs nginx | grep "404"        # Filter container logs

## Must-Know Options

| Option | Purpose                    | Example                   |
| ------ | -------------------------- | ------------------------- |
| **-i** | Case insensitive           | `grep -i error`           |
| **-v** | Invert (show non-matches)  | `grep -v OK`              |
| **-r** | Recursive (subdirectories) | `grep -r "TODO" src/`     |
| **-n** | Show line numbers          | `grep -n "fail" log.txt`  |
| **-l** | Filenames only             | `grep -l "error" *.log`   |
| **-c** | Count matches              | `grep -c "error" log.txt` |
| **-w** | Whole words only           | `grep -w "docker" output` |

---
