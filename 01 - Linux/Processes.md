---
tags:
  - Linux
---
Linux processes have states such as running/runnable, interruptible sleep, uninterruptible sleep, stopped, and zombie. The process tree records parent-child relationships. When a parent exits, orphaned children are reparented to an init process or a configured child subreaper.

## Monitoring commands

- `top`: interactive process and system monitor. It displays CPU and memory usage, task states, and load averages over 1, 5, and 15 minutes.
- `uptime`: displays uptime and the same load averages.
- `nproc`: reports the number of processing units available to the current process.
- `ps aux`: lists processes with resource and state information.
- `ps -ef`: lists processes with parent process IDs (PPIDs).
- `ps -eo pid,ppid,stat,comm,wchan`: useful for inspecting process states and wait channels.
- `htop`: an optional interactive process viewer.

### Understanding load average

Linux load average is not simply CPU utilization or a CPU run queue. It reflects the average number of tasks that are runnable or in uninterruptible sleep over 1, 5, and 15 minutes. Compare it with the number of available CPUs as a rough signal, while also investigating I/O wait and workload characteristics. A load average above the CPU count does not by itself prove CPU saturation.

## Common process states

- **R — Running or runnable:** executing on a CPU or waiting for CPU time.
- **S — Interruptible sleep:** waiting for an event and can generally be woken by signals.
- **D — Uninterruptible sleep:** usually waiting in the kernel, often for I/O. A signal such as SIGKILL can be sent, but the task cannot act on it until it leaves the uninterruptible wait. Investigate the underlying I/O or kernel wait; do not assume the process can be killed immediately.
- **T — Stopped:** stopped by a job-control or signal event, such as SIGSTOP.
- **Z — Zombie:** the process has exited, but its parent has not yet collected its exit status with wait(). A zombie is not running and cannot be killed again. The parent must reap it; if the parent exits, the zombie is reparented and normally reaped by init or a subreaper.

## Signals and process termination

- `kill PID` sends SIGTERM by default. It requests termination; the application may handle it, clean up, or ignore it.
- `kill -9 PID` sends SIGKILL. The process cannot catch or handle this signal, so it cannot perform application-level cleanup.
- Sending a signal to one PID does **not** automatically terminate all of its children.
- Process groups, sessions, service managers, or application-specific shutdown logic may be used when a whole workload must be stopped.

Examples:

```bash
ps -ef | grep '[h]ttpd'
kill 1420                 # request termination of PID 1420
kill -TERM 1420           # explicit SIGTERM
kill -KILL 1420           # force termination; use only when necessary
pkill -TERM -f httpd      # signal matching processes; verify the match first
```

For a systemd-managed service, prefer:

```bash
sudo systemctl stop httpd
sudo systemctl status httpd
journalctl -u httpd -f
```

Systemd's service cgroup and unit configuration determine how service processes are managed and stopped. Do not assume a generic `kill` command reproduces `systemctl stop` behavior.

## systemd operations

- `systemctl stop foo`: stop the running unit.
- `systemctl disable foo`: remove enablement links so the unit is not automatically started by the relevant boot target; it does **not** stop a unit that is already running.
- `systemctl disable --now foo`: disable and stop the unit.
- `systemctl daemon-reload`: reload unit files after changes.
- `journalctl -u foo`: inspect a unit's logs.

A service override can set resource controls, for example:

```ini
[Service]
MemoryMax=500M
CPUQuota=50%
```

After creating or changing a drop-in, run `systemctl daemon-reload` and restart the service if required.

## Troubleshooting notes

- Orphaned processes are not inherently resource leaks; they are reparented. Investigate what they do before taking action.
- Zombies consume a process-table entry but do not execute or consume normal process memory like a live process.
- A D-state task often points to a kernel wait such as blocked I/O. Investigate storage, filesystem, device, or network-filesystem health.
- SysRq stack dumps can help advanced kernel troubleshooting, but enabling or triggering SysRq should follow the distribution's operational and security guidance.
