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

**uptime**  Tell how long the system has been running

---

**free** shows system memory usage

- `free -h` → human-readable output.
    
- `free -b` → show values in bytes.
    
- `free -k` → show values in kilobytes.
    
- `free -m` → show values in megabytes.
    
- `free -g` → show values in gigabytes.
    
- `free -s N` → refresh output every `N` seconds.
    
- `free -c N` → repeat output `N` times, often used with `-s`.
    
- `free -w` → show buffers and cache as separate columns on supported versions.

---

**uname**  shows basic system information, mainly about the kernel and system architecture. If you run it without options, it usually prints the kernel name

- `uname -a` → show all available system information.
    
- `uname -s` → show kernel name.
    
- `uname -n` → show hostname.
    
- `uname -r` → show kernel release.
    
- `uname -v` → show kernel version.
    
- `uname -m` → show machine hardware name or architecture, such as `x86_64`.
    
- `uname -p` → show processor type.
    
- `uname -i` → show hardware platform.
    
- `uname -o` → show operating system name, often
---

**mkdir** make directory

- `mkdir -p path/to/dir` → create parent directories if needed.
    
- `mkdir -v dir` → show a message for each created directory.
    
- `mkdir -m 755 dir` → create the directory with specific permissions

---
**touch** is used to create an empty file or update the timestamps of an existing file. If the file does not exist, `touch` creates it; if it already exists, it updates its access and modification times without changing the file contents.

- `touch -a file` → update access time only.
    
- `touch -m file` → update modification time only.
    
- `touch -c file` → do not create the file if it does not exist.[](https://www.youtube.com/watch?v=bP0dwXU8B64)[](https://www.hostinger.com/in/tutorials/linux-touch-command)
    
- `touch -d "date time" file` → set a specific date and time using a string.[](https://www.hostinger.com/in/tutorials/linux-touch-command)
    
- `touch -t YYYYMMDDhhmm file` → set a specific timestamp.[](https://www.hostinger.com/in/tutorials/linux-touch-command)
    
- `touch -r ref_file file` → copy the timestamp from another file.[](https://www.youtube.com/watch?v=hUyvx7H16cg)[](https://www.hostinger.com/in/tutorials/linux-touch-command)

touch file{1..10} 
expands to: touch file1 file2 file3 file4 file5 file6 file7 file8 file9 file10

---

**cp**  copy files and directories from one location to another

cp [options] source destination 
cp [options] source1 source2 directory
you can copy more than 2 source files with `cp` as long as the last argument is a destination directory

- `cp -r dir1 dir2` or `cp -R dir1 dir2` → copy directories recursively.
    
- `cp -i file1 file2` → ask before overwrite.[](https://man7.org/linux/man-pages/man1/cp.1.html)
    
- `cp -n file1 file2` → do not overwrite existing files.
    
- `cp -v file1 file2` → show what is being copied.
    
- `cp -u file1 file2` → copy only if source is newer.
    
- `cp -p file1 file2` → preserve permissions, ownership, and timestamps.[](https://man7.org/linux/man-pages/man1/cp.1.html)
    
- `cp -a src dest` → archive mode; preserve attributes and copy recursively.[](https://man7.org/linux/man-pages/man1/cp.1.html)
    
- `cp -f file1 file2` → force copy if destination cannot be opened normally.

---

**mv** move or rename files and directories

mv [options] source destination
mv [options] source1 source2 directory
If the destination is a directory, the source is moved into it; if the destination is a new name, the source is renamed

- `mv -i src dest` → ask before overwrite.
    
- `mv -f src dest` → force overwrite without prompting.
    
- `mv -n src dest` → do not overwrite existing files.
    
- `mv -v src dest` → show what is being moved or renamed.
    
- `mv -u src dest` → move only if source is newer or destination does not exist

---

**rm**  remove files and directories from the filesystem

rm [options] file 
rm [options] file1 file2 
rm [options] directory

- `rm -i file` → ask before each removal.
    
- `rm -I files` → ask once before deleting many files or doing recursive removal.
    
- `rm -f file` → force removal and never prompt.[](https://man7.org/linux/man-pages/man1/rm.1.html)
    
- `rm -r dir` or `rm -R dir` → remove directories and their contents recursively.
    
- `rm -d dir` → remove an empty directory.
    
- `rm -v file` → show what is being removed

---
