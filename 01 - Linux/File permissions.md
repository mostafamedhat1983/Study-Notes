---
tags:
  - Linux
---
File permissions in Linux are controlled mainly by a few key commands, and each permission type has a numeric value: **`r=4`, `w=2`, `x=1`**.

---

## Basic numeric values (421 model)

Permissions are built by **adding** the numbers for each type:

- `r` (read) = **4**
    
- `w` (write) = **2**
    
- `x` (execute) = **1**
    

Each of the three positions is then:

- First digit: **owner** (user)
    
- Second digit: **group**
    
- Third digit: **others**
    

So:

- `7` = `4+2+1` → `rwx`
    
- `6` = `4+2+0` → `rw-`
    
- `5` = `4+0+1` → `r-x`
    
- `4` = `4+0+0` → `r--`
    

---

## Common permission patterns (octal)

|Octal|Symbolic|Meaning (typical use)|
|---|---|---|
|`600`|`-rw-------`|Only owner can read/write; private file.|
|`644`|`-rw-r--r--`|Owner r/w, others read only (normal file).|
|`755`|`-rwxr-xr-x`|Owner r/w/x, others can read/execute (scripts, executables).|
|`700`|`-rwx------`|Owner can do anything; others cannot access.|

---

## Common permission commands

## `chmod` – change permissions

- `chmod 644 file.txt` – standard readable‑by‑all, writable‑only‑by‑owner.
    
- `chmod 755 script.sh` – make script executable by owner and readable/executable by others.
    
- `chmod 600 ~/.ssh/id_rsa` – secure private SSH key.
    
- `chmod +x script.sh` – add execute permission for owner, group, and others.
    
- `chmod u+x script.sh` – add execute only for the owner.
    
- `chmod -R 755 dir/` – set permissions recursively on a directory tree.
    

## `ls` – view permissions

- `ls -l` – show file details including permissions (`rwx` or numbers when requested).
    
- `ls -ld dir/` – show permissions of the directory itself, not its contents.
    

## `chown` / `chgrp` – change ownership

- `chown user:group file` – change owner and group of a file.
    
- `chgrp developers dir/` – change group ownership of a directory.
    

---

## Example using 421 explicitly

If you want:

- owner: `r--` → `4`
    
- group: `rw-` → `4+2 = 6`
    
- others: `--x` → `1`
    

you get **`461`**:

bash

`chmod 461 myfile.sh`

Now:

- owner can **read** (`4`)
    
- group can **read and write** (`4+2 = 6`)
    
- others can **only execute** (`1`)
    

## Special permissions in Linux

Linux has three special permissions that add behavior beyond the normal `rwx` bits:

- `setuid` (`u+s`)  
- `setgid` (`g+s`)  
- `sticky bit` (`+t`)

### 1) setuid
- Applies mainly to executable files.
- When set, the program runs with the **owner’s** permissions, not the user who launched it.
- Numeric form: first digit `4`.
- Example: `chmod 4755 file`

### 2) setgid
- On executable files, the program runs with the **group’s** permissions.
- On directories, new files inherit the **directory’s group**.
- Numeric form: first digit `2`.
- Example: `chmod 2755 dir`

### 3) sticky bit
- Applies mainly to directories.
- In a shared directory, users can only delete their **own** files, even if they have write access to the directory.
- Common example: `/tmp`
- Numeric form: first digit `1`.
- Example: `chmod 1777 dir`

### Summary
- `4` = setuid
- `2` = setgid
- `1` = sticky bit

### Combined examples
- `chmod 4755 file` → setuid + normal permissions
- `chmod 2755 dir` → setgid + normal permissions
- `chmod 1777 /tmp` → sticky bit + full access for everyone
- `chmod 6755 file` → setuid + setgid together
- `chmod 7755 dir` → setuid + setgid + sticky bit together