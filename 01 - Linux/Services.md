---
tags:
  - Linux
---
## Managing Services in Linux

In Linux, **services** (also called daemons) are background processes that start at boot or run on demand. They are managed primarily through the **`systemctl`** command, which interacts with **systemd** to control system services.

## Common Commands

- **Check Status**
    
    - `systemctl status <service>` — Show detailed service info and logs.
        
    - `systemctl is-active <service>` — Returns “active” if running.
        
    - `systemctl is-enabled <service>` — Returns “enabled” if it starts at boot.
        
    - `systemctl is-failed <service>` — Returns “failed” if the service failed to start.
        
    - `systemctl list-units --type=service` — Lists all active services.
        
    - `systemctl list-unit-files --type=service` — Shows all installed services with their enablement state.
        
    - `systemctl --failed` — Lists all failed units.
        
- **Start & Stop Services**
    
    - `systemctl start <service>` — Starts the service immediately.
        
    - `systemctl stop <service>` — Stops the running service.
        
    - `systemctl restart <service>` — Stops and starts the service again (useful after configuration changes).
        
    - `systemctl reload <service>` — Reloads configuration without stopping the service (if supported).
        
    - `systemctl enable --now <service>` — Enables the service at boot **and** starts it immediately.
        
- **Enable or Disable at Boot**
    
    - `systemctl enable <service>` — Runs the service automatically at boot.
        
    - `systemctl disable <service>` — Prevents the service from starting at boot.
        
    - `systemctl mask <service>` — Completely blocks the service from starting (even manually); useful for disabling unwanted services.
        
    - `systemctl unmask <service>` — Removes the mask and allows the service to run again.
        
- **Other Useful Commands**
    
    - `systemctl daemon-reload` — Reloads systemd configuration after editing unit files.
        
    - `journalctl -u <service>` — Views logs for a specific service (often used after a failure).
        
    - `systemctl list-dependencies <service>` — Shows what other units this service depends on.
        
    - `systemctl show <service>` — Shows detailed properties of the service unit.
        

## Service Configuration Files

Each service has a **unit file** (usually in `/etc/systemd/system/` or `/usr/lib/systemd/system/`), typically named `<service>.service`. This file defines how the service starts, stops, and behaves.

Example path:

text

`/etc/systemd/system/multi-user.target.wants/httpd.service`

Inside, the **`ExecStart`** line specifies the command used to run the service binary.

## Quick Example

bash

`# Install and manage Apache HTTP Server 
sudo yum install httpd -y 
sudo systemctl start httpd 
sudo systemctl enable httpd 
sudo systemctl status httpd 
journalctl -u httpd`

If the server is rebooted, enabling ensures that `httpd` starts automatically on boot. Using `enable --now` would both enable it and start it in a single step.

---

