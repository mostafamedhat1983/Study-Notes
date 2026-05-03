---
tags:
  - Linux
  - Troubleshooting
---

Knowing these problems helps you diagnose and fix Linux issues faster in any environment.

---

## Main idea

Most Linux problems fall into a small number of categories:
- disk and filesystem issues
- CPU and memory pressure
- networking and DNS
- permissions and security policy
- service and process management
- time, packages, and boot

For each problem below:
- **Why it happens** = root cause
- **Quick sign** = how to recognize it fast
- **Solution** = how to fix it
- **Useful commands** = where to start

---

## 1. Disk space full

- **Why it happens:** Logs, backups, package caches, crash dumps, or temporary files grow silently until a filesystem reaches 100%.
- **Quick sign:** Writes fail with 'No space left on device' and services start behaving strangely.
- **Solution:** Use df/du to find the offender, clean safely, rotate logs, set alerts, and review retention so growth is controlled.
- **Useful commands:** `df -h, du -sh, journalctl --disk-usage, logrotate`

---

## 2. Disk shows full but files seem small

- **Why it happens:** Deleted files may still be held open by running processes, so space is not released even after cleanup.
- **Quick sign:** df is high but du does not explain the missing space.
- **Solution:** Use lsof to find deleted open files, restart the holding process if safe, and prevent repeats with log rotation and better process handling.
- **Useful commands:** `lsof +L1 or lsof | grep deleted`

---

## 3. High CPU usage

- **Why it happens:** A runaway process, bad loop, overloaded service, or inefficient query can consume cores and slow the whole host.
- **Quick sign:** Load average and response times rise while one process dominates CPU.
- **Solution:** Identify the process with top/htop/pidstat, fix the root cause, tune the workload, and add resource monitoring and limits.
- **Useful commands:** `top, htop, pidstat, ps`

---

## 4. High memory usage or OOM kills

- **Why it happens:** Applications leak memory, caches grow too much, or the host is undersized for the workload, causing the kernel OOM killer to terminate processes.
- **Quick sign:** dmesg shows OOM kills or services restart unexpectedly under memory pressure.
- **Solution:** Check free, vmstat, dmesg, and process RSS, tune app memory, add limits, reduce concurrency, or scale the machine.
- **Useful commands:** `free -m, vmstat, dmesg | grep -i oom`

---

## 5. Server is slow

- **Why it happens:** Performance issues often come from CPU, memory, disk I/O, network latency, or contention rather than one obvious failure.
- **Quick sign:** Users report slowness but the cause is not obvious at first glance.
- **Solution:** Check load, CPU steal, iowait, memory pressure, disk latency, and network health in a fixed order to isolate the bottleneck quickly.
- **Useful commands:** `uptime, top, vmstat, iostat, sar`

---

## 6. Service fails to start

- **Why it happens:** Bad configuration, missing dependency, wrong permissions, occupied ports, or invalid environment files commonly stop services from starting.
- **Quick sign:** systemctl shows failed state right after restart or reboot.
- **Solution:** Use systemctl status and journalctl, validate configs before restart, check dependencies and port conflicts, then fix the specific failure.
- **Useful commands:** `systemctl status, journalctl -xeu, ss -lntp`

---

## 7. SSH login failure

- **Why it happens:** SSHD may be down, firewall rules may block access, keys may be wrong, or account settings such as shell, expiry, or permissions may be invalid.
- **Quick sign:** SSH connections time out, refuse, or authenticate then immediately disconnect.
- **Solution:** Verify sshd status, inspect auth logs, check ~/.ssh permissions, confirm keys and port access, and test from a known-good client.
- **Useful commands:** `systemctl status sshd, journalctl -u sshd, ssh -vvv`

---

## 8. User authentication problems

- **Why it happens:** PAM, LDAP, SSSD, expired passwords, locked accounts, or broken /etc/passwd and /etc/shadow permissions can block logins.
- **Quick sign:** Users suddenly cannot log in even though the system is reachable.
- **Solution:** Review auth logs, verify account status, check NSS/PAM/SSSD configuration, and confirm system files and permissions are intact.
- **Useful commands:** `id, passwd -S, chage -l, journalctl, getent`

---

## 9. Permission denied errors

- **Why it happens:** Wrong ownership, mode bits, ACLs, mount options, or SELinux/AppArmor policy can deny access even when the file exists.
- **Quick sign:** Commands fail with 'Permission denied' despite files being present.
- **Solution:** Check ls -l, getfacl, mount options, and MAC policy status, then correct ownership, permissions, or security context carefully.
- **Useful commands:** `ls -l, chown, chmod, getfacl, mount`

---

## 10. SELinux or AppArmor denials

- **Why it happens:** Mandatory access control can block services after config changes, package changes, or custom paths that lack proper labels or rules.
- **Quick sign:** Service looks correct but access is blocked until policy or context is fixed.
- **Solution:** Read audit logs, use audit2why or equivalent tools, restore contexts, and change policy only after confirming the real requirement.
- **Useful commands:** `getenforce, ausearch, audit2why, restorecon`

---

## 11. DNS resolution failure

- **Why it happens:** Broken resolvers, bad /etc/resolv.conf, upstream DNS outages, caching issues, or search-domain mistakes prevent hostname resolution.
- **Quick sign:** Ping by IP works but hostnames fail, or some names resolve incorrectly.
- **Solution:** Test with dig/getent, verify resolver settings, inspect local caching services, and separate DNS problems from raw network problems.
- **Useful commands:** `dig, nslookup, getent hosts, resolvectl`

---

## 12. Network unreachable or packet loss

- **Why it happens:** Interface issues, bad routes, firewall rules, VLAN mistakes, or upstream network faults break connectivity.
- **Quick sign:** Server cannot reach peers, package mirrors, or upstream services.
- **Solution:** Check ip addr, ip route, firewall rules, gateway reachability, and test hop by hop with ping, traceroute, or tcpdump.
- **Useful commands:** `ip addr, ip route, ping, traceroute, tcpdump`

---

## 13. Package manager errors

- **Why it happens:** Repository metadata may be stale, mirrors unavailable, package databases locked, or dependency trees broken.
- **Quick sign:** apt/dnf/yum reports locks, broken dependencies, or unreachable repositories.
- **Solution:** Refresh metadata, verify repositories, clear caches if needed, inspect lock holders, and repair dependency issues before forcing installs.
- **Useful commands:** `apt update or dnf makecache, rpm, dpkg, repo checks`

---

## 14. Time not synchronized

- **Why it happens:** NTP/chrony may be stopped or blocked, causing clock drift that breaks TLS, Kerberos, logs, and distributed systems behavior.
- **Quick sign:** Log timestamps drift or authentication/TLS systems start failing oddly.
- **Solution:** Check timedatectl and chrony or ntpd status, confirm upstream sync, allow UDP/123 where needed, and alert on drift.
- **Useful commands:** `timedatectl, chronyc sources, ntpq -p`

---

## 15. Cron job not running

- **Why it happens:** Cron syntax errors, wrong user environment, missing PATH, non-executable scripts, or a stopped cron daemon can prevent execution.
- **Quick sign:** Scheduled jobs simply do not execute at the expected time.
- **Solution:** Check crontab syntax, service status, cron logs, script permissions, and always use full paths inside jobs.
- **Useful commands:** `crontab -l, systemctl status cron/crond, journalctl -u cron`

---

## 16. Boot failure or system not coming up

- **Why it happens:** Filesystem corruption, bad fstab entries, broken bootloader, failed updates, or initramfs issues can block boot.
- **Quick sign:** System drops into emergency mode or never reaches normal multi-user startup.
- **Solution:** Use rescue mode, inspect boot logs, test fstab entries, repair filesystems if needed, and rebuild bootloader or initramfs carefully.
- **Useful commands:** `journalctl -b, cat /etc/fstab, fsck, grub tools`

---

## 17. Filesystem mounted read-only

- **Why it happens:** Disk errors, journal problems, or kernel protection after I/O failures can remount a filesystem read-only.
- **Quick sign:** Applications suddenly cannot write although disk space looks available.
- **Solution:** Check dmesg and SMART data, inspect filesystem health, move workload off if needed, and repair during a proper maintenance window.
- **Useful commands:** `dmesg, smartctl, fsck, mount`

---

## 18. Port already in use

- **Why it happens:** A service cannot bind because another process already listens on that port or a previous instance did not exit cleanly.
- **Quick sign:** A service restart fails with bind/listen errors on startup.
- **Solution:** Use ss or lsof to identify the listener, stop or reconfigure the conflicting process, and reserve ports consistently.
- **Useful commands:** `ss -lntp, lsof -i, netstat`

---

## 19. Too many open files

- **Why it happens:** High connection counts, file descriptor leaks, or low ulimit settings exhaust available descriptors.
- **Quick sign:** Applications report file descriptor exhaustion under load.
- **Solution:** Inspect per-process FD usage, raise limits where justified, fix leaks, and monitor descriptor growth before it becomes an outage.
- **Useful commands:** `ulimit -n, lsof -p, /proc/<pid>/fd`

---

## 20. Log files grow uncontrollably

- **Why it happens:** Verbose logging, debug mode in production, failed rotation, or chatty applications can fill disks fast.
- **Quick sign:** Disk usage climbs fast, especially under /var/log or journald storage.
- **Solution:** Reduce log level, verify logrotate or journald retention, compress old logs, and set disk alerts before utilization becomes critical.
- **Useful commands:** `journalctl, logrotate -d, du -sh /var/log/*`

---

## Common mistakes

- jumping to fixes before identifying the real cause
- using `kill -9` without checking why a process is stuck
- ignoring `dmesg` and `journalctl` as first steps
- deleting files to free space without checking what is writing them
- making multiple changes at once and losing track of what fixed the issue
- not checking SELinux/AppArmor when permissions look correct
- forgetting full paths in cron jobs
- assuming all environments have the same tools installed

---

## Good practices

- always start with logs: `dmesg`, `journalctl`, `/var/log/`
- use `systemctl status` before anything else for service issues
- isolate the problem layer: hardware, OS, network, app, config
- test one change at a time
- document what you did and what changed
- set disk, memory, and CPU alerts before problems happen
- keep time synchronized on all servers
- use full paths in scripts and cron jobs

---

## Must memorize

```bash
# Disk
df -h
du -sh /*
lsof +L1

# CPU / Memory
top
free -m
dmesg | grep -i oom

# Services
systemctl status <service>
journalctl -xeu <service>

# Network
ip addr
ip route
ss -lntp
dig <hostname>

# Permissions
ls -l
getfacl
getenforce
audit2why

# Time
timedatectl
chronyc sources
```

---

## Key ideas

- Most Linux problems have a pattern: check logs, check resources, check config.
- `dmesg` and `journalctl` are your most important starting points.
- Permissions problems often have a MAC layer (SELinux/AppArmor) hidden underneath.
- Network problems should be isolated layer by layer.
- Service failures almost always explain themselves in journalctl.
- Time sync, disk space, and open file limits are easy to overlook and expensive to ignore.