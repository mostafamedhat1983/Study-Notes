---
tags:
  - Linux
---
Linux uses a standardized directory structure defined by the Filesystem Hierarchy Standard (FHS), with everything organized under the root directory `/`.

## Core Structure

All files and directories stem from `/`, treating devices, binaries, and configs as files. This single-tree layout differs from Windows' drive letters.

## Key Directories

- `/bin`: Traditionally contains essential user commands. On many modern distributions it is a symlink into `/usr/bin` as part of the merged-usr layout.
    
- `/boot`: Bootloader files, kernels, and initramfs for system startup.
    
- `/dev`: Device files representing hardware (e.g., disks, USBs as `/dev/sda`).
    
- `/etc`: System-wide configs like `/etc/passwd`, network settings.
    
- `/home`: User home dirs (e.g., `/home/mostafa` for personal files).
    
- `/lib` and `/lib64`: Traditionally contain essential shared libraries and loader files; on merged-usr systems these may be symlinks into `/usr/lib`.
    
- `/opt`: Optional third-party software installs.
    
- `/proc`: Virtual filesystem for process and kernel info (e.g., `/proc/cpuinfo`).
    
- `/root`: Root user's home directory.
    
- `/sbin`: Traditionally contains system administration commands; on merged-usr systems it may be a symlink into `/usr/sbin`.
    
- `/tmp`: Temporary files. Its contents may be removed at reboot or by periodic cleanup, depending on the distribution and configuration; do not assume files persist.
    
- `/usr`: Most user-space programs, libraries, documentation, and shared data (such as `/usr/bin` and `/usr/lib`). It is treated as static software data by the FHS, but it is not necessarily mounted read-only on every system.
    
- `/var`: Variable data like logs (/var/log), mail, spool files.
    

## Viewing Structure

Run `ls -l /` as root (via `sudo`) to list root contents.
