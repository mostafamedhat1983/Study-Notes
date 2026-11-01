---
tags:
  - Bash
  - Troubleshooting
---

Knowing these problems helps you diagnose and fix Bash scripts faster.

---

## Main idea

Most Bash problems involve quoting, expansion, environment, exit status, or input handling.

For each problem below:
- **Why it happens** = root cause
- **Quick sign** = how to recognize it fast
- **Solution** = how to fix it
- **Useful commands** = where to start

---

## 1. Permission denied when running a script

- **Why it happens:** The script lacks execute permission or its filesystem is mounted noexec.
- **Quick sign:** Direct execution fails with Permission denied.
- **Solution:** Inspect permissions and mount options; grant execute permission only where appropriate.
- **Useful commands:** `ls -l script.sh`, `chmod u+x script.sh`, `findmnt -T script.sh`

---

## 2. Bad interpreter or command contains carriage returns

- **Why it happens:** Windows CRLF line endings introduce carriage returns.
- **Quick sign:** Errors contain ^M or $'\r'.
- **Solution:** Convert to Unix line endings and configure the editor to save LF.
- **Useful commands:** `file script.sh`, `cat -v script.sh`, `dos2unix script.sh`

---

## 3. Bash syntax fails under sh

- **Why it happens:** The script uses Bash features but runs through a different shell.
- **Quick sign:** Arrays or [[ expressions produce syntax errors.
- **Solution:** Use an appropriate Bash shebang and invoke with bash, not sh.
- **Useful commands:** `head -n 1 script.sh`, `bash script.sh`, `bash --version`

---

## 4. Command not found

- **Why it happens:** A dependency is missing or PATH differs from the interactive shell.
- **Quick sign:** A command works manually but fails in automation.
- **Solution:** Check the execution environment, install required tools, and use verified paths.
- **Useful commands:** `command -v tool`, `printf '%s\n' "$PATH"`

---

## 5. Spaces break filenames or arguments

- **Why it happens:** Unquoted expansions undergo word splitting and pathname expansion.
- **Quick sign:** One filename becomes multiple arguments.
- **Solution:** Double-quote variable expansions and use arrays for argument lists.
- **Useful commands:** `shellcheck script.sh`, `printf '%s\n' "$file"`

---

## 6. Wildcards expand unexpectedly

- **Why it happens:** The shell expands unquoted glob patterns before the command runs.
- **Quick sign:** A pattern becomes a list of local files.
- **Solution:** Quote patterns intended for another tool; inspect unmatched-glob behavior when using shell globs.
- **Useful commands:** `shopt nullglob failglob`, `find . -name '*.log'`

---

## 7. Variables do not expand

- **Why it happens:** Single quotes preserve literal text.
- **Quick sign:** Output contains the variable name instead of its value.
- **Solution:** Use double quotes when expansion is intended; keep single quotes for literal strings.
- **Useful commands:** `printf '%s\n' "$HOME"`, `declare -p HOME`

---

## 8. Variables disappear after a pipeline loop

- **Why it happens:** Pipeline components commonly run in subshells.
- **Quick sign:** A counter changes inside a loop but not afterward.
- **Solution:** Redirect input into the loop or use Bash process substitution.
- **Useful commands:** `while IFS= read -r line; do count=$((count + 1)); done < input.txt`

---

## 9. Child processes cannot see a variable

- **Why it happens:** Shell variables are not automatically exported.
- **Quick sign:** A child command reports an empty setting.
- **Solution:** Export required variables; remember a child cannot change its parent's environment.
- **Useful commands:** `export APP_ENV=dev`, `printenv APP_ENV`

---

## 10. Missing arguments cause failures

- **Why it happens:** The script assumes required positional parameters exist.
- **Quick sign:** Empty paths or unbound-variable errors appear.
- **Solution:** Validate argument count before use and provide a usage message.
- **Useful commands:** `printf '%s\n' "$#"`, `: "${1:?Usage: script.sh FILE}"`

---

## 11. Test expressions fail

- **Why it happens:** Missing spaces, empty expansions, or incorrect operators break tests.
- **Quick sign:** Errors include missing ] or unary operator expected.
- **Solution:** Use [[ ]] in Bash for string tests and (( )) for arithmetic conditions.
- **Useful commands:** `[[ -n "$value" ]]`, `(( count > 0 ))`, `bash -n script.sh`

---

## 12. Arithmetic gives unexpected results

- **Why it happens:** Bash arithmetic is integer-only and leading zeros can imply octal.
- **Quick sign:** Decimal fractions fail or 08 triggers a base error.
- **Solution:** Validate input; use explicit base 10 for validated unsigned digits and a suitable tool for fractions.
- **Useful commands:** `printf '%s\n' "$((10#08))"`, `awk 'BEGIN {print 5/2}'`

---

## 13. Exit status is lost

- **Why it happens:** Another command overwrites the previous status.
- **Quick sign:** Failure checks report success after a logging command.
- **Solution:** Test the command directly or capture its status immediately.
- **Useful commands:** `if command -v git >/dev/null; then echo available; fi`

---

## 14. Pipeline failure goes unnoticed

- **Why it happens:** By default, a pipeline reports the last command's status.
- **Quick sign:** An earlier command fails while the overall pipeline succeeds.
- **Solution:** Consider pipefail and inspect PIPESTATUS immediately; handle expected nonzero statuses explicitly.
- **Useful commands:** `set -o pipefail`, `printf '%s\n' "${PIPESTATUS[@]}"`

---

## 15. set -e exits unexpectedly or misses failures

- **Why it happens:** errexit behavior depends on command context, including conditionals.
- **Quick sign:** A script stops on an expected nonzero status or continues unexpectedly.
- **Solution:** Use explicit error handling for important operations instead of relying on set -e alone.
- **Useful commands:** `help set`, `bash -x script.sh`

---

## 16. Reading lines damages input

- **Why it happens:** Default read handling strips whitespace and interprets backslashes.
- **Quick sign:** Input changes when processed.
- **Solution:** Use IFS= read -r; handle an unterminated final line if needed.
- **Useful commands:** `while IFS= read -r line || [[ -n "$line" ]]; do printf '%s\n' "$line"; done < input.txt`

---

## 17. Filename loops break

- **Why it happens:** Parsing ls output or whitespace-separated filenames loses boundaries.
- **Quick sign:** Names with spaces or newlines are processed incorrectly.
- **Solution:** Use shell globs with appropriate unmatched-pattern handling or NUL-delimited find output.
- **Useful commands:** `find . -type f -print0`, `shellcheck script.sh`

---

## 18. Relative paths fail in cron or CI

- **Why it happens:** The working directory differs from the expected directory.
- **Quick sign:** Files exist but the script cannot find them.
- **Solution:** Define a working directory explicitly and check cd failures; use absolute paths where appropriate.
- **Useful commands:** `pwd`, `realpath script.sh`, `cd /known/path || exit 1`

---

## 19. Redirection misses stderr

- **Why it happens:** Redirections are applied from left to right.
- **Quick sign:** Errors still reach the terminal while stdout goes to a file.
- **Solution:** Redirect stdout first, then duplicate stderr when both should share a destination.
- **Useful commands:** `bash script.sh >run.log 2>&1`

---

## 20. Temporary files remain after failure

- **Why it happens:** Cleanup only runs on the successful path.
- **Quick sign:** Aborted runs leave temporary data behind.
- **Solution:** Create temporary files safely and register an EXIT trap; SIGKILL cannot be trapped.
- **Useful commands:** `mktemp`, `help trap`

---

## Common mistakes

- leaving expansions unquoted
- parsing ls output
- using eval for ordinary argument handling
- assuming set -e handles every failure
- tracing secrets with set -x

---

## Good practices

- validate inputs before changing files
- use arrays for command arguments
- test with representative filenames and empty inputs
- check syntax and run ShellCheck
- avoid destructive debugging on production data

---

## Must memorize

```bash
bash -n script.sh
shellcheck script.sh
bash -x script.sh  # May expose secrets
command -v bash
set -o pipefail
# Safe temporary-file pattern
tmp=$(mktemp) || exit 1
trap 'rm -f -- "$tmp"' EXIT
```

---

## Key ideas

- Quoting preserves argument boundaries.
- Environment and working directory matter in automation.
- Check important exit statuses explicitly.
- Debug traces can leak sensitive values.
