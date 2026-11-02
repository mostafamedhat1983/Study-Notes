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
**ip route**  or  **route -n**    Displays routing table for gateway issues
**ip link show**    for link states (up/down)

**ip neighbor**    view the neighbor cache (IPv4 ARP and IPv6 neighbor discovery). It shows entries currently known to the kernel, which may be reachable, stale, failed, or permanent; it is not a complete inventory of all devices on the network.

**arp -a**      Displays the current ARP table (cache) showing IP-to-MAC address mappings.


## Ports

**netstat  -antp**     shows all TCP connections with PIDs/programs (UDP excluded). Run it with sudo for full process details.

- **`-a`**: All connections (listening + established)
- **`-n`**: Numeric IPs/ports (no DNS lookup)
- **`-t`**: TCP only
- **`-p`**: Process ID/program name owning each socket


**ss -tulnp** shows all listening TCP/UDP ports with process details (PIDs/programs). Run with sudo for full process info.

- **`-t`**: TCP sockets
- **`-u`**: UDP sockets
- **`-l`**: Listening sockets only (services waiting for connections)
- **`-n`**: Numeric IPs/ports (no DNS resolution)
- **`-p`**: Process ID and program name

**telnet** tests TCP connectivity to specific ports/services


**nmap** is a network mapper/scanner for discovering hosts, open ports, services, and OS versions. Most common use is scanning open ports on targets

## DNS

**dig** and **nslookup** are DNS lookup tools—both query DNS servers to resolve domain names to IPs (and vice versa).

## Key Differences

|Tool|Detail Level|Best For|Syntax|
|---|---|---|---|
|**dig**|**Detailed** technical output|**Scripting, troubleshooting**|`dig google.com`|
|**nslookup**|**Simple** human-readable|**Quick lookups**|`nslookup google.com`|

**dig** is better for scripting because of its consistent, machine-parseable output and non-interactive design. its output is not always identical: answers, ordering, TTLs, resolver metadata, and formatting can vary with DNS data and options

**dig +trace google.com**        # Shows full DNS resolution path
**dig google.com MX**             # Mail server records
**dig +short google.com A**    # IPv4 only, script-friendly