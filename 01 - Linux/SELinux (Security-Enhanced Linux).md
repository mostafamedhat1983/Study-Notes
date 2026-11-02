---
tags:
  - Linux
---
# SELinux (Security-Enhanced Linux)

SELinux is a Linux Security Module (LSM) that provides mandatory access control (MAC). It adds policy-based access decisions to the traditional Linux discretionary access control (DAC) checks.

## DAC compared with MAC

- **DAC:** traditional ownership and permission checks (user, group, mode bits, and related mechanisms). A privileged process such as root can bypass many ordinary DAC restrictions, but root is not universally unconstrained: Linux capabilities, namespaces, other LSMs, and kernel restrictions can still apply.
- **SELinux MAC:** policy evaluates security contexts (labels) and rules governing interactions between subjects (such as processes) and objects (such as files, ports, and other resources). SELinux policy can restrict even privileged processes.

An access is allowed only when the applicable Linux security checks permit it. SELinux is one layer of those checks; it does not replace DAC.

## SELinux modes

1. **Enforcing:** policy is enforced; denied actions are blocked and may be logged.
2. **Permissive:** policy decisions are logged, but SELinux does not block the actions.
3. **Disabled:** SELinux is not enforcing policy. The exact procedure and implications of changing to or from this mode are distribution-specific.

Do not assume Enforcing is the default on every Linux distribution or image. Check the actual host.

## Useful commands

```bash
getenforce
sestatus
ls -Z /path/to/file
ps -eZ
ausearch -m AVC -ts recent
```

- `getenforce` reports the current mode.
- `sestatus` reports SELinux status and policy information.
- `ls -Z` displays file security contexts.
- `ps -eZ` displays process contexts.
- `ausearch` can help locate SELinux access-vector-cache (AVC) denials when audit tooling is installed and configured.

## Troubleshooting approach

1. Confirm the current SELinux mode and policy.
2. Inspect AVC denials and identify the process, target, and denied operation.
3. Check whether the file or process has the expected context.
4. Correct labeling or policy using distribution-supported tools and documented policy mechanisms.
5. Re-test in Enforcing mode.

Do not solve an access problem by blindly disabling SELinux or generating and applying an allow policy for every denial. A denial may indicate incorrect labeling, an application configuration problem, or a real policy gap.

## Configuration warning

The location and supported method for changing SELinux mode depend on the distribution and release. On systems using `/etc/selinux/config`, changing the configured mode may require a reboot, and disabling SELinux can have relabeling and policy implications. Follow the official documentation for the specific distribution before changing it.

**Interview takeaway:** DAC controls access using ownership and permissions; SELinux adds policy-based MAC using security contexts. SELinux can confine processes even when they run with elevated Unix privileges.
