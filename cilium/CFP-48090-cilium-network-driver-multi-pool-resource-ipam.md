# CFP-48090: Multi-Pool Resource IPAM for the Cilium Network Driver

**SIG:** SIG-IPAM, Network Driver

**Begin Design Discussion:** 2026-08-21

**Cilium Release:** 1.21

**Authors:** Fabio Falzoi <fabio.falzoi@isovalent.com>

**Status:** Draft

**Related Issue:** [cilium/cilium#48090](https://github.com/cilium/cilium/issues/48090)

**Related CFP:** [CFP-43295: Cilium Network (DRA) Driver](./CFP-43295-cilium-network-driver-dra.md)

## Summary

This CFP proposes Multi-Pool Resource IPAM, a dedicated IP address management
subsystem for network devices allocated by the Cilium Network Driver through
Kubernetes Dynamic Resource Allocation (DRA).

Cluster operators define named, cluster-wide address pools. The Cilium operator
delegates CIDR blocks from those pools to nodes according to demand, and the
Cilium Network Driver allocates individual addresses locally while preparing DRA
`ResourceClaim` objects. Pools may optionally associate an interface prefix
length and static routes with each CIDR; the Network Driver applies that
configuration while preparing a resource. Addresses are released when claims
are unprepared.

Resource IPAM is separate from Pod IPAM. It can be enabled alongside any Pod
IPAM mode and does not allocate from, or write allocation state to,
`CiliumPodIPPool` resources.

## Motivation

The Cilium Network Driver allows workloads to claim network devices such as
SR-IOV virtual functions. A claimed device commonly needs an IPv4 address, an
IPv6 address, or both before it can be used by the workload.

Embedding static addresses in `ResourceClaim` or `ResourceClaimTemplate`
configuration is not a scalable solution. In particular, a
`ResourceClaimTemplate` is reused to create multiple claims, so a static address
in the template risks being assigned to more than one device. Static assignment
also requires cluster operators to coordinate address ownership with workload
and claim lifecycles manually.

Existing Pod IPAM modes cannot be used directly for these addresses. Pod IPAM
manages the primary Cilium interface and follows the CNI endpoint lifecycle.
Network Driver addresses belong to secondary DRA resources and follow DRA
prepare and unprepare operations. Sharing the same pool objects and allocation
state would couple otherwise independent capacity and lifecycles.

The IPAM section of CFP-43295 was deferred so that allocation models could be
evaluated separately. In particular, centrally allocating every address in the
operator can improve aggregate pool utilization, while delegating CIDRs to
nodes avoids an API and operator transaction for every claim. This proposal
selects node-local allocation from operator-delegated CIDRs, following the
existing Multi-Pool Pod IPAM control-plane model.

## Goals

- Allocate IPv4 and IPv6 addresses dynamically for network devices prepared by
  the Cilium Network Driver.
- Support multiple named, cluster-wide address pools for independent secondary
  networks and device classes.
- Keep Resource IPAM configuration, capacity, and allocation state independent
  from Pod IPAM.
- Allocate and release individual addresses locally without a Kubernetes API
  transaction for every address operation.
- Scale local pool capacity according to per-node demand and return unused delegated
  CIDRs safely.
- Preserve successful allocations across Cilium agent restarts and make DRA
  prepare and unprepare retries idempotent.
- Support IPv4-only, IPv6-only, and dual-stack Network Driver configurations
  independently of Pod IPAM.
- Apply optional per-CIDR interface prefix lengths and static routes while
  preparing DRA network resources.
- Surface allocated addresses through standard DRA allocated-device status.

## Non-Goals

- Changing how the Cilium Network Driver discovers, publishes, or allocates
  network devices.
- Managing primary Pod addresses or changing any existing Pod IPAM mode.
- Establishing or validating the underlying network connectivity, configuring
  VLANs, or integrating the secondary interface with the Cilium datapath.
- Designing a general network-configuration API for DRA resources.
- Integrating external IPAM providers in the initial implementation.
- Making IP pool capacity or topology part of DRA scheduling decisions.
- Selecting different Resource IPAM pools for IPv4 and IPv6 within one device
  request in the initial implementation.

## Proposal

### Overview

Multi-Pool Resource IPAM is an additive Network Driver capability. It is not a
new value of Cilium's global `ipam.mode` setting. The Resource IPAM components
run when the Cilium Network Driver is enabled, but address allocation is opt-in
for each DRA device request: a request that does not name a Resource IPAM pool
does not receive a dynamically allocated address from this subsystem.

The design reuses the control-plane model and allocator primitives of
Multi-Pool Pod IPAM, with separate Kubernetes resources and separate fields in
`CiliumNode`:

```mermaid
flowchart LR
    Pool[CiliumResourceIPPool] -->|pool CIDRs and node mask| Operator
    NodeRequest[CiliumNode spec.ipam.resourcePools.requested] -->|aggregate demand| Operator
    Operator[Cilium operator Resource IPAM allocator] -->|delegated CIDRs| NodeAllocation[CiliumNode spec.ipam.resourcePools.allocated]
    NodeAllocation --> AgentDriver["Cilium Network Driver<br/> local Resource IPAM allocator"]
    Claim[ResourceClaim with ip-pool] -->|prepare/unprepare claim| AgentDriver
    AgentDriver -->|allocated device data and IPs| ClaimStatus[ResourceClaim status.devices]
```

The operator owns the cluster-wide view of each Resource IPAM pool and assigns
non-overlapping CIDRs from a pool to nodes. Each network driver (embedded in the
node local agent) owns individual address allocation within its delegated CIDRs.
The driver reports aggregate demand and which delegated CIDRs it still uses;
it does not report every individual allocation through `CiliumNode`.

### Cluster-wide pools

A new cluster-scoped `CiliumResourceIPPool` (`cilium.io/v2alpha1`) resource
defines an allocation domain for Network Driver resources. Its address-family
configuration intentionally follows `CiliumPodIPPool`, but it omits Pod and
Namespace selection because a DRA device request selects the pool explicitly.

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumResourceIPPool
metadata:
  name: blue-network
spec:
  ipv4:
    cidrs:
      - 10.20.0.0/16
      - 10.30.0.0/16
    maskSize: 24
    networkConfig:
      - cidr: 10.20.0.0/16
        prefixLength: 28
        routes:
          - destination: 10.20.0.0/24
            gateway: 10.20.0.1
      - cidr: 10.30.0.0/16
        prefixLength: 30
        routes:
          - destination: 10.30.0.0/24
            gateway: 10.30.0.1
  ipv6:
    cidrs:
      - fd00:20::/48
    maskSize: 64
  allowFirstIP: false
  allowLastIP: false
```

For each family:

- `cidrs` defines the cluster-wide address space owned by the pool.
- `maskSize` defines the CIDR size delegated to an individual node. It is an
  allocation granularity and is not the prefix length configured on the
  workload interface.
- `networkConfig` optionally associates an interface `prefixLength` and a list
  of static routes with individual CIDRs in `cidrs`.
- `allowFirstIP` and `allowLastIP` control whether the first and last address of
  each delegated CIDR can be allocated. For `/31`, `/32`, `/127`, and `/128`
  CIDRs, addresses remain usable regardless of these settings.

The mask sizes and first/last-address settings are immutable because changing
their interpretation while nodes hold delegated CIDRs could make live
allocations invalid. Pool CIDR lists can be extended at runtime. Removing CIDRs
or deleting a pool stops the operator from delegating new CIDRs from that
capacity. CIDRs already delegated to nodes remain reserved until the agents can
release them safely, and their remaining local capacity may still satisfy
allocations during that transition.

The pool name scopes uniqueness. Cilium guarantees that two live allocations
from the same `CiliumResourceIPPool` do not receive the same address. Different
pools are independent allocation domains and may intentionally use overlapping
address space when they represent isolated networks. It is the responsibility
of cluster operators to ensure that pool addressing is aligned with the
underlying network infrastructure to avoid undesired side effects.

### Basic network configuration

The optional `networkConfig` list defines basic network configuration for
addresses allocated from individual pool CIDRs. A list may cover any subset of
the CIDRs for its address family, with at most one entry for each CIDR. Each
entry can define the prefix length assigned to the allocated address and static
routes to install on the claimed resource. A route contains a destination and
may contain a gateway from the same address family.

Every `networkConfig[].cidr` must exactly match a CIDR in the `cidrs` list for
the same address family. The API server rejects a pool that violates this
constraint.

The `networkConfig` field uses Kubernetes list-map semantics with `cidr` as its
key, preventing duplicate entries for the same CIDR. Removing a CIDR therefore
also requires removing its network configuration in the same update or an
earlier update.

The Network Driver watches `CiliumResourceIPPool` objects and uses its current
view of the selected pool during resource preparation. After allocating an
address, it finds the network configuration for the pool CIDR containing that
address and applies the configured prefix length and routes to the claimed
resource. If the corresponding pool CIDR has no `networkConfig` entry, the
address uses a host prefix (`/32` for IPv4 or `/128` for IPv6) and no routes are
installed from the pool.

Network configuration is applied only at preparation time. Updating an entry
has no effect on resources that were prepared using its previous value. After a
CIDR and its entry are removed from the pool, the previous configuration is not
applied to subsequent preparations, including allocations from that CIDR's
capacity while it remains delegated to a node. Resources prepared before the
change retain their existing configuration.

### Per-node demand and CIDR delegation

`CiliumNode.spec.ipam` gains a separate `resourcePools` section with the same
request/allocation handshake used by Multi-Pool Pod IPAM:

```yaml
apiVersion: cilium.io/v2
kind: CiliumNode
metadata:
  name: worker-1
spec:
  ipam:
    resourcePools:
      requested:
        - pool: blue-network
          needed:
            ipv4-addrs: 3
            ipv6-addrs: 3
      allocated:
        - pool: blue-network
          cidrs:
            - 10.20.1.0/24
            - fd00:20:0:1::/64
          allowFirstIP: false
          allowLastIP: false
```

The fields have the following ownership:

- The agent writes `requested`, expressing the number of addresses it needs
  from each pool and family.
- The operator adds CIDRs to `allocated` until the delegated usable capacity
  satisfies the request.
- The agent removes a delegated CIDR from `allocated` only after all addresses
  from it have been released and the CIDR is no longer needed.

The operator runs a second instance of the existing Multi-Pool allocator for
Resource IPAM. Separate field accessors ensure that the Pod and resource
allocators reconcile only their respective `CiliumNode` fields. Separate pool
objects and allocator instances prevent a Network Driver allocation from
consuming Pod IPAM capacity, regardless of the Pod IPAM mode in use.

### Selecting a pool from a DRA request

The Network Driver's opaque DRA configuration gains an `ip-pool` parameter. It
contains the name of a `CiliumResourceIPPool` and is associated with one or more
device requests through the standard DRA `config.requests` field:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: sriov-blue-network
spec:
  spec:
    devices:
      requests:
        - name: network-device
          exactly:
            deviceClassName: sriov.cilium.k8s.io
      config:
        - requests:
            - network-device
          opaque:
            driver: sriov.cilium.k8s.io
            parameters:
              ip-pool: blue-network
```

Note that the Resource IPAM pool is a distinct and unrelated concept from
the DRA ResourceSlice pool recorded in device allocation results.
The former selects an address allocation domain; the latter identifies
the device pool from which Kubernetes allocated the hardware resource.

There is no implicit default and no fallback to another Resource IPAM pool. If
`ip-pool` is omitted, Resource IPAM performs no allocation for that request. If
the named pool does not exist, does not provide a required address family, or
is exhausted, claim preparation fails with a retryable error rather than
configuring the device for a different network.

Address-family support for Network Driver resources is controlled independently
by the `--enable-network-driver-ipv4` and `--enable-network-driver-ipv6` options.
They respectively enable the assignment of IPv4 and IPv6 addresses, regardless
of the families enabled for the Cilium agent's Pod IPAM. This separation is
intentional because DRA resources can represent a different network topology
and dataplane from the one used by pods.

### Claim preparation and allocation

The Cilium Network Driver allocates addresses as part of
`PrepareResourceClaims`, before applying the final device configuration:

```mermaid
sequenceDiagram
    participant Kubelet
    participant Driver as Cilium Network Driver
    participant AgentIPAM as Agent Resource IPAM
    participant CiliumNode
    participant Operator as Cilium operator
    participant Claim as ResourceClaim status

    Kubelet->>Driver: PrepareResourceClaims
    Driver->>AgentIPAM: Allocate address from named pool
    alt Local delegated capacity is available
        AgentIPAM-->>Driver: IPv4 and/or IPv6 address
    else No local capacity is available
        AgentIPAM->>CiliumNode: Increase aggregate requested demand
        Operator->>CiliumNode: Add delegated CIDR to allocated
        CiliumNode-->>AgentIPAM: Observe delegated CIDR
        Driver->>AgentIPAM: Retry allocation
        AgentIPAM-->>Driver: IPv4 and/or IPv6 address
    end
    Driver->>Driver: Resolve networkConfig for allocated addresses
    Driver->>Driver: Configure prefix lengths, routes, and device
    Driver->>Claim: Persist pool, addresses, and device data
    Driver-->>Kubelet: Resource prepared
```

Note that the Network Driver and Agent Resource IPAM are shown as separate
participants to clarify their responsibilities, but both are embedded components
in the same Cilium agent binary and communicate in-process.

An address allocation is local when a delegated CIDR has free capacity. If no
address is available, the agent records a pending allocation, updates aggregate
demand in `CiliumNode`, and retries while the operator assigns capacity. The
kubelet can retry claim preparation if capacity cannot be obtained during the
current call. An unavailable operator therefore prevents allocations that need
new CIDRs, but allocations from capacity already present on the node continue
without the operator.

The driver records allocated addresses in two places in
`ResourceClaim.status.devices`:

- The driver's opaque `data` contains the selected Resource IPAM pool and the
  complete serialized device configuration, including the resolved prefix
  lengths and routes, needed for restore and release.
- Standard `networkData.ips` exposes the addresses to Kubernetes and users.

Resource IPAM uses host prefixes (`/32` for IPv4 and `/128` for IPv6) when the
pool CIDR containing an allocated address has no `networkConfig` entry. When an
entry exists, the Network Driver uses its `prefixLength` and installs its static
routes. The `spec.ipv4.maskSize` and `spec.ipv6.maskSize` values remain node
delegation granularities and are not used as interface prefixes.

If the Network Driver cannot apply a configured prefix length or route, resource
preparation fails and the error is returned to the kubelet. As with any other
failure after address allocation, the driver releases newly allocated addresses
and rolls back partial device setup. If the Kubernetes status update fails, the
device setup and address allocations performed by that preparation attempt are
also rolled back, allowing a later retry to start cleanly.

### Demand and preallocation

For each pool and family, the agent computes demand from:

- addresses currently in use by successfully prepared resources;
- allocations currently pending because local capacity is exhausted; and
- an optional per-pool preallocation buffer.

The preallocation buffer defaults to zero. A value of zero requests capacity
only in response to current or pending use. When the buffer is greater than
zero, the Network Driver requests addresses in advance so that subsequent
resource preparations can allocate locally without waiting for the operator to
delegate additional CIDRs. Demand is increased in buffer-sized steps to keep at
least the configured number of addresses available above current use once the
operator has satisfied the request.

Cluster operators should choose the buffer size based on:

- the number of DRA resources available on each node; and
- how much underutilization of the global pool is acceptable when addresses are
  delegated to nodes but remain unused.

For example, consider an IPv4 pool containing a single `/26` CIDR with a
`maskSize` of `/30`, `allowFirstIP: true`, and `allowLastIP: true`. All 64
addresses are usable, and the operator delegates them to nodes in `/30` CIDRs
containing four addresses each. The cluster has four nodes, and each Network
Driver uses a preallocation buffer of eight addresses. At startup, each driver
requests eight addresses and receives two `/30` CIDRs. The four nodes therefore
hold 32 addresses in total, leaving 32 addresses undelegated in the global
pool.

If DRA resources are then prepared only on one node, its capacity changes as
follows after the operator has reconciled each satisfiable request:

| Prepared resources | Requested capacity | Delegated capacity | Free addresses on the node | Undelegated global capacity | Result |
|---:|---:|---:|---:|---:|---|
| 0 | 8 | 8 | 8 | 32 | Startup preallocation assigns two `/30` CIDRs to every node. |
| 1 | 16 | 16 | 15 | 24 | Preparation uses local capacity; the driver then requests two more `/30` CIDRs. |
| 4 | 16 | 16 | 12 | 24 | No additional CIDRs are needed. |
| 8 | 16 | 16 | 8 | 24 | The full eight-address buffer remains available. |
| 9 | 24 | 24 | 15 | 16 | The driver requests two more `/30` CIDRs. |
| 16 | 24 | 24 | 8 | 16 | The full buffer remains available. |
| 17 | 32 | 32 | 15 | 8 | The driver requests two more `/30` CIDRs. |
| 24 | 32 | 32 | 8 | 8 | The full buffer remains available. |
| 25 | 40 | 40 | 15 | 0 | The driver receives the last two undelegated `/30` CIDRs. |
| 32 | 40 | 40 | 8 | 0 | Allocations continue from local capacity. |
| 33–40 | 48 | 40 | 7–0 | 0 | The request for more CIDRs cannot be satisfied, but preparation continues while local addresses remain. |
| 41st attempt | 56 | 40 | 0 | 0 | No local or global capacity remains, so preparation retries and eventually fails. |

The other three nodes still retain eight addresses each even if no DRA resource
is prepared on them, because those addresses satisfy their configured buffers.
They do not return the corresponding CIDRs while the buffers remain configured.
Consequently, 24 addresses remain unused when the busy node exhausts its local
capacity, illustrating the global pool underutilization that larger
preallocation buffers can introduce.

After startup restore is complete, the agent returns excess, completely unused
CIDRs by removing them from `CiliumNode.spec.ipam.resourcePools.allocated`. A
CIDR containing any restored or live allocation is never returned.

### Unprepare and restart recovery

`UnprepareResourceClaims` releases the addresses recorded for each device back
to the node-local pool. Repeated unprepare calls are safe when the allocation is
already absent.

On agent restart, the Network Driver first waits for the local `CiliumNode`
resource so that all delegated CIDRs are available. It then restores local
device and address ownership from `ResourceClaim.status.devices`. Only after
all local claims have been processed does it mark IPAM restore complete and
allow unused delegated CIDRs to be returned. This ordering prevents the agent
from releasing a CIDR before rediscovering a live address from it.

If a node is deleted, the operator can reclaim its delegated CIDRs using the
existing Multi-Pool node lifecycle. A merely unreachable node retains its
CIDRs, preventing them from being reassigned while allocations may still exist.

Successful preparations are idempotent because retries reuse the allocation
stored for the same Pod, claim, and device.

### Configuration and feature activation

Multi-Pool Resource IPAM is available only when the Cilium Network Driver is
enabled. It does not require `ipam.mode=multi-pool`; the cluster may use any
supported Pod IPAM mode.

- The Network Driver can independently enable address assignment for each
  family with `--enable-network-driver-ipv4` and `--enable-network-driver-ipv6`.
  These options default to `true` and `false`, respectively, so only IPv4
  address assignment is enabled by default.

- The operator can auto-create pools at startup with
  `--auto-create-cilium-resource-ip-pools`.
- The agent can configure warm capacity per pool with
  `--resource-ipam-multi-pool-pre-allocation=<pool>=<addresses>`. See
  [Demand and preallocation](#demand-and-preallocation) for allocation behavior
  and sizing guidance.

Pools can also be created and managed directly as Kubernetes resources. The
operator and agent receive the RBAC permissions required to observe the new
resource and reconcile the corresponding `CiliumNode` state. The CRD is
installed with the Cilium chart.

### Observability

The following Kubernetes objects expose the allocation path without recording
every IP allocation in `CiliumNode`:

- `CiliumResourceIPPool` shows the configured cluster-wide capacity, node
  allocation granularity, and optional per-CIDR network configuration.
- `CiliumNode.spec.ipam.resourcePools` shows per-node demand and delegated
  CIDRs.
- `ResourceClaim.status.devices[].networkData.ips` shows addresses allocated to
  a prepared device.
- Allocated addresses are also recorded in the
  [`networkdriver-dra-devices` StateDB table introduced by cilium/cilium#47558](https://github.com/cilium/cilium/pull/47558)
  and can be inspected with
  `cilium-dbg shell -- db/show networkdriver-dra-devices`.

Operator and agent logs report missing pools, exhausted families, pending
capacity, allocation failures, network configuration failures, and unsafe pool
updates. Resource IPAM metrics should expose delegated, used, free, and pending
address counts by pool and family, plus allocation failures, without using
individual addresses as metric labels. Sysdump collection should include
`CiliumResourceIPPool` and the relevant `CiliumNode` state.

## Impacts / Key Questions

### Allocation model: delegated CIDRs versus per-address coordination

The primary trade-off is address utilization versus allocation-path
coordination:

| Model | Allocation path | API/control-plane load | Utilization | Failure behavior | Complexity |
|---|---|---|---|---|---|
| Per-node CIDR delegation (proposed) | Agent allocates locally | One update per demand or CIDR change | May strand free addresses on nodes | Existing node capacity remains usable without the operator | Reuses Cilium Multi-Pool allocators |
| Central per-address allocation | Operator assigns every address | At least one coordinated transaction per allocation/release | High aggregate utilization | Every new allocation depends on operator and API availability | Central ownership is straightforward, but claim integration and throughput must be implemented |

This CFP chooses per-node CIDR delegation because Network Driver allocations
occur on the node, allocation latency is part of Pod startup, and the existing
Multi-Pool implementation already provides CIDR ownership, demand signaling,
restore ordering, and safe CIDR return.

### Address utilization and fragmentation

Delegating CIDRs can strand free addresses on nodes, especially when a pool is
used by few resources per node. The pool's `maskSize` bounds this cost. A `/32`
or `/128` delegation minimizes fragmentation but approaches the API behavior of
central per-address allocation; larger blocks reduce API churn and claim
latency but reserve more capacity per active node. Preallocation introduces the
same deliberate utilization/latency trade-off.

### Pool representation: CIDRs versus arbitrary ranges

For the initial implementation, `CiliumResourceIPPool` represents pool capacity
as CIDRs, following the API and implementation of Multi-Pool Pod IPAM. This
allows Resource IPAM to reuse the existing Cilium allocator interfaces and
fixed-size CIDR delegation logic.

This initial choice does not preclude adding an explicit range representation
in the future, using `startAddr` and `endAddr`. Such a representation could
describe non-CIDR-aligned address plans more compactly. Supporting it will
require defining how ranges are divided into node delegation units and how
range updates interact with existing ownership. Until then, an arbitrary range
can be expressed as a set of CIDRs, at the cost of a potentially longer pool
specification.

### Separate Resource IP pools versus reusing Pod IP pools

Reusing `CiliumPodIPPool` would reduce the number of CRDs, but it would mix
capacity and lifecycle ownership between primary Pod interfaces and DRA
resources. Pool selectors also operate on Pods and Namespaces, while Resource
IPAM selection is attached to a DRA device request.

`CiliumResourceIPPool` deliberately duplicates the address-family portion of
the Pod pool API so that the two allocation systems remain independent. Shared
Go types and allocator components limit implementation duplication. The cost is
an additional cluster-scoped CRD and separate operator reconciliation state.

### Pool-scoped versus separate network configuration

Keeping basic network configuration in `CiliumResourceIPPool` associates an
allocated address, its interface prefix length, and its static routes in one
administrative object. It also lets the Network Driver obtain the configuration
from the pool it already watches, without coordinating a second resource lookup
or allowing unrelated network configuration to select the address semantics.

The trade-off is that every address allocated from a configured CIDR receives
the same basic network configuration on every node. This API does not represent
topology-specific gateways or routes, VLANs, sysctls, or device-specific
settings. Such configuration requires a future Network Driver API rather than
expanding the pool into a general network-configuration object.

### Overlapping pools

Global overlap validation would reject valid deployments in which isolated
secondary networks reuse the same addresses. The allocator therefore treats
each `CiliumResourceIPPool` as a separate uniqueness domain. This flexibility
also makes misconfiguration possible: overlapping pools attached to the same
network can allocate duplicate addresses. The documentation and status tooling
must make pool identity visible wherever an address is shown.

### Operational complexity compared with Cluster Pool IPAM

Multi-Pool extends the Cluster Pool allocation model. In the simplest
configuration, the operator's
`--auto-create-cilium-resource-ip-pools` option creates a single `default` pool
and the cluster uses only that pool, providing practically the same operational
experience as Cluster Pool IPAM.

The additional complexity is optional: operators that need multiple allocation
domains can define separate named pools backed by different address ranges.
This provides the flexibility required when secondary networks or device
classes must draw addresses from independently managed ranges without burdening
the single-pool case.

### Coexistence with statically managed addresses

Static and dynamic address management can coexist through the
[`reservedRanges` mechanism proposed for `CiliumPodIPPool`](https://github.com/cilium/cilium/pull/46880).
It keeps reserved addresses within a pool's configured CIDRs while preventing
the Multi-Pool allocator from delegating any allocation CIDR that overlaps
them. Operators can therefore retain ranges for static address management
without risking their reuse by dynamic allocation.

Multi-Pool Resource IPAM will provide the same reservation mechanism in
`CiliumResourceIPPool`. Operators will be able to declare the portions of a
resource pool reserved for static assignments, and the operator will exclude
overlapping allocation CIDRs from node delegation. Reservations are explicit;
Resource IPAM will not discover statically assigned addresses automatically.
Because exclusion operates at delegation-CIDR granularity, pool boundaries and
`maskSize` should be chosen so that reservations do not make more dynamic
capacity unavailable than intended.

## Future Milestones

### Per-family pool selection

Allow a DRA request to select different IPv4 and IPv6 Resource IPAM pools. This
would support deployments in which address families are administered through
different allocation domains.

### Topology-aware network configuration

Extend the basic pool-scoped configuration through a future Network Driver API
for topology-specific gateways and routes, VLANs, sysctls, and other settings
that can differ between nodes while remaining consistent with the selected
Resource IPAM pool.

### Alternative allocation backends

If measurements show unacceptable fragmentation for large or sparse clusters,
add a centralized per-address or shared-pool backend behind the Network Driver
IPAM interface without changing the DRA request lifecycle.

### External IPAM integration

Allow Network Driver resource addresses to be allocated by an external IPAM
provider while preserving idempotent DRA prepare, unprepare, and restart
semantics.

## Prototype implementation

The following PRs, merged into the Network Driver `feature/dra-driver` branch,
form an earlier prototype of Multi-Pool Resource IPAM. They can be used as
implementation references, but they are non-normative and do not constitute the
definitive implementation of this CFP:

- [cilium/cilium#44081: Add the Multi-Pool Resource IPAM operator cell](https://github.com/cilium/cilium/pull/44081)
- [cilium/cilium#44124: Add agent and Network Driver support for Multi-Pool Resource IPAM](https://github.com/cilium/cilium/pull/44124)
- [cilium/cilium#46645: Add script-based coverage for Resource IPAM allocation paths](https://github.com/cilium/cilium/pull/46645)
- [cilium/cilium#46758: Support first- and last-address allocation policy](https://github.com/cilium/cilium/pull/46758)
- [cilium/cilium#46862: Correct the Resource IPAM preallocation flag description](https://github.com/cilium/cilium/pull/46862)

## References

- [Resource IPAM feature issue](https://github.com/cilium/cilium/issues/48090)
- [Cilium Network Driver feature issue](https://github.com/cilium/cilium/issues/43295)
- [CFP-43295 review discussion deferring IPAM](https://github.com/cilium/design-cfps/pull/85#discussion_r3101632653)
- [Cilium Multi-Pool Pod IPAM](https://docs.cilium.io/en/stable/network/concepts/ipam/multi-pool/)
- [Kubernetes Dynamic Resource Allocation](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/)
