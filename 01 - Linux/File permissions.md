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
    
