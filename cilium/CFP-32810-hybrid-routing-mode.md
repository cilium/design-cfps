# CFP-32810: Add hybrid routing mode in Cilium

**SIG: SIG-NAME** SIG-Datapath

**Begin Design Discussion:** 2025-06-16

**Cilium Release:** 1.19

**Authors:** Anubhab Majumdar <anmajumdar@microsoft.com>, Vamsi Kalapala <vakr@microsoft.com>, Krunal Jain <krunaljain@microsoft.com>

## Summary

Currently, Cilium supports two modes routing - [encapsulation](https://docs.cilium.io/en/stable/network/concepts/routing/#encapsulation) and [native-routing](https://docs.cilium.io/en/stable/network/concepts/routing/#native-routing). When workloads are on cross subnets with no direct connection, Cilium needs to run in encapsulation mode. This is not efficient, as every packet needs to be encapsulated, even when they are on the same subnet, thus paying the encapsulation overhead and reducing network throughput. This CFP proposes adding a third option to allow Cilium dataplane to make the decision at runtime on whether to route the packet natively or encapsulate.

## Motivation

Consider any of the following scenarios:

1. Two Kubernetes cluster connected through cluster mesh. The nodes are backed by different subnets in two different VNETs, which are connected through VNET peering. The pods are assigned IP addresses from overlay network.
2. In a large Kubernetes cluster, pod addresses are assigned from two different subnets with no direct connection.

In either case, Cilium needs to run in encapsulation routing mode. However, when routing packets in above clusters, in many situations, we can get by native routing, and not pay the encapsulation overhead that comes with encapsulation. The reason for this inflexibility comes from the fact that Cilium datapath is not aware of the underlying network topology, and hence cannot make a decision between the two routing modes during runtime.

With higher adoption in cluster mesh and large scale clusters being common to support AI workloads, it would be beneficial to have a more performant dataplane that can optimize the throughput, while providing support for both encapsulation and native-routing.

## Goals

* Retain the behavior of current routing modes as-is
* Allow users to set a new routing mode called `hybrid`
* Allow users to pass in network topologies expressed as CIDR ranges through configmap
* Support for both IPV4 and IPV6
* Allow dynamic configuration of CIDRs (add/remove) without agent restart (update w/o agent restart)

## Proposal

### Overview

Add a new option for routing-mode config as follows `routing-mode: hybrid`. Also, add a new field in config to pass information about the CIDRs and connectivity. The CIDR information will persist in pinned eBPF maps so that routing decision can be made in runtime by eBPF programs. The concept is similar to [IP Masquerading](https://docs.cilium.io/en/stable/network/concepts/masquerading/#ebpf-based) and the eBPF table it uses.

### Examples

```
subnet-topology: 10.0.0.1/24
```

If source and destination IP belongs to above CIDR, route natively. Else, preserve current behavior (encapsulate if being routed to known CIDRs/pass to Linux stack).

```
subnet-topology: 10.0.0.1/24,10.10.0.1/24
```

The above two CIDR ranges have direct connectivity. If source and destination IP belongs to either CIDR, route natively. Else, preserve current behavior (encapsulate if being routed to know CIDRs/pass to Linux stack).
```
subnet-topology: 10.0.0.1/24,10.10.0.1/24;10.20.0.1/24
```

* `10.0.0.1/24` and `10.10.0.1/24` are connected and traffic can be natively routed
* If source and destination IP belongs to `10.20.0.1/24`, route natively
* Else, preserve current behavior (encapsulate if being routed to know CIDRs/pass to Linux stack).

### Control plane implementation

Similar to dynamic flowlogs or Hubble dynamic exporter, allow users to configure the CIDRs on the fly through a configmap. The configmap will be mounted on each agent. On create/update, update `LocalNodeConfig` and trigger dataplane reconcile.

### Datapath implementation

This section provides an overview of the datapath implementation. Consider the current implementation - as long as nodes have IP connectivity, Cilium can route traffic, both intra-cluster and inter-cluster (in case of cluster mesh). 

#### Case 1

If `hybrid` mode is enabled without configuring any `subnet-topology`, routing mode will behave exactly same as encapsulation mode. Pod IPs can be from overlay or from a subnet, it will be encapsulated either way. So datapath implementation would be exactly as it is today for encapsulation routing mode.

#### Case 2

If `subnet-topology` is configured, the CIDR ranges can be configured as part of node configs, similar to [IPv4PodSubnets](https://github.com/cilium/cilium/blob/f78aca7b5dff52e6d07723f6577dd0d3f913fea6/pkg/datapath/types/node.go#L177).

Similar to how masquerade uses `NativeRoutingCIDRIPv4` to check if [SNAT is needed](https://github.com/cilium/cilium/blob/f78aca7b5dff52e6d07723f6577dd0d3f913fea6/bpf/lib/nat.h#L713), this implementation would add a similar function that can check source and destination address against multiple CIDRs. However, in this case, since we will be storing multiple CIDRs, we will use an eBPF map of type `BPF_MAP_TYPE_LPM_TRIE` to efficiently search the map. 

Next, while making routing decision today, in both `lxc` and `host` eBPF programs, a check is made to see if tunneling should be done. This implementation would add an extra check to tunneling decision. The check would use above function to figure out if native routing can be done. 
- If yes, we skip tunneling and the packet is instead sent to Linux stack for handling, where the underlying networking would route the packet to destination. 
- If no, tunneling happens exactly as it works today 

Above behavior simply optimizes current encapsulation behavior, without introducing major change to how encapsulation routing works/behaves. 

## Impacts / Key Questions

### Impact: Network throughput

Given that today Cilium is encapsulating packets that could be natively routed, removing those scenarios would yield higher throughput. This is especially true in clusters without jumbo frames.

### Key Question: Datapath latency

In a highly fragmented network topography, lookup of IP to make routing decision may not be negligible.

## Future Milestones
