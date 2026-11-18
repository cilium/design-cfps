# CFP-50093: Support Cilium Egress Gateway with IPv6 Underlay

**SIG:** SIG-DATAPATH

**Begin Design Discussion:** 2026-11-18

**Cilium Release:** 1.21 (flexible)

**Authors:** Liam Connery <lconnery@google.com>

**Status:** Draft

**Related Issue:** [cilium/cilium#50093](https://github.com/cilium/cilium/issues/50093)

## Summary

This CFP proposes extending Cilium Egress Gateway to support IPv6 gateway node
IPs so the feature can be enabled in dual-stack clusters using an IPv6 underlay
and in single-stack IPv6 clusters, across both tunnel and native routing modes.

No CRD, API, or configuration flag changes are introduced. The changes use
bytes already reserved in the Egress Gateway BPF policy maps for IPv6 gateway
addresses, update the datapath and agent to follow the configured underlay
protocol, and relax remaining IPv4-only startup checks for single-stack IPv6
clusters.

## Motivation

Operators running single-stack IPv6 clusters or dual-stack clusters on an IPv6
underlay (for example, to simplify host networking or conserve IPv4 node
addresses) still need to steer outbound traffic through dedicated perimeter
nodes, but cannot enable Egress Gateway today because it only supports IPv4
underlays and IPv4 gateway node IPs.

Most of the prerequisite work has already landed, including IPv6 Egress Gateway
policies, IPv6 BPF masquerading, and IPv6 underlay support for single-stack
([cilium/cilium#38296](https://github.com/cilium/cilium/pull/38296)) and
dual-stack ([cilium/cilium#40324](https://github.com/cilium/cilium/pull/40324))
clusters. However, the
[Egress Gateway manager](https://github.com/cilium/cilium/blob/d65ec42baf8114bbdda7655ebf7f9d459f3350cc/pkg/egressgateway/manager.go#L203-L209)
still blocks non-IPv4 underlays and unconditionally requires IPv4 masquerading,
because the policy maps, datapath tunnel handling, and agent gateway selection
still assume IPv4 gateway node addresses.

Fortunately, the existing Egress Gateway BPF policy map entries were designed
with future IPv6 gateway support in mind and already reserve the extra bytes
needed for an IPv6 gateway address and flags. Finishing IPv6 gateway node IP
support closes a remaining gap in Cilium's IPv6 support with a small, contained
change.

## Goals

* Enable Egress Gateway in dual-stack clusters using an IPv6 underlay across
  both tunnel and native routing modes.
* Enable Egress Gateway in single-stack IPv6 clusters across both tunnel and
  native routing modes.
* Preserve non-disruptive upgrades for existing IPv4-underlay deployments.

## Non-Goals

* Supporting XDP acceleration with an IPv6 underlay, which is currently
  unsupported across Cilium.

## Proposal

### Datapath

Replace the 32-bit gateway IP and reserved fields in
[bpf/lib/egress_gateway.h](https://github.com/cilium/cilium/blob/d65ec42baf8114bbdda7655ebf7f9d459f3350cc/bpf/lib/egress_gateway.h#L23-L43)
with an IPv4/IPv6 address union and a flags field (following the tunnel
endpoint pattern in ipcache), keeping the map names and entry sizes unchanged:

```c
#define EGRESS_GW_FLAG_IPV6_GATEWAY (1 << 0)

union egress_gw_addr {
	union v4addr ip4;
	union v6addr ip6;
};

struct egress_gw_policy_entry_v2 {
	__be32 egress_ip;
	union egress_gw_addr gateway_ip;
	__u32 egress_ifindex;
	__u32 flags; /* previously reserved2 */
};

struct egress_gw_policy_entry6 {
	union v6addr egress_ip;
	union egress_gw_addr gateway_ip;
	__u32 egress_ifindex;
	__u32 flags; /* previously reserved2 */
};
```

Update the matching Go map types in
[pkg/maps/egressmap](https://github.com/cilium/cilium/blob/d65ec42baf8114bbdda7655ebf7f9d459f3350cc/pkg/maps/egressmap/policy.go#L42-L66),
and update the Egress Gateway forward and reply helpers in
[bpf/lib/egress_gateway.h](https://github.com/cilium/cilium/blob/d65ec42baf8114bbdda7655ebf7f9d459f3350cc/bpf/lib/egress_gateway.h)
and [bpf/lib/nodeport.h](https://github.com/cilium/cilium/blob/d65ec42baf8114bbdda7655ebf7f9d459f3350cc/bpf/lib/nodeport.h)
to handle IPv6 gateway IPs and tunnel endpoints.

### Control Plane

* Select the gateway node's IPv6 address in
  [gateway config generation](https://github.com/cilium/cilium/blob/d65ec42baf8114bbdda7655ebf7f9d459f3350cc/pkg/egressgateway/policy.go#L137)
  when `underlay-protocol` resolves to IPv6.
* Remove the IPv4-underlay guard in the
  [Egress Gateway manager](https://github.com/cilium/cilium/blob/d65ec42baf8114bbdda7655ebf7f9d459f3350cc/pkg/egressgateway/manager.go#L203-L213)
  and gate IPv4 masquerade, reverse-path filter, and egress IP checks on IPv4
  being enabled so single-stack IPv6 clusters work as well.
* Extend the existing IPv6-underlay checks for
  [MTU](https://github.com/cilium/cilium/blob/d65ec42baf8114bbdda7655ebf7f9d459f3350cc/pkg/mtu/cell.go#L150-L151),
  [XDP acceleration](https://github.com/cilium/cilium/blob/d65ec42baf8114bbdda7655ebf7f9d459f3350cc/pkg/kpr/initializer/kube_proxy_replacement.go#L123-L126),
  and [WireGuard peer selection](https://github.com/cilium/cilium/blob/d65ec42baf8114bbdda7655ebf7f9d459f3350cc/pkg/wireguard/agent/agent.go#L607-L608)
  to also apply when Egress Gateway enables tunneling in native routing mode.

### Rollout Plan and Timeline

The work can be split into four small PRs:

1. **Core BPF and agent changes:** Map layout, datapath helpers, and gateway IP
   selection (keeping the manager guard in place).
2. **Cilium CLI connectivity tests:** Update the CLI Egress Gateway tests to
   derive expected gateway node IPs from the underlay protocol instead of
   assuming IPv4 (can be reviewed in parallel with PR 1).
3. **Dual-stack enablement:** Remove the IPv4-underlay guard, include the
   native-routing tunnel checks, wire up CI, and update docs.
4. **Single-stack IPv6 support:** Relax the remaining IPv4-only manager checks,
   wire up CI, and update docs (or fold into PR 3 if preferred).

Landing in Cilium 1.21 would be great if review bandwidth allows, but the
timeline is flexible and can move at whatever pace works best for the
maintainers (including targeting 1.22).

### Testing

Go unit tests and BPF datapath tests will cover the map, datapath, and agent
changes. For CI end-to-end tests, we will make sure the following configurations
are covered:

* Dual-stack, Egress Gateway, tunnel routing, IPv6 underlay
* Dual-stack, Egress Gateway, native routing, IPv6 underlay
* Single-stack IPv6, Egress Gateway, tunnel routing, IPv6 underlay
* Single-stack IPv6, Egress Gateway, native routing

## Impacts / Key Questions

### Upgrade and Downgrade Compatibility

Both policy maps keep their existing names, sizes, and field offsets, and the
previously reserved bytes are already zeroed. Existing IPv4-underlay entries
remain valid in place across upgrades and downgrades with no map migration.

Clusters that enable Egress Gateway with an IPv6 underlay or in single-stack
IPv6 mode must disable it before downgrading to an older release, since prior
versions reject that configuration on startup.

### Sentinel Values

Missing gateways (0.0.0.0) and excluded CIDRs (0.0.0.1) will remain stored in
the IPv4 gateway field with the IPv6 gateway flag unset, keeping existing
entries valid across upgrades. These could be moved to dedicated flag bits
instead if maintainers prefer.
