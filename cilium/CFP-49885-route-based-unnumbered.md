# CFP-49885: Route based interface discovery for BGP unnumbered

**SIG:** SIG-BGP

**Begin Design Discussion:** 2026-11-06

**Cilium Release:** TBD

**Author:** Carl Baldwin (@ecbaldwin)

**Status:** Draft

**Issue:** https://github.com/cilium/cilium/issues/49885

## Summary

Allow an unnumbered BGP peer to select its local interface from a route with an
exact destination prefix. Cilium then uses the existing unnumbered discovery
mechanism to find the peer's IPv6 link-local address on that interface. The
route's gateway address is not used as the BGP peer address.

This proposal adopts the API shape suggested during review of [PR
#49786](https://github.com/cilium/cilium/pull/49786#issuecomment-6692514258).
Default routes -- which that PR had started with -- are supported as
`0.0.0.0/0` and `::/0`.

## Motivation

Our nodes connect to a DPU that is also their default next hop; a switch would
work similarly. Interface names vary across nodes, so explicitly naming the
interface in Cilium requires additional configuration for different nodes or
hardware groups. The existing route provides a common way to identify the
appropriate interface.

For historical reasons, including PXE limitations, our nodes bootstrap with
IPv4 and acquire an IPv4 default route. We nevertheless prefer BGP unnumbered
wherever possible because it reduces address planning and configuration on both
the host and DPU. The DPU or switch can configure peering on its host-facing
interface without assigning dedicated peering addresses or tracking each host's
address.

Existing `DefaultGateway` discovery uses the route's gateway address as the BGP
peer address. Adding link-local support to that mode would help when an IPv6
default route already points to the desired link-local peer. It would not let
our IPv4 default route select an interface for IPv6 unnumbered peering.

Route based interface discovery lets us use the existing IPv4 route without
adding an IPv6 default route solely for discovery or maintaining per-node
interface configuration. An exact prefix also allows deployments to identify an
interface using a more specific route when the default route is unsuitable.

## Goals

- Select an unnumbered peering interface following a default IPv4 or IPv6 route prefix.
- Keep the route's address family independent of the IPv6 link-local BGP transport.
- Preserve explicit interface selection and existing unnumbered peer discovery.
- Reconcile interface selection when relevant routes or devices change.

## Non Goals

- Change the behavior or API of `DefaultGateway` peer-address discovery.
- Perform a longest-prefix-match lookup for a destination address.
- Create multiple BGP sessions from one configured peer.
- Add routing policy evaluation, VRF selection, or configurable routing tables
  in this initial proposal.
- Change unnumbered neighbor discovery, Router Advertisement behavior, or
  negotiated BGP address families.

## Proposal

### API

Add `deviceDiscovery` under the existing `autoDiscovery.unnumbered`
configuration, with mode `Route` selecting an interface by route prefix:

```yaml
autoDiscovery:
  mode: Unnumbered
  unnumbered:
    deviceDiscovery:
      mode: Route
      route:
        prefix: 0.0.0.0/0
```

An IPv6 or more specific prefix uses the same API:

```yaml
autoDiscovery:
  mode: Unnumbered
  unnumbered:
    deviceDiscovery:
      mode: Route
      route:
        prefix: fd00::/64
```

Existing explicit interface configuration remains valid:

```yaml
autoDiscovery:
  mode: Unnumbered
  unnumbered:
    interface: eth0
```

Require exactly one of `unnumbered.interface` and `unnumbered.deviceDiscovery`.
Initially, `Route` is the only supported device discovery mode and requires
`deviceDiscovery.route` with a valid CIDR prefix. Proposed validation requires
the network prefix to be canonical, rejecting host bits rather than silently
masking them.

The explicit device discovery mode clarifies what is being discovered and leaves
room for future methods, such as regex-based device matching.

The existing constraints on `peerAddress` and `autoDiscovery` continue to apply.
`unnumbered` is valid only with mode `Unnumbered`; `defaultGateway` remains the
configuration for mode `DefaultGateway`.

### Route selection

The following selection rules are proposed for discussion:

1. Consider unicast routes in the Linux main routing table with a destination
   prefix exactly equal to the configured prefix. A covering or more specific
   route is not a match.
2. Require a usable, non-loopback egress device, following the existing
   unnumbered interface eligibility checks.
3. Prefer eligible routes with the lowest metric.
4. If the best candidates identify different interfaces, fail discovery with a
   diagnostic rather than choosing an arbitrary interface. Multiple candidates
   identifying the same interface do not make interface selection ambiguous.

The gateway address is irrelevant to interface selection. Routes with an IPv4
gateway, an IPv6 link-local gateway, or no gateway can identify the interface.

The main-table restriction makes selection predictable without interpreting
policy-routing rules. It also excludes unrelated defaults in other tables.
Deployments whose relevant route exists only in another table would continue to
use explicit interface selection.

### Peer discovery and lifecycle

Once an interface is selected, use the existing unnumbered discovery and Router
Advertisement behavior on that interface. The discovered peer address is IPv6
link-local and scoped to the selected interface, regardless of the route prefix's
address family. Existing ambiguity handling for multiple neighbor candidates
continues to apply.

Route creation, deletion, or changes affecting selection must trigger
reconciliation for arbitrary configured prefixes, extending the current
default-route-only change tracking. Device and neighbor changes remain relevant
as in existing unnumbered discovery.

If no eligible route or unique interface exists, the peer remains unresolved and
Cilium reports the discovery failure. If a previously selected interface is no
longer selected, reconciliation must remove the stale peer and stop maintaining
its Router Advertisement state unless another configured peer still needs that
interface. Once discovery succeeds, Cilium configures the resolved peer.

The selected route must be available independently of the BGP session it
bootstraps. A DHCP, static, or connected route can serve this purpose. Depending
on a route learned only through that same session would create a circular
dependency and prevent reliable recovery.

### Compatibility and implementation

The API introduced by the merged unnumbered work in
[PR #47394](https://github.com/cilium/cilium/pull/47394) continues to support
`unnumbered.interface`. Its schema must make `interface` optional individually
while enforcing exactly one of `interface` and `deviceDiscovery`.

The sibling `defaultGateway` configuration for mode `Unnumbered` in PR #49786
would be replaced before merge. Existing `DefaultGateway` users would retain
their configuration and behavior.

Implementation includes CRD validation, configuration conversion, route
selection, change tracking, documentation, and tests. Tests should cover exact
matching, both route families, arbitrary prefixes, metric preference, interface
ambiguity, route and device changes, invalid configuration, and preservation of
explicit-interface behavior. Lifecycle tests should verify cleanup and recovery
when route selection disappears or moves to another interface.

## Impacts and Key Questions

### Routing table scope

Is main-table-only selection sufficient for the initial feature? A future table
selector could support policy-routing deployments, but would need a clear
relationship to VRFs and the BGP session's routing context.

### Multiple routes and multipath

Should equal-best routes selecting different interfaces fail discovery as
proposed? This preserves one configured peer per session and avoids relying on
iteration order. Native multipath routes also require care: the implementation
must establish what nexthops the StateDB route representation exposes and avoid
claiming complete ECMP selection semantics without that support.

### Prefix validation

Should admission reject noncanonical CIDRs or should the agent normalize them?
Rejection gives users an explicit exact-match key; normalization may be more
permissive but needs consistent validation and comparison behavior.

### Relationship to the earlier discovery CFP

[CFP-37315](https://github.com/cilium/design-cfps/blob/main/cilium/CFP-37315-bgp-auto-discovery.md)
explicitly excluded discovery from arbitrary prefixes. This proposal extends
route selection specifically for unnumbered interfaces. The route identifies
the local interface; the established unnumbered mechanism determines the peer.
It does not generalize `DefaultGateway` into arbitrary-route gateway-address
discovery.

### Alternatives

- **Support link-local addresses in DefaultGateway:** useful when the gateway
  itself is the intended BGP peer address; does not address IPv4 route selection
  for IPv6 unnumbered peering.
- **Use consistent interface names or alternative names:** can simplify explicit
  configuration, but requires provisioning and maintaining that naming convention.
  Route selection uses information already present on the node.
- **Limit selection to default routes:** meets our immediate deployment need,
  but excludes deployments whose peering interface is identified by another
  route. Exact-prefix selection includes defaults with the same API.

## Future Milestones

Configurable routing tables or VRF-aware selection can be considered separately
if concrete use cases require them.
