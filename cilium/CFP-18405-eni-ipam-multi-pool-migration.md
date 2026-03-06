# CFP-18405: ENI IPAM Multi-Pool Migration

**SIG:** SIG-IPAM

**Begin Design Discussion:** 2026-03-06

**Cilium Release:** 1.20

**Authors:** Hadrien Patte <hadrien.patte@datadoghq.com>

**Status:** Draft

## Summary

Migrate ENI IPAM mode from the CRD allocator to the multi-pool allocator to reduce Kubernetes API churn, simplify prefix delegation handling, and lay the groundwork for future IPv6 support.

## Motivation

ENI IPAM mode currently uses the CRD allocator: the operator writes individual IPs to `CiliumNode.Spec.IPAM.Pool` and the agent allocates from there, writing per-IP status back to `Status.IPAM.Used`. This design has a couple limitations.

**IPv6 is impossible.** The CRD allocator works by flattening prefixes into individual IP entries. For IPv4 prefix delegation, AWS assigns /28 prefixes (16 IPs), which is manageable to flatten. For IPv6, AWS assigns /80 prefixes (2^48 IPs). Flattening these into individual `Spec.IPAM.Pool` entries is not technically possible. Without migrating to a CIDR-aware allocator, ENI IPv6 delegated prefix support cannot be implemented.

**Excessive Kubernetes API churn.** Every IP allocation and deallocation requires a CiliumNode status update. Since all agents watch all CiliumNodes, each update fans out to every node in the cluster. In a cluster with N nodes, a single allocation event generates N watch notifications. Under sustained churn (rolling deployments, autoscaling, job workloads), this creates significant and unnecessary load on the Kubernetes API server and etcd.

## Goals

* Migrate ENI IPAM mode (both secondary IP and prefix delegation) to the multi-pool allocator
* Decouple ENI device configuration from the allocator code path
* Reduce Kubernetes API churn for IP allocation and deallocation
* Preserve backward compatibility for existing ENI IPAM users during the transition

## Non-Goals

* IPv6 support in ENI IPAM mode (deferred milestone)
* IPv6 prefix delegation support in ENI IPAM mode (deferred milestone)
* Changes to other cloud IPAM modes (Azure, AlibabaCloud)

## Proposal

### Overview

The multi-pool allocator represents IPs as CIDRs in `CiliumNode.Spec.IPAM.Pools.Allocated`. The agent allocates IPs locally from these CIDRs without writing per-IP status back to the `CiliumNode` object. Demand is communicated in aggregate via `Spec.IPAM.Pools.Requested`. This is exactly the interface needed to support both secondary IPs (as /32 CIDRs) and prefix delegation (as /28 CIDRs, without expansion).

### Current Architecture (CRD Allocator)

Secondary IPs:
1. Operator allocates ENI and secondary IPs from the AWS EC2 API.
2. Operator writes each IP individually to `CiliumNode.Spec.IPAM.Pool`.
3. Agent picks IPs from `Pool`, marks them in `Status.IPAM.Used`.
4. Each allocation or deallocation triggers a CiliumNode status patch visible to all watching agents.

Prefix delegation:
1. Operator allocates ENI and a /28 prefix from AWS.
2. Operator expands the /28 into 16 individual IPs and writes them to `Spec.IPAM.Pool`.
3. Agent allocates IPs from the expanded pool, same as secondary IPs.
4. To release a prefix, the operator must scan all 16 entries in `Status.IPAM.Used` to confirm the prefix is idle before calling `UnassignENIPrefixes`.

### Proposed Architecture (Multi-Pool Allocator)

Secondary IPs:
1. Operator allocates ENI and secondary IPs from AWS (unchanged).
2. Operator writes each IP as a /32 CIDR in `CiliumNode.Spec.IPAM.Pools.Allocated` under pool `"eni"` (see "Key Question 2").
3. Agent allocates IPs locally from the CIDR pool. No per-IP Kubernetes update is required.
4. Agent reports aggregate demand via `Spec.IPAM.Pools.Requested[pool:"eni"].needed.ipv4Addrs`.

Prefix delegation:
1. Operator allocates ENI and a /28 prefix from AWS (unchanged).
2. Operator writes the /28 CIDR directly to `Spec.IPAM.Pools.Allocated` without expansion.
3. Agent allocates IPs locally from the /28. No expansion or per-IP tracking is needed.
4. Agent reports aggregate demand; operator scales prefix count accordingly.
5. When the agent has fully released a prefix (all IPs freed), it removes the CIDR from `Allocated`. The operator detects this and calls `UnassignENIPrefixes` without needing to scan individual IP entries.

### ENI Device Configuration Decoupling

ENI device configuration is currently coupled to the CRD allocator. `configureENIDevices` is called from `nodeStore.updateLocalNodeResource` in `pkg/ipam/crd_eni.go`, which ties ENI interface setup to the CRD allocator code path. In order to allow using a different allocator, we need to restructure the device configuration integration point. `configureENIDevices` must be extracted from the CRD allocator code path before the allocator can be replaced. The proposed approach:
* Create a standalone `startENIDeviceWatcher` that sets up its own CiliumNode informer.
* This watcher calls `configureENIDevices` independently of which allocator is active.
* The CRD-allocator-specific call site in `nodeStore.updateLocalNodeResource` (`pkg/ipam/crd_eni.go`) is removed.

This ensures ENI network interfaces are configured correctly regardless of allocator choice.

### Operator-Side Changes

The AWS EC2 API interactions (ENI creation, IP assignment, prefix delegation) remain largely unchanged. The changes are limited to how results are written to CiliumNode and how demand is read:
* Write path: Write CIDRs to `Spec.IPAM.Pools.Allocated` (new format). During the transition release, also dual-write individual IPs to `Spec.IPAM.Pool` for backward compatibility (see Migration Strategy below).
* Read path: Read demand from `Spec.IPAM.Pools.Requested` when a new agent is running on the node, and from `Status.IPAM.Used` when an old agent is running. The two are mutually exclusive per node: whichever field is populated determines the agent version.
* Prefix release detection: Instead of scanning 16 individual IP entries, check whether the /28 CIDR is still present in `Allocated`.

### First/Last IP Reservation for Prefix Delegation CIDRs

The multi-pool allocator's `ipallocator.NewCIDRRange` (`pkg/ipam/service/ipallocator/allocator.go`) reserves the first and last IP of any CIDR with more than 2 addresses. This is already resolved for /32 single-IP CIDRs ([#34618](https://github.com/cilium/cilium/pull/34618)), so secondary IP CIDRs are unaffected.

For /28 prefix delegation CIDRs, this wastes 2 of 16 IPs (12.5%). Since AWS delegates the entire /28 exclusively to the node with no shared network segment, the traditional base/broadcast reservation does not apply.

The fix is to extend `ipallocator.NewCIDRRange` with an `allowFirstLastIPs` option, following the pattern already established by `CiliumLoadBalancerIPPool.AllowFirstLastIPs` (see [#28637](https://github.com/cilium/cilium/issues/28637#issuecomment-2337426529)). The ENI multi-pool allocator sets this flag when constructing CIDR allocators from `Spec.IPAM.Pools.Allocated`. This change is confined to the allocator layer (`pool.go` -> `ipallocator`) and requires no CRD schema change.

### Migration Strategy

The migration is transparent and requires no user action. It is designed to be safe for both upgrade and downgrade within Cilium's supported N-1 version policy.

Cilium upgrade order: operator upgrades first, then agents roll (DaemonSet rolling update). On downgrade: agents roll back first, then operator.

Transition release (1.20) behavior:

The new operator dual-writes to both the old and new CiliumNode fields simultaneously:
* `Spec.IPAM.Pool`: individual IPs (written for backward compatibility with old agents and downgrade safety)
* `Spec.IPAM.Pools.Allocated`: CIDRs (written for new agents)

The new agent reads from `Spec.IPAM.Pools.Allocated` and uses the multi-pool allocator. It writes aggregate demand to `Spec.IPAM.Pools.Requested` and stops writing `Status.IPAM.Used`. Eliminating the per-pod `Status.IPAM.Used` write is the primary source of churn reduction; the operator-side dual-write is low-frequency (it fires on ENI capacity changes, not on every pod event) and is an acceptable temporary cost.

Safe upgrade sequence:
1. New operator starts, all nodes still have old agents. Operator begins dual-writing `Spec.IPAM.Pool` and `Pools.Allocated` for each node.
2. Agents roll one by one. Each new agent finds `Pools.Allocated` already populated, uses multi-pool, starts writing `Pools.Requested`, stops writing `Status.IPAM.Used`.
3. New operator detects `Pools.Requested` present on a node -> reads demand from there. For nodes still running old agents, demand is still read from `Status.IPAM.Used`. Both coexist.
4. Rolling upgrade completes; all demand flows through `Pools.Requested`.

Safe downgrade sequence:
1. Agents roll back one by one. Each old agent finds `Spec.IPAM.Pool` populated (new operator is still dual-writing) and starts normally. Old agent writes `Status.IPAM.Used`.
2. New operator detects `Pools.Requested` gone and `Status.IPAM.Used` present on a node -> switches demand source for that node.
3. Rolling rollback completes, all demand flows through `Status.IPAM.Used`.
4. Old operator rolls back. It finds `Spec.IPAM.Pool` populated (was dual-written) and `Status.IPAM.Used` populated (written by old agents during step 1–3). Fully operational with no data loss.

Phase-out (1.21):

Provided all the changes described above ship with the 1.20 release, the dual-write of `Spec.IPAM.Pool` can be dropped in the 1.21 release. Cilium only officially supports N-1 upgrades, so clusters should go through 1.20 first before getting to 1.21, meaning it is safe to remove the 1.19 compatibility logic with 1.21.

## Impacts / Key Questions

### Key Question 1: Scope of Dual-Write in the Transition Release

The Migration Strategy section proposes that the new operator dual-writes `Spec.IPAM.Pool` (old format) throughout the transition release to enable safe downgrade. The question is whether this is an acceptable cost or whether we should scope it more narrowly.

#### Option 1: Always Dual-Write During Transition Release

The operator always writes both `Spec.IPAM.Pool` and `Spec.IPAM.Pools.Allocated` for all ENI nodes during the 1.20 release cycle.

##### Pros
* Simple logic: no per-node format detection needed in the operator.
* Fully safe for both upgrade and downgrade at any point.

##### Cons
* `Spec.IPAM.Pool` remains populated on all nodes even when all agents have upgraded, until 1.21.
* Slightly more data written per CiliumNode object.

#### Option 2: Temporary Opt-Out Flag to Disable Dual-Write

Introduce a temporary Helm/ConfigMap flag (e.g., `ipam.eniLegacyPoolDualWrite: false`) that disables the `Spec.IPAM.Pool` dual-write. The flag defaults to `true` (dual-write enabled). Users who have fully upgraded their cluster to 1.20, do not intend to roll back to 1.19, and want to stop populating the legacy field can explicitly set it to `false`. The flag would be removed in 1.21 when dual-write is dropped unconditionally.

##### Pros
* Gives operators a clean escape hatch without making it the default behavior.
* Explicit opt-out makes the trade-off visible: the user acknowledges that downgrade to 1.19 is no longer safe.

##### Cons
* Adds a short-lived flag that must be documented and then removed, with a deprecation notice in 1.21.
* Risk of users enabling it prematurely (before all agents have fully rolled to 1.20), leaving some nodes without a populated `Spec.IPAM.Pool`.

### Key Question 2: Pool Naming Strategy

Should all ENI-allocated addresses share a single `"eni"` pool, or should there be more purpose-named pools?

#### Option 1: Single `"eni"` Pool

All CIDRs regardless of subnet are written under a single pool name.

##### Pros
* Simpler operator logic; no pool name coordination needed.
* Straightforward for the initial migration.

##### Cons
* Cannot express per-subnet constraints; future per-subnet pod placement would require a redesign.

#### Option 2: Per-Subnet Pools

Each subnet maps to a distinct pool name (e.g., `"eni-subnet-<subnet-id>"`).

##### Pros
* Enables future per-subnet pod placement policy.
* Pools naturally map to AWS subnet boundaries.

##### Cons
* More complex operator logic for pool name management.
* Pool names must be stable across reconcile cycles; subnet IDs are suitable but verbose.

## Future Milestones

### IPv6 Secondary Addresses in ENI IPAM

With the multi-pool allocator in place, IPv6 secondary addresses can be written as CIDRs in `Spec.IPAM.Pools.Allocated` without any flattening, using the same mechanism as IPv4. This requires separate work to extend ENI device and operator logic for dual-stack.

### IPv6 Prefix Delegation

AWS assigns /80 prefixes for IPv6 ENI delegation. With the multi-pool allocator, a /80 CIDR can be written directly to `Allocated` and the agent allocates from it locally. This is the primary long-term motivation for this migration and is not implementable with the current CRD allocator.

### Per-Subnet Pod Placement

If per-subnet pools are adopted (Key Question 2, Option 2), named pools could be used to control which subnet a pod's IP is drawn from, enabling topology-aware placement without changes to the scheduling layer.
