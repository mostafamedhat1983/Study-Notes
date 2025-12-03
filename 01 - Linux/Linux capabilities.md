---
tags:
  - Linux
---
Linux capabilities are a kernel feature that split the all-powerful “root” privileges into many fine‑grained flags (capabilities) that can be independently granted to processes or executables, so you can give a program only the specific powers it needs instead of full root. [web:1][web:2]

## What Linux capabilities are

Linux used to treat UID 0 (root) as “may bypass almost all permission checks,” but capabilities decompose that into distinct privilege bits such as “bind low TCP ports,” “mount filesystems,” or “override DAC file permissions.” [web:1][web:2][web:8]  
Each thread in a process holds sets of these capabilities, and the kernel checks those sets when deciding if an operation is allowed instead of blindly trusting root. [web:1][web:4]

## Why capabilities exist

Capabilities exist to implement least privilege and to remove the need for many setuid‑root binaries, which are dangerous if they have memory‑safety or logic bugs. [web:1][web:2]  
By giving a network tool only `CAP_NET_RAW` (raw sockets) instead of full root, a compromise of that tool gives the attacker limited power rather than complete system control. [web:2][web:5]

## Capability sets (per process/thread)

Each process has several capability sets the kernel uses during checks: effective, permitted, inheritable, bounding, and ambient. [web:1][web:5][web:7]  

- Effective: bits actually used in permission checks right now. [web:1][web:5]  
- Permitted: upper bound of what can be made effective for that process. [web:1][web:4]  
- Inheritable: bits that can flow across `execve` to new programs. [web:1][web:7]  
- Bounding: global ceiling of what the process and its children can ever gain. [web:1][web:5]  
- Ambient: capabilities automatically kept across `execve` for non‑setuid binaries under certain rules (mostly for modern use cases like containers). [web:1][web:4]

## Examples of common capabilities

Capabilities are defined in `<linux/capability.h>` and current kernels expose on the order of a few dozen of them. [web:2][web:5]  
Common examples you will see in security reviews:

- `CAP_CHOWN`: change file owner regardless of normal DAC checks. [web:1][web:5]  
- `CAP_DAC_OVERRIDE`: bypass most file read/write/execute permission checks. [web:1]  
- `CAP_NET_BIND_SERVICE`: bind to ports below 1024 (e.g., 80, 443) without full root. [web:1][web:5]  
- `CAP_NET_RAW`: open raw sockets (classic for `ping` and similar tools). [web:1][web:5]  
- `CAP_SYS_ADMIN`: huge “misc admin” bucket; effectively near‑root and should be avoided whenever possible. [web:1][web:3]

## How to use capabilities (MVP)

At a minimum, you interact with capabilities in two places: processes and files. [web:1][web:2][web:8]  

- Process view: `pscap`, `getpcaps`, or `/proc/<pid>/status` (`CapEff`, `CapPrm`, etc.) show what a process currently has. [web:2][web:5]  
- File capabilities: `setcap` and `getcap` manipulate capabilities stored on executables (requires filesystem support, like ext4 with xattrs). [web:1][web:2][web:8]  

Example pattern (conceptual, not a command dump):

- Remove setuid root from a binary such as `ping`.  
- Grant only `cap_net_raw+ep` to that binary via `setcap`.  
This lets non‑root users run `ping` using only the minimal network‑socket privilege instead of inheriting full root. [web:2][web:6][web:8]

## Capabilities vs traditional root – trade‑off table

| Aspect                    | Using plain root (UID 0)                                       | Using Linux capabilities                                           |
|---------------------------|-----------------------------------------------------------------|--------------------------------------------------------------------|
| Privilege granularity     | All‑or‑nothing superuser                                       | Fine‑grained per operation (e.g., network, mount, audit) [web:1]   |
| Attack surface            | Compromise of process ⇒ full system compromise                 | Compromise ⇒ limited to granted capabilities [web:2][web:5]        |
| Implementation complexity | Operationally simple, but hard to audit safely                 | More complex to reason about sets, but easier to audit precisely   |
| Legacy compatibility      | Fully compatible with old tools                                | Some older software assumes full root and may not behave as expected [web:1][web:8] |
| Principle of least priv.  | Mostly violated                                                | Explicitly supported as a design goal [web:2][web:5]               |

## Architectural reasoning and DevOps angle

From a DevOps and container‑security perspective, capabilities are the “MVP control” for reducing container and daemon privileges without changing application code. [web:3][web:4]  
Modern runtimes (Docker, containerd, Kubernetes) run containers with a default capability set and allow you to drop or add specific bits (`CAP_NET_RAW`, `CAP_SYS_ADMIN`, etc.) in pod or container specs, which is preferred over running the container as full root. [web:3][web:4][web:8]

Pros of integrating capabilities into your designs:

- Enables principle of least privilege per service and per container. [web:2][web:3]  
- Replaces many setuid binaries with narrowly scoped privileges. [web:1][web:2]  
- Aligns with CIS benchmarks and vendor hardening guides around dropping unnecessary capabilities. [web:3][web:8]  

Cons / pitfalls:

- Misuse of broad capabilities like `CAP_SYS_ADMIN` or `CAP_SYS_MODULE` can re‑introduce near‑root power and negate the benefits. [web:1][web:3]  
- Debugging misconfigurations is non‑trivial: processes may fail in non‑obvious ways when a needed capability is missing. [web:2][web:4]  

## Security and best‑practice check

- Prefer capabilities over setuid wherever the filesystem and tooling support them. [web:1][web:2][web:8]  
- In container and Kubernetes manifests, drop all non‑required capabilities and then explicitly add back only what the workload needs; avoid `CAP_SYS_ADMIN` unless there is a well‑understood, documented reason. [web:3][web:4]  
- Monitor and audit capabilities: integrate `pscap`/`getpcaps` or equivalent checks into CI/CD and configuration audits to detect binaries or containers that hold unexpected privilege bits. [web:2][web:5]

Source of Truth (Official Doc Links)
- Linux man page `capabilities(7)`: https://man7.org/linux/man-pages/man7/capabilities.7.html [web:1]  
- Red Hat “Linux Capabilities and
