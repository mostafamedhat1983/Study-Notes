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

While a startup probe has not succeeded, Kubernetes does not run the container's liveness or readiness probes. After startup succeeds, the configured liveness and readiness probes take over.

Use for: slow-starting applications (Java apps, apps with long initialization).

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

With `failureThreshold: 30` and `periodSeconds: 10`, the configured startup failure budget is approximately 300 seconds (timing also depends on probe execution and timeout settings).

---

## Probe types

| Type | How it works |
|---|---|
| `httpGet` | HTTP GET request — success if status 200-399 |
| `tcpSocket` | TCP connection attempt — success if port opens |
| `exec` | runs a command inside the container — success if exit code 0 |
| `grpc` | Native gRPC health check (introduced as beta in Kubernetes 1.24; stable in Kubernetes 1.27) |

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

- omitting probes where the workload needs explicit startup, readiness, or liveness handling; applications can still run without probes
- using the same endpoint for liveness and readiness (wrong — they serve different purposes)
- setting `initialDelaySeconds` too low (liveness kills container before it finishes starting)
- not using a startup probe for slow-starting apps (liveness kills them too early)
- liveness probe checks deep dependencies (DB) — this causes crash loops when DB is down
- readiness probe does not check all dependencies — traffic is sent to unready Pods

---

## Good practices

- configure only the probes that match the application's health model; a startup probe is especially useful for slow initialization
- liveness probe: check only that the app process itself is alive (no external dependencies)
- readiness probe: check whether the app can currently serve its intended requests; include critical dependencies only when their failure truly makes the instance unable to serve traffic
- use a startup probe when startup duration is variable or could exceed the liveness probe's failure budget
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
- Readiness should reflect the application's ability to serve traffic; avoid marking every Pod unready for optional or non-critical dependency failures.
