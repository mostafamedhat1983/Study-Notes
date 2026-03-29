---
tags:
  - Linux
---
## Main file types

- `-` = regular file, such as text files, scripts, images, binaries, and config files.
    
- `d` = directory, which contains entries for other files and directories.
    
- `l` = symbolic link, which points to another path.
    

## Special file types

- `c` = character device, used for byte-stream devices such as terminals and similar device interfaces.
    
- `b` = block device, used for block-based devices such as disks.
    
- `p` = named pipe (FIFO), used for process-to-process data flow.
    
- `s` = socket, used for inter-process communication

## How to check

```bash
ls -l
```

```bash
file filename
```