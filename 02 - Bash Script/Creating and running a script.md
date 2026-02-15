---
tags:
  - Linux
  - Bash_Script
---
# to create a bash script file create any file with .sh extension and the first line must be
#!/bin/bash
This line is called a shebang or hashbang. It tells the operating system to execute the file using the bash interpreter located at /bin/bash.

## Shebang: `#!/bin/bash` vs `#!/usr/bin/env bash`

Both shebangs specify that Bash should interpret the script, but they locate the Bash executable differently.​

## `#!/bin/bash`

This shebang uses an absolute path to the Bash interpreter located at `/bin/bash`. When the script runs, the system directly invokes the Bash executable from that specific location. This approach assumes Bash is installed in `/bin/`, which is standard on most Linux distributions.​

**Advantages**: Faster execution since the system doesn't need to search for Bash in the PATH. It's also more predictable on systems where multiple Bash versions exist.[](https://stackoverflow.com/questions/10376206/what-is-the-preferred-bash-shebang)​

**Disadvantages**: Less portable because Bash may be installed in different locations on various systems (e.g., `/usr/local/bin/bash` on macOS or BSD systems). If Bash isn't at `/bin/bash`, the script will fail.​

## `#!/usr/bin/env bash`

This shebang uses the `env` command to search for Bash in the user's PATH environment variable. The `env` utility locates the first `bash` executable it finds in PATH and uses that interpreter.​

**Advantages**: More portable across different Unix-like systems because it doesn't assume a fixed Bash location. This is particularly useful when Bash is installed in non-standard locations or when users have custom Bash installations.​

**Disadvantages**: Slightly slower due to the PATH search. It may execute an unexpected Bash version if multiple versions exist in PATH, potentially causing compatibility issues.[](https://stackoverflow.com/questions/10376206/what-is-the-preferred-bash-shebang)​

## Which Should You Use?

Use `#!/usr/bin/env bash` for maximum portability, especially if your script will run on diverse systems like macOS, BSD, or containers where Bash locations vary. Use `#!/bin/bash` when you need consistent behavior on standard Linux systems where Bash is guaranteed to be at `/bin/bash`.​

## Don't forget to add execute permission to the script file using

chmod +x filename

## to run the script file 

bash filename

or ./filename

---

### set -euo pipefail
`set -euo pipefail` is a "strict mode" Bash declaration you put at the top of scripts to catch errors early and make them robust—perfect for your health check project.[](https://www.shkodenko.com/what-string-set-eeuo-pipefail-in-shell-script-does-mean/)

## Breakdown of Each Flag

- **`-e`**: Exit immediately if _any_ command fails (non-zero exit code). Without it, errors are ignored and the script continues.[](https://www.shkodenko.com/what-string-set-eeuo-pipefail-in-shell-script-does-mean/)​
    
- **`-u`**: Treat _unset variables_ as errors. If you reference `$foo` before setting it, the script exits—instead of silently expanding to empty.[](https://coderwall.com/p/fkfaqq/safer-bash-scripts-with-set-euxo-pipefail)​
    
- **`-o pipefail`**: In pipelines like `cmd1 | cmd2`, fail if _any_ command fails (not just the last one). Default Bash ignores early pipeline failures.[](https://gist.github.com/mikekeke/bb08b35144dc0ad5f42ab8dad6682e04)​
    

## Why Use It in Your Script

Combined, it prevents "silent failures" common in Bash: typos, missing files, bad pipes. Your health check will stop on the _first_ real problem (e.g., `df` fails on weird mount), making debugging trivial.[](https://gist.github.com/vncsna/64825d5609c146e80de8b1fd623011ca)​

**`set -euxo pipefail` produces trace output showing each command _before_ it runs, prefixed with `+`**
