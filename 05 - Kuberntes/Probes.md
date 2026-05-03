---
tags:
  - Kubernetes
---
Probes are health checks that Kubernetes runs against containers to determine if they are alive, ready to serve traffic, or still starting up.

---

## Main idea

- **Liveness probe** = is the container still alive? (restart if not)
- **Readiness probe** = is the container ready to receive traffic? (remove from Service endpoints if not)
- **Startup probe** = has the container finished starting? (give slow-starting apps more time)

---

## Liveness probe

Kubernetes restarts the container if the liveness probe fails.

Use for: detecting deadlocks, infinite loops, or frozen processes.

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 3
```

---

## Readiness probe

Kubernetes removes the Pod from Service endpoints if the readiness probe fails. The Pod is not restarted.

Use for: waiting for the app to be ready (DB connection, cache warm-up, config loading).

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
```

---

## Startup probe

Disables liveness and readiness probes until the startup probe succeeds.

Use for: slow-starting applications (Java apps, apps with long initialization).

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

This gives the app up to 30 x 10 = 300 seconds to start before liveness kicks in.

---

## Probe types

| Type | How it works |
|---|---|
| `httpGet` | HTTP GET request — success if status 200-399 |
| `tcpSocket` | TCP connection attempt — success if port opens |
| `exec` | runs a command inside the container — success if exit code 0 |
| `grpc` | gRPC health check (Kubernetes 1.24+) |

### tcpSocket example

```yaml
livenessProbe:
  tcpSocket:
    port: 5432
  initialDelaySeconds: 10
  periodSeconds: 10
```

### exec example

```yaml
livenessProbe:
  exec:
    command:
      - cat
      - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
```

---

## Key probe parameters

| Parameter | Meaning |
|---|---|
| `initialDelaySeconds` | wait before first probe after container starts |
| `periodSeconds` | how often to run the probe |
| `timeoutSeconds` | time before probe times out |
| `failureThreshold` | consecutive failures before action is taken |
| `successThreshold` | consecutive successes to be considered healthy (readiness only) |

---

## Full example with all three probes

```yaml
spec:
  containers:
    - name: app
      image: my-app:1.0
      startupProbe:
        httpGet:
          path: /healthz
          port: 8080
        failureThreshold: 30
        periodSeconds: 10
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 0
        periodSeconds: 10
        failureThreshold: 3
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        initialDelaySeconds: 0
        periodSeconds: 5
        failureThreshold: 3
```

---

## Common mistakes

- not setting probes at all (Kubernetes cannot detect unhealthy apps)
- using the same endpoint for liveness and readiness (wrong — they serve different purposes)
- setting `initialDelaySeconds` too low (liveness kills container before it finishes starting)
- not using a startup probe for slow-starting apps (liveness kills them too early)
- liveness probe checks deep dependencies (DB) — this causes crash loops when DB is down
- readiness probe does not check all dependencies — traffic is sent to unready Pods

---

## Good practices

- always configure all three probes
- liveness probe: check only that the app process itself is alive (no external dependencies)
- readiness probe: check that the app can actually serve requests (DB connection, etc.)
- use startup probe for any app that takes more than 20-30 seconds to start
- expose `/healthz` and `/ready` endpoints in your application
- tune `failureThreshold` and `periodSeconds` based on real app behavior

---

## Must memorize

```
Liveness  → container alive? → restart if fail
Readiness → ready for traffic? → remove from Service if fail
Startup   → still starting? → delays liveness/readiness until done
```

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 3
```

---

## Key ideas

- Probes are essential for production reliability.
- Liveness restarts unhealthy containers; readiness removes them from traffic.
- Startup probe prevents liveness from killing slow-starting apps.
- Liveness should not check external dependencies to avoid cascading restarts.
- Readiness should check all dependencies so unready pods get no traffic.
