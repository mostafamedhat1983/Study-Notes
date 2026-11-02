---
tags:
  - Networking
  - Troubleshooting
---

Knowing these problems helps you isolate connectivity failures layer by layer.

---

## Main idea

Most networking problems involve interface/link state, IP configuration, routing, DNS, firewall policy, NAT, or application listeners.

Commands below are primarily Linux examples. Some tools require installation or elevated permissions, and commands such as `dhclient` can change network configuration.

For each problem below:
- **Why it happens** = root cause
- **Quick sign** = how to recognize it fast
- **Solution** = how to fix it
- **Useful commands** = where to start

---

## 1. No network connectivity

- **Why it happens:** Cable, Wi-Fi, interface state, IP configuration, or upstream network is down.
- **Quick sign:** The host cannot reach its gateway or any external address.
- **Solution:** Check link state, IP address, default route, and gateway reachability in order.
- **Useful commands:** `ip link`, `ip addr`, `ip route`, `ping <gateway>`

---

## 2. No default gateway

- **Why it happens:** DHCP failed, static configuration is incomplete, or the route was removed.
- **Quick sign:** Local subnet works but external destinations do not.
- **Solution:** Verify DHCP/static settings and restore the correct default route.
- **Useful commands:** `ip route`, `ip route show default`

---

## 3. Wrong IP address or subnet mask

- **Why it happens:** DHCP assigned an unexpected address or static settings conflict with the network design.
- **Quick sign:** The host cannot reach the gateway or other hosts.
- **Solution:** Verify expected subnet, gateway, and address assignment before changing configuration.
- **Useful commands:** `ip addr`, `ip route`, `ip neigh`

---

## 4. IP address conflict

- **Why it happens:** Two devices on the same Layer 2 network use the same IP address.
- **Quick sign:** Connectivity is intermittent or ARP entries change unexpectedly.
- **Solution:** Use duplicate-address detection on the affected local link, then identify and reconfigure the conflicting device. `arping -D` sends ARP probes on the local Layer 2 segment; routers do not forward ARP broadcasts, so it cannot detect a duplicate on a different routed subnet. For conflicts across VLANs or routed networks, check DHCP/IPAM records and the relevant gateway or switch ARP tables on each segment.
- **Useful commands:** `arping -D <ip>`, `ip neigh`

---

## 5. DNS resolution fails

- **Why it happens:** Resolver configuration, DNS service, upstream connectivity, or local cache is broken.
- **Quick sign:** IP-based access works while hostname lookup fails.
- **Solution:** Test a known IP, inspect resolver settings, and query DNS servers directly.
- **Useful commands:** `ping 1.1.1.1`, `cat /etc/resolv.conf`, `dig @1.1.1.1 example.com`, `resolvectl status`

---

## 6. DNS returns wrong records

- **Why it happens:** Stale cache, incorrect zone records, split-horizon DNS, or search-domain behavior.
- **Quick sign:** A hostname resolves to an unexpected IP.
- **Solution:** Query authoritative and recursive servers separately and compare expected records.
- **Useful commands:** `dig +trace example.com`, `dig @<dns-server> example.com`, `resolvectl flush-caches`

---

## 7. Port appears closed

- **Why it happens:** The service is not running, listens on another address/port, or a firewall blocks access.
- **Quick sign:** Connection refused or timeout.
- **Solution:** Verify the listener, local firewall, and network path; distinguish refused from filtered traffic.
- **Useful commands:** `ss -lntp`, `systemctl status <service>`, `nmap -p <port> <host>`

---

## 8. Firewall blocks traffic

- **Why it happens:** Host firewall, cloud security group, or network firewall denies required traffic.
- **Quick sign:** Packets are dropped or connections time out.
- **Solution:** Inspect rules at each layer and test one rule change at a time.
- **Useful commands:** `sudo nft list ruleset`, `sudo iptables -L -n -v`, `sudo ufw status verbose`

---

## 9. NAT is not working

- **Why it happens:** NAT rules, forwarding, or upstream routing are misconfigured.
- **Quick sign:** Internal hosts cannot reach external networks.
- **Solution:** Verify forwarding, NAT rules, and upstream route; test from both source and gateway.
- **Useful commands:** `sysctl net.ipv4.ip_forward`, `sudo nft list ruleset`, `tcpdump -ni any`

---

## 10. Routing loop or black hole

- **Why it happens:** Overlapping routes, incorrect next hops, or asymmetric routing create loops or dropped packets.
- **Quick sign:** Traceroute repeats hops or packets disappear.
- **Solution:** Trace the path from both directions and correct route tables and next hops.
- **Useful commands:** `ip route`, `traceroute <host>`, `mtr <host>`

---

## 11. High latency

- **Why it happens:** Congestion, wireless interference, long path, overloaded device, or application delay.
- **Quick sign:** Response time increases while connectivity remains available.
- **Solution:** Compare path latency and packet loss, then isolate network delay from application processing time.
- **Useful commands:** `ping -c 10 <host>`, `mtr <host>`, `traceroute <host>`

---

## 12. Packet loss

- **Why it happens:** Faulty link, wireless interference, congestion, duplex mismatch, or device failure.
- **Quick sign:** Intermittent application errors and lost ping replies.
- **Solution:** Test path segments, inspect interface errors, and replace or repair the failing component.
- **Useful commands:** `mtr <host>`, `ip -s link`, `ethtool <interface>`

---

## 13. MTU mismatch

- **Why it happens:** Tunneling, VPN, or overlay networks reduce the effective MTU.
- **Quick sign:** Small packets work but large transfers or TLS handshakes stall.
- **Solution:** Discover path MTU and adjust MTU/MSS consistently across the path.
- **Useful commands:** `ping -M do -s <size> <host>`, `ip link`, `tracepath <host>`

---

## 14. VLAN tagging mismatch

- **Why it happens:** Switch port, subinterface, or bridge VLAN configuration differs.
- **Quick sign:** A VLAN cannot obtain an address or reach its gateway.
- **Solution:** Verify allowed VLANs, native VLAN, and subinterface tagging on both ends.
- **Useful commands:** `ip -d link show`, `bridge vlan show`, switch-port configuration

---

## 15. DHCP fails

- **Why it happens:** No DHCP server is reachable, the VLAN is wrong, or address pool/relay configuration is broken.
- **Quick sign:** Interface has no IPv4 address or uses an APIPA address.
- **Solution:** Verify link, VLAN, DHCP server reachability, and relay configuration.
- **Useful commands:** `ip addr`, `tcpdump -ni <interface> port 67 or port 68`

---

## 16. Wi-Fi keeps disconnecting

- **Why it happens:** Weak signal, interference, roaming, driver issues, or power management.
- **Quick sign:** Connection drops repeatedly or throughput collapses.
- **Solution:** Check signal quality and logs, disable aggressive power saving where supported, and test another band or access point.
- **Useful commands:** `iw dev`, `iw dev <interface> link`, `journalctl -u NetworkManager`

---

## 17. VPN tunnel is up but traffic fails

- **Why it happens:** Routes, split-tunnel policy, firewall rules, MTU, or DNS settings are wrong.
- **Quick sign:** Tunnel shows connected but destinations are unreachable.
- **Solution:** Verify tunnel routes, DNS, firewall policy, and MTU; test both tunnel and local-network paths.
- **Useful commands:** `ip route`, `ip addr`, `dig <internal-name>`, `ping <tunnel-peer>`

---

## 18. Container or VM has no network

- **Why it happens:** Bridge, veth, NAT, IP forwarding, or virtual-switch configuration is broken.
- **Quick sign:** Guest/container cannot reach host, LAN, or internet.
- **Solution:** Check interface existence, bridge membership, IP forwarding, NAT, and firewall rules.
- **Useful commands:** `ip link`, `ip addr`, `bridge link`, `sysctl net.ipv4.ip_forward`

---

## 19. TLS/HTTPS fails while TCP connects

- **Why it happens:** Certificate validation, SNI, proxy interception, clock skew, or protocol mismatch causes failure.
- **Quick sign:** TCP connects but HTTPS handshake fails.
- **Solution:** Verify time, certificate chain, hostname/SNI, and proxy behavior; do not disable certificate verification as a permanent fix.
- **Useful commands:** `openssl s_client -connect host:443 -servername host`, `curl -v https://host`, `timedatectl`

---

## 20. Intermittent connectivity

- **Why it happens:** Failing hardware, unstable wireless, DHCP lease issues, routing changes, or overloaded equipment.
- **Quick sign:** Problems appear periodically without a clear configuration change.
- **Solution:** Correlate logs with interface counters and path tests; monitor over time instead of relying on one snapshot.
- **Useful commands:** `mtr <host>`, `ip -s link`, `journalctl -u NetworkManager`, `dmesg`

---

## Common mistakes

- changing multiple network settings at once
- disabling a firewall or certificate verification without understanding the impact
- assuming DNS failure means total network failure
- ignoring interface errors and packet counters
- using broad security rules instead of specific sources and ports

---

## Good practices

- isolate layers: link, IP, routing, DNS, firewall, application
- test from both ends of the path
- document expected IP, gateway, DNS, and VLAN design
- monitor latency, loss, and interface errors
- make one change at a time and record the result

---

## Must memorize

```bash
ip link
ip addr
ip route
ip neigh
ping <gateway>
dig @1.1.1.1 example.com
ss -lntp
sudo nft list ruleset
mtr <host>
tcpdump -ni any
```

---

## Key ideas

- Start with link, address, route, then DNS.
- Connection refused usually means no listener; timeout often means filtering or routing.
- DNS problems can exist even when raw IP connectivity works.
- Path problems need evidence from both directions.
