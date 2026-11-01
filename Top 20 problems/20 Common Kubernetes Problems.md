---
tags:
  - Kubernetes
  - Troubleshooting
---

Knowing these problems helps you isolate Kubernetes failures layer by layer.

---

## Main idea

Most Kubernetes problems involve scheduling, container startup, health probes, networking, storage, resources, or permissions. Commands below use the current namespace unless specified; replace placeholders before running.

For each problem below:
- **Why it happens** = root cause
- **Quick sign** = how to recognize it fast
- **Solution** = how to fix it
- **Useful commands** = where to start

---

## 1. kubectl cannot reach the cluster

- **Why it happens:** The context, credentials, API endpoint, or network route is wrong.
- **Quick sign:** Requests time out or fail authentication.
- **Solution:** Verify context and endpoint access, then refresh credentials through the cluster's supported method.
- **Useful commands:** `kubectl config current-context`, `kubectl config get-contexts`, `kubectl cluster-info`

---

## 2. Resources appear missing

- **Why it happens:** kubectl targets the wrong namespace or cluster.
- **Quick sign:** Expected workloads are not listed.
- **Solution:** Check context and namespace before assuming resources were deleted.
- **Useful commands:** `kubectl get pods -A`, `kubectl config view --minify`, `kubectl get pods -n <namespace>`

---

## 3. Pod stays Pending

- **Why it happens:** Scheduling constraints, resource requests, taints, or storage prevent placement.
- **Quick sign:** The Pod has no assigned node and events show FailedScheduling.
- **Solution:** Read scheduling events and fix the specific constraint rather than deleting repeatedly.
- **Useful commands:** `kubectl describe pod <pod>`, `kubectl get nodes`, `kubectl get pvc`

---

## 4. ErrImagePull or ImagePullBackOff

- **Why it happens:** Image names, tags, registry access, or pull credentials are incorrect.
- **Quick sign:** Container waiting state reports image-pull errors.
- **Solution:** Inspect events, verify the image reference, and check imagePullSecrets in the Pod's namespace.
- **Useful commands:** `kubectl describe pod <pod>`, `kubectl get pod <pod> -o yaml`

---

## 5. CrashLoopBackOff

- **Why it happens:** A container repeatedly terminates and restart backoff increases.
- **Quick sign:** Restart count rises and the waiting reason is CrashLoopBackOff.
- **Solution:** Inspect previous logs, exit state, configuration, and probes; backoff describes the retry behavior, not the root cause.
- **Useful commands:** `kubectl logs <pod> -c <container> --previous`, `kubectl describe pod <pod>`

---

## 6. CreateContainerConfigError

- **Why it happens:** Required Secrets, ConfigMaps, or keys are missing.
- **Quick sign:** The container never starts and events mention configuration references.
- **Solution:** Correct names and keys in the same namespace; do not print secret values while debugging.
- **Useful commands:** `kubectl describe pod <pod>`, `kubectl get configmap`, `kubectl get secret`

---

## 7. Pod stuck ContainerCreating

- **Why it happens:** Volume mounting, CNI setup, or container runtime operations fail.
- **Quick sign:** The container has not started and events show mount or sandbox errors.
- **Solution:** Follow the event details to storage, networking, or the affected node.
- **Useful commands:** `kubectl describe pod <pod>`, `kubectl get events --sort-by=.metadata.creationTimestamp`

---

## 8. OOMKilled container

- **Why it happens:** Memory consumption exceeds the container limit or node memory conditions cause a kill.
- **Quick sign:** Termination reason is OOMKilled.
- **Solution:** Inspect workload memory behavior, limits, and node pressure before adjusting capacity.
- **Useful commands:** `kubectl describe pod <pod>`, `kubectl top pod <pod> --containers`, `kubectl describe node <node>`

---

## 9. Liveness probe restarts a healthy-starting app

- **Why it happens:** The probe is wrong or runs before initialization completes.
- **Quick sign:** Events show failed liveness probes and container restarts.
- **Solution:** Validate the endpoint and use a startup probe for slow initialization; avoid dependency-sensitive liveness checks.
- **Useful commands:** `kubectl describe pod <pod>`, `kubectl logs <pod>`

---

## 10. Pod runs but is not Ready

- **Why it happens:** Readiness checks or readiness gates are unsatisfied.
- **Quick sign:** Running Pod has Ready=False.
- **Solution:** Check readiness conditions and app dependencies; readiness failure does not itself restart a container.
- **Useful commands:** `kubectl describe pod <pod>`, `kubectl get pod <pod> -o yaml`

---

## 11. Service has no ready backends

- **Why it happens:** Selectors match no Pods or selected Pods are not ready.
- **Quick sign:** EndpointSlices show no eligible ready endpoints.
- **Solution:** Compare Service selectors with Pod labels and inspect readiness.
- **Useful commands:** `kubectl describe service <service>`, `kubectl get pods --show-labels`, `kubectl get endpointslices -l kubernetes.io/service-name=<service>`

---

## 12. Service connects to the wrong port

- **Why it happens:** targetPort differs from the app's actual listener or named port.
- **Quick sign:** Backends exist but connections fail.
- **Solution:** Check Service port mapping and application bind address; containerPort does not start a listener.
- **Useful commands:** `kubectl get service <service> -o yaml`, `kubectl logs <pod>`, `kubectl port-forward pod/<pod> 8080:<app-port>`

---

## 13. Cluster DNS fails

- **Why it happens:** CoreDNS, Pod DNS settings, or network policy blocks resolution.
- **Quick sign:** Service IP works while its DNS name fails.
- **Solution:** Test from a suitable diagnostic Pod and inspect DNS components and policy.
- **Useful commands:** `kubectl get pods -n kube-system -l k8s-app=kube-dns`, `kubectl logs -n kube-system -l k8s-app=kube-dns --tail=100`

---

## 14. Pod-to-Pod traffic is blocked

- **Why it happens:** NetworkPolicy, CNI routing, or application listeners deny traffic.
- **Quick sign:** Connections fail despite correct endpoints.
- **Solution:** Check source egress and destination ingress policies, then routing and listeners; policy enforcement depends on the network plugin.
- **Useful commands:** `kubectl get networkpolicy -A`, `kubectl describe networkpolicy <policy>`, `kubectl get pods -o wide`

---

## 15. Ingress returns 404, 502, or 503

- **Why it happens:** Controller/class, host/path rules, backend ports, or readiness are incorrect.
- **Quick sign:** External requests fail while internal access may work.
- **Solution:** Verify an appropriate controller exists, then trace the request from rule to Service to ready Pod.
- **Useful commands:** `kubectl get ingressclass`, `kubectl describe ingress <ingress>`, `kubectl get service`

---

## 16. PVC stays Pending

- **Why it happens:** No matching volume, provisioner failure, or StorageClass constraints prevent binding; WaitForFirstConsumer may deliberately delay it.
- **Quick sign:** The claim is unbound.
- **Solution:** Inspect claim events, StorageClass behavior, provisioner health, and consuming Pod scheduling.
- **Useful commands:** `kubectl describe pvc <claim>`, `kubectl get storageclass`, `kubectl get pv`

---

## 17. Volume attachment or mount fails

- **Why it happens:** Access-mode conflicts, CSI faults, topology, or filesystem permissions block use.
- **Quick sign:** Events show FailedAttachVolume or FailedMount.
- **Solution:** Inspect Pod and CSI events; resolve attachment ownership carefully and never delete a PVC just to retry.
- **Useful commands:** `kubectl describe pod <pod>`, `kubectl describe pvc <claim>`, `kubectl get volumeattachment`

---

## 18. Node becomes NotReady or Pods are evicted

- **Why it happens:** Kubelet/runtime failure, connectivity loss, or resource pressure affects the node.
- **Quick sign:** Node conditions report NotReady, DiskPressure, or MemoryPressure.
- **Solution:** Check node conditions and host logs; plan any drain around disruption budgets and storage safety.
- **Useful commands:** `kubectl describe node <node>`, `kubectl get pods -A -o wide`, `journalctl -u kubelet`

---

## 19. Forbidden RBAC error

- **Why it happens:** The identity lacks the requested verb on the resource or namespace.
- **Quick sign:** The API returns Forbidden.
- **Solution:** Verify identity, Roles, and bindings; grant only required permissions rather than cluster-admin.
- **Useful commands:** `kubectl auth can-i get pods -n <namespace>`, `kubectl get rolebinding -n <namespace>`

---

## 20. Deployment rollout stalls

- **Why it happens:** New Pods cannot start or become ready, or rollout constraints block progress.
- **Quick sign:** Available replicas remain below the desired count.
- **Solution:** Inspect ReplicaSets and Pods; roll back only after considering configuration and database compatibility.
- **Useful commands:** `kubectl rollout status deployment/<name>`, `kubectl describe deployment <name>`, `kubectl rollout history deployment/<name>`

---

## Common mistakes

- deleting Pods before collecting logs and events
- confusing Running with Ready
- raising limits without checking workload behavior
- deleting claims to fix storage errors
- changing the wrong context or namespace

---

## Good practices

- verify context before any write
- start with describe, events, and previous logs
- trace traffic layer by layer
- set realistic requests and health probes
- protect secrets in logs and exported manifests

---

## Must memorize

```bash
kubectl config current-context
kubectl get pods -A -o wide
kubectl describe pod <pod>
kubectl logs <pod> -c <container>
kubectl logs <pod> -c <container> --previous
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl get endpointslices
kubectl describe node <node>
kubectl rollout status deployment/<name>
# Requires a working resource metrics API
kubectl top pods
```

---

## Key ideas

- Status describes a symptom; events and logs explain its cause.
- Scheduling, startup, readiness, and traffic are separate layers.
- Requests influence scheduling; limits constrain resource use.
- Storage deletion can turn an outage into data loss.
