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

## Replacing Text in Vim

- **:s** : replace text in Vim using the substitute command.

- Basic syntax:
`:s/old/new/`


- Replace first match in the current line.
`:s/old/new/`

- Replace all matches in the current line.
`:s/old/new/g`

- Replace all matches in the whole file.
`:%s/old/new/g`

- Replace in whole file with confirmation.
`:%s/old/new/gc`

- Replace between specific lines.
`:10,20s/old/new/g`

- Replace from current line to end of file.

`:.,$s/old/new/g`

- Replace whole word only.
`:%s/\<old\>/new/g`

- Case-insensitive replace.
`:%s/old/new/gi`

## Useful flags

- **g**: replace all matches in a line.
    
- **c**: confirm each replacement.
    
- **i**: ignore case.
    
- **I**: case-sensitive search.

