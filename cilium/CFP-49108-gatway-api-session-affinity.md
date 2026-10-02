# CFP-49108: Gateway API Session Affinity (Consistent-Hash Load Balancing)

**SIG:** SIG-ServiceMesh

**Begin Design Discussion:** 2026-09-30

**Cilium Release:** 1.21

**Authors:** Renars G. <rgrinbergs@evolution.com>

**Status:** Draft

## Summary

Cilium's Gateway API implementation (north/south `Gateway` routes and GAMMA
`Service`-parented routes) load balances every backend with Envoy's
`ROUND_ROBIN` policy. There is no way to route requests that share an
application-supplied key (an HTTP header, cookie, query parameter, or path
segment) to the same backend Pod, and no client-address affinity in the L7
paths either: `Service.spec.sessionAffinity: ClientIP` is honored by the eBPF
datapath but lost once traffic is redirected to Envoy.

This proposal adds *session affinity* as defined by
[GEP-1619](https://gateway-api.sigs.k8s.io/geps/gep-1619/#the-relationship-of-session-persistence-and-session-affinity)
(hash-based, "weak" affinity, as distinct from the cookie-encoded *session
persistence* delivered by CFP-47089), implemented with Envoy's consistent-hash
load balancers (Maglev, ring hash) and a cluster-level `hash_policy`
(`HttpProtocolOptions.hash_policy`, Envoy 1.37+; Cilium ships cilium-envoy
1.38.4). The property delivered is *key affinity*: the same
application-supplied key reaches the same Pod regardless of which caller sends
it or which node it enters on.

Upstream Gateway API has no API for load-balancing algorithms or session
affinity yet. The proposal is therefore split into an API-independent part
(internal model and Envoy translation) and a small user-facing surface that
extends the existing, documented `service.cilium.io/lb-l7-algorithm` Service
annotation and adds one companion annotation for the hash key. That surface is
expected to be superseded by the upstream `Backend` resource once it can
represent in-cluster destinations. The internal model is built so that the
migration changes the ingestion only.

## Motivation

Actor and entity-sharding systems (Akka Cluster Sharding, Orleans, Dapr actors)
are the representative workload that needs "same key, same Pod" routing without
a strong guarantee. Each entity is owned by one Pod at a time, and ownership is
established lazily: the Pod that receives the first request for an entity,
identified by a sharding key in the request (for example
`?shardKey=order-8813`), activates it. Every later request for that entity,
from any service, should reach the owning Pod. Under round robin the framework
must, for each misrouted request, either forward it internally to the owner (an
extra hop, extra connections, a cluster-wide ownership lookup on the hot path)
or re-allocate the actor to the receiving Pod (deactivate, persist state,
re-activate elsewhere, invalidate cached ownership lookups). Consistent hashing
on the sharding key makes the first hop correct in steady state for every
caller, because the mapping depends only on the key and the endpoint set. When
Pods are added or removed, only the entities whose key remaps (about 1/N)
change owner, which is the rebalancing these frameworks already implement.

This traffic is service-to-service (east/west): clients are not browsers, and a
natural key is already on the request. The cookie-encoded session persistence
from CFP-47089 does not cover it: it pins each caller to whichever Pod its first
request reached, so two services asking for the same entity are pinned
independently (see "Key affinity versus caller affinity"). Its other drawbacks
for east/west traffic (a cookie round trip, a Pod address in a client-visible
token, no GAMMA support today) are secondary.

### Key affinity versus caller affinity

Session persistence: the routing key is minted by the proxy. A caller's first
request is load balanced normally and the response carries a cookie encoding
the chosen Pod's address; later requests from that caller present the cookie
and return to that Pod. The application's own key is not consulted. The mapping
is *caller session → Pod*, established randomly.

Consistent hashing: the routing key is supplied by the request (header, query
parameter, client-set cookie, path segment), and the Pod is a deterministic
function of the key and the current endpoint set. The caller is irrelevant. The
mapping is *key → Pod*, identical on every node.

Services `orders` and `billing` both call `cart` for tenant `42`:

| Mechanism                                   | `orders` → `cart` Pod                 | `billing` → `cart` Pod                | Same Pod for tenant `42`?                                  |
| ------------------------------------------- | ------------------------------------- | ------------------------------------- | ---------------------------------------------------------- |
| Round robin (today)                         | any                                   | any                                   | No                                                         |
| Session persistence (CFP-47089)             | the Pod its first request reached     | the Pod *its* first request reached   | Only by chance (1/M for M Pods); the tenant is not consulted |
| Consistent hash on `x-tenant-id` (this CFP) | `hash(42)`                            | `hash(42)`                            | Yes, from any caller on any node                           |

GEP-1619's "strong" and "weak" describe the durability of a single caller's
pinning, not cross-caller convergence: the probability that N callers with the
same key land on one Pod is 1/M^(N-1) under persistence and 1 under hashing.

The same distinction applies within consistent hashing. A per-caller hash
source (downstream source IP, or a cookie the proxy generates when none is
present) gives caller affinity only; the hash-source table below states this
per source. Source-IP hashing is the degenerate case: the key is unique per
caller, so the effect is that of client-IP session persistence (GEP-3798,
`Service.spec.sessionAffinity: ClientIP`), differing in mechanism only. Hashing
is stateless and deterministic (the same client address maps to the same Pod
on every node and across proxy restarts; about 1/N of clients remap when the
endpoint set changes), whereas client-IP persistence keeps a server-side stick
table (random first assignment, held for a configured duration, unaffected by
endpoints being added). Both depend on the address being unique per client:
clients behind NAT, hostNetwork Pods sharing a node address, or requests
arriving through an external load balancer share the key and therefore share a
Pod.

### State of Cilium

- `operator/pkg/model/translation/envoy_cluster.go` applies
  `withClusterLbPolicy(Cluster_ROUND_ROBIN)` to every HTTP and TCP cluster,
  including the `http:`/`grpc:`-prefixed ext_authz clusters and the clusters of
  request-mirror targets. Every HTTP cluster already carries
  `envoy.extensions.upstreams.http.v3.HttpProtocolOptions`, whose
  `hash_policy` field (Envoy 1.37+) is unused. No `hash_policy` is emitted
  anywhere. The agent qualifies cluster names with the CEC namespace and name
  (`pkg/ciliumenvoyconfig/cec_resource_parser.go`), so a Service referenced
  from N Gateways or GAMMA parents has N identical clusters per node.
- CFP-47089 / cilium/cilium#48029 added cookie-based session persistence on
  `HTTPRoute`/`GRPCRoute` rules and rejects it for GAMMA routes with
  `Accepted=False` / `UnsupportedValue`.
- The Proxy Load Balancing beta (`service.cilium.io/lb-l7=enabled`) documents
  `service.cilium.io/lb-l7-algorithm` with values `round_robin`,
  `least_request`, `random`. The implementation
  (`operator/pkg/ciliumenvoyconfig/annotations.go`) maps the value onto the
  Envoy `Cluster.LbPolicy` enum via `Cluster_LbPolicy_value[strings.ToUpper(v)]`:
  `maglev` and `ring_hash` are accepted but, with no `hash_policy`, Envoy
  selects hosts at random; unknown values map to `0` (`ROUND_ROBIN`) silently;
  `CLUSTER_PROVIDED` and `LOAD_BALANCING_POLICY_CONFIG` are accepted. This path
  (`operator/pkg/ciliumenvoyconfig/envoy_config.go`) builds its Listener,
  RouteConfiguration and Cluster directly, does not use `operator/pkg/model`,
  and is the only consumer of Helm `loadBalancer.l7.algorithm`
  (`--loadbalancer-l7-algorithm`).
- The eBPF datapath offers per-Service algorithm selection (CFP-34577,
  `service.cilium.io/lb-algorithm`); it hashes the L4 tuple and does not apply
  to traffic redirected to Envoy.
- Cilium pins `cilium-envoy:v1.38.4` (`images/cilium/Dockerfile`).
  `cilium/proxy` compiles `envoy.load_balancing_policies.maglev`, `ring_hash`,
  `least_request` and `random`
  (`envoy_build_config/extensions_build_config.bzl`) and already emits
  `SafeRegex` matchers and `RegexMatchAndSubstitute` rewrites, so the RE2
  engine is present. Envoy 1.37 added cluster-level
  `HttpProtocolOptions.hash_policy`, 1.36 `WeightedCluster.use_hash_policy`,
  and since 1.35 request headers are finalized before host selection. No
  dataplane build change is required.
- A hand-written `CiliumEnvoyConfig` can express a `MAGLEV` cluster with a
  `hash_policy` today but cannot coexist with the CEC the operator generates
  for the same Service (cilium/cilium#33767).

### State of upstream Gateway API

- GEP-1619 defines *session persistence* (strong; backend identifier encoded in
  a cookie or header) and *session affinity* (weak; best-effort, typically
  deterministic hashing), and a two-tier data-plane decision: honor a
  persistence identity if present, otherwise load balance taking affinity into
  account. It left room for an affinity API but it was never specified. Requests
  for a standard load-balancing policy (kubernetes-sigs/gateway-api#992, #1778)
  did not progress. GEP-3798 (client IP-based session persistence, a
  server-side stick table with a duration and optional subnet mask) targeted
  Experimental in v1.4.0 and is Deferred; it may be withdrawn or folded into
  GEP-1619's alternatives.
- kubernetes-sigs/gateway-api discussion #4462 (January–June 2026) made the
  `Backend` resource (GEP-4894, formerly GEP-4488) the primary attachment
  point for session persistence (#4876, merged); route-inline and
  `XBackendTrafficPolicy` fields are to be deprecated once `Backend`
  persistence reaches Standard. GEP-4894 lists load-balancing algorithm
  selection, including consistent hashing, as future inline `Backend`
  configuration.
- `XBackend` on `main` (2026-09-30): `type` (`ExternalHostname` |
  `EndpointSelector`), `port.number`, `endpointSelector.matchLabels`,
  `sessionPersistence` (top level of `spec`; valid only for
  `type: EndpointSelector`), `protocol`, `tls`. `EndpointSelector` resolves
  through an implementation-created stop-gap Service (#5255, merged
  2026-09-14) until the upstream `EndpointSelector` resource (KEP-6116)
  exists; #5298 (open) proposes nesting `sessionPersistence` under
  `spec.endpointSelector`. Cilium does not implement `Backend`. A portable API
  for this feature is a 2027+ item; CFP-47089 shipped route-inline persistence
  under the same reasoning, with maintainers accepting the later churn
  (cilium/cilium#47089).
- Other implementations expose consistent hashing in three placements, none
  portable:
  - *Destination-side, algorithm and key together:* Istio
    (`DestinationRule.trafficPolicy.loadBalancer.consistentHash`: header /
    cookie / query parameter / source IP; ring hash or Maglev), Kong
    (`KongUpstreamPolicy` referenced from a Service annotation:
    `algorithm: consistent-hashing`, `hashOn` plus `hashOnFallback`), kgateway
    (`BackendConfigPolicy.spec.loadBalancer.<ringHash|maglev>.hashPolicies`
    targeting the Service, ordered with `terminal`), NGINX Gateway Fabric
    (`UpstreamSettingsPolicy` on the Service: `ip_hash`, or `hash` /
    `hash consistent` with `hashMethodKey` set to any NGINX variable), GKE
    (`GCPTrafficDistributionPolicy` on the Service, GA 2026-08-31:
    `HEADER_FIELD`, `CLIENT_IP`, cookie types, `RING_HASH`; its route-rule
    `GCPSessionAffinityFilter` is stateful cookie persistence, not hashing).
  - *Route-side, algorithm and key together:* Envoy Gateway
    (`BackendTrafficPolicy` targeting an `HTTPRoute` or `Gateway`;
    `loadBalancer.consistentHash` with SourceIP / Header / Cookie; Maglev
    only, ring hash requested in envoyproxy/gateway#8467).
  - *Split, mirroring Envoy's cluster/route split:* Azure Application Gateway
    for Containers (`BackendLoadBalancingPolicy` with `strategy: ring-hash` on
    the Service; cookie affinity in a `RoutePolicy` targeting the `HTTPRoute`).
  No implementation has hashing fields inline in `HTTPRoute`. Upstream's
  direction (`Backend`) is destination-side; this CFP proposes that placement
  and works out the route-side placement as Option D in Key Question 1.

## Goals

- Consistent-hash load balancing (Maglev or ring hash) for backend Services
  reached through Cilium Gateway API routes and GAMMA routes.
- Key affinity: requests carrying the same application-supplied key reach the
  same Pod regardless of caller or ingress node, for as long as the endpoint
  set is unchanged.
- Hash key from an HTTP request header, a cookie, a query parameter, a capture
  group of the request path, or the downstream source IP, with documentation of
  which sources give key affinity and which give caller affinity only.
- One meaning for `service.cilium.io/lb-l7-algorithm` across Gateway API,
  GAMMA and Proxy Load Balancing; `maglev` / `ring_hash` actually consistent in
  all three.
- The same semantics for ClusterMesh backends, both global Services
  (`service.cilium.io/global`) and MCS-API `ServiceImport`s, including key
  affinity across clusters whenever the clusters select the same endpoint set.
- GEP-1619 two-tier semantics when a route also configures session
  persistence: persistence identity first, hash second.
- Invalid configuration reported through route status and Kubernetes events.
- Internal model and Envoy translation independent of the user-facing API, so
  a future `Backend`-based API changes ingestion only.

## Non-Goals

- A new Cilium CRD or Cilium-specific policy-attachment resource.
- New fields on upstream CRDs, or Cilium annotations on `HTTPRoute` /
  `GRPCRoute`.
- Implementing the upstream `Backend` resource (Future Milestones).
- Strong persistence guarantees. Consistent hashing remaps a fraction of keys
  when the endpoint set changes.
- Consistency across weighted `backendRefs` in the first version. The hash
  selects an endpoint *within* a backend Service; selection *between* weighted
  backends stays weighted-random. Envoy 1.36+ can hash that choice too
  (`WeightedCluster.use_hash_policy`; Future Milestones).
- A Helm-level default algorithm for Gateway API and GAMMA clusters. Helm
  `loadBalancer.l7.algorithm` applies to Proxy Load Balancing only, as today;
  Gateway API and GAMMA clusters default to `ROUND_ROBIN` (Future Milestones).
- Changes to eBPF (L4) load balancing or `service.cilium.io/lb-algorithm`.
- Cross-cluster key affinity when clusters intentionally select different
  endpoint sets (`service.cilium.io/affinity: local|remote`,
  `service.cilium.io/shared: "false"`). Affinity is then per cluster.
- Locality-weighted distribution between local and remote endpoints inside
  Envoy. Cilium emits a flat endpoint list without locality or priority; the
  local/remote choice is made before EDS by the existing ClusterMesh backend
  selection (Future Milestones).
- ClusterMesh cluster-aware addressing (overlapping PodCIDRs) in the L7 path.
- Signed or encrypted cookies, cookie security attributes, header-based session
  persistence.

## Proposal

### Overview

1. **User-facing configuration.** Two annotations on the *backend* Service
   (the Service named in `backendRefs`; the GAMMA backend facet), which maps
   1:1 onto the Envoy cluster Cilium generates. A GAMMA `parentRef` Service is
   an interception point and carries no load-balancing configuration (see
   "GAMMA: frontend facet versus backend facet").
   - `service.cilium.io/lb-l7-algorithm` (existing): documented values
     `maglev` and `ring_hash` added.
   - `service.cilium.io/lb-l7-hash-policy` (new): the hash key.
   For ClusterMesh, the same annotations are read from each cluster's Service
   copy (global Services) or arrive through `ServiceExport.spec.exportedAnnotations`
   (MCS-API); see "ClusterMesh: global Services and MCS-API".
2. **Internal model.** `model.Backend` gains a `LoadBalancing` attribute
   (algorithm and hash policy), populated during ingestion like `AppProtocol`
   and `TLS`.
3. **Envoy translation.** Both on the cluster: algorithm → `lb_policy` (plus
   Maglev / ring-hash config); hash key → `HttpProtocolOptions.hash_policy`
   (Envoy 1.37+), which Envoy consults before any route-level policy. Routes
   are unchanged. One cluster mutator serves the Gateway API / GAMMA translator
   and the Proxy Load Balancing reconciler.

The configuration attaches to the Service and is applied to every route
cluster Cilium generates for that `namespace:name:port` (one per referencing
CEC), so there is no route-vs-route conflict and all copies are identical.

### User-facing configuration

Callers address Service `cart`. A producer route attached to `cart` forwards to
Service `cart-v1`, whose Pods are selected by consistent hash on
`x-tenant-id`. The annotations go on `cart-v1`; `cart` is an ordinary Service
without Cilium annotations and is omitted.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cart-v1
  namespace: shop
  annotations:
    service.cilium.io/lb-l7-algorithm: maglev
    service.cilium.io/lb-l7-hash-policy: "header:x-tenant-id"
spec:
  ports:
  - name: http
    port: 8080
    appProtocol: http
  selector:
    app: cart
    version: v1
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: cart
  namespace: shop
spec:
  parentRefs:
  - group: ""
    kind: Service
    name: cart          # parent: traffic to cart's ClusterIP is intercepted here
    port: 8080
  rules:
  - backendRefs:
    - name: cart-v1     # backend: cart-v1's endpoints are hashed over
      port: 8080
```

Every request to `cart:8080` carrying the same `x-tenant-id` reaches the same
`cart-v1` Pod, from any caller on any node, while `cart-v1`'s endpoint set is
unchanged.

The same annotations on the same Service have the same effect when the Service
is a `backendRef` of a north/south `HTTPRoute` behind a `Gateway`, or carries
`service.cilium.io/lb-l7=enabled`.

#### `service.cilium.io/lb-l7-algorithm`

| Value           | Envoy `lb_policy` | Notes                                                                     |
| --------------- | ----------------- | ------------------------------------------------------------------------- |
| `round_robin`   | `ROUND_ROBIN`     | Current default; unchanged.                                               |
| `least_request` | `LEAST_REQUEST`   | Already accepted by Proxy Load Balancing.                                 |
| `random`        | `RANDOM`          | Already accepted by Proxy Load Balancing.                                 |
| `maglev`        | `MAGLEV`          | Consistent. Requires `lb-l7-hash-policy`; Envoy default table size 65537. |
| `ring_hash`     | `RING_HASH`       | Consistent. Requires `lb-l7-hash-policy`; Envoy default ring size 1024.   |

Values are case-insensitive, matching Proxy Load Balancing. `maglev` and
`ring_hash` without `lb-l7-hash-policy` are rejected (see Validation) instead
of degrading to random.

#### `service.cilium.io/lb-l7-hash-policy`

One hash source per Service, `<type>[:<name>][;<attr>=<value>...]`:

| Value                                          | Envoy `hash_policy`                              | Affinity provided                                                                                       |
| ---------------------------------------------- | ------------------------------------------------ | ------------------------------------------------------------------------------------------------------- |
| `header:<name>`                                | `header { header_name }`                         | Key: same header value, same Pod, from any caller                                                       |
| `query-parameter:<name>`                       | `query_parameter { name }`                       | Key: same parameter value, same Pod, from any caller                                                    |
| `cookie:<name>`                                | `cookie { name }`                                | Key if the application sets the cookie to a shared value; caller only if it is a per-client session ID  |
| `cookie:<name>;ttl=<duration>[;path=<path>]`   | `cookie { name, ttl, path }`                     | Caller: the proxy generates a random per-caller value when the cookie is absent                         |
| `source-ip`                                    | `connection_properties { source_ip: true }`      | Caller: one client Pod (or last hop), one backend Pod                                                   |
| `path-regex:<RE2 pattern>`                     | `header { header_name: ":path", regex_rewrite }` | Key: first capture group of the pattern applied to the request path; same capture, same Pod, from any caller |

Semantics:

- `header`, `query-parameter` and `cookie` hash the client-sent value. If the
  key is absent, Envoy computes no hash and the consistent-hash balancer
  selects a host at random for that request (Envoy's documented behavior).
- Key affinity requires a value chosen by the application and shared by every
  caller that should converge: a tenant, user, account or shard identifier in a
  header, query parameter, or application-set cookie.
- `cookie` with `ttl`: Envoy generates and `Set-Cookie`s a random value when
  the cookie is absent, so later requests hash consistently. Without `ttl`,
  only client-provided cookies are hashed. This is the Envoy Gateway and Istio
  contract. The generated value is a random hash seed, not a Pod address, but
  it is minted per caller and therefore yields caller affinity only.
- `source-ip` hashes the downstream address. Cilium delivers GAMMA and Gateway
  traffic to Envoy via TPROXY, so for GAMMA this is the client Pod IP; for
  north/south traffic behind an external load balancer it is the last hop, and
  all clients behind that hop share one Pod. `X-Forwarded-For` is not consulted
  in the first version. This source gives caller affinity, equivalent in effect
  to client-IP session persistence (see "Key affinity versus caller
  affinity"); it is included because every surveyed implementation offers it
  and because it is the basis for honoring `Service.spec.sessionAffinity:
  ClientIP` in L7 paths (Future Milestones).
- `path-regex` extracts the key from the request path (REST-style
  `/orders/{id}/...`, gRPC method paths). Everything after the first `:` in
  the annotation value is the RE2 pattern verbatim (`;` and `=` are not
  special for this type). The key is the first capture group of the first match
  anywhere in the path; the pattern must contain at least one capture group.
  Envoy supports this through `regex_rewrite` on the header hash policy applied
  to `:path`. Differences from the other sources:
  - The path is always present, so there is no "key absent, load balance
    randomly" case. A path that does not match hashes on its full path:
    deterministic, but all requests for one non-matching path (a health-check
    endpoint) land on one Pod. Scope such rules to their own `backendRefs`, or
    write a pattern that matches every path the Service serves.
  - Envoy's `:path` includes the query string; exclude `?` from captured
    segments (`[^/?]+`).
  - Since Envoy 1.35 request headers are finalized (`URLRewrite` path
    rewrites, `RequestHeaderModifier`) before host selection, so the pattern
    is matched against the path as sent upstream. A rule that rewrites the
    path and forwards to a hashed backend must write the pattern against the
    rewritten form.
  - The pattern is validated at ingestion with Go's RE2-compatible `regexp`
    (compiles; at least one capture group) so Envoy never rejects the
    configuration. Envoy's RE2 program-size limit applies as for every regex
    Cilium emits.

One hash source per Service matches Envoy Gateway and Istio; kgateway and Kong
offer ordered fallbacks. Ordered fallback chains are a Future Milestone.

#### Interaction with existing annotations

- The annotations are evaluated only when Cilium generates a route cluster for
  the Service: as a `backendRef` or `RequestMirror` target of a Gateway API or
  GAMMA route, or with `service.cilium.io/lb-l7=enabled`. On a Service that
  Cilium does not proxy at L7 they have no effect and no event is emitted. A
  Service that appears only as a GAMMA `parentRef` gets no cluster and is not
  evaluated. The `http:`/`grpc:`-prefixed ext_authz clusters of an
  `ExternalAuth` Service keep `ROUND_ROBIN`.
- `service.cilium.io/lb-algorithm` (eBPF, CFP-34577) is unrelated and still
  governs the L4 datapath; both may be set on one Service.
- Helm `loadBalancer.l7.algorithm` remains the default for Proxy Load
  Balancing Services without `lb-l7-algorithm`. Gateway API and GAMMA clusters
  default to `ROUND_ROBIN`, as today (Non-Goals).

#### ClusterMesh: global Services and MCS-API

Remote endpoints already reach Envoy through the path this proposal hashes
over. `computeLoadAssignments` in `pkg/ciliumenvoyconfig/controller.go` builds
each cluster's EDS assignment from `writer.SelectBackends(txn, bes, svc, nil)`.
With ClusterMesh enabled, `pkg/clustermesh/loadbalancer/selectbackends.go`
replaces that selection function and applies the ClusterMesh annotations
before EDS:

| Annotation on the Service                     | Backends Envoy receives                                                                     |
| --------------------------------------------- | ------------------------------------------------------------------------------------------- |
| no `service.cilium.io/global`                 | local only                                                                                  |
| `global: "true"`, `affinity` unset or `none`  | local and remote                                                                            |
| `global: "true"`, `affinity: local`           | local; remote only while no local backend is active                                         |
| `global: "true"`, `affinity: remote`          | remote; local only while no remote backend is healthy                                       |
| `global: "true"`, `shared: "false"`           | as above for what this cluster consumes; this cluster's backends are not exported to others |

The result is one `LocalityLbEndpoints` entry without locality or priority,
addressed by Pod IP, which is unique across the mesh (non-overlapping
PodCIDRs). Envoy therefore hashes over exactly the set the eBPF datapath would
use, and no change is needed for consistent hashing to include remote
endpoints. Key affinity across clusters follows from the property in
"Behavior and operational notes": the hash is a function of the key and the
selected endpoint set, so two clusters that select the same set map a key to
the same Pod, wherever the Pod runs.

| Placement of Pods and annotations                                | Cluster A hashes over | Cluster B hashes over | Key affinity                                                                              |
| ---------------------------------------------------------------- | --------------------- | --------------------- | ----------------------------------------------------------------------------------------- |
| A has no Pods, B has Pods (any `affinity`)                       | B's Pods              | B's Pods              | Across clusters                                                                           |
| Pods in both, `affinity` unset or `none`                         | A ∪ B                 | A ∪ B                 | Across clusters; keys owned by the other cluster cross the mesh on every request           |
| Pods in both, `affinity: local`                                  | A                     | B                     | Per cluster: one owner per key per cluster                                                |
| Pods in both, `affinity: local`, no A Pod ready                  | B                     | B                     | Across clusters while A has no ready Pod; every key seen by A remaps at failover and at recovery |
| Pods in both, `shared: "false"` on B                             | A                     | A ∪ B                 | None across clusters (asymmetric by design)                                               |

The first row is the pattern of a cluster that hosts only a Service object
(cilium/cilium#40956: with global Services, the Service must exist in every
cluster that consumes it, even without local Pods). Rows three to five are
configurations in which clusters select different sets on purpose; hashing is
still consistent within each cluster, and the Non-Goals state that
cross-cluster affinity is not provided there. Row four follows Cilium's view
of health (`be.State`, `be.Unhealthy` in `selectbackends.go`): Pods that are
not ready or are terminating leave the set. Failures only Envoy observes
(connection errors, 5xx) cause per-node outlier ejection and do not add remote
backends.

Where the annotations are read differs by ClusterMesh model:

- **Global Services.** Each cluster evaluates its own Service object, the one
  its routes name in `backendRefs`. The two annotations must be identical on
  every cluster's copy. Cilium has no cross-cluster view of Service
  annotations (the shared `ClusterService` carries labels, `includeExternal`
  and `shared`, but no annotations), so drift is not detected: a cluster whose copy lacks `lb-l7-hash-policy` load balances
  round robin while the others hash, and cross-cluster affinity breaks
  silently (Key Question 7).
- **MCS-API.** A `backendRef` of kind `ServiceImport` resolves to the derived
  Service (`derived-<hash>`) that
  `pkg/clustermesh/mcsapi/service_controller.go` creates, and
  `backendToModelBackend` receives that derived Service. Its annotations are a
  copy of the `ServiceImport`'s, which are the
  `ServiceExport.spec.exportedAnnotations` of the oldest `ServiceExport`
  (`serviceimport_controller.go`); differing exports are reported as
  `AnnotationsConflict` on the `ServiceExport` status. Setting the two
  annotations in `exportedAnnotations` therefore gives one source of truth and
  conflict detection with no new code. Annotations placed directly on a
  derived Service are overwritten on reconcile. `ServiceImport` backends are
  supported on Gateway routes; GAMMA ingestion (`ingestion/gamma.go`) does not
  resolve `ServiceImport`s, so GAMMA across clusters uses the global-Service
  model.

Global-Service example. Cluster B runs the Pods; cluster A holds only the
Service object and the route. The Service manifest is applied to both
clusters unchanged:

```yaml
# Both clusters
apiVersion: v1
kind: Service
metadata:
  name: cart-v1
  namespace: shop
  annotations:
    service.cilium.io/global: "true"
    service.cilium.io/lb-l7-algorithm: maglev
    service.cilium.io/lb-l7-hash-policy: "header:x-tenant-id"
spec:
  ports:
  - name: http
    port: 8080
    appProtocol: http
  selector:
    app: cart
    version: v1
---
# Cluster A (and B, if its callers should be routed the same way)
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: cart
  namespace: shop
spec:
  parentRefs:
  - group: ""
    kind: Service
    name: cart          # local frontend facet; intercepted in the cluster of the caller
    port: 8080
  rules:
  - backendRefs:
    - name: cart-v1     # local object; endpoints come from cluster B via ClusterMesh
      port: 8080
```

MCS-API example (Gateway routes only). The exporting cluster owns the
configuration; importing clusters receive it through the `ServiceImport`:

```yaml
# Exporting cluster
apiVersion: multicluster.x-k8s.io/v1beta1
kind: ServiceExport
metadata:
  name: cart-v1
  namespace: shop
spec:
  exportedAnnotations:
    service.cilium.io/lb-l7-algorithm: maglev
    service.cilium.io/lb-l7-hash-policy: "header:x-tenant-id"
---
# Any cluster
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: cart
  namespace: shop
spec:
  parentRefs:
  - name: my-gateway
  rules:
  - backendRefs:
    - group: multicluster.x-k8s.io
      kind: ServiceImport
      name: cart-v1
      port: 8080
```

Failure behavior:

- ClusterMesh propagates endpoint changes through the clustermesh-apiserver or
  kvstore, so clusters see a change at slightly different times; the sets
  diverge briefly, as nodes within one cluster do.
- Envoy outlier detection ejects an endpoint only on the nodes whose Envoy
  fails to reach it. During an inter-cluster network partition, the nodes that
  lost connectivity hash over the endpoints they can still reach while the
  others keep the full set; the sets converge when the partition heals. Keys
  remap on the partitioned nodes twice, as in the `affinity: local` failover
  row.
- `source-ip` hashing is unaffected: client Pod addresses are unique across
  the mesh.

#### GAMMA: frontend facet versus backend facet

GAMMA distinguishes the *frontend* facet of a Service (name and cluster IP;
bound by `parentRef`) from its *backend* facet (endpoint IPs; named by
`backendRef`). Cilium translates them into different Envoy objects:

| Facet    | Named in      | Cilium translation                                                                  | Load-balancing role                                                                         | Annotations evaluated |
| -------- | ------------- | ----------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- | --------------------- |
| Frontend | `parentRefs`  | TPROXY redirect of ClusterIP:port traffic to Envoy; Envoy listener and route table  | None. No endpoint selection here; the Service's eBPF load balancing is bypassed once intercepted | No                    |
| Backend  | `backendRefs` | Envoy cluster with EDS endpoints                                                    | `lb_policy` and `hash_policy` on the cluster                                                | Yes                   |

The hash chooses one endpoint out of the set a `backendRef` resolves to, so it
is a backend-facet property. The parent facet is an interception point,
equivalent to a north/south `Gateway` listener.

One object may play both roles: `parentRefs: [cart]` with
`backendRefs: [cart]` attaches L7 behavior to a Service without redirecting
its traffic; the data plane routes to `cart`'s endpoints directly, not back
through its ClusterIP. The upstream Mesh conformance manifests `mesh-frontend`,
`mesh-ports` and `mesh-consumer-route` use this shape (`echo-vN` → `echo-vN`);
`mesh-split` fans out to other Services. In this shape the annotations on
`cart` are read because `cart` appears in `backendRefs`; the parent role never
evaluates them.

When the route splits to `cart-v1` and `cart-v2`, the annotations belong on
`cart-v1` and `cart-v2`; an annotation on `cart` alone has no effect:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cart-v1
  namespace: shop
  annotations:
    service.cilium.io/lb-l7-algorithm: maglev
    service.cilium.io/lb-l7-hash-policy: "header:x-tenant-id"
---
apiVersion: v1
kind: Service
metadata:
  name: cart-v2
  namespace: shop
  annotations:
    service.cilium.io/lb-l7-algorithm: maglev
    service.cilium.io/lb-l7-hash-policy: "header:x-tenant-id"
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: cart
  namespace: shop
spec:
  parentRefs:
  - group: ""
    kind: Service
    name: cart          # frontend facet: intercepted, not hashed
    port: 8080
  rules:
  - backendRefs:
    - name: cart-v1     # backend facet: hashed within cart-v1's endpoints
      port: 8080
      weight: 90
    - name: cart-v2     # backend facet: hashed within cart-v2's endpoints
      port: 8080
      weight: 10
```

A given `x-tenant-id` reaches the same Pod within `cart-v1` and the same Pod
within `cart-v2`; the choice between the two Services remains weighted-random
(Non-Goals).

A parent-level default (annotating `cart` to hash every backend of every route
attached to it) is not proposed: it gives the frontend facet a load-balancing
meaning it does not have, duplicates configuration the producer can place on
each backend, and a backend Service reachable through several parents or
Gateways would need precedence rules.

### Internal model

```go
// operator/pkg/model/model.go (existing fields abbreviated)

type Backend struct {
	Name        string
	Namespace   string
	Port        *BackendPort
	AppProtocol *string
	TLS         *BackendTLSOrigination
	Weight      *int32

	// LoadBalancing configures how Envoy selects an endpoint of this
	// backend. If unset, the translator default (ROUND_ROBIN) applies.
	LoadBalancing *LoadBalancing `json:"load_balancing,omitempty"`
}

type LoadBalancing struct {
	Algorithm  LoadBalancingAlgorithm `json:"algorithm"`
	HashPolicy *HashPolicy            `json:"hash_policy,omitempty"`
}

type LoadBalancingAlgorithm string

const (
	LoadBalancingRoundRobin   LoadBalancingAlgorithm = "round_robin"
	LoadBalancingLeastRequest LoadBalancingAlgorithm = "least_request"
	LoadBalancingRandom       LoadBalancingAlgorithm = "random"
	LoadBalancingMaglev       LoadBalancingAlgorithm = "maglev"
	LoadBalancingRingHash     LoadBalancingAlgorithm = "ring_hash"
)

// Exactly one of the fields is set.
type HashPolicy struct {
	Header         *HashPolicyHeader   `json:"header,omitempty"`
	QueryParameter *string             `json:"query_parameter,omitempty"`
	Cookie         *HashPolicyCookie   `json:"cookie,omitempty"`
	SourceIP       *HashPolicySourceIP `json:"source_ip,omitempty"`
}

// HashPolicySourceIP hashes the downstream address; a non-nil value selects it.
type HashPolicySourceIP struct{}

// HashPolicyHeader hashes a request header, optionally after extracting a
// part of it. path-regex is represented as Name ":path" with a Regex.
type HashPolicyHeader struct {
	Name  string           `json:"name"`
	Regex *HashPolicyRegex `json:"regex,omitempty"`
}

// HashPolicyRegex maps onto Envoy's RegexMatchAndSubstitute. The annotation
// surface fixes Substitution to the first capture group; the model keeps it
// explicit so a future API can expose it.
type HashPolicyRegex struct {
	Pattern      string `json:"pattern"`
	Substitution string `json:"substitution"`
}

type HashPolicyCookie struct {
	Name string         `json:"name"`
	TTL  *time.Duration `json:"ttl,omitempty"`
	Path string         `json:"path,omitempty"`
}
```

The model is a strict subset of what Envoy supports and a strict superset of
what the annotations express, so a later `Backend`-based or policy-based
ingestion populates it without translation changes.

### Ingestion

`backendToModelBackend(svc corev1.Service, ...)` in
`operator/pkg/model/ingestion/gateway.go` already receives the backend Service
(it reads `appProtocol` from it). It gains a call to a shared parser,
`ingestion.LoadBalancingFromService(svc) (*model.LoadBalancing, error)`, which
reads and validates both annotations, accepts only the five listed algorithm
values and rejects unknown ones (today they map to `ROUND_ROBIN` silently).
The Proxy Load Balancing reconciler (`operator/pkg/ciliumenvoyconfig`) calls
the same parser in place of `lbModeClusterMutator`, so the two paths cannot
drift.

Gateway API ingestion (`Input.Services`) and GAMMA ingestion
(`GammaInput.Services`) already carry the Service objects. The Gateway
reconciler watches backend Services without an update predicate
(`watchhandlers.EnqueueRequestForBackendService`), so an annotation change on a
backend re-renders the affected Gateways. The GAMMA reconciler is keyed on the
parent Service and returns early for a Service that is not a parent. When the
backend is a different object from the parent (`cart` → `cart-v1`), a
backend-Service-to-parent mapping is added (reusing
`indexers.BackendServiceHTTPRouteIndex` / `BackendServiceGRPCRouteIndex` where
possible) so that an annotation change on `cart-v1` re-renders the CEC of
parent `cart`. When one object plays both roles, the existing parent watch
already triggers reconciliation.

### Envoy translation

Cluster (`operator/pkg/model/translation/envoy_cluster.go`):

- `desiredEnvoyCluster` resolves `getLoadBalancing(m, ns, name, port)` next to
  `getAppProtocol` and `getTLSOrigination` and passes it to `httpCluster` /
  `clusterMutators`, where `withLoadBalancing(lb)` replaces
  `withClusterLbPolicy(Cluster_ROUND_ROBIN)`. The ext_authz clusters
  (`getHTTPExtAuthClusterName`, `getGRPCExtAuthClusterName`) keep
  `ROUND_ROBIN`.
- `maglev` → `lb_policy: MAGLEV` and `maglev_lb_config { table_size }`;
  `ring_hash` → `lb_policy: RING_HASH` and
  `ring_hash_lb_config { minimum_ring_size }`. Envoy defaults unless tuned
  (Key Question 5).
- The hash key is emitted as `HttpProtocolOptions.hash_policy` (Envoy 1.37+)
  in the cluster's existing `TypedExtensionProtocolOptions`. Envoy's router
  consults the cluster-level policy before any route-level one
  (`Router::Filter::computeHashKey`). The policy travels with the chosen
  cluster: a route splitting to backends with different keys hashes each
  backend on its own key (Key Question 2), and requests the async client sends
  to the cluster (request mirrors) are hashed the same way, although they never
  see route-level hash policies.
- `path-regex` is emitted as `header { header_name: ":path", regex_rewrite:
  { pattern: { regex: "^.*?(?:<pattern>).*$" }, substitution: "\1" } }`.
  Envoy's `regex_rewrite` replaces only the matched portion and keeps the rest,
  so without the wrapper a pattern that does not consume the whole path would
  leave path fragments in the key. The wrapper's group is non-capturing;
  capture-group numbering is unchanged.
- TCP (TLS passthrough) clusters are unchanged; `TLSRoute` traffic carries no
  L7 key. `source-ip` hashing for TCP clusters is a Future Milestone.

Route (`operator/pkg/model/translation/envoy_virtual_host.go`): unchanged. No
`RouteAction.hash_policy` is emitted; redirect and direct-response routes are
untouched.

Proxy Load Balancing (`operator/pkg/ciliumenvoyconfig/envoy_config.go`):

- `getClusterResources` builds the cluster's `HttpProtocolOptions` itself and
  applies `lbModeClusterMutator`. It switches to the shared parser and the
  same `withLoadBalancing` mutator, gaining `hash_policy`, the Maglev /
  ring-hash config and value validation. `getVirtualHost` is unchanged.

The `stateful_session` filter from CFP-47089 is unaffected. Envoy evaluates
the persistence cookie before the load balancer runs, so a route with both
persistence and a hashed backend follows GEP-1619's two tiers: persistence
identity first, consistent hash for requests without one.

Consequence to document: combining both on one route removes key affinity for
cookie-bearing callers. A caller whose first request for key `K` hashed to Pod
`Y` receives a persistence cookie for `Y` and returns to `Y`; if `Y`
disappears, that caller is pinned to whichever Pod its next request reaches,
while other callers hash `K` to the new `hash(K)`. Once a caller holds a
cookie, the key no longer determines its Pod. Key affinity requires hashing
alone. Cilium rejects `sessionPersistence` on GAMMA routes today, so the
combination cannot occur east/west; a north/south rule that sets
`sessionPersistence` and forwards to a hashed Service receives a `Warning`
event.

### Validation and status

Parsing errors, unknown values, and `maglev` / `ring_hash` without a hash
policy (fail-open; Key Question 4 has the fail-closed alternative):

- The backend keeps the translator default (`ROUND_ROBIN`). The route stays
  `Accepted=True` and is programmed.
- Every route referencing the backend gets an implementation-specific
  condition on the affected parent status (working name
  `io.cilium/BackendLoadBalancingValid=False`, reason `UnsupportedValue`;
  condition type to be settled in review) naming the Service and annotation.
  The check is added to `operator/pkg/gateway-api/routechecks/route_checks.go`
  next to `CheckSessionPersistence`.
- A `Warning` event is emitted on the Service.
- For Proxy Load Balancing (`lb-l7=enabled`), which has no route object, the
  Service event is the only signal.

### Behavior and operational notes

- **Consistency across nodes and callers.** Cilium runs one Envoy per node;
  GAMMA traffic enters Envoy on the client's node, so two callers on two nodes
  are served by two Envoy instances. The hash is a pure function of the key
  (xxHash64) and the healthy endpoint set: Envoy hashes endpoints by address
  and every agent programs the same backend set into EDS, so all nodes, and
  all CECs referencing the Service, build the same Maglev table or ring
  without coordination. Nodes disagree while an endpoint change propagates,
  while a cilium-envoy rollout leaves nodes on different Envoy builds, and for
  the duration of an outlier ejection, which is per node: an endpoint ejected
  on one node (default `base_ejection_time` 30 s; at most
  `max_ejection_percent` 10 % of hosts) leaves that node's table only, so keys
  mapped to it reach a different Pod from that node.
- **Consistency across clusters.** The same property extends to ClusterMesh:
  clusters that select the same endpoint set map a key to the same Pod. Which
  configurations select the same set, and which do not, is tabulated in
  "ClusterMesh: global Services and MCS-API".
- **Endpoint churn.** Adding or removing one endpoint remaps about 1/N of the
  keys (Maglev) or the keys owned by the removed host (ring hash). Outlier
  ejection has the same effect as removal, on the ejecting node only.
- **Weighted backends.** Envoy picks a cluster by weight, then hashes within
  it with that cluster's own policy. Same key, same Pod holds per backend
  Service, not across a weighted split (Non-Goals; Future Milestones).
- **Changing the algorithm.** Updating `lb-l7-algorithm` changes `lb_policy`
  on a live cluster; Envoy rebuilds the balancer and existing mappings are
  reshuffled once.
- **Existing `maglev` / `ring_hash` Proxy Load Balancing users** currently get
  random selection. After this change the value is invalid without
  `lb-l7-hash-policy`: the cluster falls back to `ROUND_ROBIN` and a Service
  warning event is emitted until a hash policy is added. Documented in the
  upgrade notes; alternatives in Key Question 4.

### Testing

- Unit tests for the annotation parser: valid values, case handling, cookie
  attributes, `path-regex` patterns containing `;`, `:` and `=`, patterns
  without a capture group, rejections.
- Golden-fixture translation tests in `operator/pkg/model/translation`:
  `maglev`, `ring_hash`, each hash source including the `path-regex` wrapper,
  a split to backends with different keys, an annotated Service that is also
  an `ExternalAuth` backend (ext_authz cluster stays `ROUND_ROBIN`), a
  request-mirror target, persistence plus affinity.
- Proxy Load Balancing tests in `operator/pkg/ciliumenvoyconfig`: cluster
  `lb_policy` and `hash_policy` from the annotations; `maglev` without a hash
  policy falls back to `ROUND_ROBIN`; unknown values are rejected.
- Ingestion tests for Gateway API and GAMMA inputs.
- Route-status tests in `operator/pkg/gateway-api` mirroring the CFP-47089
  reconciliation tests.
- End-to-end check in the Gateway API / GAMMA CI workflow: N requests with the
  same header land on one Pod; N distinct headers spread across Pods; the same
  header sent from two client Pods on two different nodes lands on the same
  backend Pod; the same holds through `service.cilium.io/lb-l7=enabled`.
- ClusterMesh check in the multi-cluster CI workflow, global-Service model: a
  Service object without Pods in cluster A and with Pods in cluster B; the same
  header sent from a caller in A and a caller in B lands on the same Pod in B.
  With Pods in both clusters and `affinity` unset, the same holds over the
  union; with `affinity: local`, each cluster's callers land on one Pod of
  their own cluster. MCS-API variant on a Gateway route with a `ServiceImport`
  backend and the annotations in `exportedAnnotations`.

### Documentation

- New page `Documentation/network/servicemesh/gateway-api/session-affinity.rst`
  next to the session-persistence page: both annotations, fallback semantics,
  the traffic-splitting caveat, key affinity versus caller affinity with the
  `orders` / `billing` / `cart` example, the interaction with session
  persistence, the per-node outlier-ejection caveat, and the ext_authz /
  request-mirror behavior.
- Cross-reference from the session-persistence page for readers who need
  cross-caller convergence.
- A ClusterMesh section on the new page (the placement table, the
  annotation-drift caveat for global Services, `exportedAnnotations` for
  MCS-API), and a cross-reference from the ClusterMesh services documentation.
- Proxy Load Balancing annotation table updated with `maglev`, `ring_hash` and
  `lb-l7-hash-policy`.
- Upgrade note for the behavior change above.

## Impacts / Key Questions

### Key Question 1: Where should the configuration live?

The internal model is identical for every option. Options A to C share the
cluster-level translation; Option D moves the key to `RouteAction.hash_policy`
and adds a cluster-naming rule.

#### Option A (proposed): Service annotations extending `service.cilium.io/lb-l7-*`

Pros

- No new CRD and no new annotation family: `lb-l7-algorithm` is documented and
  already selects the L7 algorithm for a Service; the hash-policy annotation
  completes a half-exposed feature.
- Matches Cilium's cluster model (one policy per Service:port, one Envoy
  cluster per Service:port); no precedence rules.
- Destination-side placement, like Istio, Kong, NGF, GKE's header-field
  affinity, and the future upstream `Backend` field. Migration is annotation
  value to `Backend` field, one Service at a time.
- Small implementation, realistic for one release.

Cons

- Annotations are hard to deprecate and have no schema validation; Cilium
  committers have expressed a preference against growing annotation surface in
  the Gateway API implementation (cilium/cilium#43532, #43534; TODO: link the
  review comment).
- One key per Service; different keys per path are not expressible.
- Configuration is split across Service and route; not discoverable from the
  `HTTPRoute`.

#### Option B: Cilium policy CRD attached to Services (GEP-713 direct policy)

Pros

- Structured, schema-validated, status-bearing; closest to Envoy Gateway's
  `BackendTrafficPolicy` and upstream `XBackendTrafficPolicy`.

Cons

- Cilium has declined Gateway API CRDs other than `CiliumGatewayClassConfig`
  because of long-term maintenance and reconciler cost, and because upstream
  intends `Backend` to absorb this.
- Deprecated on the same timeline as Option A, at higher cost.

#### Option C: Wait for and implement the upstream `Backend` resource

Pros

- Portable; the stated upstream direction (GEP-4894, discussion #4462).

Cons

- `XBackend` cannot yet represent in-cluster destinations without the stop-gap
  Service, and Cilium has not implemented `Backend`. Long-term destination, not
  a near-term option; both A and B migrate to it (Future Milestones).

#### Option D (alternative proposal): HTTPRoute-level configuration through an `ExtensionRef` filter

Route-side placement, following the Azure split and Envoy's own cluster/route
split: the hash key is a route-rule property (`RouteAction.hash_policy`), the
algorithm a destination property (cluster `lb_policy`). Inline `HTTPRoute`
fields are not available (Cilium cannot add fields to an upstream CRD) and a
route annotation would be the least portable choice, so the mechanism is the
`ExtensionRef` filter, Gateway API's extension point for implementation-specific
per-rule behavior (used in GEP-4894's own examples for a `RateLimitPolicy`).

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumHTTPHashPolicy            # name to be settled in review
metadata:
  name: by-tenant
  namespace: shop
spec:
  algorithm: Maglev                   # Maglev | RingHash; applied to the rule's backends
  hashPolicies:                       # ordered; Envoy semantics, including terminal
  - header:
      name: x-tenant-id
    terminal: true
  - pathRegex:
      pattern: '^/orders/([^/?]+)'
  - sourceIP: {}
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: cart
  namespace: shop
spec:
  parentRefs:
  - group: ""
    kind: Service
    name: cart
    port: 8080
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /orders
    filters:
    - type: ExtensionRef
      extensionRef:
        group: cilium.io
        kind: CiliumHTTPHashPolicy
        name: by-tenant
    backendRefs:
    - name: cart-v1
      port: 8080
  - backendRefs:                      # no filter: this rule stays round robin
    - name: cart-v1
      port: 8080
```

Semantics:

- The filter sets the `hash_policy` of the rule's `RouteAction`. Only requests
  matching the rule are hashed, even when other rules forward to the same
  Service.
- The algorithm is a cluster property and other rules may want round robin for
  the same Service, so Cilium generates one Envoy cluster per
  `(namespace, name, port, algorithm)`: hashed rules point at
  `shop:cart-v1:8080:maglev`, other rules at the existing cluster. Both
  subscribe to the same EDS resource; the agent's endpoint programming is
  unchanged. These clusters carry no cluster-level `hash_policy`, which would
  override the rule's. Cost: a second cluster and Maglev table per hashed
  Service per referencing CEC.
- Rule-vs-rule conflicts are impossible by construction (Key Question 2
  disappears). A rule with several weighted backends applies the same key
  within each backend, as in Option A.
- Unresolvable references follow the specification for `ExtensionRef`
  filters: the rule is not silently downgraded, the route reports the failure
  in status, and matching requests receive an error response.
- The same kind applies to `GRPCRoute` rules.
- A `targetRefs` (GEP-713 direct policy) form, as Envoy Gateway uses, is a
  variant with the same cost. `ExtensionRef` is preferred because it is
  visible in the route it affects and selects one rule without `sectionName`
  plumbing.

Pros

- Per-rule keys, which Option A cannot express: a Service serving several
  entity types under different paths gets a key per rule, and `path-regex` is
  natural because the rule's path match is known.
- Discoverable from the route; schema-validated; carries status.
- Ordered hash-policy chains with `terminal` on day one (kgateway and Kong
  expose them).
- The route author configures hashing without touching the Service.

Cons

- A new Cilium CRD (cilium/cilium#43534), and a net-new mechanism: Cilium
  does not implement `ExtensionRef`; unknown filter types are skipped in
  ingestion. Adds CRD installation, RBAC, a watch from the filter object to its
  routes, reference resolution and status handling. Roughly three to four times
  the implementation surface of Option A for the same data-plane result.
- Cluster duplication per hashed Service and referencing CEC; additional
  Maglev tables per node.
- Furthest from upstream's destination-side direction: `Backend` is per
  destination, a filter is per rule, so migration to `Backend.spec.loadBalancing`
  is a re-modelling rather than a field move.
- Istio declined route-level hash configuration in its core API
  (istio/api#3180) as niche.

Recommendation: Option A now, with the model built for Option C. Option D is
the fallback if reviewers consider per-rule key selection a requirement; the
internal model serves both, and Option D differs in ingestion, key placement
(route-level) and the cluster-naming rule.

### Key Question 2: Routes with multiple backends carrying different hash policies

Resolved by the cluster-level placement: each backend's cluster carries its
own `hash_policy`, so a split to `cart-v1` (hashed on `x-tenant-id`) and
`cart-v2` (hashed on `x-user-id`) hashes each within its own key and no route
is rejected. Route-level `hash_policy` could not do this: Envoy folds all
route-level policies into one value (`hash = rotl(old, 1) ^ new`), so the Pod
chosen within `cart-v1` for one tenant would change with the user ID. Option D
(one policy list per rule) is unaffected.

### Key Question 3: Lift the GAMMA ban on session persistence?

CFP-47089 rejects `sessionPersistence` on GAMMA routes without recording a
reason; none is given in the CFP text, the documentation page,
`CheckSessionPersistence` or the cilium/cilium#47089 thread, where a user has
asked for GAMMA support. This CFP keeps the ban out of scope and asks the
CFP-47089 authors to document the reason (cookie `Secure` attribute on
cleartext east/west traffic? Pod-address exposure to in-cluster clients?) so a
follow-up can address it.
Lifting the ban is not a substitute for this CFP: persistence pins callers,
not keys, and cannot make two services converge on one Pod for a shared
identifier.

### Key Question 4: Fail open or fail closed on invalid annotations?

CFP-47089 fails closed (`Accepted=False`, `UnsupportedValue`), appropriate for
configuration written on the route by the route author. Here the configuration
lives on a Service the route author may not own, referenced by many routes.

- Option 1 (proposed): fail open. Route accepted and programmed with
  `ROUND_ROBIN`; implementation-specific
  `io.cilium/BackendLoadBalancingValid=False` condition on the route; `Warning`
  event on the Service. A typo cannot take a Service offline and is no longer
  silent (today unknown values select `ROUND_ROBIN` without any signal).
- Option 2: fail closed as CFP-47089 (`Accepted=False`, `UnsupportedValue` on
  every referencing route). Consistent with existing behavior, impossible to
  overlook, but an annotation typo becomes an outage for every referencing
  route.

The same choice applies to existing Proxy Load Balancing users with `maglev` or
`ring_hash` (currently random selection): under Option 1 they get a Service
event and `ROUND_ROBIN` until they add a hash policy. A third possibility,
keeping Envoy's random fallback for them and activating the new behavior only
when `lb-l7-hash-policy` is present, avoids any behavior change but leaves the
misconfiguration silent.

### Key Question 5: Tuning knobs (table size, ring size)

Envoy's defaults (Maglev 65537, ring minimum 1024) suit typical Service sizes.
Exposing them means more annotations or Helm values
(`loadBalancer.l7.maglev.tableSize`, `loadBalancer.l7.ringHash.minimumRingSize`).
Proposal: Helm-level defaults only, no per-Service knobs. Envoy's Maglev table
is built once per Envoy cluster, that is per referencing CEC (tens of KiB in
the compact representation for fewer
than 256 hosts, up to about 1 MiB at the default table size), negligible next
to the per-node eBPF Maglev memory discussed in CFP-34577.

### Key Question 6: Ship `path-regex` in the first version?

Implementation cost is small: one optional struct on the header hash policy,
one parser branch (the remainder of the value is the pattern), about six lines
of translation, regex validation at ingestion. Envoy supports it natively and
cilium-envoy ships RE2. The cost is semantics and documentation: no "key
absent" fallback, `:path` including the query string, `URLRewrite` ordering.

Prior art is mixed. Istio declined an equivalent (istio/api#3180, regex capture
on the hashed header, motivated by the path-key case) as too niche for its core
API; Envoy Gateway does not expose it. Envoy, NGINX (`hash $1` after a
`map`/`location` capture), NGINX Gateway Fabric (`hash consistent` with
`hashMethodKey`, for example `$request_uri`) and HAProxy (`balance uri`)
support path-derived keys, and the actor-sharding case is naturally a path or
query key.

- Option 1 (proposed): include it; the annotation grammar and model already
  accommodate it and deferring saves little.
- Option 2: defer to a follow-up, keeping the first review on the four sources
  every other implementation has.

### Key Question 7: Detect annotation drift between clusters for global Services?

For global Services, each cluster reads its own Service copy and Cilium cannot
see the other clusters' annotations. A copy missing or differing in the two
annotations silently breaks cross-cluster key affinity.

- Option 1 (proposed): document the requirement that copies be identical and
  point users who need enforcement to MCS-API, where `exportedAnnotations` is
  the single source of truth and `AnnotationsConflict` detection already
  exists.
- Option 2: propagate the `service.cilium.io/lb-l7-*` annotations through
  ClusterMesh (`ClusterService` in the kvstore), compare on import, and emit a
  `Warning` event on the local Service when a remote cluster's copy differs.
  Detects drift for global Services too, at the cost of extending the shared
  service schema and its compatibility rules.

### Impact: Security

Hash sources are attacker-controlled input (headers, cookies, query
parameters, paths, source addresses). A client that can choose keys can
concentrate load on one Pod, but no more than by connecting to that Pod
directly; no new information is disclosed and no Pod address is encoded in a
token. The generated hash cookie (when `ttl` is set) is a random value. This is
a smaller threat surface than session persistence, as GEP-1619 notes.

### Impact: Deprecation path and the future `Backend` shape

When Cilium implements the upstream `Backend` resource with load-balancing
fields, the annotations become deprecated: Cilium emits a `Warning` event and a
route condition when both are present on one destination, `Backend`
configuration takes precedence, and the annotations are removed after the
standard deprecation window. The internal model is unchanged.

The upstream field shape is not decided. GEP-4894 lists load balancing,
retries, timeouts and health checks as inline `Backend` fields for follow-on
GEPs; `sessionPersistence` already exists on `XBackend` at the top level of
`spec`, valid only for `type: EndpointSelector` (gateway-api `main`,
2026-09-30; #5298 proposes nesting it under `endpointSelector`). The example
below follows that convention and the Envoy Gateway, kgateway and Istio APIs,
on the `XBackend` that exists today. Cilium would ingest it into the same
`model.LoadBalancing` the annotations populate:

```yaml
apiVersion: gateway.networking.x-k8s.io/v1alpha1   # gateway.networking.k8s.io once graduated
kind: XBackend
metadata:
  name: cart-v1
  namespace: shop
spec:
  type: EndpointSelector
  endpointSelector:
    matchLabels:
      app: cart
      version: v1
  port:
    number: 8080
  protocol: HTTP
  # Illustrative follow-on GEP field, not part of GEP-4894; valid only for
  # type: EndpointSelector, like sessionPersistence.
  loadBalancing:
    type: ConsistentHash                 # RoundRobin | LeastRequest | Random | ConsistentHash
    consistentHash:
      algorithm: Maglev                  # Maglev | RingHash
      hashOn:
      - type: Header                     # Header | Cookie | QueryParameter | SourceIP
        header: x-tenant-id
        terminal: true
      - type: SourceIP
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: cart
  namespace: shop
spec:
  parentRefs:
  - group: ""
    kind: Service
    name: cart                           # GAMMA parent stays a Service (frontend facet)
    port: 8080
  rules:
  - backendRefs:
    - group: gateway.networking.x-k8s.io
      kind: XBackend
      name: cart-v1                      # the backend facet as a resource of its own
      port: 8080
```

Consequences for Cilium:

- `Backend` makes the backend facet a first-class resource: the GAMMA parent
  remains a Service, the destination becomes a `Backend`, and load balancing
  attaches to the destination. Option A's Service annotations map onto this one
  Service at a time; Option D's per-rule filters do not.
- Implementing `Backend` is a prerequisite this CFP does not deliver: resolving
  an `EndpointSelector` (via the GEP's stop-gap Cilium-owned headless Service
  until the upstream `EndpointSelector` resource exists) and accepting `Backend`
  kinds in `backendRefs`, including GAMMA routes.
- Translation is unchanged: `backendToModelBackend` gains a second source for
  `model.LoadBalancing`; `envoy_cluster.go` does not change.

## Future Milestones

### Upstream: load balancing on `Backend`

Draft a GEP proposing `Backend.spec.loadBalancing` (algorithm, consistent-hash
key, tuning) on GEP-4894, referencing GEP-1619's affinity definition and the
implementations surveyed above; implement `Backend` in Cilium. This is the
portable replacement for Option A.

### Honor `Service.spec.sessionAffinity: ClientIP` in L7 paths

When Cilium redirects a Service to Envoy (GAMMA, Gateway API backend, Proxy
Load Balancing), `sessionAffinity: ClientIP` is silently lost. Source-IP
consistent hashing gives the same caller affinity without new API and without
the stick table `timeoutSeconds` implies (see "Key affinity versus caller
affinity" for the differences). GEP-1619 requires rejecting routes that combine
Service session affinity with Gateway API session persistence, so an
interaction rule is needed. The upstream API for this, GEP-3798, is Deferred.

### Regex extraction for arbitrary headers

`path-regex` is the `:path` special case of Envoy's header `regex_rewrite`. The
model carries header name and regex separately, so `header:<name>;regex=<pattern>`
is a parser change. Left out of the first version to keep the annotation
grammar unambiguous (`;` inside a pattern).

### Ordered hash-policy chains

Envoy supports multiple `hash_policy` entries with `terminal` flags ("header if
present, else source IP"). The model extends to a list without changing the
translation approach.

### Consistent selection across weighted `backendRefs`

Envoy 1.36+ `WeightedCluster.use_hash_policy` lets the route-level hash also
choose the weighted cluster, so one key reaches one Pod across a canary split.
It needs a route-level `hash_policy` mirroring the backends' shared key and
applies only when every backend of the rule hashes the same key.

### Helm default algorithm for Gateway API and GAMMA clusters

Plumb `--loadbalancer-l7-algorithm` into the CEC translator so Helm
`loadBalancer.l7.algorithm` applies to all three L7 paths (today: Proxy Load
Balancing only).

### Locality-aware hashing for ClusterMesh

Emit `LocalityLbEndpoints` with priorities (local 0, remote 1) instead of a
flat list, so that `service.cilium.io/affinity: local` is enforced inside
Envoy with a Maglev table per priority level. This changes failover semantics:
Envoy spills traffic to the next priority by health percentage
(overprovisioning) rather than switching sets when the last local backend goes
unhealthy, so the current `SelectBackends` behavior and the Envoy behavior
would need to be reconciled first.

### `source-ip` affinity for TCP / TLS passthrough clusters

`RING_HASH` / `MAGLEV` with a source-IP hash on `TLSRoute` / `TCPRoute`
clusters.

### Lift the GAMMA restriction on session persistence

See Key Question 3.

## References

- [GEP-1619: Session Persistence](https://gateway-api.sigs.k8s.io/geps/gep-1619/)
- [GEP-4894: Backend Resource](https://gateway-api.sigs.k8s.io/geps/gep-4894/)
- [GEP-3798: Client IP-Based Session Persistence (Deferred)](https://gateway-api.sigs.k8s.io/geps/gep-3798/)
- [kubernetes-sigs/gateway-api discussion #4462: Which attachment model for Session Persistence?](https://github.com/kubernetes-sigs/gateway-api/discussions/4462)
- [kubernetes-sigs/gateway-api#1778: Add support for Load Balancing Policy](https://github.com/kubernetes-sigs/gateway-api/issues/1778)
- [kubernetes-sigs/gateway-api#4876: GEP-1619: Backend as attachment point for Session Persistence](https://github.com/kubernetes-sigs/gateway-api/pull/4876), [#5255: GEP-4894: Add EndpointSelector to Backend API](https://github.com/kubernetes-sigs/gateway-api/pull/5255), [#5298: GEP-1619: Add Session Persistence API to Backend](https://github.com/kubernetes-sigs/gateway-api/pull/5298)
- [CFP-47089: Gateway API Session Persistence](https://github.com/cilium/design-cfps/blob/main/cilium/CFP-47089-gateway-api-session-persistence.md), cilium/cilium#47089, cilium/cilium#48029
- [CFP-34577: Per-service Load Balancing Algorithm Selection](https://github.com/cilium/design-cfps/blob/main/cilium/CFP-34577-per-service-lb-algorithm-selection.md)
- [cilium/cilium#43532: CFP: Per-Service Envoy Circuit Breaker Configuration](https://github.com/cilium/cilium/issues/43532) and its PR [cilium/cilium#43534](https://github.com/cilium/cilium/pull/43534) (committer feedback on new CRDs and annotations; TODO: link the comment)
- [cilium/cilium#33767: Gateway API cannot be used with custom Envoy configs](https://github.com/cilium/cilium/issues/33767)
- [cilium/cilium#40956: Exposing services in ClusterMesh and creating an L7 load balancer using Gateway API](https://github.com/cilium/cilium/issues/40956)
- [Cilium docs: ClusterMesh load-balancing and service discovery (global, shared, affinity annotations)](https://docs.cilium.io/en/stable/network/clustermesh/services/)
- [Cilium docs: Multi-Cluster Services API (MCS-API) support](https://docs.cilium.io/en/stable/network/clustermesh/mcsapi/)
- [Cilium docs: Proxy Load Balancing for Kubernetes Services (beta)](https://docs.cilium.io/en/stable/network/servicemesh/envoy-load-balancing/)
- [Envoy: load balancers (ring hash, Maglev)](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/load_balancers)
- [Envoy: `RouteAction.HashPolicy`](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#config-route-v3-routeaction-hashpolicy)
- [Envoy: `HttpProtocolOptions.hash_policy` (cluster-level, Envoy 1.37+)](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/upstreams/http/v3/http_protocol_options.proto)
- [Envoy: `WeightedCluster.use_hash_policy` (Envoy 1.36+)](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#config-route-v3-weightedcluster)
- [Envoy Gateway: Load Balancing](https://gateway.envoyproxy.io/docs/tasks/traffic/load-balancing/)
- [Istio: `LoadBalancerSettings.ConsistentHashLB`](https://istio.io/latest/docs/reference/config/networking/destination-rule/#LoadBalancerSettings-ConsistentHashLB)
- [istio/api#3180: regex rewrite of the hashed header (declined)](https://github.com/istio/api/pull/3180)
- [kgateway: Consistent hashing (`hashPolicies` on `BackendConfigPolicy`)](https://kgateway.dev/docs/envoy/latest/traffic-management/session-affinity/consistent-hashing/)
- [NGINX Gateway Fabric: `UpstreamSettingsPolicy` (`ip_hash`, `hash`, `hash consistent`)](https://docs.nginx.com/nginx-gateway-fabric/traffic-management/upstream-settings/)
- [Envoy Gateway issue #8467: RingHash in `BackendTrafficPolicy`](https://github.com/envoyproxy/gateway/issues/8467)
- [Azure Application Gateway for Containers: Load balancing strategies](https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/load-balancing-strategies)
- [GKE Gateway: GCPSessionAffinityFilter / GCPTrafficDistributionPolicy](https://github.com/GoogleCloudPlatform/gke-gateway-api), [GKE Gateway traffic management](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/traffic-management)
- [Kong: KongUpstreamPolicy consistent hashing](https://developer.konghq.com/operator/dataplanes/how-to/configure-upstream-policy/)