---
tags:
  - Linux
---
Linux processes show states like running, sleeping, stopped, zombie, and uninterruptible sleep (D) in tools like `top`. Systemd (PID 1) manages the process tree, spawning children and adopting orphans when parents die.

## Monitoring Commands

**`top`** provides real-time views of CPU, RAM, load average (1/5/15-min queues), tasks (total, running, sleeping), and sorts processes by resource use—press `q` to quit. Load average rises with CPU saturation; values > CPU cores (check `nproc`) mean queuing.

**`ps aux`** lists all processes statically: PID, user, %CPU/%MEM, status (S=sleeping, R=running, Z=zombie, T=stopped, **D=uninterruptible I/O wait**), sorted by PID.[](https://www.youtube.com/watch?v=cFWPx54EsT0)

**`ps -ef`** adds parent PID (PPID) to trace hierarchy; PID 1 (systemd) parents orphans from killed services like httpd.[](https://www.youtube.com/watch?v=cFWPx54EsT0)

**`htop`** improves on top: colors, tree view (F5), mouse support, search—install if missing (`sudo apt install htop`).[](https://xtom.com/blog/top-vs-htop-linux-process-monitoring/)

## Full Process States

- **R**: Running/runnable (CPU-ready).
    
- **S**: Interruptible sleep (signal-wakeable).
    
- **D**: **Uninterruptible sleep** (I/O wait like disk; ignores `kill -9`, check `iotop` or `ps -eo stat,wchan`).
    
- **T**: Stopped/suspended (Ctrl+Z or SIGSTOP).
    
- **Z**: Zombie (dead entry; kill parent or reboot).[](https://www.cbtnuggets.com/blog/certifications/open-source/what-are-the-5-linux-process-states)
    

This flowchart shows transitions—D-state often needs I/O fixes, not kills.

## Killing Processes

**`kill PID`** (SIGTERM): Polite; closes children first.  
**`kill -9 PID`** (SIGKILL): Forceful; orphans kids to PID 1 (systemd cleans via Type=notify).

**Examples** (httpd):  
`ps -ef | grep httpd | grep -v grep` finds PIDs.  
`kill 1420` stops parent+children.  
`ps -ef | grep httpd | awk '{print $2}' | xargs kill -9` mass-kills (use cautiously).[](https://www.youtube.com/watch?v=cFWPx54EsT0)

**Better**: `systemctl stop httpd` (handles tree). Or `pkill -f httpd` (name-based).[](https://www.digitalocean.com/community/tutorials/how-to-use-systemctl-to-manage-systemd-services-and-units)

## systemd Best Practices

- **Logs**: `journalctl -u httpd -f` tails service logs.[](https://www.digitalocean.com/community/tutorials/how-to-use-systemctl-to-manage-systemd-services-and-units)
    
- **Limits**: Add to `/etc/systemd/system/httpd.service.d/override.conf`:
    
    text
    
    `[Service] MemoryMax=500M CPUQuota=50%`
    
    Then `systemctl daemon-reload && systemctl restart httpd`.
    
- **Review**: `systemctl list-unit-files --state=enabled` → disable unneeded: `systemctl disable foo`.[](https://wafaicloud.com/blog/managing-systemd-services-for-optimal-performance/)
    

**Pro tip**: Orphans (PPID=1) waste resources; zombies clog tables (not RAM). D-state signals trouble—use `echo w > /proc/sysrq-trigger` for stack traces.[](https://www.suse.com/support/kb/doc/?id=000016919)

Like systemd as family patriarch: forks kids, reparents orphans, but D-state kids stonewall signals until I/O finishes.[](https://www.youtube.com/watch?v=cFWPx54EsT0)

