---
tags:
  - Linux
---
Linux capabilities divide many traditionally root-associated privileged operations into fine-grained permission bits that can be independently granted to processes or executables. Capabilities reduce privilege when used carefully, but they do not mean every operation available to UID 0 is represented by a single simple capability list.

## What Linux capabilities are

UID 0 retains special privilege semantics, but the kernel also checks capability sets, namespaces, and other security mechanisms. Capabilities include permissions such as binding low TCP ports, mounting filesystems, and overriding discretionary access controls. Capability sets are maintained per thread.

## Why capabilities exist

Capabilities exist to implement least privilege and to remove the need for many setuid‑root binaries, which are dangerous if they have memory‑safety or logic bugs.
By giving a network tool only `CAP_NET_RAW` (raw sockets) instead of full root, a compromise of that tool gives the attacker limited power rather than complete system control.

## Capability sets (per process/thread)

Each process has several capability sets the kernel uses during checks: effective, permitted, inheritable, bounding, and ambient.

- Effective: bits actually used in permission checks right now.
- Permitted: upper bound of what can be made effective for that process.
- Inheritable: bits that can flow across `execve` to new programs.
- Bounding: global ceiling of what the process and its children can ever gain.
- Ambient: capabilities automatically kept across `execve` for non‑setuid binaries under certain rules (mostly for modern use cases like containers).

## Examples of common capabilities

Capabilities are defined in `<linux/capability.h>` and current kernels expose on the order of a few dozen of them.
Common examples you will see in security reviews:

- `CAP_CHOWN`: change file owner regardless of normal DAC checks.
- `CAP_DAC_OVERRIDE`: bypass most file read/write/execute permission checks.
- `CAP_NET_BIND_SERVICE`: bind to ports below 1024 (e.g., 80, 443) without full root.
- `CAP_NET_RAW`: open raw sockets (classic for `ping` and similar tools).
- `CAP_SYS_ADMIN`: huge “misc admin” bucket; effectively near‑root and should be avoided whenever possible.

## How to use capabilities (MVP)

At a minimum, you interact with capabilities in two places: processes and files.

- Process view: `pscap`, `getpcaps`, or `/proc/<pid>/status` (`CapEff`, `CapPrm`, etc.) show what a process currently has.
- File capabilities: `setcap` and `getcap` manipulate capabilities stored on executables (requires filesystem support, like ext4 with xattrs).

Example pattern (conceptual, not a command dump):

- Remove setuid root from a binary such as `ping`.
- Grant only `cap_net_raw+ep` to that binary via `setcap`.
On systems where `ping` uses raw sockets and the installed binary is configured accordingly, `CAP_NET_RAW` can allow it to run without full root privileges. Some implementations use other mechanisms, so inspect the actual binary and system configuration.

## Capabilities vs traditional root – trade‑off table

| Aspect                    | Using plain root (UID 0)                                       | Using Linux capabilities                                           |
|---------------------------|-----------------------------------------------------------------|--------------------------------------------------------------------|
| Privilege granularity     | UID 0 has broad traditional privilege, subject to kernel controls | Capabilities can grant selected privileged operations          |
| Attack surface            | A privileged process can have broad impact, but other controls still apply | A process is limited by its effective capabilities and other controls |
| Implementation complexity | Operationally simple, but hard to audit safely                 | More complex to reason about sets, but easier to audit precisely   |
| Legacy compatibility      | Fully compatible with old tools                                | Some older software assumes full root and may not behave as expected |
| Principle of least priv.  | Requires additional privilege restriction and careful configuration | Supports least privilege when capability sets are scoped correctly |

## Architectural reasoning and DevOps angle

From a DevOps and container‑security perspective, capabilities are the “MVP control” for reducing container and daemon privileges without changing application code.
Modern runtimes (Docker, containerd, Kubernetes) run containers with a default capability set and allow you to drop or add specific bits (`CAP_NET_RAW`, `CAP_SYS_ADMIN`, etc.) in pod or container specs, which is preferred over running the container as full root.

Pros of integrating capabilities into your designs:

- Enables principle of least privilege per service and per container.
- Replaces many setuid binaries with narrowly scoped privileges.
- Aligns with CIS benchmarks and vendor hardening guides around dropping unnecessary capabilities.

Cons / pitfalls:

- Misuse of broad capabilities like `CAP_SYS_ADMIN` or `CAP_SYS_MODULE` can re‑introduce near‑root power and negate the benefits.
- Debugging misconfigurations is non‑trivial: processes may fail in non‑obvious ways when a needed capability is missing.

## Security and best‑practice check

- Prefer capabilities over setuid wherever the filesystem and tooling support them.
- In container and Kubernetes manifests, drop all non‑required capabilities and then explicitly add back only what the workload needs; avoid `CAP_SYS_ADMIN` unless there is a well‑understood, documented reason.
- Monitor and audit capabilities: integrate `pscap`/`getpcaps` or equivalent checks into CI/CD and configuration audits to detect binaries or containers that hold unexpected privilege bits.

Source of Truth (Official Doc Links)
- Linux man page `capabilities(7)`: https://man7.org/linux/man-pages/man7/capabilities.7.html
- Red Hat “Linux Capabilities and
