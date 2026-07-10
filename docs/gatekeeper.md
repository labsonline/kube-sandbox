# Gatekeeper and Cilium: Kubernetes zero-trust in practice

**OPA Gatekeeper and Cilium form a two-layer zero-trust architecture where Gatekeeper prevents misconfigurations at the Kubernetes API layer and Cilium enforces network segmentation at the kernel level.** Neither tool alone provides complete coverage: Gatekeeper cannot inspect network packets, and Cilium cannot prevent a misconfigured workload from entering the cluster. Together, they close each other's blind spots. This guide provides the complete, production-ready policy set — four Gatekeeper ConstraintTemplates with their Constraints, a cluster-wide default-deny CiliumClusterwideNetworkPolicy, audit configuration, and operational procedures — to enforce zero-trust from admission through dataplane.

---

## 1. Two layers that cover each other's blind spots

The defense model operates across five distinct layers, each catching a class of violations that the others cannot. **Layer 1** is Gatekeeper admission control. When a user runs `kubectl apply`, the request hits the Kubernetes API server, passes authentication and RBAC, then enters the admission webhook chain. Gatekeeper's mutating webhook fires first (sequential, can modify objects), followed by its validating webhook (parallel, accept or reject only). A rejected request never reaches etcd — the workload never exists. This is where pod security violations, unapproved images, missing labels, and overly permissive CiliumNetworkPolicies are caught.

**Layer 2** is a single CiliumClusterwideNetworkPolicy that establishes cluster-wide default deny. Every pod that passes Gatekeeper and starts running is immediately subject to this policy, which blocks all ingress and egress except DNS resolution to kube-dns. No traffic flows until an explicit allow rule exists.

**Layer 3** consists of per-namespace CiliumNetworkPolicy and standard Kubernetes NetworkPolicy resources that grant fine-grained allow rules. These policies select endpoints by the exact labels that Gatekeeper enforces at admission (`app`, `team`, `env`), creating a closed loop where admission validation guarantees that runtime network policy selectors match correctly.

**Layer 4** is WireGuard transparent encryption. Cilium establishes WireGuard tunnels between every pair of nodes automatically — no certificate management, no application changes. All pod-to-pod traffic crossing a node boundary is encrypted at Layer 3 in the Linux kernel. **Layer 5** is Hubble observability, which taps eBPF events to provide flow logs, DNS visibility, dropped-packet reasons, and service dependency maps, all annotated with Cilium security identity metadata.

### The admission-to-enforcement pipeline

The exact order of operations from `kubectl apply` to eBPF enforcement is:

```text
kubectl apply
  → API server authentication + RBAC authorization
    → Gatekeeper mutating webhook (sequential, can modify the object)
      → OpenAPI schema validation
        → Gatekeeper validating webhook (parallel, accept/reject)
          → Persist to etcd (object now exists)
            → kube-scheduler assigns pod to node
              → kubelet creates containers + network namespace
                → Cilium agent detects endpoint
                  → Label resolution → security identity allocation
                    → eBPF program compilation + map insertion
                      → Traffic flows per policy (state: ready)
```

Gatekeeper operates entirely before the object is persisted. Cilium operates entirely after the pod is running. There is no overlap — and no gap, because Gatekeeper ensures every pod that reaches Cilium carries the labels and security context that Cilium's policies expect.

### Cluster-wide default deny

This single CiliumClusterwideNetworkPolicy implements Layer 2. It matches every endpoint outside kube-system and blocks all traffic except egress to kube-dns on UDP/53 with DNS inspection enabled:

```yaml
# Layer 2: cluster-wide default deny with DNS exception
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: default-deny                          # single policy, cluster-wide scope
  annotations:
    description: "Baseline default deny — blocks all traffic except DNS"
    owner: "platform-security"
    security.mycompany.io/wide-network-approved: "SECREVIEW-4421"
spec:
  description: "Default deny all ingress and non-DNS egress cluster-wide"
  endpointSelector:                           # selects all endpoints except kube-system
    matchExpressions:
      - key: io.kubernetes.pod.namespace
        operator: NotIn
        values:
          - kube-system
  ingress: []                                 # empty ingress list = deny all ingress
  egress:
    - toEndpoints:                            # allow only kube-dns egress
        - matchLabels:
            io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: UDP
            - port: "53"
              protocol: TCP
          rules:
            dns:
              - matchPattern: "*"             # enables Cilium DNS proxy for FQDN visibility
```

An empty `ingress: []` array means "this policy governs ingress, and no ingress is allowed." Combined with the egress section that only permits DNS, every pod starts fully isolated. Per-namespace CiliumNetworkPolicy resources then punch specific holes for legitimate traffic.

---

## 2. Replicating cluster state for cross-resource Rego rules

Gatekeeper's Rego engine can only see the object under admission review by default. To write referential constraints — policies that cross-reference other resources in the cluster — those resources must be replicated into OPA's in-memory cache via the Config resource. Once synced, resources are accessible at `data.inventory.namespace[<ns>][<apiVersion>][<kind>][<name>]` for namespaced resources and `data.inventory.cluster[<apiVersion>][<kind>][<name>]` for cluster-scoped ones.

```yaml
apiVersion: config.gatekeeper.sh/v1alpha1
kind: Config
metadata:
  name: config                                # must be named "config"
  namespace: gatekeeper-system                # must be in gatekeeper-system namespace
spec:
  sync:
    syncOnly:
      - group: ""                             # core API group
        version: "v1"
        kind: "Namespace"                     # needed for namespace label referential checks
      - group: "cilium.io"
        version: "v2"
        kind: "CiliumNetworkPolicy"           # enables referential policies on CNPs
  match:                                      # process exclusions for system namespaces
    - excludedNamespaces:
        - "kube-system"
        - "gatekeeper-system"
      processes: ["*"]                        # exclude from all processes: webhook, audit, sync
```

Syncing Namespaces enables any future Rego rule that needs to look up namespace metadata — for example, verifying that a pod's namespace carries the required `team` label before the pod itself is admitted. Syncing CiliumNetworkPolicy enables audit of existing policies and potential future referential constraints that check whether a namespace has at least one CNP. With `auditFromCache: true` in the Helm values, the audit controller evaluates only the synced resources, which reduces API server load in large clusters.

---

## 3. Locking down pod security contexts at admission

This ConstraintTemplate enforces a comprehensive pod security baseline: no privileged containers, mandatory `runAsNonRoot`, read-only root filesystems, drop-ALL capabilities with an explicit allowlist for additions, and no host namespace sharing. It covers `containers`, `initContainers`, and `ephemeralContainers` uniformly.

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8spodsecuritycontext
  annotations:
    metadata.gatekeeper.sh/title: "Pod Security Context"
    metadata.gatekeeper.sh/version: "1.0.0"
    description: >-
      Enforces pod security context requirements: no privileged containers,
      runAsNonRoot, readOnlyRootFilesystem, drop ALL capabilities, no host
      namespaces. Supports exemptImages glob and allowedCapabilities list.
spec:
  crd:
    spec:
      names:
        kind: K8sPodSecurityContext
      validation:
        openAPIV3Schema:
          type: object
          properties:
            exemptImages:                     # glob patterns for images to skip
              type: array
              items:
                type: string
            allowedCapabilities:              # capabilities permitted in add list
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8spodsecuritycontext

        import future.keywords.in

        # --- container iterators (covers all three container types) ---

        _input_containers[c] {
          c := input.review.object.spec.containers[_]
        }
        _input_containers[c] {
          c := input.review.object.spec.initContainers[_]
        }
        _input_containers[c] {
          c := input.review.object.spec.ephemeralContainers[_]
        }

        # --- image exemption via glob matching ---

        _is_exempt(container) {
          exempt := object.get(input.parameters, "exemptImages", [])
          pattern := exempt[_]
          glob.match(pattern, [], container.image)
        }

        # --- violation: privileged container ---

        violation[{"msg": msg}] {
          c := _input_containers[_]
          not _is_exempt(c)
          c.securityContext.privileged == true
          msg := sprintf(
            "DENIED: container %q is privileged; set securityContext.privileged=false",
            [c.name],
          )
        }

        # --- violation: runAsNonRoot not explicitly true ---

        violation[{"msg": msg}] {
          c := _input_containers[_]
          not _is_exempt(c)
          not c.securityContext.runAsNonRoot == true
          msg := sprintf(
            "DENIED: container %q must set securityContext.runAsNonRoot=true",
            [c.name],
          )
        }

        # --- violation: readOnlyRootFilesystem not explicitly true ---

        violation[{"msg": msg}] {
          c := _input_containers[_]
          not _is_exempt(c)
          not c.securityContext.readOnlyRootFilesystem == true
          msg := sprintf(
            "DENIED: container %q must set securityContext.readOnlyRootFilesystem=true",
            [c.name],
          )
        }

        # --- violation: capabilities.drop does not include ALL ---

        violation[{"msg": msg}] {
          c := _input_containers[_]
          not _is_exempt(c)
          sc := object.get(c, "securityContext", {})
          caps := object.get(sc, "capabilities", {})
          drop_list := object.get(caps, "drop", [])
          not "ALL" in drop_list
          msg := sprintf(
            "DENIED: container %q must include 'ALL' in securityContext.capabilities.drop",
            [c.name],
          )
        }

        # --- violation: disallowed capability in add list ---

        violation[{"msg": msg}] {
          c := _input_containers[_]
          not _is_exempt(c)
          sc := object.get(c, "securityContext", {})
          caps := object.get(sc, "capabilities", {})
          add_list := object.get(caps, "add", [])
          cap := add_list[_]
          allowed := object.get(input.parameters, "allowedCapabilities", [])
          not cap in allowed
          msg := sprintf(
            "DENIED: container %q capability %q is not in allowedCapabilities %v",
            [c.name, cap, allowed],
          )
        }

        # --- violation: hostPID ---

        violation[{"msg": msg}] {
          input.review.object.spec.hostPID == true
          msg := "DENIED: pod must not set spec.hostPID=true"
        }

        # --- violation: hostIPC ---

        violation[{"msg": msg}] {
          input.review.object.spec.hostIPC == true
          msg := "DENIED: pod must not set spec.hostIPC=true"
        }

        # --- violation: hostNetwork ---

        violation[{"msg": msg}] {
          input.review.object.spec.hostNetwork == true
          msg := "DENIED: pod must not set spec.hostNetwork=true"
        }
```

The Constraint applies this template to all Pods, excluding `kube-system` and `gatekeeper-system` where system components legitimately need elevated privileges. The `allowedCapabilities` list below permits only `NET_BIND_SERVICE` (needed by services binding to ports below 1024):

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPodSecurityContext
metadata:
  name: pod-security-context
  annotations:
    description: "Enforce baseline pod security context on all workload pods"
spec:
  enforcementAction: deny                     # reject non-compliant pods at admission
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces:
      - kube-system                           # system components need elevated privileges
      - gatekeeper-system                     # gatekeeper itself needs to run
  parameters:
    exemptImages:
      - "registry.k8s.io/pause:*"            # pause container is infrastructure
    allowedCapabilities:
      - "NET_BIND_SERVICE"                    # only capability addition permitted
```

When Gatekeeper evaluates a pod, it checks every container (including init and ephemeral) against every violation rule. A container missing `securityContext` entirely triggers violations for `runAsNonRoot`, `readOnlyRootFilesystem`, and `capabilities.drop` simultaneously — the developer receives all failures in a single response rather than playing whack-a-mole.

---

## 4. Only approved registries can run in the cluster

This template uses OPA's built-in `strings.any_prefix_match` for efficient prefix-based matching against an approved registry allowlist. Every container image — across `containers`, `initContainers`, and `ephemeralContainers` — must begin with an approved prefix.

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sapprovedregistries
  annotations:
    metadata.gatekeeper.sh/title: "Approved Container Registries"
    metadata.gatekeeper.sh/version: "1.0.0"
    description: >-
      Blocks containers whose image does not start with an approved registry
      prefix. Uses strings.any_prefix_match for efficient matching.
spec:
  crd:
    spec:
      names:
        kind: K8sApprovedRegistries
      validation:
        openAPIV3Schema:
          type: object
          properties:
            approvedRegistries:               # list of allowed registry prefixes
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sapprovedregistries

        import future.keywords.in

        # --- container iterators ---

        _all_containers[c] {
          c := input.review.object.spec.containers[_]
        }
        _all_containers[c] {
          c := input.review.object.spec.initContainers[_]
        }
        _all_containers[c] {
          c := input.review.object.spec.ephemeralContainers[_]
        }

        # --- violation: image not from approved registry ---

        violation[{"msg": msg}] {
          c := _all_containers[_]
          registries := input.parameters.approvedRegistries
          not strings.any_prefix_match(c.image, registries)
          msg := sprintf(
            "DENIED: container %q image %q is not from an approved registry; approved prefixes: %v",
            [c.name, c.image, registries],
          )
        }
```

The Constraint defines approved prefixes. **Every prefix must end with a trailing `/`** to prevent subdomain bypass attacks. Without the trailing slash, a prefix like `registry.example.com` would also match `registry.example.com.evil.com/backdoor:latest` because the malicious domain starts with the same string. The trailing slash — `registry.example.com/` — ensures only paths within the legitimate registry match:

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sApprovedRegistries
metadata:
  name: approved-registries
  annotations:
    description: "Only images from approved registries may run in the cluster"
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces:
      - kube-system
      - gatekeeper-system
  parameters:
    approvedRegistries:
      - "registry.example.com/"               # trailing / prevents subdomain bypass
      - "gcr.io/my-project/"                  # GCR project scope
      - "us-docker.pkg.dev/my-project/"       # Artifact Registry
      - "registry.k8s.io/"                    # upstream Kubernetes images
```

Images that lack a registry prefix (e.g., `nginx:latest`, which Docker resolves to `docker.io/library/nginx:latest`) will fail this check because the literal string `nginx:latest` does not start with any approved prefix. This is intentional — all images should be fully qualified in production manifests.

---

## 5. Required labels keep Cilium identities correct

This section is the critical bridge between Gatekeeper admission control and Cilium runtime enforcement. Cilium derives a **security identity** for every endpoint by collecting the pod's identity-relevant labels, serializing them into a lookup key, and mapping that key to a **numeric uint32 identity ID** stored in the cluster's identity allocator (kvstore or CRD-backed). That numeric ID is then programmed into eBPF maps on every node, where it serves as the primary key for all policy lookups. When a packet arrives at an endpoint, Cilium resolves the source identity from packet metadata (e.g., the VXLAN header for overlay networks carries the 24-bit identity) and performs an eBPF map lookup to determine if the traffic is allowed.

**When a pod is missing a label that CiliumNetworkPolicy selectors depend on, the consequences are silent and dangerous.** The pod receives a different identity than expected — one that doesn't match any `endpointSelector` in the intended allow rules. Under default deny, the pod is completely isolated with no indication of why. Under permissive policies, the pod may match an overly broad selector and receive more access than intended. Neither failure produces an error message. Cilium does not know what labels *should* exist — it only works with what it receives.

Gatekeeper solves this by making the labels mandatory at admission time. The ConstraintTemplate below validates that every resource carries specified labels whose values match regex patterns:

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
  annotations:
    metadata.gatekeeper.sh/title: "Required Labels with Regex Validation"
    metadata.gatekeeper.sh/version: "1.0.0"
    description: >-
      Requires specified labels on resources with optional regex validation
      on label values. Supports both pod and namespace targets.
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              description: "List of required labels with optional regex patterns"
              items:
                type: object
                properties:
                  key:
                    type: string
                    description: "The label key that must be present"
                  regex:
                    type: string
                    description: "Regex pattern the label value must match (optional)"
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        import future.keywords.in

        # --- violation: required label is missing ---

        violation[{"msg": msg}] {
            required := input.parameters.labels[_]
            label_key := required.key
            provided := object.get(input.review.object.metadata, "labels", {})
            not provided[label_key]
            msg := sprintf(
                "DENIED: %s/%s is missing required label %q",
                [input.review.kind.kind, input.review.object.metadata.name, label_key],
            )
        }

        # --- violation: label value does not match regex ---

        violation[{"msg": msg}] {
            required := input.parameters.labels[_]
            label_key := required.key
            required.regex != ""
            provided := object.get(input.review.object.metadata, "labels", {})
            label_value := provided[label_key]
            not re_match(required.regex, label_value)
            msg := sprintf(
                "DENIED: %s/%s label %q value %q does not match required pattern %q",
                [input.review.kind.kind, input.review.object.metadata.name,
                 label_key, label_value, required.regex],
            )
        }
```

Two Constraints reuse this single template. The first enforces pod identity labels — `app`, `team`, and `env` — with regexes that ensure values are DNS-compatible and from a controlled vocabulary for environment:

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: pod-cilium-identity-labels
  annotations:
    description: >-
      Pods must carry app, team, and env labels for Cilium identity
      derivation and CiliumNetworkPolicy selector matching.
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces:
      - kube-system
      - gatekeeper-system
  parameters:
    labels:
      - key: "app"                            # primary Cilium endpointSelector key
        regex: "^[a-z][a-z0-9-]{1,62}$"      # DNS-compatible, 2-63 chars
      - key: "team"                           # maps to CiliumNetworkPolicy namespaceSelector
        regex: "^[a-z][a-z0-9-]{2,30}$"      # team slug, 3-31 chars
      - key: "env"                            # environment discriminator
        regex: "^(production|staging|development|test)$"
```

The second Constraint enforces namespace labels. Cilium propagates namespace labels to endpoint identities under the prefix `io.cilium.k8s.namespace.labels.<key>`. A CiliumClusterwideNetworkPolicy can then select source endpoints by namespace label — for example, `io.cilium.k8s.namespace.labels.team: backend`. If the namespace is missing its `team` label, that selector silently matches nothing, and traffic is dropped under default deny with no error:

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: namespace-identity-labels
  annotations:
    description: >-
      Namespaces must carry team and environment labels. Cilium propagates
      these as io.cilium.k8s.namespace.labels.* for policy selectors.
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Namespace"]
    excludedNamespaces:
      - kube-system
      - gatekeeper-system
      - default
  parameters:
    labels:
      - key: "team"                           # becomes io.cilium.k8s.namespace.labels.team
        regex: "^[a-z][a-z0-9-]{2,30}$"
      - key: "environment"                    # becomes io.cilium.k8s.namespace.labels.environment
        regex: "^(production|staging|development|test|sandbox)$"
```

---

## 6. CiliumNetworkPolicy validation prevents overly broad rules

A CiliumNetworkPolicy with `fromEntities: ["world"]` or an empty `endpointSelector: {}` can punch a hole through the entire default-deny posture. Cilium itself does not judge whether a policy is "too broad" — it enforces exactly what it receives. Gatekeeper fills this gap by validating CiliumNetworkPolicy and CiliumClusterwideNetworkPolicy CRDs at admission time, requiring documentation annotations and blocking dangerous configurations unless explicitly approved.

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sciliumnetworkpolicyvalidation
  annotations:
    metadata.gatekeeper.sh/title: "CiliumNetworkPolicy Validation"
    metadata.gatekeeper.sh/version: "1.0.0"
    description: >-
      Validates CiliumNetworkPolicy and CiliumClusterwideNetworkPolicy CRDs:
      requires documentation annotations, spec.description, and blocks
      dangerous fromEntities/toEntities or empty endpointSelectors without
      an explicit approval annotation.
spec:
  crd:
    spec:
      names:
        kind: K8sCiliumNetworkPolicyValidation
      validation:
        openAPIV3Schema:
          type: object
          properties:
            requiredAnnotations:              # annotation keys that must be present
              type: array
              items:
                type: string
            approvalAnnotation:               # annotation key that approves dangerous rules
              type: string
            dangerousEntities:                # entity values that require approval
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sciliumnetworkpolicyvalidation

        import future.keywords.in

        # --- helper: check if approval annotation is present ---

        _has_approval {
            anno := input.review.object.metadata.annotations
            key := input.parameters.approvalAnnotation
            anno[key]
        }

        # --- helper: resolve dangerous entity set from parameters ---

        _dangerous[entity] {
            entity := input.parameters.dangerousEntities[_]
        }

        # --- violation: missing required annotation ---

        violation[{"msg": msg}] {
            required_key := input.parameters.requiredAnnotations[_]
            annotations := object.get(
                input.review.object.metadata, "annotations", {},
            )
            not annotations[required_key]
            msg := sprintf(
                "DENIED: %s %q must have annotation %q",
                [input.review.kind.kind, input.review.object.metadata.name,
                 required_key],
            )
        }

        # --- violation: missing spec.description ---

        violation[{"msg": msg}] {
            not input.review.object.spec.description
            msg := sprintf(
                "DENIED: %s %q must set spec.description",
                [input.review.kind.kind, input.review.object.metadata.name],
            )
        }

        # --- violation: dangerous fromEntities without approval ---

        violation[{"msg": msg}] {
            rule := input.review.object.spec.ingress[_]
            entity := rule.fromEntities[_]
            entity in _dangerous
            not _has_approval
            msg := sprintf(
                "DENIED: ingress.fromEntities contains %q — requires annotation %q",
                [entity, input.parameters.approvalAnnotation],
            )
        }

        # --- violation: dangerous toEntities without approval ---

        violation[{"msg": msg}] {
            rule := input.review.object.spec.egress[_]
            entity := rule.toEntities[_]
            entity in _dangerous
            not _has_approval
            msg := sprintf(
                "DENIED: egress.toEntities contains %q — requires annotation %q",
                [entity, input.parameters.approvalAnnotation],
            )
        }

        # --- violation: dangerous fromEntities in specs[] ---

        violation[{"msg": msg}] {
            spec_rule := input.review.object.specs[_]
            rule := spec_rule.ingress[_]
            entity := rule.fromEntities[_]
            entity in _dangerous
            not _has_approval
            msg := sprintf(
                "DENIED: specs[].ingress.fromEntities contains %q — requires annotation %q",
                [entity, input.parameters.approvalAnnotation],
            )
        }

        # --- violation: dangerous toEntities in specs[] ---

        violation[{"msg": msg}] {
            spec_rule := input.review.object.specs[_]
            rule := spec_rule.egress[_]
            entity := rule.toEntities[_]
            entity in _dangerous
            not _has_approval
            msg := sprintf(
                "DENIED: specs[].egress.toEntities contains %q — requires annotation %q",
                [entity, input.parameters.approvalAnnotation],
            )
        }

        # --- violation: empty endpointSelector without approval ---

        violation[{"msg": msg}] {
            selector := object.get(input.review.object.spec, "endpointSelector", {})
            _is_empty_selector(selector)
            not _has_approval
            msg := sprintf(
                "DENIED: %s %q has empty endpointSelector (matches all pods) — requires annotation %q",
                [input.review.kind.kind, input.review.object.metadata.name,
                 input.parameters.approvalAnnotation],
            )
        }

        # --- helper: detect empty selectors ---

        _is_empty_selector(selector) {
            count(selector) == 0
        }

        _is_empty_selector(selector) {
            ml := object.get(selector, "matchLabels", {})
            count(ml) == 0
            me := object.get(selector, "matchExpressions", [])
            count(me) == 0
        }
```

The Constraint targets both CiliumNetworkPolicy and CiliumClusterwideNetworkPolicy. The `dangerousEntities` parameter blocks `"world"` and `"all"` — entities that open the policy to traffic from outside the cluster or from every endpoint. The `approvalAnnotation` provides an escape hatch for legitimate use cases (e.g., the default-deny policy itself, ingress controllers) after security review:

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sCiliumNetworkPolicyValidation
metadata:
  name: cilium-policy-validation
  annotations:
    description: >-
      Validates all CiliumNetworkPolicy and CiliumClusterwideNetworkPolicy
      resources for documentation and dangerous entity usage.
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: ["cilium.io"]              # Cilium CRD API group
        kinds:
          - "CiliumNetworkPolicy"
          - "CiliumClusterwideNetworkPolicy"
  parameters:
    requiredAnnotations:
      - "description"                         # human-readable policy intent
      - "owner"                               # team or person responsible
    approvalAnnotation: "security.mycompany.io/wide-network-approved"
    dangerousEntities:
      - "world"                               # all external traffic
      - "all"                                 # every endpoint + external
```

---

## 7. Audit catches resources that predate your policies

Gatekeeper's validating webhook only evaluates resources at creation and update time. Resources that existed before a Constraint was deployed, or resources admitted during a brief webhook outage (with `failurePolicy: Ignore`), are invisible to admission control. The **audit controller** fills this gap. It runs as a separate deployment (`gatekeeper-audit`) that periodically lists all cluster resources matching each Constraint's `match` criteria and evaluates them against the Rego policy. Violations are written to each Constraint's `.status.violations` field, where they can be scraped by monitoring systems.

### Production audit Helm values

```yaml
# Helm values for gatekeeper audit configuration
auditInterval: 300                            # audit every 5 minutes (default: 60s)
constraintViolationsLimit: 50                 # max violations stored per constraint (default: 20)
auditFromCache: false                         # true = audit only synced resources; false = list from API
auditMatchKindOnly: true                      # only audit resources matching constraint kinds (faster)
auditChunkSize: 500                           # paginate API server list calls
```

Setting `auditInterval` to **300 seconds** balances timeliness against API server load. In clusters with thousands of resources, aggressive intervals cause unnecessary churn. The `constraintViolationsLimit` of **50** ensures enough violations are surfaced for triage without bloating the Constraint status object.

### Reading violations

Each Constraint object's `.status` reports the timestamp, total count, and individual violations:

```yaml
# kubectl get k8srequiredlabels pod-cilium-identity-labels -o yaml
status:
  auditTimestamp: "2026-03-06T14:30:00Z"
  totalViolations: 3
  violations:
    - enforcementAction: deny
      group: ""
      version: v1
      kind: Pod
      namespace: legacy-app
      name: worker-7f8b9c4d6-xkl2m
      message: 'DENIED: Pod/worker-7f8b9c4d6-xkl2m is missing required label "env"'
    - enforcementAction: deny
      group: ""
      version: v1
      kind: Pod
      namespace: legacy-app
      name: api-5d6f7a8b9-mn3op
      message: 'DENIED: Pod/api-5d6f7a8b9-mn3op is missing required label "team"'
```

### Prometheus metrics for monitoring

Gatekeeper exposes four metrics critical for audit and webhook observability. **`gatekeeper_violations`** is a gauge labeled by `enforcement_action` (deny, warn, dryrun) that reports the total count of audited violations across all constraints — this is the primary alert signal. **`gatekeeper_audit_duration_seconds`** is a histogram tracking how long each audit cycle takes; a sudden increase indicates the cluster has grown or a constraint is expensive to evaluate. **`gatekeeper_audit_last_run_time`** is a gauge with the Unix timestamp of the last audit start — alert if this falls more than `2 × auditInterval` behind wall clock time, which indicates the audit controller is stuck or crashed. For webhook latency, **`gatekeeper_validation_request_duration_seconds`** is a histogram labeled by `admission_status` (allow/deny) that tracks how long each admission decision takes; P99 above 500ms signals Rego evaluation is too expensive or the webhook needs more replicas.

### Safe rollout: dryrun → warn → deny

Deploy every new Constraint in three phases. Start with `enforcementAction: dryrun` — the webhook allows all requests silently, but the audit controller records violations in `.status.violations`. This reveals how many existing resources violate the new policy without any user-facing impact. After remediating existing violations, switch to `enforcementAction: warn` — the webhook still allows requests but returns a **warning header** that `kubectl` displays to the user (e.g., `Warning: DENIED: container "app" must set runAsNonRoot=true`). This trains developers to expect the policy. Finally, switch to `enforcementAction: deny` to hard-block non-compliant admissions. **Never skip the dryrun phase in a cluster with existing workloads** — doing so risks blocking legitimate workloads during audit-revealed remediation.

---

## 8. What happens when a pod is missing its identity labels

This section walks through the concrete failure mode that Gatekeeper prevents and explains the Cilium identity mechanism in detail.

### Without Gatekeeper: silent policy bypass

Consider a Deployment where a developer forgets the `app` label on the pod template. The pod is admitted without issue — the API server does not validate label completeness. Cilium's agent on the scheduled node detects the new endpoint, collects its labels (`team: payments`, `env: production` — but no `app`), and allocates a security identity for this exact label set. That identity is a different numeric ID than the one for pods with `app: payment-api, team: payments, env: production`. Every CiliumNetworkPolicy that selects `endpointSelector: matchLabels: app: payment-api` does not match this pod. Under default deny, the pod is completely isolated. Under more permissive policies, it may receive a broad fallback identity that grants unintended access. The developer sees connection timeouts in their application logs, files a networking ticket, and the platform team spends hours tracing through Hubble flows before discovering the missing label. **The feedback loop is measured in hours or days.**

### With Gatekeeper: immediate, actionable rejection

The same `kubectl apply` hits Gatekeeper's validating webhook. The `K8sRequiredLabels` Constraint matches the Pod, finds that `app` is missing, and returns:

```log
Error from server (Forbidden): admission webhook "validation.gatekeeper.sh" denied the request:
[pod-cilium-identity-labels] DENIED: Pod/payment-api-7f8b9c-xk2lm is missing required label "app"
```

The developer adds the label and re-applies. Total time to resolution: **seconds**. The pod enters the cluster with the correct identity, matches the intended CiliumNetworkPolicy selectors, and traffic flows as designed.

### How Cilium identity derivation works

Cilium collects all **identity-relevant labels** from a pod. By default, this includes all user-defined pod labels plus system labels (`io.kubernetes.pod.namespace`, `io.cilium.k8s.namespace.labels.*`, `io.cilium.k8s.policy.serviceaccount`, `io.cilium.k8s.policy.cluster`). These labels are serialized into a canonical key and compared against the cluster's identity store. If an identical label set has been seen before, the existing numeric identity is returned. If the set is new, a new identity is allocated from the cluster-local range (**256 to 65535** for single-cluster deployments). That numeric ID is written into eBPF policy maps on every node. When traffic arrives, Cilium extracts the source identity from packet metadata and performs an O(1) eBPF map lookup to determine the verdict. **A single missing label creates an entirely different identity**, and no policy written for the "correct" label set will match.

### Namespace labels in CiliumClusterwideNetworkPolicy

Cilium automatically propagates namespace labels to endpoint identities under the `io.cilium.k8s.namespace.labels.<key>` prefix. This enables CiliumClusterwideNetworkPolicy selectors to match by namespace attributes — but only when those namespace labels exist. Here is a complete example that allows pods in namespaces labeled `team=backend` to reach pods labeled `app=api`:

```yaml
# This policy relies on Gatekeeper enforcing the "team" label on namespaces.
# Without namespace-identity-labels Constraint, the fromEndpoints selector
# would silently match nothing if a namespace lacks team=backend.
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: allow-backend-to-api
  annotations:
    description: "Allow backend namespaces to reach api pods"
    owner: "platform-networking"
spec:
  description: "Ingress from team=backend namespaces to app=api endpoints"
  endpointSelector:
    matchLabels:
      app: api                                # selects destination pods
  ingress:
    - fromEndpoints:
        - matchLabels:
            io.cilium.k8s.namespace.labels.team: backend
            # Cilium resolves this against the source endpoint's identity,
            # which includes propagated namespace labels. If the source
            # namespace lacks team=backend, no identity carries this label
            # and the rule matches nothing — traffic stays blocked.
```

The `io.cilium.k8s.namespace.labels.*` prefix is only functional in **CiliumClusterwideNetworkPolicy**, not namespaced CiliumNetworkPolicy. In a namespaced CNP, Cilium implicitly scopes `fromEndpoints` to the policy's own namespace and silently ignores namespace-label selectors. This is an important operational detail: namespace-based cross-namespace policies require CCNP.

Gatekeeper's `namespace-identity-labels` Constraint ensures that every namespace carries the `team` and `environment` labels, which in turn guarantees that every pod's identity includes the corresponding `io.cilium.k8s.namespace.labels.team` and `io.cilium.k8s.namespace.labels.environment` entries. Without this enforcement, any namespace created without `team=backend` would produce pods whose identities silently fail to match the `fromEndpoints` selector above, and inter-service communication would break with no error.

---

## 9. Webhook resilience, testing, and GitOps ordering

### Webhook failurePolicy: choosing between safety and availability

Gatekeeper's validating webhook defaults to `failurePolicy: Ignore`, meaning that if the webhook is unreachable (pods crashed, network partition, OOMKilled), the API server allows requests through without validation. This is fail-open — safe for cluster availability but a security gap. Switching to `failurePolicy: Fail` makes the webhook fail-closed: if Gatekeeper is down, all CREATE and UPDATE requests matching the webhook rules are rejected. This is more secure but creates a hard dependency on Gatekeeper availability for cluster operations.

For production clusters running `failurePolicy: Fail`, three mitigations are non-negotiable. First, run **3 replicas** of the controller-manager with pod anti-affinity across nodes and a `system-cluster-critical` priority class to prevent preemption. Second, deploy a **PodDisruptionBudget** with `minAvailable: 2` to prevent voluntary disruptions from killing quorum. Third, document a **break-glass procedure**: delete the `ValidatingWebhookConfiguration` named `gatekeeper-validating-webhook-configuration` to immediately disable all Gatekeeper validation. This always works because Kubernetes does not subject webhook configuration changes to webhook admission:

```bash
# Emergency: disable Gatekeeper validation immediately
kubectl get validatingwebhookconfiguration \
  gatekeeper-validating-webhook-configuration -o yaml > /tmp/gk-webhook-backup.yaml
kubectl delete validatingwebhookconfiguration \
  gatekeeper-validating-webhook-configuration
# Re-enable: reapply or let Helm reconcile
kubectl apply -f /tmp/gk-webhook-backup.yaml
```

### Namespace exclusions: three methods

The first method is a **label on the namespace** — `admission.gatekeeper.sh/ignore: "true"` causes the webhook's `namespaceSelector` to skip the namespace entirely. This is the most efficient exclusion: the API server never sends the request to Gatekeeper. The second method is the **Config resource `match` block** with `processes: ["*"]`, which excludes the namespace from webhook evaluation, audit, sync, and mutation processing within Gatekeeper. The third method is **per-constraint `match.excludedNamespaces`**, which scopes the exclusion to individual policies. Use the label method for system namespaces (`kube-system`, `gatekeeper-system`) and per-constraint exclusions for application-specific exceptions.

### Testing policies before deployment

The `gator verify` CLI runs ConstraintTemplate + Constraint + test-case suites locally without a cluster. Each suite references a template, a constraint, and a list of test objects with expected violation counts:

```yaml
# suite.yaml — test the pod-security-context policy
apiVersion: test.gatekeeper.sh/v1alpha1
kind: Suite
tests:
  - name: pod-security-context
    template: k8spodsecuritycontext-template.yaml
    constraint: pod-security-context-constraint.yaml
    cases:
      - name: compliant-pod
        object: tests/compliant-pod.yaml      # all fields correct
        assertions:
          - violations: no
      - name: privileged-container
        object: tests/privileged-pod.yaml     # privileged: true
        assertions:
          - violations: yes
          - message: "privileged"
            violations: 1
      - name: missing-everything
        object: tests/insecure-pod.yaml       # missing runAsNonRoot, RO fs, drop ALL
        assertions:
          - violations: 3                     # exactly three violations expected
```

Run with `gator verify ./...` in CI. For unit-testing the Rego logic independently (faster iteration, no YAML overhead), extract the Rego into a standalone `.rego` file and use `opa test`:

```rego
# k8spodsecuritycontext_test.rego
package k8spodsecuritycontext

test_privileged_denied {
    inp := {"review": {"object": {"spec": {"containers": [
        {"name": "bad", "image": "nginx",
         "securityContext": {"privileged": true}}
    ]}}}, "parameters": {"exemptImages": [], "allowedCapabilities": []}}
    results := violation with input as inp
    count(results) > 0
}

test_compliant_allowed {
    inp := {"review": {"object": {"spec": {"containers": [
        {"name": "good", "image": "nginx",
         "securityContext": {
             "privileged": false,
             "runAsNonRoot": true,
             "readOnlyRootFilesystem": true,
             "capabilities": {"drop": ["ALL"]}
         }}
    ]}}}, "parameters": {"exemptImages": [], "allowedCapabilities": []}}
    results := violation with input as inp
    count(results) == 0
}
```

Run with `opa test . -v` for verbose output or `opa test . --coverage` for coverage metrics.

### GitOps sync ordering with Flux

Gatekeeper ConstraintTemplates dynamically create CRDs in the API server. A Constraint referencing a kind that doesn't yet exist will fail with `no matches for kind`. In Flux, enforce ordering with `dependsOn` across separate Kustomization resources. The chain is: Gatekeeper controller → Config + ConstraintTemplates → Constraints → workloads:

```yaml
# 1. Gatekeeper controller (Helm or raw manifests)
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: gatekeeper-system
  namespace: flux-system
spec:
  interval: 10m
  path: ./infrastructure/gatekeeper/controller
  prune: true
  wait: true                                  # block until deployment is healthy
  sourceRef:
    kind: GitRepository
    name: flux-system
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: gatekeeper-controller-manager
      namespace: gatekeeper-system
    - apiVersion: apps/v1
      kind: Deployment
      name: gatekeeper-audit
      namespace: gatekeeper-system
---
# 2. Config resource + ConstraintTemplates
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: gatekeeper-templates
  namespace: flux-system
spec:
  dependsOn:
    - name: gatekeeper-system                 # wait for controller to be ready
  interval: 10m
  path: ./infrastructure/gatekeeper/templates
  prune: true
  wait: true                                  # wait for CRDs to be registered
  sourceRef:
    kind: GitRepository
    name: flux-system
---
# 3. Constraint instances
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: gatekeeper-constraints
  namespace: flux-system
spec:
  dependsOn:
    - name: gatekeeper-templates              # CRDs from templates must exist first
  interval: 10m
  path: ./infrastructure/gatekeeper/constraints
  prune: true
  wait: true
  sourceRef:
    kind: GitRepository
    name: flux-system
---
# 4. Application workloads
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: workloads
  namespace: flux-system
spec:
  dependsOn:
    - name: gatekeeper-constraints            # policies must be active before workloads
  interval: 10m
  path: ./apps/production
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
```

The `wait: true` directive on each Kustomization ensures Flux waits until all resources in that layer are fully reconciled and healthy before proceeding. Without this, Flux might apply Constraints before the ConstraintTemplate CRDs are registered, causing reconciliation failures and retry loops.

---

## 10. Every violation class and where it gets caught

The table below maps each violation class to the layer that detects it, when detection occurs, and how the operator receives feedback. The core design principle is that **Gatekeeper catches configuration errors before they become runtime problems**, while **Cilium catches runtime network violations that no admission controller can see**.

| Violation class                                    | Gatekeeper                         | Cilium                                     | When detected                      | Feedback mechanism                                 |
| -------------------------------------------------- | ---------------------------------- | ------------------------------------------ | ---------------------------------- | -------------------------------------------------- |
| Missing pod labels (`app`, `team`, `env`           | ✅ Admission denied                | ❌ Wrong identity assigned silently        | Admission time (seconds)           | Webhook rejection message                          |
| Unapproved container image                         | ✅ Admission denied                | ❌ Cannot inspect                          | Admission time (seconds)           | Webhook rejection message                          |
| Privileged container                               | ✅ Admission denied                | ❌ Cannot inspect                          | Admission time (seconds)           | Webhook rejection message                          |
| Root execution (`runAsNonRoot` missing)            | ✅ Admission denied                | ❌ Cannot inspect                          | Admission time (seconds)           | Webhook rejection message                          |
| Dangerous capabilities                             | ✅ Admission denied                | ❌ Cannot inspect                          | Admission time (seconds)           | Webhook rejection message                          |
| Overly permissive CiliumNetworkPolicy              | ✅ Admission denied                | ❌ Enforces as written                     | Admission time (seconds)           | Webhook rejection message                          |
| Cilium identity misconfiguration                   | ✅ Prevented via label enforcement | ⚠️ Detectable via Hubble identity mismatch | Admission time (prevented)         | Webhook rejection / Hubble flows                   |
| Unauthorized network connection                    | ❌ Cannot inspect packets          | ✅ eBPF drops packet                       | Runtime (milliseconds)             | Hubble flow log with DROPPED verdict               |
| L7 protocol violation (e.g., disallowed HTTP path) | ❌ Cannot inspect L7               | ✅ Envoy proxy rejects                     | Runtime (milliseconds)             | Hubble L7 flow log                                 |
| DNS exfiltration to unauthorized domain            | ❌ Cannot inspect DNS              | ✅ DNS proxy blocks                        | Runtime (milliseconds)             | Hubble DNS log with DROPPED verdict                |
| Unencrypted pod-to-pod traffic (cross-node)        | ❌ Cannot enforce encryption       | ✅ WireGuard encrypts transparently        | Runtime (always on)                | `cilium status` shows WireGuard peers              |
| Pre-existing non-compliant resource                | ✅ Audit detects                   | ❌ N/A                                     | Next audit cycle (≤ auditInterval) | Constraint `.status.violations` + Prometheus gauge |

The table reveals a clean separation: the left side of the stack (Gatekeeper) handles all configuration validation — labels, images, security contexts, policy CRDs — at admission time with immediate developer feedback. The right side (Cilium) handles all runtime enforcement — packet filtering, L7 inspection, DNS control, encryption — at the kernel level with Hubble-based observability. The overlap zone is Cilium identity integrity, where Gatekeeper's label enforcement at admission prevents the silent identity misconfiguration that would otherwise only be discoverable through runtime debugging.

## Conclusion

The integration between Gatekeeper and Cilium is not about redundancy — it is about **complementary coverage** across two fundamentally different enforcement planes. Gatekeeper operates on Kubernetes API objects before they are persisted, catching misconfigurations that would create silent failures at runtime. Cilium operates on network packets after pods are running, enforcing segmentation rules that no admission controller can express. The critical integration point is label integrity: Gatekeeper guarantees that every pod and namespace carries the labels that Cilium's identity mechanism and policy selectors depend on, transforming what would be a silent identity misconfiguration into an immediate, actionable admission rejection.

The four ConstraintTemplates presented here — pod security context, approved registries, required labels, and CiliumNetworkPolicy validation — form a minimal but complete admission policy set for a Cilium-based zero-trust cluster. The operational procedures (dryrun → warn → deny rollout, gator testing in CI, Flux dependency ordering, webhook resilience) ensure these policies can be deployed safely in production without disrupting existing workloads. The cluster-wide default deny CiliumClusterwideNetworkPolicy ensures that the Cilium side of the architecture starts from a zero-trust baseline, and the end-to-end decision matrix provides a clear mental model for which layer catches which violation class.
