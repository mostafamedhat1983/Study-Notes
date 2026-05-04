---
tags:
  - Linux
---
Linux uses a standardized directory structure defined by the Filesystem Hierarchy Standard (FHS), with everything organized under the root directory `/`.

## Core Structure

All files and directories stem from `/`, treating devices, binaries, and configs as files. This single-tree layout differs from Windows' drive letters.

## Key Directories

- `/bin`: Essential user binaries like `ls`, `cp` for basic operations, available in single-user mode.
    
- `/boot`: Bootloader files, kernels, and initramfs for system startup.
    
- `/dev`: Device files representing hardware (e.g., disks, USBs as `/dev/sda`).
    
- `/etc`: System-wide configs like `/etc/passwd`, network settings.
    
- `/home`: User home dirs (e.g., `/home/mostafa` for personal files).
    
- `/lib` and `/lib64`: Shared libraries for `/bin` and `/sbin` binaries.
    
- `/opt`: Optional third-party software installs.
    
- `/proc`: Virtual filesystem for process and kernel info (e.g., `/proc/cpuinfo`).
    
- `/root`: Root user's home directory.
    
- `/sbin`: System admin binaries like `fdisk`, `reboot`.
    
- `/tmp`: Temporary files, cleared on reboot.
    
- `/usr`: User programs and data (/usr/bin for apps, /usr/lib for libs, read-only).
    
- `/var`: Variable data like logs (/var/log), mail, spool files.
    

## Viewing Structure

Run `ls -l /` as root (via `sudo`) to list root contents.
