---
tags:
  - Linux
---
## SELinux, which stands for Security-Enhanced Linux, is a security module built into the Linux kernel

### The key concept behind SELinux is Mandatory Access Control (MAC). Here’s how it differs from the standard Linux security model:

  

   * Standard Linux (Discretionary Access Control - DAC): Access is based on user and group ownership. If you own a file, you can change its permissions and decide who gets to see or modify it. The root user is all-powerful and can access anything.

   * SELinux (Mandatory Access Control - MAC): Access is determined by a system-wide security policy created by the administrator. Every single process, file, and port is assigned a security
     "label" or "context." SELinux then enforces rules about which labels can interact with which other labels. Access is denied by default and only allowed if a specific policy rule permits it. Crucially, these rules apply to all users, including root.

  SElinux can operate in three modes:
   1. Enforcing: (Default) The security policy is actively enforced. Any action that violates the policy is blocked and logged.
   2. Permissive: The policy is not enforced, but any violations are logged. This is useful for troubleshooting or developing new policies without breaking functionality.
   3. Disabled: SELinux is completely turned off.

### Install

sudo yum install selinux-policy selinux-policy-targeted policycoreutils policycoreutils-python-utils setools setroubleshoot

### disable

sudo vi /etc/selinux/config

SELINUX=disabled 

sudo reboot

sestatus