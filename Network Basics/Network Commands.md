---
tags:
  - Network
  - Linux
---
## To show ip

**ifconfig**
**ip addr**
**hostname -I**  Lists only non-loopback IPs
**ipconfig** (windows)

## Traceroute

**tracert** or **traceroute**
**mtr**   **Interactive output updates live (like `top` for networks).**
**mtr -r**  report mode ,similar to normal traceroute

## Connectivity

**ping ip**
**ip route**    Displays routing table for gateway issues
**ip link show**    for link states (up/down)

**ip neighbor**    view the ARP table, this lists IP-to-MAC mappings for local network neighbors. It show **Only**known** neighbors**—devices your machine has recently communicated with directly , not all network

**arp -a**      Displays the current ARP table (cache) showing IP-to-MAC address mappings.


## Ports

**netstat  -antp**     shows all TCP connections with PIDs/programs (UDP excluded). Run it with sudo for full process details.

- **`-a`**: All connections (listening + established)
- **`-n`**: Numeric IPs/ports (no DNS lookup)
- **`-t`**: TCP only
- **`-p`**: Process ID/program name owning each socket


