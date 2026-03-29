---
tags:
  - Linux
---
 **ps**   - report a snapshot of the current processes.

---

**time** - run programs and summarize system resource usage
time ls
aws  awscliv2.zip  my-project-Infra  session-manager-plugin.deb

real    0m0.003s
user    0m0.003s
sys     0m0.000s

---
**more** - is a pager utility that displays the contents of text files one screen or page at a time, making it easier to read long files in the terminal. It allows you to scroll through large files without overwhelming your screen with text all at once
## Navigation Commands

While viewing a file with `more`, you can use these keyboard shortcuts:​

- **Spacebar** - Move forward one full screen
    
- **Enter** - Move forward one line at a time
    
- **b** - Move back one screen
    
- **q** - Quit and exit the viewer
    
- **/pattern** - Search for text pattern within the file
    
- **n** - Jump to next occurrence of search pattern
    
- **h** - Display help with available commands

---

**whoami**  shows current username

___

**pwd**  print working directory

---
**ls**  list directory contents

- `ls -a`  
    Lists **all files**, including hidden ones (names starting with `.`).
    
- `ls -l`  
    Shows **long format** with permissions, owner, group, size, and modification time.
    
- `ls -la`  
    Long format plus all files (including hidden ones).
    

## Format and readability

- `ls -h`  
    Shows sizes in **human‑readable units** (KB, MB, etc.), usually with `-l` as `ls -lh`.
    
- `ls -F`  
    Appends type indicators (e.g., `/` for directories, `*` for executables).
    
- `ls -1`  
    Lists one entry per line instead of in columns.
    

## Sorting and ordering

- `ls -t`  
    Sorts by **modification time**, newest first.
    
- `ls -r`  
    **Reverses** the sort order (e.g., `ls -tr` for oldest first).
    
- `ls -S`  
    Sorts by **file size**, largest first.
    

## Directory and recursion

- `ls -R`  
    Lists contents **recursively** (including subdirectories).
    
- `ls -d`  
    Lists **directories themselves**, not their contents (often used with patterns like `ls -d */`).
    

---

## Quick reference table

|Option|What it does|
|---|---|
|`ls`|List non‑hidden files and dirs in current dir.|
|`ls -a`|Show all files, including hidden ones.|
|`ls -l`|Long format with detailed info (perms, owner, size, time).|
|`ls -la`|Long format + all files (including hidden).|
|`ls -lh`|Long format with human‑readable sizes.|
|`ls -F`|Add type indicators (e.g., `/` for dirs).|
|`ls -1`|One entry per line.|
|`ls -t`|Sort by modification time, newest first.|
|`ls -r`|Reverse sort order.|
|`ls -S`|Sort by size, largest first.|
|`ls -R`|Recursively list all subdirectories.|
|`ls -d`|List directory entries, not their contents.|

---

**cat**  concatenate files and print on the standard output

**cat /etc/os-release**
prints the operating‑system identification data of your Linux system
 
---
**sudo** run a command with elevated privileges, usually as `root`. It uses the system’s sudo policy and typically asks for **your** password

- `sudo -u user command` → run as another user.
    
- `sudo -i` → open a login shell as root.
    
- `sudo -s` → open a shell with elevated privileges.
    
- `sudo -l` → list allowed commands.
    
- `sudo -v` → refresh cached credentials.
    
- `sudo -k` → require password again next time.
    
- `sudo -e file` → safely edit a protected file.
    
- `sudo -h` → help.
    
- `sudo -V` → version info.

---
**cd** change the current working directory

- `cd` → go to your home directory.[](https://en.wikipedia.org/wiki/Cd_\(command\))
    
- `cd /path/to/dir` → go to an absolute path.
    
- `cd dir_name` → go to a subdirectory using a relative path.
    
- `cd ..` → go up one directory.
    
- `cd -` → go back to the previous directory.[](https://www.youtube.com/watch?v=HErmdjrblvI)
    
- `cd /` → go to the root directory.[](https://www.geeksforgeeks.org/linux-unix/cd-command-in-linux-with-examples/)

---

**clear**  clear the terminal screen

---
