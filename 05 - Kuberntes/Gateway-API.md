# Gateway API and Choosing an Implementation

## Overview
Gateway API is the newer Kubernetes networking model for exposing traffic into a cluster, and Kubernetes documentation positions it as a more expressive direction than legacy Ingress.[cite:56][cite:59] It separates the standard API objects from the actual controller that enforces them, which makes the design more portable across environments.[cite:58][cite:65]

## Core idea
The easiest way to think about it is:
- **Gateway API** = the Kubernetes standard objects you write.[cite:56][cite:58]
- **Implementation** = the controller/product that reads those objects and actually routes traffic.[cite:60][cite:65]

Gateway API itself does not route traffic on its own; it needs an implementation such as Envoy Gateway, Traefik, Kong, Istio, Cilium, HAProxy, or a cloud-managed option.[cite:60][cite:65]

## Main resources
### GatewayClass
`GatewayClass` defines which implementation should manage a Gateway, similar to choosing a storage backend with `StorageClass`.[cite:58][cite:65]

### Gateway
`Gateway` defines the entry point that listens for traffic, such as port 80 or 443, protocol, hostname, and TLS behavior.[cite:58][cite:60]

### Routes
Route resources such as `HTTPRoute` define how traffic is matched and forwarded to backend Services based on hostnames, paths, headers, and filters.[cite:58][cite:60]

### Service
A Kubernetes `Service` still represents the stable backend destination for application Pods; Gateway API sits in front of Services rather than replacing them.[cite:58][cite:60]

## Quick mental model
A simple flow looks like this:

```text
Client -> Gateway -> HTTPRoute rules -> Service -> Pods
```

In older terms, what used to be packed into one Ingress object is now split into clearer responsibilities across Gateway and Route resources.[cite:58][cite:59]

## Mapping from Ingress
- **Ingress** -> roughly becomes `Gateway` + `HTTPRoute`.[cite:58][cite:59]
- **Ingress controller** -> becomes a Gateway API implementation.[cite:60][cite:65]
- **Annotations and controller-specific tricks** -> replaced by more structured API resources, though exact feature support still depends on the implementation.[cite:60][cite:65]

## Why this matters
The model separates infrastructure ownership from app routing ownership, so platform teams can manage shared edge entry points while app teams manage their own routes.[cite:58][cite:59] That makes Gateway API cleaner for multi-team platforms than older annotation-heavy Ingress setups.[cite:59][cite:62]

## Choosing an implementation
The best implementation depends on the environment and operational goals, not only on popularity.[cite:65] A practical way to choose is:

| Environment | Good fit | Reason |
|---|---|---|
| GKE | Google Gateway support [cite:70] | Native integration with GCP load balancing [cite:70] |
| Service mesh-heavy platform | Istio [cite:65] | Strong mesh policy and mTLS capabilities [cite:65] |
| Bare metal or multi-cloud | Envoy Gateway, Traefik, Kong, HAProxy, or Cilium [cite:65] | More flexibility when the edge is self-managed [cite:65] |
| Simpler developer-focused setup | Traefik [cite:62][cite:65] | Known for easier setup and developer-friendly operations [cite:62] |
| Advanced L7 policy/extensibility | Envoy Gateway or Kong [cite:60][cite:65] | Stronger gateway and policy features [cite:60][cite:65] |

## Practical takeaway
A good learning path is:
1. Learn the Gateway API resources: `GatewayClass`, `Gateway`, `HTTPRoute`, and how they connect to Services.[cite:58][cite:60]
2. Pick one implementation that matches the target environment.[cite:65]
3. Learn that implementation's installation, GatewayClass name, supported features, and operational model.[cite:65]

For a DevOps engineer, this is a more future-proof path than investing heavily in retired `ingress-nginx` behavior, because the API concepts transfer across products and platforms.[cite:56][cite:65]
