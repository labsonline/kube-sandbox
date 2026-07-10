# Kubernetes zero-trust networking with Cilium

**Cilium, powered by eBPF, delivers the most complete zero-trust networking stack available for Kubernetes today** — combining identity-based L3–L7 policy enforcement, transparent encryption, and deep observability in a single agent. This guide walks through implementing zero-trust from the ground up, starting with standard Kubernetes NetworkPolicy wherever possible and falling back to Cilium's CRDs (CiliumNetworkPolicy, CiliumClusterwideNetworkPolicy) only where the native API falls short. The core strategy: establish a default-deny baseline, layer on least-privilege allow rules, encrypt everything in transit, and observe continuously with Hubble.

Zero-trust in Kubernetes means no pod trusts any other pod by default — every connection must be explicitly authorized, identity-verified, and encrypted. Kubernetes NetworkPolicy handles the L3/L4 foundation but cannot enforce L7 rules, cluster-wide policies, or FQDN-based egress. Cilium fills every gap.

---

## 1. Zero-trust principles mapped to Kubernetes

Zero-trust networking rests on four pillars. Each maps to specific Kubernetes and Cilium capabilities — and each has gaps that must be addressed.

**Least privilege** means every pod can only communicate with the exact services it needs, on the exact ports and protocols required. In Kubernetes, this translates to default-deny NetworkPolicies with explicit per-service allow rules. Cilium extends this to L7 granularity: not just "frontend can reach the API on port 8080," but "frontend can only `GET /api/v1/*` on the API."

**Identity-based access** replaces IP-based trust with workload identity. Kubernetes NetworkPolicy uses label selectors as a rudimentary identity system. Cilium goes further by assigning **numeric security identities** derived from pod labels, evaluated in the kernel via eBPF at wire speed. For cryptographic identity, Cilium integrates SPIFFE/SPIRE for mutual TLS authentication.

**Mutual verification** requires both sides of a connection to prove their identity. Kubernetes NetworkPolicy provides no encryption or authentication. Cilium delivers transparent WireGuard or IPsec encryption for all pod-to-pod traffic and supports mTLS via SPIFFE/SPIRE without sidecars.

**No implicit trust** means the network fabric itself is untrusted. Even pods in the same namespace, on the same node, must have explicit authorization. This requires default-deny as the baseline, continuous monitoring via Hubble, and encryption of all traffic regardless of network topology.

---

## 2. Kubernetes NetworkPolicy as the L3/L4 foundation

Standard Kubernetes NetworkPolicy (`networking.k8s.io/v1`) is the portable, CNI-agnostic starting point for zero-trust segmentation. It operates at L3/L4 — controlling traffic by IP, port, and protocol (TCP/UDP/SCTP). Every policy you can express with the native API should use it for maximum portability.

### How the API works

A NetworkPolicy selects pods via `spec.podSelector` and declares allowed `ingress` and `egress` rules. An **empty `podSelector: {}`** selects all pods in the namespace. Policies are purely **additive** — multiple policies selecting the same pod produce the union of all their allow rules. There are no deny rules in the v1 API.

The critical behavioral model: pods with **no** NetworkPolicy selecting them allow all traffic (fully open). The moment any NetworkPolicy selects a pod for a given direction (ingress or egress), that pod enters **default-deny** for that direction — only traffic matching explicit allow rules is permitted.

Each peer in `from`/`to` rules can be a `podSelector` (same namespace), a `namespaceSelector` (cross-namespace), or an `ipBlock` (CIDR range). When `podSelector` and `namespaceSelector` appear in the **same peer entry**, they combine with AND logic. When they appear as **separate list items**, they combine with OR logic — a subtle distinction that causes frequent misconfiguration.

### Where Kubernetes NetworkPolicy falls short

The native API has significant limitations that prevent a complete zero-trust implementation:

- **No L7 filtering.** Cannot filter by HTTP method, path, headers, gRPC service, or DNS query. A rule allowing port 8080 allows all HTTP traffic on that port — GET, POST, DELETE, everything.
- **No cluster-wide policies.** NetworkPolicy is namespaced. Default-deny must be applied to every namespace individually, and new namespaces are unprotected until a policy is created.
- **No FQDN-based egress.** Cannot specify "allow egress to `api.github.com`" — only CIDR blocks. External service IPs change frequently, making CIDR-based rules fragile.
- **No audit or log mode.** Policies enforce immediately with no dry-run capability. No native flow logs show what was allowed or denied.
- **No explicit deny rules.** Cannot write "deny traffic from namespace X" while allowing everything else.
- **No host or node policies.** Cannot control traffic to/from the node itself or host-networked pods.
- **No service account selectors.** Identity is limited to pod and namespace labels.
- **CNI-dependent enforcement.** Kubernetes does not enforce policies — the CNI must. Flannel, for example, silently ignores all NetworkPolicy objects.

These gaps are precisely where Cilium's CRDs become necessary.

---

## 3. Why Cilium is the right CNI for zero-trust

Cilium replaces the entire traditional networking stack (iptables, kube-proxy, sidecar proxies) with eBPF programs loaded directly into the Linux kernel. This architectural choice makes it uniquely suited for zero-trust.

**eBPF dataplane performance** eliminates iptables chain traversal. At scale, iptables evaluates rules sequentially (O(n)), while Cilium uses eBPF hash maps with **O(1) lookup times**. Policy evaluation happens at the kernel level with minimal context switching, enabling thousands of fine-grained rules without measurable latency overhead.

**Identity-based enforcement** is Cilium's defining feature. Rather than matching packets by IP address, Cilium assigns each pod a numeric security identity derived from its Kubernetes labels. These identities are encoded in packet metadata and evaluated by eBPF programs in the datapath. When a pod is rescheduled to a new node with a new IP, its identity remains the same — policies follow the workload, not the address.

**Hubble observability** is built into the Cilium agent, providing L3/L4/L7 flow visibility, policy verdict monitoring, DNS query logging, and service dependency maps — all without additional agents. This is essential for the "continuous verification" pillar of zero-trust.

**Unified policy stack** supports standard Kubernetes NetworkPolicy (fully conformant), CiliumNetworkPolicy for L7/FQDN/entity rules, and CiliumClusterwideNetworkPolicy for cluster-wide baselines — all evaluated together in a single coherent model.

| Feature                | Cilium                   | Calico                      | Flannel       |
| ---------------------- | ------------------------ | --------------------------- | ------------- |
| Dataplane              | eBPF (native)            | iptables or eBPF            | VXLAN overlay |
| L7 policy              | Native (HTTP, gRPC, DNS) | Limited (add-ons)           | None          |
| Encryption             | WireGuard / IPsec        | WireGuard                   | None          |
| Observability          | Hubble (built-in, L3–L7) | Felix metrics + third-party | None          |
| Service mesh           | Sidecar-free             | DaemonSet Envoy             | None          |
| NetworkPolicy support  | Full                     | Full                        | **None**      |
| kube-proxy replacement | Full eBPF                | eBPF mode                   | No            |

---

## 4. Establishing a default-deny baseline

Default-deny is the foundation of zero-trust. Without it, every pod is fully open. The implementation strategy starts with Kubernetes NetworkPolicy for per-namespace coverage, then layers on CiliumClusterwideNetworkPolicy for true cluster-wide enforcement.

### Per-namespace default-deny with Kubernetes NetworkPolicy

Apply this to every namespace to block all ingress and egress:

```yaml
# default-deny-all.yaml
# Blocks ALL ingress and egress for every pod in this namespace.
# Must be applied to each namespace individually.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production          # Change per namespace
spec:
  podSelector: {}                # Selects ALL pods in the namespace
  policyTypes:
    - Ingress
    - Egress
  # No ingress or egress rules = everything blocked
```

Immediately after applying egress deny, **DNS resolution breaks**. Every namespace needs a companion DNS allow policy:

```yaml
# allow-dns.yaml
# Restores DNS resolution after egress default-deny.
# Targets kube-dns pods in kube-system by label.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production          # Change per namespace
spec:
  podSelector: {}                # All pods in the namespace
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:           # AND: must match both selectors
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

The per-namespace approach has an operational weakness: every new namespace is unprotected until someone applies the policy. Automation via Kyverno, OPA Gatekeeper, or GitOps (ArgoCD/Flux) can auto-generate default-deny policies on namespace creation, but this is fragile.

### Cluster-wide default-deny with CiliumClusterwideNetworkPolicy

For true cluster-wide coverage, CiliumClusterwideNetworkPolicy (CCNP) is necessary. A single resource locks down every pod in the cluster — including pods in namespaces created in the future:

```yaml
# cluster-default-deny.yaml
# Blocks all ingress and egress for every pod outside kube-system.
# Allows DNS to kube-dns with DNS proxy interception for FQDN support.
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: default-deny
spec:
  description: "Cluster-wide default deny with DNS allowance"
  endpointSelector:
    matchExpressions:
      # Exclude kube-system to avoid breaking cluster services
      - key: io.kubernetes.pod.namespace
        operator: NotIn
        values:
          - kube-system
  ingress:
    - {}                         # Empty ingress rule triggers default-deny
  egress:
    # Allow only DNS to kube-dns
    - toEndpoints:
        - matchLabels:
            io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: UDP
          rules:
            dns:
              - matchPattern: "*" # Intercept all DNS for FQDN policy support
```

This CCNP achieves what would otherwise require applying the Kubernetes NetworkPolicy to every namespace manually. The `dns` rule with `matchPattern: "*"` also enables Cilium's DNS proxy, which is required for `toFQDNs` egress policies later.

**Why exclude kube-system?** CoreDNS, the Cilium agent, kube-apiserver, and other critical components run here. Overly restrictive policies on kube-system can break the entire cluster. Apply targeted allow policies to kube-system separately.

---

## 5. Identity-based segmentation

### Kubernetes-native label selectors

Standard NetworkPolicy provides identity through label selectors. This is sufficient for most L3/L4 segmentation:

```yaml
# namespace-isolation.yaml
# Allows the API pods in "backend" to receive traffic only from
# pods in the "frontend" namespace on port 8080.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-api
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: api-server
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: frontend
          podSelector:
            matchLabels:
              app: web-frontend    # AND: pods must match both selectors
      ports:
        - protocol: TCP
          port: 8080
```

For multi-tenant isolation, apply custom labels to namespaces (`team: team-a`) and use `namespaceSelector` to restrict cross-tenant communication. Kubernetes automatically labels every namespace with `kubernetes.io/metadata.name: <name>`, which is reliable for targeting specific namespaces.

### Cilium's extended identity model

Cilium goes beyond labels by assigning **numeric security identities** to each unique set of security-relevant labels. The process works as follows: when a pod starts, the Cilium agent extracts its labels, hashes the security-relevant subset, and either reuses an existing identity or allocates a new one. This numeric identity is then used in eBPF map lookups for **O(1) policy evaluation** in the kernel datapath.

Identities are stored as `CiliumIdentity` CRDs (default) or in an external etcd (for very large clusters). Multiple pods with identical labels share a single identity, so policy rules scale with the number of unique label combinations — not pod count.

Cilium extends the identity model with capabilities unavailable in native NetworkPolicy:

**Service account selectors** — Cilium automatically derives the label `io.cilium.k8s.policy.serviceaccount` from a pod's `serviceAccountName`:

```yaml
# service-account-policy.yaml
# Only pods running as service account "order-processor" can access the
# payment service. This is impossible with standard Kubernetes NetworkPolicy.
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: payment-service-access
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: payment-service
  ingress:
    - fromEndpoints:
        - matchLabels:
            io.cilium.k8s.policy.serviceaccount: order-processor
```

**Entity selectors** — pre-defined identity groups for infrastructure components:

| Entity           | Description                          | Typical use                       |
|------------------|--------------------------------------|-----------------------------------|
| `world`          | All traffic from outside the cluster | Control north-south traffic       |
| `cluster`        | All endpoints inside the cluster     | Allow intra-cluster communication |
| `host`           | The local node the pod runs on       | Allow kubelet probes              |
| `remote-node`    | All other cluster nodes              | Allow inter-node traffic          |
| `kube-apiserver` | API server endpoints                 | Allow webhook/API calls           |
| `health`         | Cilium health check endpoints        | Always allow for monitoring       |

These entities let you write policies like "deny all traffic from the internet" or "allow pods to reach the API server" without knowing specific IPs or labels — critical for infrastructure-level zero-trust controls.

---

## 6. L7 policy enforcement requires CiliumNetworkPolicy

Kubernetes NetworkPolicy cannot inspect application-layer traffic. It allows port 8080 or it doesn't — there is no way to distinguish between `GET /healthz` and `DELETE /admin/users`. For zero-trust at the application layer, CiliumNetworkPolicy with L7 rules is required.

### HTTP path and method filtering

```yaml
# l7-http-policy.yaml
# Restricts the API service to accept only specific HTTP methods and paths
# from the frontend. All other requests return 403 Forbidden.
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: api-l7-http-policy
  namespace: production
spec:
  description: "L7 HTTP path-based access control for API service"
  endpointSelector:
    matchLabels:
      app: api
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
          rules:
            http:
              # Allow read-only access to the API
              - method: "GET"
                path: "/api/v1/.*"     # Extended POSIX regex
              # Allow health checks
              - method: "GET"
                path: "/healthz"
              # Allow creating orders (but not updating or deleting)
              - method: "POST"
                path: "/api/v1/orders"
```

When traffic is redirected through Cilium's Envoy proxy for L7 inspection, non-matching requests receive an HTTP **403 Forbidden** response (not a silent drop), providing explicit feedback to clients.

### gRPC method filtering

gRPC maps to HTTP/2 POST requests, so Cilium uses the same HTTP rule syntax with gRPC-specific paths:

```yaml
# grpc-method-policy.yaml
# Restricts which gRPC methods a client can invoke on the server.
# Blocked methods return gRPC status code 7 (PERMISSION_DENIED).
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: grpc-method-filter
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: grpc-server
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: grpc-client
      toPorts:
        - ports:
            - port: "50051"
              protocol: TCP
          rules:
            http:
              - method: "POST"
                path: "/mypackage.MyService/GetResource"
              - method: "POST"
                path: "/mypackage.MyService/ListResources"
              # UpdateResource and DeleteResource are implicitly DENIED
```

Cilium returns **gRPC status code 7 (PERMISSION_DENIED)** for blocked methods rather than dropping the connection, giving clients actionable error information.

### DNS query filtering

Cilium's DNS proxy can filter which domains pods are allowed to resolve:

```yaml
# dns-filter-policy.yaml
# Restricts DNS resolution to only specific domains.
# Queries for other domains receive REFUSED or NXDOMAIN.
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: restrict-dns-queries
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: my-app
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
              - matchName: "api.example.com"
              - matchName: "postgres.production.svc.cluster.local"
              - matchPattern: "*.svc.cluster.local"
```

---

## 7. FQDN-based egress controls

Kubernetes NetworkPolicy can only restrict egress by CIDR block — useless when external APIs have dynamic IP addresses behind CDNs or load balancers. CiliumNetworkPolicy's `toFQDNs` solves this by intercepting DNS responses and dynamically populating allow rules with resolved IPs.

```yaml
# fqdn-egress.yaml
# Allows egress only to api.example.com on port 443.
# The DNS proxy intercepts resolution responses and maps the domain to IPs.
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: fqdn-egress-policy
  namespace: production
spec:
  description: "Restrict external egress to specific FQDNs"
  endpointSelector:
    matchLabels:
      app: my-app
  egress:
    # RULE 1: Allow and intercept DNS queries (REQUIRED for toFQDNs)
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
              - matchPattern: "*"  # Intercept all DNS to feed the FQDN cache

    # RULE 2: Allow HTTPS to api.example.com only
    - toFQDNs:
        - matchName: "api.example.com"
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP

    # RULE 3: Allow HTTPS to any *.amazonaws.com endpoint
    - toFQDNs:
        - matchPattern: "*.amazonaws.com"
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
```

**How it works internally:** The Cilium DNS proxy intercepts all DNS responses (that's why the DNS rule with `matchPattern: "*"` is required). When the application resolves `api.example.com`, the proxy captures the returned IP addresses and injects corresponding CIDR-based allow rules into the eBPF datapath. TTLs are respected — when a DNS record expires, the CIDR rule is removed. There is a brief window (~100ms, configurable via `--tofqdns-proxy-response-max-delay`) between DNS resolution and datapath update where the initial SYN packet may be dropped under high load.

**Critical gotcha for Alpine/musl-based containers:** Cilium's DNS proxy returns `REFUSED` for denied queries by default. musl libc (used in Alpine) interprets `REFUSED` as fatal and stops all DNS resolution. Set `--tofqdns-dns-reject-response-code=nameError` in the Cilium configuration to return `NXDOMAIN` instead.

---

## 8. mTLS and transparent encryption

Encryption in transit is a non-negotiable zero-trust requirement. Cilium provides two transparent encryption mechanisms that require zero application changes.

### WireGuard encryption (recommended)

WireGuard provides node-to-node encryption using the modern ChaCha20-Poly1305 cipher. Each node automatically generates a key pair on startup, distributes its public key via the `CiliumNode` CRD, and creates a `cilium_wg0` tunnel device.

**Enable via Helm:**

```bash
helm install cilium cilium/cilium --version 1.19.1 \
  --namespace kube-system \
  --set encryption.enabled=true \
  --set encryption.type=wireguard \
  --set encryption.nodeEncryption=true  # Also encrypts node-to-node traffic
```

**Verify:**

```bash
kubectl -n kube-system exec -ti ds/cilium -- cilium status | grep Encryption
# Encryption: Wireguard [cilium_wg0 (Pubkey: ..., Port: 51871, Peers: 3)]
```

Key rotation is automatic via WireGuard's Noise protocol — keys are renegotiated every 2 minutes. UDP port **51871** must be open between all cluster nodes. Requires Linux kernel **5.6+** with `CONFIG_WIREGUARD`. Control-plane nodes are automatically excluded from node-to-node encryption to avoid bootstrap issues.

### IPsec encryption (for FIPS compliance)

Choose IPsec when compliance mandates FIPS-validated cryptographic algorithms (GCM-AES-128):

```bash
# Create the IPsec key secret
kubectl create -n kube-system secret generic cilium-ipsec-keys \
  --from-literal=keys="3 rfc4106(gcm(aes)) $(dd if=/dev/urandom count=20 bs=1 2>/dev/null | xxd -p -c 64) 128"

# Install with IPsec
helm install cilium cilium/cilium --version 1.19.1 \
  --namespace kube-system \
  --set encryption.enabled=true \
  --set encryption.type=ipsec \
  --set encryption.nodeEncryption=true
```

IPsec requires manual key management via Kubernetes secrets with explicit rotation. Key IDs cycle from 1–15. During rotation, old and new keys coexist for approximately 5 minutes.

| Aspect             | WireGuard              | IPsec                       |
| ------------------ | ---------------------- | --------------------------- |
| Key management     | Automatic              | Manual (K8s secrets)        |
| Performance        | Faster (modern cipher) | Slightly more overhead      |
| FIPS compliance    | No                     | Yes                         |
| Kernel requirement | 5.6+                   | Any modern kernel           |
| Configuration      | Simple Helm values     | Requires pre-created secret |

---

## 9. Cilium service mesh and SPIFFE/SPIRE identity

For the strongest zero-trust posture, encryption alone is insufficient — you need **cryptographic identity verification** via mutual TLS. Cilium's service mesh provides this without sidecar proxies.

### Sidecar-free architecture

Traditional service meshes (Istio, Linkerd) inject a sidecar proxy into every pod. Cilium eliminates this overhead: eBPF programs in the kernel handle L3/L4 traffic management, while a single per-node Envoy proxy (DaemonSet) handles L7 processing. The result is lower latency, lower resource consumption, and no sidecar lifecycle management.

### SPIFFE/SPIRE mutual authentication (beta)

Cilium integrates with SPIFFE (Secure Production Identity Framework for Everyone) and SPIRE for cryptographic workload identity. Each Cilium-managed endpoint receives a SPIFFE ID in the format `spiffe://spiffe.cilium/identity/<cilium-identity-id>`. The mTLS handshake is performed **out-of-band** by the Cilium agent — applications are completely unaware.

**Enable SPIRE with Cilium:**

```bash
helm install cilium cilium/cilium --version 1.19.1 \
  --namespace kube-system \
  --set authentication.mutual.spire.enabled=true \
  --set authentication.mutual.spire.install.enabled=true
```

**Require mutual authentication in a policy:**

```yaml
# mutual-auth-policy.yaml
# Requires cryptographic identity verification for all traffic to the
# payment service. Connections without valid SPIFFE identity are rejected.
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: require-mtls
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: payment-service
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: order-service
      authentication:
        mode: "required"         # Enforce mutual TLS authentication
```

**Current limitations (Cilium 1.19/1.20):** Mutual authentication is **beta**. It is validated only with SPIRE (other SPIFFE implementations are untested), is not compatible with ClusterMesh, and requires a PersistentVolumeClaim for the SPIRE server in production.

---

## 10. Hubble observability for policy compliance

Hubble is the observability layer that makes zero-trust operationally viable. Without visibility into what traffic is flowing, which policies are allowing or denying it, and where gaps exist, zero-trust is blind enforcement.

### Architecture and deployment

Hubble runs as three components: the **Hubble Server** embedded in each Cilium agent (collects eBPF-derived flow data), **Hubble Relay** (aggregates flows cluster-wide via gRPC), and **Hubble UI** (web-based service dependency maps).

```bash
# Enable Hubble with Relay and UI
helm upgrade cilium cilium/cilium --version 1.19.1 \
  --namespace kube-system --reuse-values \
  --set hubble.enabled=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true

# Access Hubble CLI
cilium hubble port-forward
hubble status
# Healthcheck (via localhost:4245): Ok
# Current/Max Flows: 16380/16380 (100.00%)
# Connected Nodes: 4/4
```

### Essential commands for zero-trust auditing

```bash
# See all dropped traffic (policy denials) — the most important zero-trust query
hubble observe --verdict DROPPED

# Filter drops by namespace
hubble observe --verdict DROPPED --namespace production

# Watch L7 HTTP traffic with full method/path/status
hubble observe --namespace production --protocol http

# Find failed HTTP requests (5xx errors)
hubble observe --protocol http --http-status 500-599

# Monitor DNS resolution failures
hubble observe -t l7 --protocol DNS --verdict DROPPED

# Filter by source and destination labels
hubble observe --from-label "app=frontend" --to-label "app=backend"

# JSON output for automation and alerting pipelines
hubble observe --verdict DROPPED -o json | jq '.flow.drop_reason_desc'
```

Hubble flow records include source/destination pod name, namespace, labels, security identity, IP, port, **verdict** (FORWARDED, DROPPED, AUDIT), drop reason, and L7 protocol details (HTTP method/path/status, DNS query/answer, gRPC status).

### Prometheus and Grafana integration

Enable Hubble metrics for continuous monitoring:

```bash
helm upgrade cilium cilium/cilium --version 1.19.1 \
  --namespace kube-system --reuse-values \
  --set prometheus.enabled=true \
  --set hubble.metrics.enableOpenMetrics=true \
  --set hubble.metrics.enabled="{dns,drop,tcp,flow,icmp,httpV2:exemplars=true;labelsContext=source_ip\,source_namespace\,source_workload\,destination_ip\,destination_namespace\,destination_workload\,traffic_direction}"
```

The key metric for zero-trust compliance is **`hubble_drop_total`** (broken down by reason, including "Policy denied") and **`hubble_policy_verdicts_total`** (counts of ALLOWED/DENIED by source/destination context). Set Grafana alerts on unexpected drops to detect misconfigurations or unauthorized access attempts.

---

## 11. Complete YAML reference

### Namespace isolation with Kubernetes NetworkPolicy

```yaml
# Allows the backend API to receive traffic only from the frontend
# namespace on a specific port. Pure Kubernetes-native — no Cilium CRDs.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: api-server
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: frontend
      ports:
        - protocol: TCP
          port: 8080
```

### Default-deny all ingress and egress (Kubernetes-native)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

### Allow DNS egress (Kubernetes-native)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

### L7 HTTP path-based policy (CiliumNetworkPolicy)

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: api-l7-http-filter
  namespace: production
spec:
  description: "Allow only GET /api/v1/* from frontend to API pods"
  endpointSelector:
    matchLabels:
      app: api
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
          rules:
            http:
              - method: "GET"
                path: "/api/v1/.*"
              - method: "GET"
                path: "/healthz"
```

### FQDN egress (CiliumNetworkPolicy)

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: fqdn-egress
  namespace: production
spec:
  description: "Allow egress only to api.stripe.com on 443"
  endpointSelector:
    matchLabels:
      app: payment-service
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
    - toFQDNs:
        - matchName: "api.stripe.com"
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
```

### Cluster-wide default-deny (CiliumClusterwideNetworkPolicy)

```yaml
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: cluster-default-deny
spec:
  description: "Cluster-wide default deny (except DNS), excludes kube-system"
  endpointSelector:
    matchExpressions:
      - key: io.kubernetes.pod.namespace
        operator: NotIn
        values:
          - kube-system
  ingress:
    - {}
  egress:
    - toEndpoints:
        - matchLabels:
            io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: UDP
          rules:
            dns:
              - matchPattern: "*"
```

---

## 12. Gotchas, debugging, and best practices

### The ten most common pitfalls

**1. DNS breaks after egress deny.** This is the single most frequent outage caused by network policies. Every egress-deny policy must include a DNS allow rule. Test DNS resolution immediately after applying any egress policy.

**2. AND vs OR confusion in selectors.** When `namespaceSelector` and `podSelector` appear in the *same* peer entry, they combine with AND. When they appear as *separate list items*, they combine with OR. This is a YAML indentation issue that causes silent misconfiguration.

**3. First policy triggers instant default-deny.** In Cilium's `default` enforcement mode, applying the first policy that selects a pod flips it from allow-all to default-deny for that direction. If the policy doesn't cover all necessary traffic flows, the pod loses connectivity immediately.

**4. K8s NetworkPolicy and CiliumNetworkPolicy interaction.** When both policy types select the same pod, their allow rules are unioned and deny rules (Cilium only) take precedence. This can lead to unintended allow behavior if not carefully tracked.

**5. toFQDNs without DNS rule.** `toFQDNs` requires a companion DNS egress rule in the same policy — the DNS proxy must intercept responses to learn IP mappings. Without it, the FQDN rule matches nothing.

**6. Alpine containers and DNS REFUSED.** Cilium's DNS proxy returns `REFUSED` for denied queries. musl libc treats this as fatal. Fix: `--tofqdns-dns-reject-response-code=nameError`.

**7. Host-networked pods are invisible.** Pods with `hostNetwork: true` use the node's IP and are not managed by Cilium by default. Network policies do not apply to them unless you use host-level policies via CCNP with `nodeSelector`.

**8. CIDR rules don't match managed pods.** In CiliumNetworkPolicy, `toCIDR`/`fromCIDR` rules only apply to endpoints Cilium doesn't manage. You cannot use CIDR rules to match internal pod IPs — use `toEndpoints`/`fromEndpoints` with label selectors instead.

**9. Existing connections survive policy changes.** Cilium uses eBPF conntrack. Established TCP connections may continue after a deny policy is applied. Restart pods if immediate termination is required.

**10. Forgetting new namespaces.** Without CiliumClusterwideNetworkPolicy or automation, newly created namespaces have no default-deny and are fully open.

### Graduated rollout strategy

The safest path to zero-trust follows three phases:

**Phase 1 — Observe.** Enable Hubble and monitor traffic for 1–2 weeks. Build a service dependency map using Hubble UI. Identify all legitimate communication paths. Use `hubble observe --namespace <ns>` to catalog traffic patterns per namespace.

**Phase 2 — Audit.** Apply candidate policies with Cilium's policy audit mode enabled (`--policy-audit-mode=true`). Policies are evaluated but not enforced — traffic that would be denied appears with verdict `AUDIT` in Hubble. Validate: `hubble observe --verdict AUDIT`. The `enableDefaultDeny` toggle on CiliumNetworkPolicy also enables safe first-policy deployment without triggering default-deny.

**Phase 3 — Enforce.** Disable audit mode and switch to enforcement. Apply default-deny baselines, then layer on per-service allow rules. Monitor `hubble observe --verdict DROPPED` continuously for unexpected denials.

### Debugging denied traffic step-by-step

```bash
# 1. Check Cilium health
cilium status

# 2. Find the affected endpoint
kubectl -n kube-system exec -ti ds/cilium -- cilium-dbg endpoint list

# 3. Watch for policy drops via Hubble
hubble observe --verdict DROPPED --namespace production --follow

# 4. Real-time policy verdicts inside the Cilium agent
kubectl -n kube-system exec -ti ds/cilium -- cilium-dbg monitor --type policy-verdict

# 5. Inspect which policies are applied to an endpoint
kubectl -n kube-system exec -ti ds/cilium -- cilium-dbg policy get

# 6. Check the BPF policy map for a specific endpoint
kubectl -n kube-system exec -ti ds/cilium -- cilium-dbg bpf policy get <endpoint-id>

# 7. Verify identity resolution
kubectl -n kube-system exec -ti ds/cilium -- cilium-dbg identity list

# 8. For FQDN issues, check the DNS cache
kubectl -n kube-system exec -ti ds/cilium -- cilium-dbg fqdn cache list

# 9. Run the full connectivity test suite
cilium connectivity test

# 10. Collect a full diagnostic bundle for support
cilium sysdump
```

### Decision framework: when to use which policy type

| Need                                       | Use                             | Why                                |
|--------------------------------------------|---------------------------------|------------------------------------|
| L3/L4 pod-to-pod, same or cross-namespace  | Kubernetes NetworkPolicy        | Portable, sufficient, CNI-agnostic |
| L7 HTTP/gRPC/DNS filtering                 | CiliumNetworkPolicy             | K8s NP has no L7 support           |
| FQDN-based egress                          | CiliumNetworkPolicy (`toFQDNs`) | K8s NP has no FQDN support         |
| Cluster-wide default-deny                  | CiliumClusterwideNetworkPolicy  | K8s NP is per-namespace only       |
| Service account–based policy               | CiliumNetworkPolicy             | K8s NP has no SA selectors         |
| Entity-based rules (world, kube-apiserver) | CiliumNetworkPolicy/CCNP        | K8s NP has no entity concept       |
| Explicit deny rules                        | CiliumNetworkPolicy/CCNP        | K8s NP is allow-only               |
| Node/host firewall                         | CiliumClusterwideNetworkPolicy  | K8s NP cannot target nodes         |

## Conclusion

Implementing zero-trust in Kubernetes is a layered process, not a single configuration change. The most effective approach starts with Kubernetes-native NetworkPolicy for L3/L4 segmentation — it's portable, well-understood, and sufficient for basic isolation. CiliumClusterwideNetworkPolicy then fills the critical gap of cluster-wide default-deny, eliminating the per-namespace management burden. CiliumNetworkPolicy adds the L7 intelligence (HTTP paths, gRPC methods, DNS filtering, FQDN egress) that transforms coarse port-level rules into true least-privilege access. WireGuard encryption ensures all traffic is encrypted without touching application code, and SPIFFE/SPIRE integration (currently beta) adds the cryptographic identity verification that completes the zero-trust model.

The operational insight that separates successful zero-trust deployments from failed ones is the graduated rollout: observe with Hubble first, audit candidate policies second, enforce only after validating coverage. The most dangerous moment is applying the first policy to a namespace — it flips the default from allow-all to deny-all in one step. Hubble's policy verdict monitoring, combined with `enableDefaultDeny: false` for safe first policies, makes this transition manageable. Treat network policies as code, store them in Git, validate them in CI, and monitor their effects continuously.
