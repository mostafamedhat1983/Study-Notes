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

**find** locates files and directories by name, type, size, time, permissions—recursively searches directory trees. 

## Core Syntax

```bash
find [start_path] [options] [expression] 
find ~ -name "*.yaml"                   # Your home dir, YAML files 
find /etc -type f -name "*.conf"        # Config files only
```

## Essential Options (Your Notes)

| Option              | Purpose                         | Example                             |
| ------------------- | ------------------------------- | ----------------------------------- |
| **-name "pattern"** | Filename match (case-sensitive) | `find . -name "docker-compose.yml"` |
| **-iname**          | Case-insensitive name           | `find ~ -iname "*.conf"`            |
| **-type f/d**       | Files only / Dirs only          | `find /var/log -type f`             |
| **-size +10M**      | Size > 10MB                     | `find / -size +100M 2>/dev/null`    |
| **-mtime -7**       | Modified <7 days ago            | `find /tmp -mtime +7 -delete`       |
| **-user mostafa**   | Your files only                 | `find /home -user mostafa`          |

---

