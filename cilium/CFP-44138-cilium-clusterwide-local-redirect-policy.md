# CFP-44138: Cilium Clusterwide Local Redirect Policy

**SIG:** SIG-LB

**Begin Design Discussion:** 2026-02-03

**Cilium Release:** TBD

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

Overall, it is believed issues have contributed to a fragile implementation that incurs an ongoing technical burden to the project. Recent examples include:

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

The new resource should be cluster-scoped so that its scope is visible to administrators and so that cluster-wide use cases are represented by a cluster-wide object.

### Address-based redirects are too broad by default

An address-based CLRP can match an arbitrary IP address. Historically, this could include ClusterIPs and Pod IPs, allowing a misconfigured policy to interfere with unrelated services or workloads.

Cilium provides an `addressMatchCIDRs` configuration option, but the safe scope is not part of the resource API and is not enforced by default.

CCLRP should restrict address-based redirects to link-local address ranges. This preserves the primary address-based use case \- node-local or host-provided services such as a metadata endpoint \- without allowing a policy to override arbitrary in-cluster destinations.

### Multi-port matching depends on implicit names

The current API leans on port declarations in 3 key touch points.

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
* Multi-port mapping does not work unless the pod has a matching `port[]` entry, even if an application is listening on the expected port.  
* The port mapping logic differs between single-port and multi-port configurations, and cases where `serviceMatcher.toPorts[]` is omitted.  
* There are special cases that depend on port list ordering. Incorrect ordering between frontend, backend and pod spec can lead to mapping failures.

CCLRP should express the relationship directly as `port` to `targetPort`, following the general Kubernetes Service model. Port names should not be required to connect the frontend and backend sides of a redirect.

### Loop prevention exposes datapath details

The `skipRedirectFromBackend` field exposes an implementation detail of the socket load balancer and requires users to make the same runtime decision independently for every policy. However, there is no known useful purpose for redirecting a backend's traffic back to itself.

Loop prevention should be an agent-level behaviour, enabled when the required kernel capability is available and reported when it is not.

## Goals

* Make the cluster-wide impact of local redirects explicit in the API.  
* Provide a safe default scope for address-based redirects.  
* Replace implicit port-name joins with explicit frontend-to-backend port mappings.  
* Remove datapath loop-prevention configuration from individual policies.  
* Reduce silent failure modes through validation and observable reconciliation errors.  
* Preserve the existing local redirect use cases, especially node-local DNS and host-provided link-local services.  
* Allow CCLRP and CLRP to coexist during adoption of the new resource.

## Non-Goals

* Redesign the eBPF local redirect datapath.  
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

Because CCLRP is cluster-scoped, selectors are evaluated across namespaces. The current proposal follows the cluster-wide policy convention of restricting a selector with the `io.kubernetes.pod.namespace` label. The safety of relying on that convention is discussed in a key questions below.

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

CCLRP does not expose `skipRedirectFromBackend` within the API.

Instead, the agent should determine whether the host kernel supports the mechanism required to identify backend-originated traffic at runtime, and enable it for all CCLRPs if available. Otherwise, the agent should log a warning to report that loop prevention is not available.

*Note: `skipRedirectFromBackend` field will still remain opt-in for CLRP.*

The safety of this change is not guaranteed. The original implementation of CLRP was augmented with loop prevention via socket lookups (`bpf_sk_lookup_*` helpers) in October 2020. This appears to have suffered from incorrect behaviour under certain scenarios and was replaced with the current skip-map implementation in April 2024.

At the time of the original augmentation, queries were raised about any known use cases that may require a request from an LRP backend to be sent back to that same backend. The original contributor confirmed there were no known use cases.

At the time of writing, when reviewing this feature within the context of the aforementioned use cases above, it is believed there are still no known use cases that would require a request from an LRP backend to be sent back to that same backend.

Original socket lookup implementation: [https://github.com/cilium/cilium/pull/13287](https://github.com/cilium/cilium/pull/13287)  
Revised "skip-map" implementation: [https://github.com/cilium/cilium/pull/26144](https://github.com/cilium/cilium/pull/26144)

### Validation and conflicts

The CCLRP controller should reject a policy that has:

* neither or both of `addressMatcher` and `serviceMatcher`  
* an address outside the link-local ranges  
* an invalid port, protocol, or port mapping  
* a service reference that cannot be resolved  
* a selector or mapping that cannot produce a valid local redirect frontend

Policies that attempt to claim the same concrete frontend must have deterministic behaviour. The initial implementation should allow only one active owner for a frontend within the same policy type and report conflicts as errors.

CCLRP should take precedence over CLRP when both resources identify the same frontend, so that an explicitly migrated CCLRP is not shadowed by the legacy policy while both resources coexist.

The precise status representation is an implementation detail for the initial release, but validation and reconciliation failures must be visible through agent diagnostics, logs, and events or equivalent Kubernetes-facing feedback. A full status API is a possible follow-up milestone.

### Observability

The existing local redirect debugging commands should be extended to identify:

* whether an entry came from CLRP or CCLRP;  
* the policy that owns the entry;  
* the resolved frontend address and port;  
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

The implementation will add the CCLRP CRD, permissions, and agent reconciliation required to translate policy intent into the existing load-balancer frontend and backend state.

The implementation should keep the following boundaries clear:

1. The Kubernetes-facing layer validates and reflects CCLRP intent.  
2. A reconciliation layer resolves Services, ports, and node-local backend endpoints.  
3. The existing load-balancer integration applies concrete local redirect entries.

The internal tables, Go structures, and controller decomposition may evolve during implementation. They should not become additional user-facing API commitments in this CFP.

A prototype implementation will be used to validate the API and reconciliation behaviour with tests covering address matching, Service matching, namespace-restricted selectors, explicit port translation, invalid configuration, backend churn, nodes without local backends preserving the original frontend behaviour, and kernel capability handling.

The initial CRD will be implemented with the `cilium.io/v2alpha1` API version. During the alpha phase, CLRP will remain supported but feature-frozen, except for critical fixes and changes required to enable migration to CCLRP.

The CCLRP CRD will only be promoted to the `cilium.io/v2` API version once it is stable and migration-ready, at which point the existing CLRP CRD will be marked as deprecated.

It is expected that both promotion and deprecation will be implemented in the same Cilium minor release. CCLRP and CLRP can continue to co-exist for several additional minor releases, providing sufficient time for administrators to migrate to the new resource.

It is expected that CLRP will be removed in a future Cilium minor release; this scheduling is outside the scope of this document.

### Migration approach

There will be no automatic object conversion from CLRP to CCLRP.

Administrators will need to translate an existing CLRP into a CCLRP and explicitly add any namespace restriction that was previously implied by the namespaced resource. For example, a backend selector that was previously limited to `kube-system` must include `io.kubernetes.pod.namespace: kube-system` in the CCLRP selector.

It is reasonable to expect the intent of a CLRP, re-declared via CCLRP, to reconcile to the same final state (subject to any behavioural differences such as loop prevention).

CLRP and CCLRP resources will be able to co-exist to facilitate migration. The initial proposal gives CCLRP precedence when both resource types attempt to claim the same concrete frontend. This avoids depending on creation order, allowing a CCLRP to override the effective behaviour of an existing CLRP. Reconciled forwarding state, previously derived from the CLRP, is expected to transition to the CCLRP.

Administrators will need to verify the CCLRP operation before deciding whether to proceed by removing the legacy CLRP, or roll back by deleting the CCLRP.

Migration documentation will provide a mechanical conversion checklist and explain how to roll out a CCLRP before removing its corresponding CLRP. The expected behaviour will be documented, surfaced through diagnostics, and covered by upgrade and migration tests.

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

### Key Question: should address matching support multiple IPs?

The proposal keeps one IP per `addressMatcher`. This means a dual-stack configuration uses two resources, but each resource has simple validation and conflict semantics.

Supporting multiple IPs in one resource would improve convenience, but would require defining all-or-nothing behaviour, conflict reporting, partial failure, and update/deletion semantics.

The initial proposal defers that extension until a concrete use case requires it. The only theorised use case at the time of writing is a non-Kubernetes application that may also need to be exposed on IPv6 alongside IPv4.

### Key Question: should port mappings be top-level or nested?

This proposal places `ports` at the policy level because each entry relates the selected frontend to the selected local backend, regardless of whether the frontend was identified by a Service or an address. This also makes the mapping visible and removes the existing nested `redirectFrontend` and `redirectBackend` wrappers.

The trade-off is that the relationship between `ports.port` and the frontend matcher, and between `ports.targetPort` and the backend selector, is less visually nested. The API documentation and examples should explain that relationship directly.

### Key Question: should CCLRP deprecate inherited all-port behaviour?

The current proposal allows `ports` to be omitted for a `serviceMatcher`, in which case all supported Service ports are redirected and their mappings are inherited from the Service. This preserves a convenient form of the existing API, but it conflicts with the goal of making frontend-to-backend mappings explicit.

It also means that adding a new port to a Service can change the effective scope of a redirect without changing the CCLRP. That may be surprising for a cluster-scoped resource.

#### Option 1: Retain inherited all-port behaviour

Pros:

* Keeps the node-local DNS configuration concise.  
* Automatically includes new Service ports.  
* Preserves the current convenience behaviour for ServiceMatcher users.

Cons:

* The effective set of redirected ports is implicit and can change over time.  
* It weakens the explicit `port` to `targetPort` contract.  
* It makes auditing and migration less predictable.

#### Option 2: Deprecate inherited all-port behaviour

Require `ports` for CCLRP, while retaining the legacy omission behaviour for CLRP during the transition period.

Pros:

* Every CCLRP declares its redirected frontend ports explicitly.  
* Adding a Service port cannot silently broaden an existing redirect.  
* It gives the new API one consistent port model.

Cons:

* Policies are more verbose, especially for Services with many ports.  
* Administrators must update CCLRP when they want a newly added Service port redirected.  
* Migration requires enumerating the Service ports that were previously inherited.

The proposed direction is to deprecate inherited all-port behaviour for CCLRP and retain it only for legacy CLRP compatibility until CLRP itself is deprecated.

### Key Question: how should inherited Service target ports be resolved?

If inherited Service mappings remain supported, Kubernetes Service port and `targetPort` semantics must remain understandable.

* Numeric target ports should map directly.  
* Named target ports need to be resolved according to Kubernetes conventions against the selected local backend pods.

The implementation must define and test behaviour when a named target port is not present on a selected backend pod. The result should be an observable validation or reconciliation error, not a silently omitted mapping.

## Future Milestones

### CCLRP status conditions

Add a stable status model for accepted, rejected, partially resolved, and backend-unavailable conditions, including the concrete frontends and node-local backends produced by a policy.

### CCLRP promotion and CLRP deprecation

The CCLRP CRD will be promoted to stable and the legacy CLRP CRD will be marked as deprecated in the same future Cilium minor release. See Implementation approach for further details.

### CLRP removal

The CLRP CRD will be scheduled for removal from the codebase after several additional Cilium minor releases. See Implementation approach for further details.  
