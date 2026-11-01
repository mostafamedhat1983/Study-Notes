---
tags:
  - Python
  - Troubleshooting
---

Knowing these problems helps you diagnose Python environment, dependency, syntax, and runtime failures faster.

---

## Main idea

Most Python problems involve interpreter/version mismatch, virtual environments, dependencies, imports, encoding, or error handling.

For each problem below:
- **Why it happens** = root cause
- **Quick sign** = how to recognize it fast
- **Solution** = how to fix it
- **Useful commands** = where to start

---

## 1. Wrong Python version runs

- **Why it happens:** PATH, alias, shebang, or virtual environment selects a different interpreter.
- **Quick sign:** A feature fails although it is supported in another Python version.
- **Solution:** Verify the active interpreter and use the intended executable explicitly.
- **Useful commands:** `which python3`, `python3 --version`, `python3 -c 'import sys; print(sys.executable)'`

---

## 2. ModuleNotFoundError

- **Why it happens:** The package is not installed in the active interpreter or the module path is wrong.
- **Quick sign:** Import fails at runtime.
- **Solution:** Confirm the interpreter, install into that environment, and verify the import path.
- **Useful commands:** `python3 -m pip show <package>`, `python3 -m pip install <package>`, `python3 -c 'import <module>'`

---

## 3. Packages install globally by mistake

- **Why it happens:** No virtual environment is active or the package manager targets the system interpreter.
- **Quick sign:** Dependencies conflict or system tools break.
- **Solution:** Create and activate a project virtual environment, then install dependencies there.
- **Useful commands:** `python3 -m venv .venv`, `source .venv/bin/activate`, `python3 -m pip list`

---

## 4. Virtual environment is not active

- **Why it happens:** Activation was skipped, a new shell was opened, or the wrong environment path was used.
- **Quick sign:** Installed packages are not found.
- **Solution:** Activate the intended environment and verify its interpreter and package list.
- **Useful commands:** `source .venv/bin/activate`, `which python3`, `python3 -m pip list`

---

## 5. Dependency version conflict

- **Why it happens:** Packages require incompatible versions of the same dependency.
- **Quick sign:** Resolver errors or runtime import failures appear.
- **Solution:** Pin compatible versions, inspect dependency metadata, and test in a clean environment.
- **Useful commands:** `python3 -m pip check`, `python3 -m pip show <package>`, `python3 -m venv .venv`

---

## 6. requirements.txt is incomplete

- **Why it happens:** Direct dependencies were recorded but transitive dependencies or versions were not captured.
- **Quick sign:** A clean install fails or behaves differently.
- **Solution:** Generate a complete lock/requirements file from a verified environment and test a clean installation.
- **Useful commands:** `python3 -m pip freeze > requirements.txt`, `python3 -m pip install -r requirements.txt`

---

## 7. SyntaxError

- **Why it happens:** Invalid indentation, missing punctuation, or Python-version-incompatible syntax.
- **Quick sign:** Python fails before running the program.
- **Solution:** Read the reported file and line, fix indentation, and verify the interpreter version.
- **Useful commands:** `python3 -m py_compile script.py`, `python3 --version`

---

## 8. IndentationError

- **Why it happens:** Tabs and spaces are mixed or indentation levels are inconsistent.
- **Quick sign:** Python reports inconsistent use of tabs and spaces.
- **Solution:** Use one indentation style consistently; configure the editor to insert spaces.
- **Useful commands:** `python3 -m py_compile script.py`, `cat -A script.py`

---

## 9. NameError

- **Why it happens:** A variable or function is used before definition, misspelled, or outside its scope.
- **Quick sign:** Runtime reports a name is not defined.
- **Solution:** Check spelling, definition order, and scope; initialize variables before use.
- **Useful commands:** `python3 -m pdb script.py`, `grep -n "<name>" script.py`

---

## 10. TypeError

- **Why it happens:** An operation receives an incompatible type, such as adding a string and integer.
- **Quick sign:** Runtime reports unsupported operand or argument type.
- **Solution:** Inspect value types at the failure point and convert or validate inputs explicitly.
- **Useful commands:** `python3 -m pdb script.py`, add `print(type(value))` temporarily at the failure point

---

## 11. KeyError

- **Why it happens:** Code assumes a dictionary key exists.
- **Quick sign:** Runtime reports the missing key.
- **Solution:** Validate input data or use a safe access pattern with an appropriate default.
- **Useful commands:** `python3 -m pdb script.py`, add `print(data.keys())` temporarily at the failure point

---

## 12. IndexError

- **Why it happens:** Code accesses a list position beyond its length.
- **Quick sign:** Runtime reports list index out of range.
- **Solution:** Check collection length and boundary conditions, especially empty inputs.
- **Useful commands:** `python3 -m pdb script.py`, add `print(len(items))` temporarily at the failure point

---

## 13. AttributeError

- **Why it happens:** An object does not have the accessed method or attribute, often because it is None or another type.
- **Quick sign:** Runtime reports object has no attribute.
- **Solution:** Inspect the object's actual type and handle None or unexpected shapes before access.
- **Useful commands:** `python3 -m pdb script.py`, add `print(type(obj), dir(obj))` temporarily at the failure point

---

## 14. ImportError or circular import

- **Why it happens:** Modules import each other or an import occurs before initialization completes.
- **Quick sign:** Partially initialized module errors appear.
- **Solution:** Restructure shared code into a separate module or move imports to the required scope.
- **Useful commands:** `python3 -m module_name`, `python3 -c 'import module_name'`

---

## 15. Script cannot find relative files

- **Why it happens:** The working directory differs from the script directory.
- **Quick sign:** File exists but open() reports FileNotFound.
- **Solution:** Build paths relative to the script file and log the resolved path.
- **Useful commands:** `pwd`, add this inside the script: `from pathlib import Path; print(Path(__file__).resolve().parent)`

---

## 16. Encoding or Unicode errors

- **Why it happens:** Files use an unexpected encoding or text is decoded incorrectly.
- **Quick sign:** UnicodeDecodeError or unexpected characters appear.
- **Solution:** Specify encoding explicitly when reading/writing files and inspect the source encoding.
- **Useful commands:** `file data.txt`, `python3 -c "print(open('data.txt', encoding='utf-8').read())"`

---

## 17. Unhandled exception hides the cause

- **Why it happens:** Broad exception handling suppresses the original error or traceback.
- **Quick sign:** The program exits silently or with a vague message.
- **Solution:** Catch specific exceptions, log the traceback, and avoid bare except blocks.
- **Useful commands:** `python3 -m pdb script.py`, `python3 -X dev script.py`

---

## 18. Program hangs

- **Why it happens:** A network call, lock, input prompt, or infinite loop waits indefinitely.
- **Quick sign:** The process remains alive without output.
- **Solution:** Add timeouts, inspect threads/processes, and use a debugger or profiler to locate the wait.
- **Useful commands:** `ps -ef`, `py-spy dump --pid <pid>`, `timeout 30 python3 script.py`

---

## 19. Tests fail only in CI

- **Why it happens:** Python version, dependencies, environment variables, working directory, or external services differ.
- **Quick sign:** Tests pass locally but fail in the pipeline.
- **Solution:** Compare interpreter, installed packages, environment, and test command; make CI deterministic.
- **Useful commands:** `python3 --version`, `python3 -m pip freeze`, `python3 -m pytest -v`

---

## 20. Secrets appear in logs or code

- **Why it happens:** Credentials are hardcoded, printed, or included in error messages.
- **Quick sign:** A token or password appears in output, logs, or Git history.
- **Solution:** Rotate the secret immediately, load it from a secure source, and redact sensitive values in logs.
- **Useful commands:** `git log --all -- path/to/file`, `grep -RIn --exclude-dir=.git 'password\\|token' .`

---

## Common mistakes

- installing packages into the system interpreter
- using bare except blocks
- relying on the current working directory for file paths
- committing virtual environments or secrets
- assuming local and CI Python environments are identical

---

## Good practices

- use one virtual environment per project
- pin dependencies and test clean installs
- validate inputs and handle expected failures specifically
- use pathlib for file paths
- keep secrets outside source code and logs

---

## Must memorize

```bash
python3 --version
which python3
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install -r requirements.txt
python3 -m pip check
python3 -m py_compile script.py
python3 -m pytest -v
```

---

## Key ideas

- Confirm which interpreter is actually running.
- Virtual environments isolate dependencies and prevent conflicts.
- Identify the exception type/message and inspect the relevant stack frame.
- Environment differences explain many local-versus-CI failures.
