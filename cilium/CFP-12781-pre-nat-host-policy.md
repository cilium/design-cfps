# CFP-12781: Pre-NAT Host Policy Stage (`preNATIngress`)

**SIG: SIG-Policy** ([View all current SIGs](https://docs.cilium.io/en/stable/community/community/#all-sigs))

**Begin Design Discussion:** 2026-09-01

**Cilium Release:** 1.20

**Authors:** Jakub Hlavnicka <jakub.hlavnicka@illumio.com>

**Status:** Draft

**Issue:** [cilium/cilium#12781](https://github.com/cilium/cilium/issues/12781)

> **Note:** This design supersedes the earlier revision of this CFP, which proposed a
> `hostFirewall.enforceBeforeNodePortDNAT` configuration flag. It was rewritten in response to review
> feedback suggesting that the pre-NAT hook point be exposed as an explicit policy API field rather
> than as a configuration flag that silently changes the meaning of the existing `ingress` field. The
> earlier revision remains available in the history of
> [cilium/design-cfps#97](https://github.com/cilium/design-cfps/pull/97).
>
> **One question remains open for SIG-Policy:** whether this stage belongs on
> `CiliumClusterwideNetworkPolicy` as a new field or in a dedicated kind — see [Open Question: New
> Field vs. New Kind](#open-question-new-field-vs-new-kind). Every other question raised in review is
> settled in [Resolved Design Decisions](#resolved-design-decisions).

## Summary

Introduce a second, explicitly addressable ingress policy enforcement stage for the host endpoint,
which runs at the `from-netdev` hook **before** NodePort load balancing performs DNAT and SNAT. The
stage is expressed in the API as a new `preNATIngress` field on `CiliumClusterwideNetworkPolicy`,
using the same rule syntax as the existing `ingress` field.

Because the new stage is only enforced when a policy explicitly populates `preNATIngress`, the
semantics of every existing policy are unchanged and no configuration flag or opt-in migration is
required.

## Motivation

Host firewall policy today evaluates NodePort traffic **after** the load balancer has rewritten the
packet. By that point the destination has been DNATed to the backend pod IP and port, and — for
remote backends — the source has been SNATed to the node IP. Two categories of policy are therefore
inexpressible:

1. Denying traffic to the NodePort range (30000-32767) from external sources.
2. Allowing specific external source IPs to reach specific NodePort ports.

The following policy reads as if it does both, but today rules (1) and (3) silently fail to match
the traffic the author intended, because the packet the host firewall sees no longer carries the
NodePort destination port:

```yaml
apiVersion: cilium.io/v2
kind: CiliumCIDRGroup
metadata:
  name: allowed-nodeport-source
spec:
  externalCIDRs:
  - 142.217.23.90/32
---
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: host-baseline
spec:
  nodeSelector: {}
  ingress:
  - fromEntities:
    - world
    toPorts:
    - ports:
      - port: "1"
        endPort: 30000
        protocol: TCP
  - fromEntities:
    - cluster
  - fromCIDRSet:
    - cidrGroupRef: allowed-nodeport-source
    toPorts:
    - ports:
      - port: "30500"
        protocol: TCP
```

The gap is not only that the rules do not work — it is that there is no way for the author to *tell*
that they do not work. The policy is accepted, the selectors are valid, and the ports are in range.
This design makes the enforcement point part of the API, so that a rule which requires pre-NAT
matching is written as a pre-NAT rule and is enforced as one.

**Related Issue:** [cilium/cilium#12781](https://github.com/cilium/cilium/issues/12781)

## Goals

* Allow `CiliumClusterwideNetworkPolicy` to match the original destination port of NodePort traffic,
  before DNAT rewrites it to the backend pod port.
* Allow `CiliumClusterwideNetworkPolicy` to match the original external source IP, before SNAT
  replaces it with the node IP.
* Make the enforcement point explicit in the policy object, so that the same rule cannot mean
  "pre-NAT" on one cluster and "post-NAT" on another.
* Require no configuration change, no opt-in flag, and no behaviour change for clusters that do not
  use the new field.

## Non-Goals

* L7 policy at the pre-NAT stage. The packet has not yet been steered to a proxy and the connection
  has no established endpoint context; only L3/L4 rules are supported.
* Egress. A symmetric post-SNAT egress stage is plausible but is deliberately out of scope; see
  [Egress Counterpart](#egress-counterpart) for the rationale.
* Replacing or deprecating the existing `ingress` field. Both stages coexist and both are enforced.
* Per-service policy. This is node-scoped policy; it matches on the packet as it arrives on the
  wire, not on the Kubernetes Service object it will be resolved to.

## Proposal

### API

A new field `preNATIngress` is added to `CiliumClusterwideNetworkPolicySpec` (and to the entries of
`specs`). Its type is `[]IngressRule` — identical to `ingress` — so all existing selector syntax
(`fromEntities`, `fromCIDR`, `fromCIDRSet` with `cidrGroupRef` and `except`, `fromEndpoints`,
`fromNodes`, `toPorts` with `endPort` ranges) is available unchanged.

The motivating policy becomes:

```yaml
apiVersion: cilium.io/v2
kind: CiliumCIDRGroup
metadata:
  name: allowed-nodeport-source
spec:
  externalCIDRs:
  - 142.217.23.90/32
---
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: host-baseline
spec:
  nodeSelector:
  kubernetes.io/hostname: node-host-name
  # Evaluated on the packet as it arrives on the native device, before NodePort
  # DNAT/SNAT. Destination ports are the ports on the wire; source IPs are the
  # original client addresses.
  preNATIngress:
  - fromEntities:
    - cluster
  - fromEntities:
    - world
    toPorts:
    - ports:
      - port: "1"
        endPort: 29999
        protocol: TCP
      - port: "1"
        endPort: 29999
        protocol: UDP
  - fromCIDRSet:
    - cidrGroupRef: allowed-nodeport-source
    toPorts:
    - ports:
      - port: "30500"
        protocol: TCP

  # Unchanged semantics: evaluated on traffic terminating on the host endpoint,
  # after LB translation.
  ingress:
  - fromEntities:
    - cluster

  egress:
  - toEntities:
    - world
    - cluster
    - health
```

The author can now read the enforcement point off the object. A rule under `preNATIngress` matching
`port: "30500"` matches the NodePort; the same rule under `ingress` matches a host-terminated port
30500 and will never see NodePort traffic.

**Default-deny is scoped to the stage.** Cilium's rule is that an endpoint selected by any rule in a
direction becomes default-deny in that direction. Here that rule is applied *per stage*: a policy
that sets only `ingress` does not put a node's pre-NAT stage into default-deny, and a policy that
sets only `preNATIngress` does not put the host endpoint's `ingress` stage into default-deny. The
pre-NAT stage is enforced only on nodes selected by at least one policy that populates
`preNATIngress`; everywhere else it passes traffic through untouched. This is what makes the change
non-breaking — no existing object can activate the new stage.

### Datapath

The stage is implemented at the `from-netdev` hook, ahead of the NodePort lookup:

```
External packet arrives on native device
    |
    v
tail_handle_ipv4_from_netdev
    |
    +-- TC_INDEX_F_SKIP_PRE_NAT_POLICY set? --Yes--> NodePort / LB processing
    |
    +-- Pre-NAT stage active on this node? --No----> NodePort / LB processing
    |
    Yes
    v
tail_ipv4_pre_nat_host_policy            (CILIUM_CALL_IPV4_PRE_NAT_HOST_POLICY)
    |
    |   lookup src identity in ipcache using the ORIGINAL saddr
    |   lookup CT entry; CT_ESTABLISHED / CT_REPLY --> allow, no policy eval
    |   policy lookup in cilium_policy_prenat_v2 with
    |       (src identity, ORIGINAL dport, proto, ingress)
    v
[verdict]
    |
    +-- deny --> DROP (new drop reason: DROP_POLICY_PRE_NAT)
    |
    allow
    v
set TC_INDEX_F_SKIP_PRE_NAT_POLICY
    |
    v
recirculate: CILIUM_CALL_IPV4_FROM_NETDEV
    |
    v
NodePort / LB processing (DNAT, SNAT) --> existing flow unchanged
```

Notes on the mechanism:

* The policy check runs in its own tail call frame (`CILIUM_CALL_IPV4_PRE_NAT_HOST_POLICY` /
  `CILIUM_CALL_IPV6_PRE_NAT_HOST_POLICY`) to stay within the 512-byte BPF stack limit, then
  recirculates through `CILIUM_CALL_IPV4/6_FROM_NETDEV` with a skip flag so the second pass goes
  straight to LB processing.
* The skip flag is a new `TC_INDEX_F_SKIP_PRE_NAT_POLICY` bit rather than a reuse of
  `TC_INDEX_F_SKIP_HOST_FIREWALL`, so that the pre-NAT stage and the existing host firewall stage
  can be skipped independently. Reusing the existing bit would suppress the host `ingress` check for
  host-terminated traffic that had only passed the pre-NAT check.
* **Policy map.** The stage gets its own per-node ingress-only policy map,
  `cilium_policy_prenat_v2`, populated by the existing policy distillation pipeline with the host
  endpoint's node labels as the selector context. Reusing the map format means selector resolution,
  ipcache identity allocation, `CiliumCIDRGroup` expansion, incremental updates, and
  `cilium bpf policy get` all work without new machinery. It does mean a second map's worth of
  entries per node; the expected rule count for this stage is small.
* **Activation.** The tail call target is populated only when host firewall and NodePort are both
  enabled. Whether the stage is *enforced* is a per-node runtime property derived from whether any
  selected policy carries `preNATIngress` — the same mechanism that decides whether an endpoint is
  in default-deny. No compile-time or Helm flag participates in the decision.
* An advanced agent flag `--disable-pre-nat-host-policy` (default `false`) is provided purely as an
  operational escape hatch; setting it causes policies with `preNATIngress` to be reported as not
  enforced via a status condition, rather than silently ignored.

### Validation

CRD validation and the agent's policy parser reject configurations whose enforcement point cannot be
honoured, rather than accepting them and under-enforcing:

* `preNATIngress` is only valid on `CiliumClusterwideNetworkPolicy` with a `nodeSelector`. It is
  rejected on `CiliumNetworkPolicy` and on `CiliumClusterwideNetworkPolicy` with an
  `endpointSelector`, since there is no per-endpoint pre-NAT stage.
* `toPorts[].rules` (L7) is rejected — no proxy exists at this hook point.
* `authentication` / mutual-auth is rejected for the same reason.
* `icmps` is supported; `toPorts` with `serverNames` or TLS config is rejected.
* Rules that would be silently non-matching are rejected where detectable, e.g. `fromEndpoints` with
  a selector that can only resolve to local pods, since local pod traffic does not traverse the
  native device hook.

### Observability

* New drop reason `DROP_POLICY_PRE_NAT`, distinct from `DROP_POLICY`, so that operators can tell a
  pre-NAT denial from a host firewall or pod policy denial. Surfaced in `cilium monitor`, Hubble
  flows, and the `cilium_drop_count_total` metric.
* Hubble policy verdict events carry a stage discriminator so that
  `hubble observe --verdict DROPPED` output identifies which of the two host stages dropped the
  packet.
* `cilium policy get` and `cilium endpoint get <host>` report pre-NAT enforcement state
  (`Disabled` / `Enforcing` / `Audit`) alongside the existing ingress and egress enforcement state.
* A `CiliumClusterwideNetworkPolicy` status condition reports when `preNATIngress` is present but
  the stage is unavailable on a node (host firewall disabled, NodePort disabled, or the escape-hatch
  flag set), so that a policy which cannot be enforced is visibly not enforced.

## Why `loadBalancerSourceRanges` Is Not Sufficient

Kubernetes `Service.spec.loadBalancerSourceRanges` provides per-service source IP filtering for
LoadBalancer services. While superficially similar, it does not address the use cases motivating
this CFP.

### 1. Not a Centralized Policy Mechanism

`loadBalancerSourceRanges` is a per-service field that must be set on each individual Service or
Gateway object. An external controller or platform operator would need to dynamically patch the
configuration of user-owned services with allowed source IP ranges. This is invasive and
conflict-prone — in environments where services are managed by GitOps tooling such as ArgoCD or
Flux, external mutations to service specs are detected as drift and reverted, creating a
reconciliation loop. A `CiliumClusterwideNetworkPolicy` is a single cluster-wide object that is
independent of individual service definitions.

### 2. No Support for Exclusions (Allow + Deny Combinations)

`loadBalancerSourceRanges` is whitelist-only — it can only specify which source CIDRs are allowed.
There is no way to express exclusion rules or to combine allow and deny semantics.
`CiliumClusterwideNetworkPolicy` supports richer expressions including `fromCIDRSet` with `except`
fields, and — under this proposal — the same expressiveness at the pre-NAT stage.

### 3. Requires Non-Default Cilium Configuration

By default, Cilium only enforces `loadBalancerSourceRanges` on LoadBalancer and ExternalIPs service
frontends. Extending enforcement to NodePort frontends requires `bpf.lbSourceRangeAllTypes=true`.
This is a non-starter on natively integrated Cilium deployments such as GKE Datapath v2 and AKS
ACNS, where platform operators do not control or expose Cilium configuration flags.

### 4. Kubernetes API Rejects `loadBalancerSourceRanges` on NodePort Services

The Kubernetes API server validates that `spec.loadBalancerSourceRanges` (and the legacy annotation
`service.beta.kubernetes.io/load-balancer-source-ranges`) may only be set when `spec.type` is
`LoadBalancer`:

```
The Service "example" is invalid: spec.LoadBalancerSourceRanges: Forbidden: may only be used when `type` is 'LoadBalancer'
```

Source range filtering via this mechanism is therefore unavailable for NodePort services at the
Kubernetes API level, regardless of what the underlying CNI supports.

## Resolved Design Decisions

These were raised as open questions in review. They are recorded here as settled, with the rejected
alternatives and the costs accepted retained so that they do not have to be rediscovered.

### Field Naming

**Decision: `preNATIngress`.**

The field was suggested in review as `preSNATIngress`. `preNATIngress` is used instead because the
hook precedes both DNAT (destination rewrite) and SNAT (source rewrite), and the primary motivating
rule matches a *destination port* — a DNAT concern. A user reading `preSNATIngress` has no reason to
expect that it fixes port matching, which understates the change. The cost accepted is that "NAT" is
broad: the datapath performs several translations, so documentation must state which ones this hook
precedes.


### Egress Counterpart

**Decision: ingress only in the first iteration.**

A symmetric post-SNAT egress stage is plausible, but there is no concrete demand for it, and adding
it would double the datapath and test surface for speculative benefit. Shipping ingress alone matches
the reported use cases and keeps the first iteration reviewable.

The risk accepted is that the naming and structure chosen now constrain the egress side if it is
added later. `preNATIngress` admits an obvious `preNATEgress` counterpart, which bounds that risk.

## Open Question: New Field vs. New Kind

This is the one decision the design does not settle, and the question this CFP is being taken to
SIG-Policy to resolve.

An alternative to extending `CiliumClusterwideNetworkPolicy` is a dedicated kind, e.g.
`CiliumNodePerimeterPolicy`.

### Option 1: New field on `CiliumClusterwideNetworkPolicy` (proposed)

#### Pros

* Reuses `nodeSelector`, the rule types, `CiliumCIDRGroup` references, status reporting, and the
  entire policy pipeline.
* Node policy stays in one object, so an operator reads one YAML to know what a node enforces.

#### Cons

* Adds to an already-large CRD, and makes `CiliumClusterwideNetworkPolicy` mean two different things
  depending on which fields are set.

### Option 2: A separate CRD

#### Pros

* Clean separation; the new stage's restrictions (no L7, no auth, node-scoped only) are expressed by
  the type rather than by validation rules on a general-purpose type.
* Independent RBAC, so the perimeter policy can be owned by the platform team while application
  teams retain `CiliumClusterwideNetworkPolicy`.

#### Cons

* Duplicates a large amount of schema and controller code.
* Two objects must now be read together to understand what a node enforces, and their interaction
  becomes a documentation burden.
