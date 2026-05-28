# Cilium Gateway API — Hairpin Traffic 403 "Access denied"

## Problem

Internal pods connecting to services through the Gateway API VIP (e.g. `https://bitwarden.shjem.dk`) receive HTTP 403 with body `Access denied`, while the same URL works fine from external clients.

This affects any cluster using Cilium with:
- Gateway API (or Ingress) with L7 LB
- External Envoy proxy mode (`envoy.enabled: true`)
- `policyEnforcement: default` (the default setting)
- A broad CiliumClusterwideNetworkPolicy that selects most/all pods

## Root Cause

When hairpin traffic flows through the Gateway API L7 load balancer, Cilium's external Envoy proxy enforces **two** levels of network policy:

1. **Pod policy** — the egress policy of the *source pod* (the pod making the request)
2. **Ingress policy** — the policy of the `reserved:ingress` endpoint

The critical detail is that the Pod policy check uses the **post-NAT backend destination** (e.g. `10.52.2.247:80`), not the original VIP address (`172.16.40.100:443`). This means the source pod's egress policy must explicitly allow traffic to the backend pod's identity and port.

### Why external traffic is unaffected

For external clients, the source IP doesn't match any local endpoint in the NPDS (Network Policy Discovery Service). The Pod policy check is skipped entirely, and only the Ingress policy is evaluated.

For internal (hairpin) traffic, the source IP belongs to a local pod. Envoy resolves it via BPF metadata, retrieves the pod's egress policy from NPDS, and enforces it against the backend destination.

### The typical trigger

A broad CCNP like `dns-visibility` that selects most pods but only allows limited egress:

```yaml
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: dns-visibility
spec:
  endpointSelector:
    matchExpressions:
      - key: reserved:ingress
        operator: DoesNotExist      # matches ALL non-ingress pods
  egress:
    - toEndpoints: [...]            # DNS only (port 53)
      toPorts: [...]
    - icmps: [...]                  # ICMP only
```

With `policyEnforcement: default`, once any policy selects a pod, all traffic not matching a rule is denied. Since `dns-visibility` only allows DNS and ICMP egress, any hairpin traffic to backend ports (80, 443, etc.) is denied by the L7 LB proxy — even though the pod can reach the VIP at the L3/L4 level.

## How to Diagnose

### 1. Enable Envoy filter debug logging

```bash
# Find the cilium agent pod on the affected node
kubectl -n kube-system exec <cilium-agent-pod> -c cilium-agent -- \
  cilium-dbg envoy admin logging set loggers filter=debug
```

> **Note:** The syntax is `filter=debug` (with `=`), not `filter:debug`.

### 2. Trigger a hairpin request

```bash
kubectl run curl-test --image=curlimages/curl --rm -i --restart=Never \
  --overrides='{"spec":{"nodeName":"<affected-node>"}}' \
  -- curl -sk --max-time 5 https://your-service.example.com/path
```

### 3. Check envoy logs for the denial

```bash
kubectl -n kube-system logs <cilium-envoy-pod> --since=30s | grep -E "Pod policy|DENY"
```

The smoking gun is:
```
Pod policy DENY on proxy_id: <listener_port> id: <dest_identity> port: <backend_port> sni: "..."
```

This confirms the source pod's egress policy is blocking traffic to the backend.

### 4. Reset logging when done

```bash
kubectl -n kube-system exec <cilium-agent-pod> -c cilium-agent -- \
  cilium-dbg envoy admin logging set loggers filter=info
```

## Solution

Add an egress rule to the broad CCNP that allows traffic to cluster endpoints:

```yaml
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: dns-visibility
spec:
  endpointSelector:
    matchExpressions:
      - key: reserved:ingress
        operator: DoesNotExist
  egress:
    - toEndpoints:
        - matchLabels:
            io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: ANY
          rules:
            dns:
              - matchPattern: "*"
    - icmps:
        - fields:
            - family: IPv4
              type: 8
            - family: IPv4
              type: 0
    # Required for Gateway API hairpin traffic.
    # Without this, the L7 LB denies internal pods reaching backends
    # because their egress policy doesn't cover backend ports.
    - toEntities:
        - cluster
```

### Alternative (more restrictive)

If `toEntities: [cluster]` is too broad, you can restrict to specific ports:

```yaml
    - toEntities:
        - cluster
      toPorts:
        - ports:
            - port: "80"
              protocol: TCP
            - port: "443"
              protocol: TCP
```

Or allow only traffic to the ingress identity (covers Gateway API backends):

```yaml
    - toEntities:
        - ingress
```

## Key Takeaways

1. **Cilium's L7 LB enforces source pod egress against the post-NAT destination.** This is by design but often unexpected. A pod's ability to connect to the VIP at L3/L4 does not mean the L7 proxy will forward it.

2. **Broad CCNPs with restrictive egress break hairpin routing.** Any CCNP that selects many pods (e.g. for DNS visibility) must also allow egress to the ports used by Gateway API backends.

3. **`policyEnforcement: never` bypasses the issue** because no policies are enforced at all. This is useful as a diagnostic step but not a production fix.

4. **External traffic is immune** because external source IPs have no NPDS entry, so the Pod policy check is skipped entirely.

5. **The HTTP 403 body `Access denied`** comes from Cilium's Envoy network filter (`cilium.network`), not from the upstream service. The `envoy_cilium_access_denied` counter increases with each denial.
