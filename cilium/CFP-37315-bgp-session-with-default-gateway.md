# CFP-37315

**SIG: SIG-BGP**

**Begin Design Discussion:** 2024-02-24

**Cilium Release:** X.XX

**Authors:** Naveen Achyuta <naveen.achyuta22@gmail.com>

## Summary

Cilium’s current BGP implementation requires specifying the peer IP address in the BGP cluster configuration. In large-scale environments with thousands of Kubernetes nodes, managing distinct BGP configuration files (one per node) becomes impractical.

This proposal offers a simpler alternative: rely on the IPv4 default gateway of ToR switches to automatically create BGP sessions when no peer IP is provided in the BGP cluster configuration.

## Motivation

Large networks with thousands of Kubernetes nodes cannot feasibly manage the peer IP addresses of all ToR switches within numerous BGP cluster configurations. Allowing Cilium to automatically discover and use the IPv4 default gateway as the peer simplifies BGP session creation, reducing operational overhead and configuration complexity.

## Goals

### Auto-Discovery of Peer IPs
Allow Cilium to establish BGP sessions by discovering the IPv4 default gateway on the node, when no peer IP is explicitly provided in the BGP cluster configuration.
### Single Config File for All Nodes
Enable large networks to use a generic BGP configuration file without needing to specify unique peer IP addresses on a per-node basis.

## Non-Goals

* _List aspects which are specifically out of context for this CFP._

## Proposal

### Overview

At a high level, this proposal modifies how Cilium handles BGP peer IP configuration. If a user omits the peer IP for a BGP process, the Cilium agent attempts to retrieve the node’s IPv4 default gateway from statedb (which stores local routing information). Cilium can then use this default gateway as the peer IP to establish the BGP session. If the default gateway is not found, the agent skips the neighbor and logs an error for it.

### BGP Cluster Config

If the user skips the peer ip for a bgp process, cilium agent should not return back an error when its creating new bgp sessions. It should first rely on statedb to get ipv4 default gateway. If not found, it should skip the neighbor and log an error.

### Routes from Statedb

cilium agent should rely on statedb to get the ipv4 default gateway to create the bgp session. we can inject statedb into neighbor reconcillation struct and when we loop through neighbors, we can populate the peer ip by looking for default gateway of 0.0.0.0/0 route in routes table of statedb.


## Impacts / Key Questions

### Impact 1: Multi-Homing Scenario

In multi-homed environments (i.e., multiple default gateways leading to different ASNs), this proposal does not address which gateway belongs to which peer ASN. For example:

bgpInstances:
  - name: "65001"
    localASN: 65001
    peers:
    - name: "65000"
      peerASN: 65000
      peerConfigRef:
        name: "cilium-peer"
    - name: "65011"
      peerASN: 65011
      peerConfigRef:
        name: "cilium-peer"

If multiple default gateways exist, we cannot reliably match them to different peer ASNs. One workaround is to specify local or remote interface names for each peer to ensure the correct gateway is used.


### Key Question

Is there a way to identify which default gateway should be used for a specific BGP peer ASN in a multi-homed environment without requiring manual configuration?


### Impact 2: Multiple BGP Instances

If the user does not provide peer IP when multiple bgp instances are configured, the proposal  cannot assign the correct default gateway to a peer of a bgp process

### Key Question

Should the system return an error if the user omits peer IPs when multiple BGP processes are defined?

### Impact 3: Dual-Stack (IPv4 and IPv6)

If the user wants to create a v4 and v6 bgp session (dual stack) without providing the peer ip, the proposal will not work because the system assumes that the user will leverage IPv4 bgp session to send both v4/v6 traffic.

### Key Question

Should we stick IPv4 session for this proposal or should we create both v4 and v6 and have a config knob incase the user wants to disable any of the address families?


### Option 1:

When the peer IP is absent and there is exactly one peer, use the discovered IPv4 default gateway to create a single BGP session.

#### Pros

Simple implementation
Significantly reduces configuration overhead

#### Cons

Does not cover multi-homed networks or multiple peers.
Does not handle IPv6 or dual-stack scenarios.

### Option 2:

If both IPv4 and IPv6 default gateways are present and a single peer is defined without a peer IP, create two BGP sessions (IPv4 and IPv6). Provide configuration knobs to disable any undesired address families.

#### Pros

Adds flexibility for dual-stack environments.
Reduces the need for separate peer IP configurations for IPv4 and IPv6 by skipping the peer ip config

#### Cons

Deviates from the current "one bgp session per configured peer" design.
Still does not address multi-homed or multiple-peer scenarios.

## Future Milestones

_List things that this CFP will enable but that are out of scope for now. This can help understand the greater impact of a proposal without requiring to extend the scope of a CFP unnecessarily._

### Deferred Milestone 1

_Description of deferred milestone_

### Deferred Milestone 2
