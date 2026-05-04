---
tags:
  - Linux
---
**Library files in Linux** are precompiled reusable code files that programs use to perform common tasks without rewriting the same code again.

There are two main types of libraries in Linux:

- **Static libraries (`.a`)**: their code is copied into the program at build time.
    
- **Shared libraries (`.so`)**: their code is loaded at runtime, and many programs can use the same library file.
    

Library files are commonly stored in directories like `/lib`, `/usr/lib`, and `/usr/local/lib`. Essential shared libraries needed for booting and core commands are typically found under `/lib` or `/lib64`.


A library may provide functions for:

- **File operations** like open, read, write, and close files.
    
- **Memory management** like allocating and freeing memory.
    
- **String handling** like copying, comparing, or joining text.
    
- **Math operations** like square roots, powers, or trigonometric calculations.
    
- **Process control** like creating or managing processes.
    
- **Network operations** like sending and receiving data over sockets.

## Very short version

**A Linux library is compiled reusable code that programs use so developers do not have to write common code from scratch.**
