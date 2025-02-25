# CFP-37315

**SIG: SIG-BGP**

**Begin Design Discussion:** 2024-02-24

**Cilium Release:** X.XX

**Authors:** Naveen Achyuta <naveen.achyuta22@gmail.com>

## Summary

Cilium’s current BGP implementation requires specifying the peer IP address in the BGP cluster configuration. In large-scale environments with thousands of Kubernetes nodes, managing distinct BGP configuration files (one per node) becomes impractical.

This proposal offers a simpler alternative: rely on the default gateway of ToR switches to automatically create BGP sessions when no peer IP is provided in the BGP cluster configuration.

## Motivation

Large networks with thousands of Kubernetes nodes cannot feasibly manage the peer IP addresses of all ToR switches within numerous BGP cluster configurations. Allowing Cilium to automatically discover and use the  default gateway as the peer simplifies BGP session creation, reducing operational overhead and configuration complexity.

## Goals

### Auto-Discovery of Peer IPs
Allow Cilium to establish BGP sessions by discovering the default gateway on the node, when no peer IP is explicitly provided in the BGP cluster configuration.
### Single Config File for All Nodes
Enable large networks to use a generic BGP configuration file without needing to specify unique peer IP addresses on a per-node basis.

## Non-Goals

* _List aspects which are specifically out of context for this CFP._

## Proposal

### Overview

At a high level, this proposal modifies how Cilium handles BGP peer IP configuration. If a user omits the peer IP for a BGP process, the Cilium agent attempts to retrieve the node’s default gateway from statedb (which stores local routing information). Cilium can then use this default gateway as the peer IP to establish the BGP session. If the default gateway is not found, the agent skips the neighbor and logs an error for it.

### BGP Cluster Config

If the user skips the peer ip for a bgp process and enables auto discovery, cilium agent should not return back an error when its creating new bgp sessions. It should first rely on statedb to get default gateway. If not found, it should skip the neighbor and log an error.
Every peer will have to specify the address family so that we fetch the default gateway for that address family

config proposal:

bgpInstances:
  - name: "65001"
    localASN: 65001
    autoDiscovery: true
    peers:
    - name: "peer1 - 65000"
      peerASN: 65000
      afi: ipv4
      peerConfigRef:
        name: "cilium-peer"
    - name: "peer2 - 65000"
      peerASN: 65000
      afi: ipv6
      peerConfigRef:
        name: "cilium-peer"

In multi-homed environments (i.e., multiple default gateways leading to different ASNs), this proposal does not address which gateway belongs to which peer ASN.

example multi-homed config:
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

If multiple default gateways exist, we cannot reliably match them to different peer ASNs. One workaround is to use same local ASN on both the ToRs for bgp sessions with the k8s node, and bgp cluster config will have one peer configured.

proposed multi-homed config:
bgpInstances:
  - name: "65001"
    localASN: 65001
    peers:
    - name: "65000"
      peerASN: 65000
      afi: ipv4
      peerConfigRef:
        name: "cilium-peer"

proposed config in the device (arista):
bgp listen range 10.2.0.0/24 peer-group SERVERS remote-as 65001
router bgp 65100
   router-id 10.1.1.1
   timers bgp 1 3
   neighbor SERVERS local-as 65000 no-prepend replace-as


### Routes from Statedb

cilium agent should rely on statedb to get the default gateway to create the bgp session. we can inject statedb into neighbor reconcillation struct and when we loop through neighbors, we can populate the peer ip by looking for default gateway of 0.0.0.0/0 or ::0/0 route in routes table of statedb.


## Impacts / Key Questions

### Impact 1: Multi-Homing Scenario

In multi-homed environments, we will be breaking the design of one peer per bgp session because we will create two bgp sessions with one peer defined in the config. One of the workarounds could be to define two peers with same config and a unique name.

### Key Question

Is there a different way to accomplish this?


### Impact 2: Multiple BGP Instances

The proposal does not take into account multiple bgp instances so auto-discovery will work in that case.

### Key Question

What are use-cases of having multiple bgp instances? (bgp instances is a list in config)

### Impact 3: config knob for auto-discovery

config knob to enable auto-discovery will be crucial because that enables the users to understand the dependencies before making the changes. We can have documentation for auto-discover and provide the pre-requisites for enabling it.
Without the knob, the user may use auto-discovery without understanding the requirements for it.

### Key Question

are we okay to add a knob for auto-discovery or should we enable auto-discovery by default if peerAddress is not specified?


### Option 1:

If the user wants to create bgp sessions without specifying peerAddress, the user will remove peerAddress field, enable auto-discovery in the config and add ipv4 and/or ipv6 peers explicitly with "afi" field

#### Pros

Adds flexibility to create ipv4 and/or ipv6 sessions.
auto-discovery field will help users to understand the requirements provided in the documentation before using it. (User can obviously enable it without knowing the requirements, but, we can error out with clear error messages)

#### Cons

For multi-homed environments, it deviates from the current "one bgp session per configured peer" design if we decide to use only one bgp peer for both sessions.

### Option 2:

do nothing

#### Pros



#### Cons

User will have to manage bgp cluster files for all the k8s nodes and updates to these files can be cumbersome

## Future Milestones

_List things that this CFP will enable but that are out of scope for now. This can help understand the greater impact of a proposal without requiring to extend the scope of a CFP unnecessarily._

### Deferred Milestone 1

_Description of deferred milestone_

### Deferred Milestone 2
