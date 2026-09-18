# CFP-44138: Cilium Clusterwide Local Redirect Policy

**SIG:** SIG-LB

**Begin Design Discussion:** 2026-02-03

**Cilium Release:** 1.22

**Authors:** Yusuke Suzuki, Alasdair McWilliam

**Status:** Draft

**Issue:** [https://github.com/cilium/cilium/issues/44138](https://github.com/cilium/cilium/issues/44138)

## Summary

This CFP proposes a new cluster-scoped `CiliumClusterwideLocalRedirectPolicy` (CCLRP) custom resource for configuring local traffic redirection.

CCLRP makes the cluster-wide effect of a local redirect explicit, restricts address-based redirects to link-local addresses, replaces implicit named-port joins with explicit port mappings, and removes datapath loop-prevention details from the policy API.

The existing namespace-scoped `CiliumLocalRedirectPolicy` (CLRP) will remain available while CCLRP matures. Deprecation and eventual removal of CLRP are future milestones and are not required for the initial CCLRP release.

## Motivation

CLRP redirects traffic destined for a frontend to node-local backend pods.

It is currently configured through a namespaced resource, even though the redirect affects traffic originating from workloads throughout the cluster.

The current [API](https://docs.cilium.io/en/stable/network/kubernetes/local-redirect-policy/) also combines Kubernetes service references, arbitrary addresses, backend pod selection, and port matching rules with several implicit behaviours. These behaviours are difficult to reason about for the reasons that are discussed in the following sections.

Overall, these issues are believed to have contributed to a fragile implementation that incurs an ongoing technical burden to the project. Recent examples include:

* CiliumLocalRedirectPolicy: panic if redirectBackend.toPorts is defined but empty  
  * [https://github.com/cilium/cilium/issues/48482](https://github.com/cilium/cilium/issues/48482)  
* LRP shows invalid IP for NodeLocal DNS when applying the example manifest  
  * [https://github.com/cilium/cilium/issues/43944](https://github.com/cilium/cilium/issues/43944)  
* LRP backend is not selected when Pod container ports are not specified  
  * [https://github.com/cilium/cilium/issues/43929](https://github.com/cilium/cilium/issues/43929)  
* LRP does not use the right port number  
  * [https://github.com/cilium/cilium/issues/41589](https://github.com/cilium/cilium/issues/41589)  
* CiliumLocalRedirectPolicy doesn’t work with skipRedirectFromBackend  
  * [https://github.com/cilium/cilium/issues/40450](https://github.com/cilium/cilium/issues/40450)

### Resource scope does not match traffic scope

`CiliumLocalRedirectPolicy` is a namespaced resource and its Service and backend references are namespaced. However, the resulting redirect is not limited to traffic from that namespace. This makes it possible for a policy created in one namespace to change traffic from workloads in other namespaces without the resource scope making that impact obvious.

The new resource will be cluster-scoped so that its scope is visible to administrators and so that cluster-wide use cases are represented by a cluster-wide object.

### Address-based redirects are too broad by default

An address-based CLRP can match an arbitrary IP address. Historically, this could include ClusterIPs and Pod IPs, allowing a misconfigured policy to interfere with unrelated services or workloads.

Cilium provides an `addressMatchCIDRs` configuration option, but the safe scope is not part of the resource API and is not enforced by default.

CCLRP will restrict address-based redirects to link-local address ranges. This preserves the primary address-based use case—node-local or host-provided services such as a metadata endpoint—without allowing a policy to override arbitrary in-cluster destinations.

### Multi-port matching depends on implicit names

The current API leans on port declarations in three key touchpoints.

* CLRP `redirectFrontend.(addressMatcher|serviceMatcher).toPorts[]` provides a list of target frontend ports.  
* CLRP `redirectBackend.toPorts[]` provides a list of target backend ports.  
* Pod `ports[]` provides a list of ports a container is intended to listen on as informational metadata.

The API may or may not use port names as a join key between these touch points depending on the CLRP declaration.

When a **single** frontend port is specified:

* Port names in the CLRP declaration are ignored.  
* The first entry in the frontend `toPorts[]` is mapped directly to the first entry in the backend `toPorts[]`.  
* Pod `ports[]` metadata is ignored.

When **multiple** frontend ports are specified:

* Port names in the CLRP declaration are mandatory.  
* The port name is used as a join key to map frontend `toPorts[]` to backend `toPorts[]`.  
* A corresponding named entry in the Pod `ports[]` list must also exist and match.

When frontend ports are **omitted**: 

* An `addressMatcher` CLRP will not apply.  
* A `serviceMatcher` CLRP will redirect all service ports.

Problems:

* The numeric port mapping cannot be determined directly from the manifests.  
* The pod `ports[]` list is informational in Kubernetes, yet it affects CLRP backend selection.  
* Multi-port mapping does not work unless the pod has a matching `ports[]` entry, even if an application is listening on the expected port.
* The port mapping logic differs between single-port and multi-port configurations, and configurations where `serviceMatcher.toPorts[]` is omitted.
* There are special cases that depend on port list ordering. Incorrect ordering between frontend, backend and pod spec can lead to mapping failures.

CCLRP will express the relationship directly as `port` to `targetPort`, following the general Kubernetes Service model. Port names will not be required to connect the frontend and backend sides of a redirect.

### Loop prevention exposes datapath details

The `skipRedirectFromBackend` field exposes a datapath implementation detail and requires users to make the same runtime decision independently for every policy. However, there is no known useful purpose for redirecting a backend's traffic back to itself.

Loop prevention will be mandatory for CCLRP and implemented as an agent-level behaviour. During the alpha phase, an agent without the required kernel capability will report an error and leave CCLRP unavailable without preventing existing CLRP use. Once CLRP is removed, the capability will be required whenever local redirects are enabled.

## Goals

* Make the cluster-wide impact of local redirects explicit in the API.  
* Provide a safe default scope for address-based redirects.  
* Replace implicit port-name joins with explicit frontend-to-backend port mappings.  
* Remove datapath loop-prevention configuration from individual policies.  
* Reduce silent failure modes through validation and observable reconciliation errors.  
* Preserve the existing local redirect use cases, especially node-local DNS and host-provided link-local services.  
* Allow CCLRP and CLRP to coexist during adoption of the new resource.
* Preserve an active local redirect while a CCLRP takes over a frontend from a CLRP to facilitate zero-downtime migration.

## Non-Goals

* Redesign the eBPF local redirect datapath beyond any skip map changes selected by this CFP.
* Change the fundamental local redirect behaviour: traffic is still redirected only to node-local backend pods selected by the policy.  
* Provide an automatic conversion of existing CLRP objects into CCLRP objects.  
* Deprecate or remove CLRP as part of the initial CCLRP implementation.  
* Support arbitrary address ranges through CCLRP.  
* Generalise the API to express an arbitrary set of frontend types or routing behaviours.

## Use Cases

### Cloud-provider-managed service

Cloud providers may expose infrastructure services to workloads through a well-known DNS name and link-local IP address. The service may be managed outside the tenant's Kubernetes namespace and may have a node-local implementation on each host.

Workloads should be able to use the provider's normal endpoint while Cilium redirects traffic to the local implementation on the node.

This use case is represented by an `addressMatcher` for the link-local address and a backend selector for the local service pods.

CCLRP provides a cluster-scoped mechanism for expressing this class of redirect; the mechanism used to reach a backend listening on an address different from its Pod IP is outside the scope of this CFP.

The cluster-wide scope is important because the endpoint is a shared cluster facility rather than an application-owned namespaced Service.

### Node-local DNS

In large clusters, a centralised CoreDNS or kube-dns deployment may become a scalability bottleneck for DNS request volume. Running a DNS cache on each node reduces traffic to the centralised DNS service and distributes request processing across the cluster.

Workloads should continue to query the cluster DNS Service, while Cilium redirects DNS traffic to the node-local cache on the same node.

This use case is represented by a `serviceMatcher` for the cluster DNS Service and explicit mappings for both UDP and TCP port 53.

## Proposal

### Overview

Introduce the following cluster-scoped resource:

```
apiVersion: cilium.io/v2alpha1
kind: CiliumClusterwideLocalRedirectPolicy
metadata:
  name: <name>
spec:
  # Exactly one of addressMatcher and serviceMatcher must be specified.
  addressMatcher:
    ip: <link-local-ip>
  # serviceMatcher:
  #   namespace: <service-namespace>
  #   service: <service-name>

  localEndpointSelector:
    matchLabels:
      app: <local-backend>
      io.kubernetes.pod.namespace: <backend-namespace>

  ports:
    - port: <frontend-port>
      targetPort: <backend-port>
      protocol: <TCP, UDP>
```

`addressMatcher` and `serviceMatcher` are mutually exclusive.

The policy is cluster-scoped, so it has no Kubernetes namespace. The backend selector is evaluated against pods in all namespaces.

The current proposal expresses namespace restriction in the selector, using the same namespace-label convention used by `CiliumClusterwideNetworkPolicy`. The safety of this choice is questioned below.

The API describes policy intent. The implementation may resolve a service into one or more concrete ClusterIP frontends and may resolve a selector into the node-local backend endpoints available on each node. Those internal representations are implementation details and are not part of the CCLRP API contract.

### Address matching

An `addressMatcher` contains one IP address. The address must be within a link-local range:

* IPv4 link-local: `169.254.0.0/16`  
* IPv6 link-local: `fe80::/10`

The `addressMatcher` is intended for destinations that are not represented by a Kubernetes Service, such as a host-provided metadata endpoint. It must not be used to override a Kubernetes Service ClusterIP or a Pod IP.

The initial API intentionally supports one IP per `addressMatcher`. An administrator that needs both IPv4 and IPv6 can create separate policies. This keeps conflict handling, validation, updates, and deletion semantics straightforward while leaving room for a future multi-address extension.

### Service matching

A `serviceMatcher` identifies a Kubernetes Service by namespace and name. The Service must have a ClusterIP that Cilium represents as a concrete frontend within the load-balancer implementation. Services without a ClusterIP are not supported by `serviceMatcher`.

When `ports` is present, it selects the Service ports to redirect and explicitly maps each frontend port to a backend `targetPort`.

When `ports` is omitted, frontend-to-backend port mappings are derived from the Service and all ports are redirected. This preserves the existing `serviceMatcher` convention, while facilitating an explicit list when a subset of Service ports should be redirected, or when the local backend uses different ports. However, maintaining this behaviour is raised in a key question below.

The controller must validate that every selected Service port can be represented by the requested mapping. Invalid or ambiguous mappings must be reported as reconciliation errors; they must not be silently skipped.

### Backend selection

`localEndpointSelector` selects the pods that can receive redirected traffic. Only backend pods on the node where the traffic is processed are eligible for the corresponding local redirect entry.

Because CCLRP is cluster-scoped, selectors are evaluated across namespaces. The current proposal follows the cluster-wide policy convention of restricting a selector with the `io.kubernetes.pod.namespace` label. The safety of relying on that convention is discussed in a key question below.

Example:

```
localEndpointSelector:
  matchLabels:
    app: node-local-dns
    io.kubernetes.pod.namespace: kube-system
```

The policy must not redirect traffic to a backend pod on another node. If no eligible local backend exists on a node, Cilium must not install a local redirect entry on that node. Traffic should be processed as if the policy were absent; it must not be redirected to a remote backend or blackholed. This preserves the existing fail-open behaviour introduced by [PR #41463](https://github.com/cilium/cilium/pull/41463).

### Port mapping

Each entry in `ports` represents one frontend-to-backend mapping:

| Field | Meaning | Required |
| :---- | :---- | :---- |
| `port` | Destination port on the matched frontend | Yes |
| `targetPort` | Port on the selected local backend pod | Yes |
| `protocol` | Transport protocol used by both sides | No; defaults to `TCP` |
| `name` | Optional descriptive port name | No; not used for matching |

The frontend and backend port relationship is determined by the numeric `port` and `targetPort` fields. CCLRP must not require the two ports to have the same name, and it must not require a matching `containerPort` declaration solely to establish the mapping.

For an `addressMatcher`, at least one `ports` entry is required because the `addressMatcher` alone does not identify a complete load-balancer frontend.

For a `serviceMatcher`, `ports` is currently optional; the implications of inheriting the Service mapping when it is omitted are discussed below.

### Loop prevention

[PR #13287](https://github.com/cilium/cilium/pull/13287) introduced the original implementation of CLRP loop prevention to the datapath using socket lookups (`bpf_sk_lookup_*` helpers). The PR discussion noted that this depended on the relevant `sk_lookup` helpers being available, but did not expose a per-policy opt-out. Listening sockets that happened to match the destination port could cause translation to be skipped incorrectly, as reported in [issue #15195](https://github.com/cilium/cilium/issues/15195) and [issue #20164](https://github.com/cilium/cilium/issues/20164).

[PR #26144](https://github.com/cilium/cilium/pull/26144) replaced the socket-lookup implementation with netns-cookie-based skip maps and introduced `skipRedirectFromBackend` into the CLRP as an opt-in field. The linked history does not document why that behaviour changed. One possible contributing reason is kernel compatibility: [`bpf_get_netns_cookie`](https://github.com/torvalds/linux/commit/f318903c0bf4) was introduced in Linux 5.7, while the userspace access through [`SO_NETNS_COOKIE`](https://github.com/torvalds/linux/commit/e8b9eab99232c4e62ada9d7976c80fd5e8118289) required by Cilium landed upstream in Linux 5.14 and was [backported to Linux 5.12.18](https://cdn.kernel.org/pub/linux/kernel/v5.x/ChangeLog-5.12.18). The additional kernel-version requirement may have influenced the decision to make the revised behaviour opt-in, but this remains an inference rather than a documented rationale.

During the review of [PR #13287](https://github.com/cilium/cilium/pull/13287), the possibility of redirecting traffic from a CLRP backend back to that same backend was considered, but no current use case was identified. The available history to date does not identify a subsequent use case.

Based on the above history, CCLRP must always enable loop prevention and must not expose a per-policy opt-out.

During the `v2alpha1` phase, the agent must check for `SO_NETNS_COOKIE` support at startup. If the capability is unavailable, the agent must log an actionable error and continue to start so that existing CLRP use is not disrupted. CCLRP reconciliation must fail on that node, no CCLRP redirect should be installed, and traffic must behave as if the CCLRP were absent.

*Note: This CFP does not change CLRP: `skipRedirectFromBackend` remains opt-in and existing capability handling remains unchanged.*

#### Existing CLRP skip map behaviour

The existing netns-cookie implementation uses two BPF skip maps: one per address family. The existing maps and their key structures are provided below for reference. Only key presence matters; the stored map value is not interpreted.

```
// IPv4

struct skip_lb4_key {
	__u64 netns_cookie;     /* Source pod netns cookie */
	__u32 address;          /* Destination service virtual IPv4 address */
	__u16 port;             /* Destination service virtual layer4 port */
	__u16 pad;
};

struct {
	__uint(type, BPF_MAP_TYPE_HASH);
	__type(key, struct skip_lb4_key);
	__type(value, __u8);
	__uint(pinning, LIBBPF_PIN_BY_NAME);
	__uint(max_entries, CILIUM_LB_SKIP_MAP_MAX_ENTRIES);
	__uint(map_flags, BPF_F_NO_PREALLOC | BPF_F_RDONLY_PROG_COND);
} cilium_skip_lb4 __section_maps_btf;

// IPv6

struct skip_lb6_key {
	__u64 netns_cookie;     /* Source pod netns cookie */
	union v6addr address;   /* Destination service virtual IPv6 address */
	__u32 pad;
	__u16 port;             /* Destination service virtual layer4 port */
	__u16 pad2;
};

struct {
	__uint(type, BPF_MAP_TYPE_HASH);
	__type(key, struct skip_lb6_key);
	__type(value, __u8);
	__uint(pinning, LIBBPF_PIN_BY_NAME);
	__uint(max_entries, CILIUM_LB_SKIP_MAP_MAX_ENTRIES);
	__uint(map_flags, BPF_F_NO_PREALLOC | BPF_F_RDONLY_PROG_COND);
} cilium_skip_lb6 __section_maps_btf;
```

Only a CLRP with `skipRedirectFromBackend=true` creates skip map entries. The control plane adds entries for the selected backend pods and redirected frontends.

When the datapath processes traffic from an endpoint, it looks up the source network namespace cookie and destination frontend in the skip map for that address family. This check is implemented in both the [socket LB](https://github.com/cilium/cilium/blob/v1.21.0-pre.2/bpf/bpf_sock.c#L365-L369) and [per-packet LB](https://github.com/cilium/cilium/blob/v1.21.0-pre.2/bpf/bpf_lxc.c#L250-L257) paths. If an entry exists, the datapath skips the redirect translation, preventing traffic from a backend pod from being redirected back to itself.

The control plane reconciles a frontend as an `L3n4Addr`, but the skip map key contains only the backend network namespace cookie, frontend IP address, and frontend port. It does not contain the L4 protocol. Consequently, TCP and UDP frontends with the same IP address and port share skip map state.

*Note: This proposal does not change the field or the existing protocol-neutral key semantics for CLRP.*

#### CCLRP skip map behaviour

CCLRP represents a frontend as an `L3n4Addr`, including the L4 protocol. Reusing the existing skip map unchanged would mean that TCP and UDP frontends with the same IP address and port continue to share skip map state. Whether the skip map should be extended to differentiate between L4 protocols is discussed as a key question below.

To facilitate migration, a CCLRP should take precedence over a CLRP that claims the same concrete frontend. Only the effective policy should contribute skip map entries for the overlap. If the CCLRP is removed, the CLRP should become effective and its entries should be reconciled again.

#### Skip map capacity

The existing IPv4 and IPv6 skip maps are sized independently, and each is currently limited to [100 entries](https://github.com/cilium/cilium/blob/v1.21.0-pre.2/pkg/loadbalancer/maps/types.go#L1267-L1276). Existing CLRP entries count towards this limit; CCLRP entries would count towards the same limit.

The following illustrative projection assumes one backend pod per use case and that the resulting keys do not overlap. It uses the current protocol-neutral key and applies per node and per address family.

* `NF` is the number of distinct redirected frontend `L3n4Addr` tuples.
* `NK` is the number of distinct frontend keys produced for the skip map.
  * The current key uses `(IP, port)`
  * A protocol-differentiated key would use `(IP, port, protocol)`.
* `NB` is the number of distinct backend network namespaces.
* The number of skip map entries is `NK × NB`.

| Use case | Redirected frontends | `NF` | `NK` | `NB` | Current skip map entries (`NK × NB`) |
| :--- | :--- | ---: | ---: | ---: | ---: |
| Cloud-managed service | `169.254.169.254:80/TCP`, `169.254.169.254:443/TCP` | 2 | 2 | 1 | 2 |
| Node-local DNS | `<DNS-IP>:53/TCP`, `<DNS-IP>:53/UDP` | 2 | 1 | 1 | 1 |
| **Total** | | | | | **3** |

If CCLRP retains the current protocol-neutral key, equivalent CLRPs and CCLRPs selecting the same frontends and backend network namespaces should require the same skip map capacity. If protocol differentiation is selected, `NK` becomes equal to `NF` and the projected usage increases from 3 to 4 IPv4 entries per node. Both projections remain small relative to the current limit of 100. The initial implementation should therefore retain the existing limit, subject to prototype validation of the expected usage for supported deployment shapes. If a supported configuration can exceed the limit, the map size or configurability must be revisited before release.

Frontend and skip map state are reconciled by separate reconciliation loops and may be programmed at different times. This proposal does not change that model or introduce activation ordering between them. Failure to install a required CCLRP skip map entry, including because the map is full, must be reported by the skip map reconciler.

### Validation and conflicts

The CCLRP controller should reject a policy that has:

* neither or both of `addressMatcher` and `serviceMatcher`  
* an address outside the link-local ranges  
* an invalid port, protocol, or port mapping  
* a `serviceMatcher` that resolves to an unsupported Service
* a selector or mapping that cannot produce a valid local redirect frontend

If the referenced Service does not currently exist or cannot yet be resolved, the policy should remain inactive and report an observable reconciliation condition until the dependency becomes available.

Policies that attempt to claim the same concrete frontend must have deterministic behaviour. The initial implementation should allow only one active owner for a frontend within the same policy type and should report conflicts as errors.

CCLRP should take precedence over CLRP when both resources identify the same frontend, so that an explicitly migrated CCLRP is not shadowed by the legacy policy while both resources coexist.

Validation and reconciliation failures must be visible through CCLRP status, agent diagnostics, logs, and events or equivalent Kubernetes-facing feedback.

### Observability

The existing local redirect debugging commands should be extended to identify:

* whether an entry came from CLRP or CCLRP;  
* the policy that owns the entry;  
* the resolved frontend address, port, and protocol;
* the selected node-local backends;  
* validation or reconciliation errors.

This is required to make cluster-wide redirects diagnosable during the coexistence period.

### Examples

#### Link-local address redirect

```
apiVersion: cilium.io/v2alpha1
kind: CiliumClusterwideLocalRedirectPolicy
metadata:
  name: cloud-metadata-redirect
spec:
  addressMatcher:
    ip: 169.254.169.254
  localEndpointSelector:
    matchLabels:
      app: metadata
      io.kubernetes.pod.namespace: cloud-infra
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
    - port: 443
      targetPort: 8443
      protocol: TCP
```

This redirects traffic to the link-local metadata address to a node-local backend selected from the `cloud-infra` namespace.

#### Node-local DNS redirect

```
apiVersion: cilium.io/v2alpha1
kind: CiliumClusterwideLocalRedirectPolicy
metadata:
  name: node-local-dns-redirect
spec:
  serviceMatcher:
    namespace: kube-system
    service: kube-dns
  localEndpointSelector:
    matchLabels:
      app: node-local-dns
      io.kubernetes.pod.namespace: kube-system
  ports:
    - port: 53
      targetPort: 53
      protocol: UDP
    - port: 53
      targetPort: 53
      protocol: TCP
```

This redirects both DNS protocols for the cluster DNS Service to the node-local DNS backend.

#### Comparison with CiliumLocalRedirectPolicy

When comparing the above CCLRP examples with the existing CLRP facility:

* The cluster-wide impact of this redirect is explicit in both resource type and lack of namespace in the resource metadata.  
* The frontend-to-backend port mapping is explicitly declared in one place.  
* Differing behaviour based on port list declaration is broadly removed.  
* Ordering constraints across multiple port lists is no longer a concern.  
* Unnecessary nesting of fields within the API is minimised.

### Implementation approach

The implementation should add the CCLRP CRD, permissions, and agent reconciliation required to translate policy intent into the existing load-balancer frontend and backend state.

The implementation should keep the following boundaries clear:

1. The Kubernetes-facing layer validates and reflects CCLRP intent.  
2. A reconciliation layer resolves Services, ports, and node-local backend endpoints.  
3. The existing load-balancer integration applies concrete local redirect entries.

The internal tables, Go structures, and controller decomposition may evolve during implementation. They should not become additional user-facing API commitments in this CFP.

The implementation should include CCLRP status conditions for accepted, rejected, partially resolved, and backend-unavailable states, including the concrete frontends and node-local backends produced by a policy. The status model may evolve during `v2alpha1`, but must be stable before promotion to `v2`.

A prototype implementation should be used to validate the API and reconciliation behaviour with tests covering address matching, Service matching, namespace-restricted selectors, explicit port translation, invalid configuration, backend churn, and nodes without local backends preserving the original frontend behaviour.

Kernel capability tests must cover non-fatal startup with CCLRP rejection while CLRP remains supported.

Skip map tests must cover the selected protocol semantics, TCP and UDP frontends with the same IP address and port, the capacity boundary and exhaustion, independent frontend and skip map reconciliation, failure reporting, agent restart, and pinned-map compatibility if the key changes.

Migration tests must cover CLRP and CCLRP coexistence, zero-downtime takeover, and rollback.

This CFP was authored midway through the Cilium 1.21 development cycle. To avoid coupling the public alpha API to the shortened implementation window, delivery is split across releases: prototype and non-user-facing foundational work may land in 1.21, while the `cilium.io/v2alpha1` CCLRP API is targeted for 1.22. The complete versioned schedule is defined under Future Milestones.

### Migration approach

No automatic object conversion from CLRP to CCLRP will be provided.

Administrators will need to translate an existing CLRP into a CCLRP and explicitly add any namespace restriction that was previously implied by the namespaced resource. For example, a backend selector that was previously limited to `kube-system` must include `io.kubernetes.pod.namespace: kube-system` in the CCLRP selector.

It is reasonable to expect the intent of a CLRP, re-declared via CCLRP, to reconcile to the same final state (subject to any behavioural differences such as loop prevention).

CLRP and CCLRP resources must be able to coexist to facilitate migration. CCLRP should take precedence when both resource types attempt to claim the same concrete frontend, avoiding any dependency on creation order.

If the CCLRP can be fully reconciled, it should take ownership of the frontend, and the overlapping CLRP should become dormant while its resource remains present. The administrator can then remove the CLRP after verifying the CCLRP.

If the CCLRP cannot be reconciled, the takeover should not occur and the CLRP should remain active.

During takeover, the frontend should transition directly from the CLRP-derived redirect target to the CCLRP-derived redirect target without an intermediate non-redirected state.

Administrators will need to verify the CCLRP operation before deciding whether to proceed by removing the legacy CLRP, or roll back by deleting the CCLRP.

Migration documentation should provide a mechanical conversion checklist and explain how to roll out a CCLRP before removing its corresponding CLRP. The expected behaviour should be documented, surfaced through diagnostics, and covered by upgrade and migration tests.

## Impacts / Key Questions

### Impact: cluster-wide permissions and blast radius

CCLRP is intentionally cluster-scoped. Creating or modifying one requires cluster-level authorisation, and the resulting redirect can affect traffic from workloads in any namespace. This is a larger authorisation boundary than the current resource presents, but it accurately reflects the effect of the feature.

The documentation and RBAC examples must make this explicit. Namespace-specific ownership or delegation is outside the initial proposal.

### Key Question: how should backend namespace scope be expressed?

The examples above make use of the `io.kubernetes.pod.namespace` label in `localEndpointSelector` to align with the selector conventions of `CiliumClusterwideNetworkPolicy`. However, this creates a safety foot gun for local redirects: omitting that label broadens the selector to pods in every namespace, which can cause unintended pods to become redirect backends.

This is a more serious failure mode than an overly broad policy selector because the selected pods change the load-balancer destination for matching traffic. The API should make namespace scope explicit and should not silently default to all namespaces as a consequence of a missing label.

#### Option 1: Make namespace scope a first-class field

Add an explicit namespace field to the backend selection criteria, for example:

```
localEndpointSelector:
  namespace: kube-system
  matchLabels:
    app: node-local-dns
```

Pros:

* The namespace boundary is visible in every policy.  
* Omitting a label cannot accidentally widen the backend selection.  
* It matches the primary use cases, where local backend pods normally belong to one namespace.

Cons:

* Selecting backends from multiple namespaces would require additional API semantics.  
* This differs from the existing cluster-wide network-policy selector convention.

#### Option 2: Retain namespace selection through labels

Keep using `io.kubernetes.pod.namespace` in the label selector and require administrators to include it when a namespace restriction is desired.

Pros:

* It follows the existing `CiliumClusterwideNetworkPolicy` convention.  
* It retains the flexibility of selecting pods across multiple namespaces.

Cons:

* A missing label silently changes the policy's scope.  
* Reviewers and operators must remember that a cluster-scoped local redirect has a different blast radius from an ordinary policy selector.

The safer default is to require an explicit namespace boundary, or otherwise require an explicit opt-in for selecting backends across all namespaces.

### Key Question: should the skip map key distinguish between L4 protocols?

The existing skip map key contains the frontend IP address and port, but not the L4 protocol. CCLRP represents frontends as `L3n4Addr` tuples, so the skip map implementation must either continue to share state between TCP and UDP frontends with the same IP address and port or be extended to distinguish them.

#### Option 1: Differentiate between L4 protocols

Extend the skip map implementation so that TCP and UDP frontends produce distinct keys.

Pros:

* Establishes a one-to-one relationship between each concrete CCLRP frontend/backend pod pair and a skip map entry.
* Prevents a backend network namespace associated with one protocol from affecting loop prevention for another protocol.
* Allows TCP and UDP frontends with the same IP address and port to use different backend pods without sharing skip map state.

Cons:

* Requires a skip map key and datapath lookup change.
* Increases entry usage when the same IP address and port are redirected for multiple protocols.
* Requires compatibility testing for existing pinned maps and for CLRP and CCLRP coexistence.

#### Option 2: Retain protocol-neutral skip map keys

Reuse the existing skip map implementation for CCLRP without adding the L4 protocol to the key.

Pros:

* Preserves the existing CLRP datapath and map key semantics.
* Avoids pinned-map compatibility changes.
* Uses one entry when TCP and UDP share the same frontend IP address, port, and backend network namespace.

Cons:

* Does not preserve the one-to-one relationship between each concrete CCLRP frontend/backend pod pair and a skip map entry, because frontends that differ only by protocol share the same entry.
* Configurations that use different backend pods for TCP and UDP may require validation or restriction to avoid incorrect loop-prevention behaviour.

The prototype should validate whether protocol-neutral state can produce incorrect behaviour for supported CCLRP configurations. The selected option must be reflected in the final capacity assessment and migration testing.

### Key Question: should address matching support multiple IPs?

The proposal keeps one IP per `addressMatcher`. This means a dual-stack configuration uses two resources, but each resource has simple validation and conflict semantics.

Supporting multiple IPs in one resource would improve convenience, but would require defining all-or-nothing behaviour, conflict reporting, partial failure, and update/deletion semantics.

The initial proposal defers that extension until a concrete use case requires it. The only theorised use case at the time of writing is a non-Kubernetes application that may also need to be exposed on IPv6 alongside IPv4.

### Key Question: should port mappings be top-level or nested?

This proposal places `ports` at the policy level because each entry relates the selected frontend to the selected local backend, regardless of whether the frontend was identified by a Service or an address. This also makes the mapping visible and removes the existing nested `redirectFrontend` and `redirectBackend` wrappers.

The trade-off is that the relationship between `ports.port` and the frontend matcher, and between `ports.targetPort` and the backend selector, is less visually nested. The API documentation and examples should explain that relationship directly.

### Key Question: should CCLRP support inherited all-port behaviour?

The current proposal allows `ports` to be omitted for a `serviceMatcher`, in which case all supported Service ports are redirected and their mappings are inherited from the Service. This preserves a convenient form of the existing API, but it conflicts with the goal of making frontend-to-backend mappings explicit.

It also means that adding a new port to a Service can change the effective scope of a redirect without changing the CCLRP. That may be surprising for a cluster-scoped resource.

#### Option 1: Retain inherited all-port behaviour

Pros:

* Keeps the node-local DNS configuration concise.  
* Automatically includes new Service ports.  
* Preserves the current convenience behaviour for `serviceMatcher` users.

Cons:

* The effective set of redirected ports is implicit and can change over time.  
* It weakens the explicit `port` to `targetPort` contract.  
* It makes auditing and migration less predictable.
* Requires port mappings to be inferred from Service and backend state, including load-balancer state that CCLRP reconciliation intends to modify, creating circular reconciliation dependencies.

#### Option 2: Require explicit port mappings

Require `ports` for CCLRP, while retaining the legacy omission behaviour for CLRP during the transition period.

Pros:

* Every CCLRP declares its redirected frontend ports explicitly.  
* Adding a Service port cannot silently broaden an existing redirect.  
* It gives the new API one consistent port model.
* Keeps the desired port mappings explicit, simplifying reconciliation and avoiding circular dependencies on derived load-balancer state.

Cons:

* Policies are more verbose, especially for Services with many ports.  
* Administrators must update CCLRP when they want a newly added Service port redirected.  
* Migration requires enumerating the Service ports that were previously inherited.

CCLRP should require explicit port mappings. Inherited all-port behaviour should remain available only for CLRP compatibility.

### Key Question: how should inherited Service target ports be resolved?

If inherited Service mappings remain supported, Kubernetes Service port and `targetPort` semantics must remain understandable.

* Numeric target ports should map directly.  
* Named target ports need to be resolved according to Kubernetes conventions against the selected local backend pods.

The implementation must define and test behaviour when a named target port is not present on a selected backend pod. The result should be an observable validation or reconciliation error, not a silently omitted mapping.

## Future Milestones

### Foundational work (1.21)

Prototype and API-neutral foundational work should land in 1.21. This may include shared local-redirect reconciliation, ownership and conflict handling, and test infrastructure. The release must not install or register the CCLRP CRD or watch CCLRP resources.

### CCLRP `v2alpha1` introduction (1.22)

The CCLRP CRD should be introduced as `cilium.io/v2alpha1`.

CLRP should remain supported but feature-frozen, except for critical fixes and changes required to enable migration to CCLRP.

The two resources should be able to coexist, with CCLRP taking precedence when both claim the same concrete frontend.

### CCLRP promotion and CLRP deprecation (1.23)

CCLRP should be promoted to `cilium.io/v2` once it is stable and migration-ready.

CLRP should be marked as deprecated in the same release.

Starting with 1.23, a two-release window should be provided for migration from CLRP to CCLRP. Both resources should remain supported during this window. New local redirect deployments should use CCLRP, while CLRP remains available for migration and rollback.

### CLRP removal (1.25 or later)

CLRP should not be removed before 1.25. Removal remains conditional on CCLRP stability, documented migration procedures, and evidence that supported deployment shapes can migrate without interrupting active redirects.

As part of CLRP removal, `SO_NETNS_COOKIE` support should become a prerequisite whenever local redirects are enabled. If the capability is unavailable, the agent should fail to start with an actionable error. This change must be documented as an upgrade impact.
