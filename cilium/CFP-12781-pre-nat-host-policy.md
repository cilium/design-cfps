# CFP-12781: Pre-NAT Host Policy Stage (`preNATIngress`)

**SIG: SIG-Policy** ([View all current SIGs](https://docs.cilium.io/en/stable/community/community/#all-sigs))

**Begin Design Discussion:** 2026-09-01

**Cilium Release:** 1.20

**Authors:** Jakub Hlavnicka <jakub.hlavnicka@illumio.com>

**Status:** Draft

**Issue:** [cilium/cilium#12781](https://github.com/cilium/cilium/issues/12781)

> **Note:** This is an alternative design to
> [CFP-12781-host-firewall-before-nodeport-dnat.md](./CFP-12781-host-firewall-before-nodeport-dnat.md),
> written in response to review feedback suggesting that the pre-NAT hook point be exposed as an
> explicit policy API field rather than as a configuration flag that changes the meaning of the
> existing `ingress` field. See [Comparison With the Configuration-Flag
> Design](#comparison-with-the-configuration-flag-design).
>
> The field was suggested in review under the name `preSNATIngress`. This document uses
> `preNATIngress`, because the hook point precedes both DNAT and SNAT; the naming is itself an open
> question, see [Key Question: Field Naming](#key-question-field-naming).

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
  [Key Question: Egress Counterpart](#key-question-egress-counterpart).
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

### Enforcement Semantics

**Scope.** `preNATIngress` applies to traffic arriving on native devices on nodes selected by
`nodeSelector`, evaluated before the NodePort/LB logic in `from-netdev`. It applies to *all* such
traffic, not only traffic that turns out to match an LB frontend — see [Key Question: Scope of the
Pre-NAT Stage](#key-question-scope-of-the-pre-nat-stage).

**Default-deny is scoped to the stage.** Cilium's rule is that an endpoint selected by any rule in a
direction becomes default-deny in that direction. Here that rule is applied *per stage*:

| Cluster state | Pre-NAT stage | `ingress` stage |
| --- | --- | --- |
| No policy with `preNATIngress` selects the node | allow all (pass through) | unchanged |
| At least one policy with `preNATIngress` selects the node | default-deny; only listed traffic allowed | unchanged |

A policy that sets only `ingress` does **not** put the node's pre-NAT stage into default-deny, and a
policy that sets only `preNATIngress` does **not** put the host endpoint's `ingress` stage into
default-deny. This is what makes the change non-breaking: existing objects cannot activate the new
stage.

**The two stages are conjunctive.** A packet that is subject to both must be allowed by both. For
NodePort traffic to a pod backend, the packet passes the pre-NAT stage, is DNATed, and is then
subject to the *backend pod's* ingress policy as it is today — the host `ingress` field is not
consulted for forwarded traffic. For traffic terminating on the host itself, the packet passes the
pre-NAT stage and then the host `ingress` stage. The pre-NAT stage is therefore strictly additive:
it can only cause drops, never allow traffic that a later stage denies.

**Connection tracking.** Only the first packet of a connection in the forward direction is evaluated
against `preNATIngress`. Established connections and reply traffic bypass the stage, using the same
conntrack semantics as the existing host firewall. This is required for correctness — the reply of a
NodePort connection arrives already-translated and must not be matched against pre-NAT rules — and
it bounds the per-packet cost to new flows.

**Auditing.** `enable-policy: audit` and per-node policy audit mode apply to the pre-NAT stage as
well; denied packets are reported and forwarded rather than dropped, so the stage can be rolled out
observably.

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

Notes on the mechanism, largely carried over from the configuration-flag design:

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

## Comparison With the Configuration-Flag Design

The companion CFP proposes `hostFirewall.enforceBeforeNodePortDNAT`, a cluster-wide flag that moves
where the existing `ingress` field is evaluated. The two designs solve the same datapath problem and
share most of the BPF work; they differ in what the API says.

| | Config flag (`enforceBeforeNodePortDNAT`) | Policy field (`preNATIngress`) |
| --- | --- | --- |
| Meaning of an `ingress` rule | depends on cluster configuration | fixed |
| Breaking change risk | real; flag changes existing policies' behaviour | none; existing objects unaffected |
| Opt-in granularity | whole cluster | per rule |
| Can express "deny at the wire, allow at the host" | no — one stage, one field | yes — the stages are separate |
| Portability of a policy object | needs matching cluster config | self-describing |
| Path to default-on | breaking; needs a major release | not needed; nothing to flip |
| API surface added | one Helm value | one CRD field, plus validation |
| Datapath work | new tail calls, skip flag, recirculation | same, plus a second policy map |

The decisive argument for the field is portability. A `CiliumClusterwideNetworkPolicy` is an object
that gets committed to a repository and applied to many clusters. Under the flag design, the same
YAML enforces different things depending on a Helm value the policy author may not control and
cannot see from the object — and the failure mode of the mismatch is silent under-enforcement of a
security policy. Under the field design, the enforcement point travels with the rule.

The secondary argument is expressiveness. Several of the motivating deployments want *both* stages:
a tight allowlist on what may enter the node from the outside world, and a separate, looser policy
on what may terminate on the host. The flag design forces a single choice for the whole cluster; the
field design lets one object state both.

The cost is a permanent addition to the policy API, which is a higher bar than a Helm value and
harder to withdraw.

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

## Impacts / Key Questions

### Impact: Policy API Growth

Adding a third top-level rule list to `CiliumClusterwideNetworkPolicy` — alongside `ingress`,
`ingressDeny`, `egress`, `egressDeny` — enlarges an API that is already large, and invites the
question of whether `preNATIngressDeny` must follow. The proposal is to ship `preNATIngress` alone
and add a deny variant only if a concrete need appears; `fromCIDRSet.except` covers the known
exclusion use cases.

### Impact: A New Silent-Failure Mode of Its Own

If an operator writes `preNATIngress` on a cluster where host firewall or NodePort is disabled, the
rules do nothing. This is mitigated by the status condition described under
[Observability](#observability), but it is a real cost of the design and worth weighing against the
flag alternative, where the same mismatch is at least visible in one place.

### Impact: Per-Packet Cost on New Flows

New flows arriving on native devices take an extra tail call, an ipcache lookup, and a policy map
lookup before LB processing. Established flows are unaffected. The cost is only paid on nodes where
the stage is active. Benchmarking on the NodePort request-rate path is required before merge, with
an explicit target of no measurable regression when no `preNATIngress` policy exists.

### Key Question: Field Naming

The field names the hook point, so the name should be accurate about where it is.

#### Option 1: `preNATIngress`

##### Pros

* Accurate: the hook precedes DNAT (destination rewrite) and SNAT (source rewrite), and the
  motivating use cases depend on both.
* Reads correctly for the port-matching use case, which is a DNAT concern, not an SNAT one.

##### Cons

* "NAT" is broad; a reader may wonder which NAT, given the datapath performs several.

#### Option 2: `preSNATIngress` (as suggested in review)

##### Pros

* Names the transformation operators most often complain about — losing the client source IP.

##### Cons

* Understates the change: the primary motivating rule matches a *destination port*, which is a DNAT
  concern. A user reading `preSNATIngress` has no reason to expect it fixes port matching.

#### Option 3: A stage discriminator instead of a new field

```yaml
spec:
  ingress:
  - enforcementStage: PreNAT      # default: PostNAT
    fromEntities: [world]
```

##### Pros

* No new top-level field; extends to future stages without further API growth.
* Deny variants come for free.

##### Cons

* Mixing stages within one list makes default-deny scoping harder to read, since the reader must
  scan every rule to know which stages are activated.
* A per-rule field in a list is a weaker signal than a top-level field for something as consequential
  as enforcement point.

### Key Question: Scope of the Pre-NAT Stage

Does `preNATIngress` apply to all traffic entering the native device, or only to traffic that
resolves to an LB frontend?

#### Option 1: All traffic on the native device (proposed)

##### Pros

* Simple, uniform mental model: "this is policy on the packet as it arrived".
* Lets the field express a general node-perimeter policy, which is what several of the motivating
  deployments actually want.
* No dependency on LB lookup ordering; the verdict does not change when a Service is created or
  deleted.

##### Cons

* Broader blast radius: an incomplete allowlist can black-hole traffic that has nothing to do with
  NodePort, including control plane and health traffic.
* Requires authors to think about VXLAN/Geneve, WireGuard, and IPsec traffic that terminates on the
  node.

#### Option 2: Only traffic matching an LB frontend

##### Pros

* Narrowly targeted at the reported problem; much smaller blast radius.

##### Cons

* Requires the LB lookup before the policy check, so the "original" packet is only original by
  bookkeeping, and the verdict becomes coupled to Service churn.
* Cannot express "block the NodePort range", since a port with no Service behind it does not match a
  frontend — which is exactly rule (1) of the motivating example.

### Key Question: Interaction With Encapsulated and Encrypted Traffic

Traffic arriving in a VXLAN/Geneve tunnel, or as ESP under IPsec, is opaque at the `from-netdev`
hook. The proposal is that `preNATIngress` matches the *outer* packet — the tunnel or ESP packet as
it arrived — and that inner traffic continues to be subject to the existing post-decapsulation
policy stages. This is defensible but needs to be stated loudly in documentation, since a rule
allowing `fromEntities: cluster` to the tunnel port is what keeps a cluster reachable, and its
absence under default-deny will partition the cluster. An alternative is to exempt tunnel and ESP
traffic from the stage entirely; that is safer but makes the stage unable to police the tunnel
endpoint itself.


#### Option 1: Ingress only, now (proposed)

##### Pros

* Matches the reported use cases; no speculative API.
* Keeps the first iteration reviewable.

##### Cons

* If the egress side is added later, the naming and structure chosen now constrain it.

#### Option 2: Both directions in one change

##### Pros

* Symmetric API; one design discussion instead of two.

##### Cons

* No concrete demand for the egress side; doubles datapath and test surface for speculative benefit.

### Key Question: New Field vs. New Kind

An alternative to extending `CiliumClusterwideNetworkPolicy` is a dedicated kind, e.g.
`CiliumNodePerimeterPolicy`.

#### Option 1: New field on `CiliumClusterwideNetworkPolicy` (proposed)

##### Pros

* Reuses `nodeSelector`, the rule types, `CiliumCIDRGroup` references, status reporting, and the
  entire policy pipeline.
* Node policy stays in one object, so an operator reads one YAML to know what a node enforces.

##### Cons

* Adds to an already-large CRD, and makes `CiliumClusterwideNetworkPolicy` mean two different things
  depending on which fields are set.

#### Option 2: A separate CRD

##### Pros

* Clean separation; the new stage's restrictions (no L7, no auth, node-scoped only) are expressed by
  the type rather than by validation rules on a general-purpose type.
* Independent RBAC, so the perimeter policy can be owned by the platform team while application
  teams retain `CiliumClusterwideNetworkPolicy`.

##### Cons

* Duplicates a large amount of schema and controller code.
* Two objects must now be read together to understand what a node enforces, and their interaction
  becomes a documentation burden.

The RBAC separation is a genuine argument for the separate kind and matches the motivating
deployment model, where a platform operator — not the application owner — sets the node perimeter.
It is the strongest counter-argument to the proposal as written and is offered here for discussion
rather than settled.
