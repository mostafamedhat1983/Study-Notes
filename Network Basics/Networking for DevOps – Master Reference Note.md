---
tags:
  - Network
  - Docker
  - Kubernetes
  - Linux
---
## 1. Big picture of networking for DevOps

- Networking is **how services talk to each other** across machines, containers, VPCs, and the internet.  
- As a DevOps engineer, you must understand IP, DNS, ports, routing, NAT, firewalls, VPCs, and Kubernetes networking so you can:
  - design secure, scalable architectures,
  - answer “how traffic flows” questions (e.g., from browser → LB → service → DB),
  - debug connectivity issues at any layer.

When interviewers ask:
> “What is networking for DevOps?”  
> “Explain what happens when you type `https://api.example.com`.”  
> “How do services in Kubernetes talk to each other?”

You can explain it as a **chain** from client → DNS → LB → Service → pod → DB, each secured and controlled by networking rules.

---

## 2. IP addressing basics

- **IP address** = “postal address” of a host or service (IPv4: `A.B.C.D`).  
- **Private IPs** (RFC 1918):
  - `10.0.0.0/8`  
  - `172.16.0.0/12`  
  - `192.168.0.0/16`  
- **Public IPs** = directly reachable on the internet.  
- In cloud:
  - VMs/pods get **private IPs** inside the VPC.  
  - Public IPs attach to **load balancers, NAT Gateways, or Elastic IPs** for internet access.  

Interview‑style context:
> “We use private IPs inside the VPC for security and scale, and only expose public IPs on load balancers and gateways.”

---

## 3. Subnets, CIDR, and subnetting

- A **subnet** = a smaller network inside a bigger one (e.g., `10.0.0.0/16` → `10.0.1.0/24`).  
- **Common CIDR examples**:
  - `/24` → 256 addresses, 254 usable hosts.  
  - `/25` → 128, `/26` → 64, `/27` → 32, `/28` → 16, `/29` → 8, `/30` → 4.  
- **Network ID** = first IP, **broadcast** = last IP, middle IPs = usable hosts.  
- In cloud you usually split into:
  - **public subnets** (web, LB, internet‑facing workloads),
  - **private subnets** (app, DB, internal services) for security and routing.  

This is often the **design‑interview** topic:
> “How do you design a VPC with public and private subnets?”  
You explain:
- public subnet for LB/web,  
- private subnet for app/DB,  
- route tables pointing to IGW (public) and NAT Gateway (private).

---

## 4. DNS, ports, and “how services are found”

### DNS
- **DNS** maps domain names → IPs:
  - `api.example.com` → `3.8.1.5` (or LB IP).  
  - In Kubernetes: `service.namespace.svc.cluster.local` → `ClusterIP`.  
- Changing DNS allows blue‑green, canary, or failover without changing IPs.  

Useful tools: `dig`, `nslookup`, `host`.

When asked:
> “How does a client find your service?”  
you can say:
- The domain is resolved via DNS to the LB or Ingress IP,  
- then the LB routes to the right service/pod.

### Ports
- A **port** = like a “door number” on a host (e.g., `:80`, `:443`, `:3306`).  
- One IP can have many services as long as they use different ports.  
- You expose ports via:
  - `docker run -p 8080:80` (host → container mapping),  
  - security groups / firewalls (which ports are allowed),  
  - Kubernetes Services and Ingress.  

You’ll often hear:
> “Why do we use `80` and `443`?”  
You can say:
- `80` is HTTP (unencrypted), `443` is HTTPS (TLS‑encrypted).  
- Load balancers and proxies terminate TLS on `443` and forward to internal ports (e.g., `8080`).

---

## 5. Commonly used ports (DevOps / cloud)

Know these at minimum for interview‑style questions:

- **22** – SSH (secure remote access).  
- **80** – HTTP (unsecured web traffic).  
- **443** – HTTPS (secure web traffic, TLS).  
- **53** – DNS (DNS queries over UDP/TCP).  
- **25, 587, 465** – SMTP (email sending).  
- **389, 636** – LDAP / LDAPS (directory services).  
- **3306** – MySQL.  
- **5432** – PostgreSQL.  
- **6379** – Redis.  
- **3000** – Grafana.  
- **9090** – Prometheus.  
- **9092** – Kafka.  
- **8080** – Jenkins / common web apps.  
- **9200** – Elasticsearch HTTP API.  
- **5601** – Kibana.  
- **6443** – Kubernetes API Server.  
- **2375 / 2376** – Docker Daemon API (2375 unencrypted, 2376 TLS‑enabled).  

When asked:
> “Name some ports you use in DevOps.”  
You can say:
- “I commonly use 22 for SSH, 80/443 for web, 53 for DNS, 3306/5432 for databases, 6379 for Redis, 3000/9090 for Grafana/Prometheus, and 6443 for Kubernetes API.”

Interview‑style elaboration:
> “We keep strict firewall rules: only allow SSH from trusted IPs, web ports from the internet, and DB/Redis ports from app subnets, and never expose management ports like 2375 or 6443 to the public internet.”

---

## 6. ARP, switching, and Layer 2

- A **switch** works at **Layer 2 (data‑link)** using **MAC addresses**.  
- **ARP** maps IP ↔ MAC on the same subnet:
  - Host A asks: “Who has `10.0.1.20`?”  
  - Host B replies with its MAC; the switch learns which port that MAC lives on.  
- ARP is local to a subnet; beyond that, **IP routing** takes over.  

This is relevant for:
> “How do devices find each other on the same network?”  
You can say:
- Switches forward frames based on MACs,  
- ARP helps each host discover the MAC behind an IP on the same subnet.

---

## 7. Routing, gateways, and route tables

- A **router** decides “where a packet should go next” using **routing tables**.  
- **Static route** = manually configured path:
  - “Send `10.0.2.0/24` traffic via `10.0.1.1`.”  
- **Default route** = `0.0.0.0/0` → gateway (e.g., `10.0.1.1`).  
- In cloud:
  - Each subnet has a **route table**:
    - Public subnet: `0.0.0.0/0` → **Internet Gateway (IGW)**.  
    - Private subnet: `0.0.0.0/0` → **NAT Gateway** (so instances can reach internet, but not vice versa).  

Typical interview context:
> “How do you route traffic from private subnets to the internet?”  
You can say:
- Private subnets use a route table pointing to a NAT Gateway,  
- so traffic goes out with a public IP, but the internet cannot initiate connections back to the private instances.

---

## 8. Firewalls, NAT, and cloud security

### Firewalls / security groups
- Control:
  - which ports are open (e.g., `22`, `80`, `443`, `3306`),  
  - from which IPs/CIDRs.  
- Examples:
  - AWS **Security Groups** (instance‑level, stateful).  
  - AWS **NACLs** (subnet‑level, more like classic firewall rules).  

Common interview angle:
> “How do you secure your VPC?”  
You can say:
- Use security groups with least‑privilege rules (only required ports from required CIDRs),  
- Use NACLs as an extra layer if needed,  
- Keep DBs and Redis only reachable from app subnets, not from the internet.

### NAT (Network Address Translation)
- **SNAT** (source NAT): private IP → public IP when going out (e.g., private DB using NAT Gateway).  
- **DNAT** (destination NAT): public IP/port → internal IP/port (e.g., LB → backend service).  
- Very common in:
  - private subnets needing internet access,  
  - exposing internal services via public IPs/LBs.  

You can explain:
> “NAT lets private hosts share a public IP for outbound traffic, and DNAT lets load balancers forward to internal backends without exposing them directly.”

---

## 9. VPC / cloud networking (AWS‑style view)

- **VPC** = your private network in the cloud, e.g., `10.0.0.0/16`.  
- Inside a VPC:
  - create **public subnets** (`10.0.1.0/24`) and **private subnets** (`10.0.2.0/24`),  
  - attach **route tables**:
    - public → **Internet Gateway (IGW)**,  
    - private → **NAT Gateway**,  
  - attach **Security Groups** (and optionally NACLs) for least‑privilege control.  

Typical design‑interview question:
> “How do you place web, app, and DB in a VPC?”  
You can say:
- Web/LB in public subnets,  
- App in private subnets,  
- DB in private subnets,  
- Allow web → app, app → DB, and only required ports,  
- Use NAT Gateway for app → internet and security groups for everything else.

---

## 10. Docker / container networking

- **Bridge network** (default Docker network):
  - each container gets its own IP,  
  - containers can talk to each other by IP or service name,  
  - port mapping: `docker run -p 8080:80` → host port `8080` → container port `80`.  
- Later in Kubernetes, **CNI plugins** (e.g., Calico, Cilium, Flannel) handle pod‑level bridge‑style networking at scale.  

When asked:
> “How do containers communicate?”  
You can say:
- Docker uses bridge networks with isolated IPs,  
- Kubernetes replaces that with CNI‑managed pod IPs and Services for stable endpoints.

---

## 11. Kubernetes networking (pods, services, Ingress)

### Pods
- Each **pod** gets its own IP.  
- Containers in the same pod share:
  - same IP,  
  - `localhost` communication.  

### Services
- **Service** = stable IP/DNS name for pods:
  - `ClusterIP`: internal only,  
  - `NodePort`: opens a port on each node,  
  - `LoadBalancer`: cloud LB → set of pods behind the Service.  
- Used by:
  - other pods in the cluster,  
  - external systems (via `LoadBalancer` or Ingress).  

You can expect:
> “How do pods talk to each other in Kubernetes?”  
“Where do you put your frontend/backend?”  
You answer:
- Pods have IPs; Services give stable endpoints;  
- Frontend can be a `LoadBalancer` or Ingress, backend can be `ClusterIP` only reachable from inside the cluster.

### Ingress
- **Ingress** = L7 HTTP/HTTPS entry point.  
- Routes by:
  - `host` (e.g., `api.example.com`),  
  - `path` (e.g., `/orders`, `/users`).  
- Behind the scenes:
  - Ingress controller (e.g., Nginx, Traefik, AWS ALB Ingress Controller),  
  - rules map host+path → Service → pods.  

You can explain:
> “Ingress is how external HTTP traffic reaches specific Services in Kubernetes, allowing multiple APIs or apps on the same domain with different paths.”

---

## 12. Troubleshooting flow (DevOps‑style)

When a service is unreachable, follow this flow:

1. **DNS**  
   - Can the client resolve `api.example.com`? Use `dig`, `nslookup`, `host`.

2. **Reachability (IP)**  
   - Can you ping the IP? If ICMP is blocked, try `curl` or `telnet` on the service port.

3. **Port and service**  
   - Is the port open and listening? Use `telnet`, `nc`, `curl`.  
   - Is something bound to that port? Use `ss -tlnp` or `netstat -tlnp`.

4. **Firewall / security group / NACL**  
   - Is the port allowed from the source IP/CIDR?  
   - Is NAT configured correctly for outbound traffic?

5. **LB / Ingress**  
   - Is the target registered (target group, Service endpoints)?  
   - Are health checks passing?  
   - Are Ingress rules correct (host/path)?

6. **Application logs**  
   - Once connectivity is “ok”, the problem is usually in the app or config.

This is the classic interview line:
> “When a service is unreachable, how do you debug it?”  
You walk through this **layer‑by‑layer** story.

---

## 13. Key CLI tools to know

- `ping` – basic reachability (ICMP).  
- `traceroute` / `tracert` – trace path to a host.  
- `dig` / `nslookup` / `host` – DNS queries.  
- `curl` / `wget` – test HTTP/HTTPS endpoints.  
- `telnet` / `nc` (netcat) – test if a TCP port is open.  
- `ss` / `netstat` – list listening ports and connections.  
- `tcpdump` / Wireshark – capture packets (advanced).

You can say in interviews:
> “When debugging connectivity, I start with `dig` and `ping`, then `telnet`/`curl` on the port, then `ss`/`netstat` to see if something is listening, and finally check firewalls, routing, and logs.”

---

