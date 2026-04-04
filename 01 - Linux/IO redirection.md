---
tags:
  - Linux
---
In Bash‑style shells, `>`, `>>`, `1>`, `2>`, and `&>` are **output redirection** operators that control where a command’s output (and errors) go.

---

## `>` – overwrite normal output

Writes standard output (stdout, file descriptor `1`) to a file, **overwriting** it if it exists:

```bash
echo "hello" > log.txt
```
Every time you run this, `log.txt` is cleared and replaced with `hello`.

---

## `>>` – append normal output

Same as `>`, but **appends** instead of overwriting:
```bash
echo "line 1" > log.txt 
echo "line 2" >> log.txt
```

Now `log.txt` contains both lines.

---

## `1>` and `2>` – explicit file descriptors

- `1>` is the same as `>`: stdout explicit.
    
- `2>` redirects **stderr** (standard error) to a file:
```bash
echo "hello" 1>out.log 2>err.log
```


    - `stdout` → `out.log`
        
    - `stderr` → `err.log`
    

---

## `2>&1` – merge error into normal output

`2>&1` means “redirect stderr to wherever stdout is going right now”:

```bash
echo "hello" >all.log 2>&1
```
- stdout and stderr both go to `all.log`.
    
- Order matters: `2>&1 >all.log` is **not** the same; `>all.log` comes too late, so stderr still goes to the terminal.
    

---

## `&>` and `>&` – redirect both stdout and stderr

`&>` (or `>&` in some shells) is a shortcut to send **both** stdout and stderr to the same place:

```bash
echo "hello" &> combined.log
```

This is equivalent to:

```bash
echo "hello" >combined.log 2>&1
```

Both versions send all output and errors to `combined.log`


`/dev/null` is a **special “null device”** that **discards everything written to it** and always returns end-of-file (EOF) when read. In practice, it is used as a **black hole** for unwanted output
`/dev/null` **is where you send output (or error) when you want it “thrown away”** instead of seeing it or writing it to a real file.

---
