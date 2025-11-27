---
tags:
  - Linux
---
dnf (Dandified YUM) is the next-generation version of yum (Yellowdog Updater, Modified) and has largely replaced it in
  modern RPM-based Linux distributions, such as:

   * Fedora (since version 18, became default in 22)
   * CentOS Stream
   * Red Hat Enterprise Linux (RHEL) (since RHEL 8)
   * AlmaLinux
   * Rocky Linux

  While they are not exactly the same internally, dnf aims to be compatible with yum's command-line interface, so many
  yum commands work with dnf without modification. In fact, on many systems where dnf is the default, running yum often
  executes dnf as an alias for backward compatibility.

 Key differences and reasons for the switch to `dnf`:

   * Improved Performance: dnf offers better performance for dependency resolution and package management operations.
   * Stronger Dependency Resolution: It uses a more advanced dependency resolver (libsolv) that is generally more
     reliable and handles complex dependency chains better.
   * Modern API: dnf has a well-defined Python API for extensions and integrations, making it easier for developers.