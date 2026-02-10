# Shell vs Terminal vs Console vs CLI vs TTY vs POSIX (Linux)

## Console
- Historically: a physical screen + keyboard attached to a machine
- In Linux: the system’s main input/output device
- Device file: /dev/console
- Used for boot messages and recovery

## TTY (Teletype)
- Originally: physical teleprinters
- In Linux: a text communication device
- Virtual consoles: /dev/tty1, /dev/tty2 (Ctrl+Alt+F1–F6)
- Pseudo terminals: /dev/pts/0, /dev/pts/1
- Check current TTY:
  tty

## Terminal
- A device or application that connects you to a TTY
- Usually a terminal emulator
- Examples: GNOME Terminal, xterm, Windows Terminal, iTerm2
- Creates pseudo terminals (/dev/pts/*)

## Shell
- A command interpreter program
- Runs inside a terminal
- Executes commands and scripts
- Common shells: bash, zsh, sh, fish
- Check current shell:
  echo $SHELL

## CLI (Command-Line Interface)
- A text-based way to interact with software
- Not a program itself
- Examples: git, docker, kubectl, aws
- Used through a shell

## POSIX
- A standard, not a tool
- Defines shell behavior (sh), system calls, and core utilities
- Ensures portability across Unix-like systems
- POSIX shell example:
  #!/bin/sh

## How Everything Fits Together

Keyboard
 ↓
Terminal Emulator
 ↓
TTY (/dev/tty or /dev/pts/*)
 ↓
Shell (bash / sh / zsh)
 ↓
CLI tools (git, docker, kubectl)
 ↓
Linux Kernel

## One-line Mental Model
- Console → system screen
- TTY → communication channel
- Terminal → window/application
- Shell → command interpreter
- CLI → text-based interaction
- POSIX → rules/standard
