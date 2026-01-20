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