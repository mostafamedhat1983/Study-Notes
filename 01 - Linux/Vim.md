---
tags:
  - Linux
---
Vim is a **free, open‑source text editor** that runs in the terminal and also has a GUI version

Vim works with **modes** instead of a normal “type‑as‑you‑go” editor. The main ones in your notes can be:

- **Normal mode** – default; you run commands like move, delete, copy, search, etc. Press `Esc` to return here.
    
- **Insert mode** – where you actually type text (like most editors). Enter with `i`, `a`, `o`, `O`, etc., and go back with `Esc`.
    
- **Visual mode** – you select text with `v`, then apply commands (delete, copy, etc.) on that selection.

| Command       | What it does (in Normal mode)     |
| ------------- | --------------------------------- |
| `i`           | enter Insert mode at cursor       |
| `Esc`         | go back to Normal mode            |
| `h j k l`     | move left, down, up, right        |
| `x`           | delete one character under cursor |
| `dd`          | delete current line               |
| `yy`          | copy (yank) current line          |
| `p`           | paste after cursor                |
| `/word`       | search for `word`                 |
| `u`           | undo last change                  |
| `:w`          | save file                         |
| `:q`          | quit                              |
| `:q!`         | quit without saving               |
| `:wq` or `:x` | save and qui                      |

