POSIX stands for **Portable Operating System Interface**, a family of standards developed by the IEEE Computer Society that defines operating system interfaces to maintain compatibility between different systems. The standard specifies application programming interfaces (APIs), command-line shells, and utility interfaces primarily based on Unix, enabling software portability across Unix, Linux, and other compatible operating systems.​

## Purpose and Benefits

POSIX was created in 1988 to address the fragmentation problem that emerged as multiple Unix variants proliferated, threatening the promise of cross-platform application portability. The standard enables a "write once, adopt everywhere" approach where developers can create applications that run on any POSIX-compliant system without modifying the source code. This reduces development and maintenance costs by eliminating the need for separate codebases for different operating systems.​

## What POSIX Standardizes

The current specification (IEEE Std 1003.1-2017) consists of four major components:​

- **Base Definitions** - Common terms, concepts, syntax, and C-language header definitions
    
- **System Interfaces** - System service functions and subroutines for the C programming language, covering process management, file I/O, and inter-process communication
    
- **Shell and Utilities** - Command interpreter and common utility programs like `awk`, `echo`, and `ed`
    
- **Rationale** - Historical information explaining why features were included or excluded
    

## POSIX-Compliant Systems

Unix systems like AIX, HP-UX, Solaris, and macOS are POSIX-compliant, as are Unix-like systems such as Linux and BSD. The standard is intended for both application developers seeking portability and system implementors building compatible operating systems.​


Bash is **mostly POSIX-compliant**, but with important distinctions. Bash conforms to the POSIX.1-2008 standard and is intended as a conformant implementation of the IEEE POSIX Shell and Tools specification. However, bash includes many additional features beyond what POSIX defines, commonly called "bashisms".​

## Bashisms vs POSIX Features

Bashisms are features not included in POSIX standards, such as arrays, associative arrays, extended test operators (`[[`), and certain function syntax. When you write scripts using these bash-specific features, they won't run on strictly POSIX-compliant shells like `dash` or `ksh`.​

## Running Bash in POSIX Mode

You can make bash behave more strictly according to POSIX standards in several ways:​

- Start bash with the `--posix` command-line option
    
- Execute `set -o posix` in a running bash session
    
- Invoke bash as `sh` (when bash is symlinked to `/bin/sh`)
    
- Set the `POSIXLY_CORRECT` environment variable
    

## Version Differences

Earlier versions of bash weren't fully POSIX-compliant, but **Bash 4.0 and later versions are fully POSIX-compliant**. If you write scripts using only POSIX-compliant syntax in bash, they can run on most Unix and Linux-based operating systems.[](https://www.baeldung.com/linux/test-posix-compliance-shell-scripts)​

## Testing for POSIX Compliance

You can test if your bash scripts are POSIX-compliant using the ShellCheck tool with the `--shell=sh` option, which identifies bashisms and other non-POSIX constructs in your code.​[](https://www.baeldung.com/linux/test-posix-compliance-shell-scripts)​