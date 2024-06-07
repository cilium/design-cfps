# CFP-14339: Cilium MTU model

**SIG:** SIG-Datapath

**State:** Dormant[^a]

**Begin Design Discussion:** Feb 26, 2021

**Authors:** Joe Stringer <joe@cilium.io>

**Issue:** https://github.com/cilium/cilium/issues/14339

## Summary

Cilium 1.10 or earlier does not provide deterministic Ethernet MTU (Maximum
Transmission Unit) size configuration in environments where multiple networks
are exposed to the nodes where Cilium runs. This feature aims to configure MTU
consistently and optimally based on the available networks.

### Motivation

Cilium issue #14339[^o], Cilium is using wrong interface for MTU detection.

It is common in cloud environments to have multiple networks exposed to Cilium
where the MTU of the interfaces attached to those networks may vary quite
widely, such as allowing 1500 byte ethernet payloads via one network while
supporting as much as 9000 byte payloads via another network. Misconfiguration
of the MTU across different nodes can cause connectivity disruption,
particularly for UDP packets.

In some user environments, the Cilium 1.9 MTU calculation behaviour is
inconsistent as it is determined by either:

* `--mtu` flag (administrator override)
* `InitDefaultPrefix()` configures the node IP address
   * `->firstGlobalV4Addr()`
      * List all IPs from interfaces, except for interfaces that are down and
        docker interfaces
      * Pick the first one in the sorted list


We also currently expose only MTU and not MRU. Rather than considering transmit
and receive separately to determine the maximum sizes that can be viably sent
vs. received depending on the attached networks, we select a single MTU and
derive the MRU based on the configured/detected MTU and other configuration
such as ipsec, tunnel. This means that if the MTU we detected is incorrect,
then we may configure the MRU in a way that may cause traffic drops on receive
in cases where the packets could otherwise have been successfully received.

### Goals

1. Consistently calculate MTU and MRU (Maximum Receive Unit) sizes on each node.
2. Guarantee that packets of any reasonable size may be successfully received,
   regardless of the attached networks.
3. Provide visibility into the MTU and MRU size calculation and status on each
   node.
4. In cloud environments with East-West MTU 9K + North-South MTU 1.5K,
   automatically configure in a way that allows pods to utilize 9K MTU.[^b][^c]
5. Nice-to-haves:
   1. Minimize the likelihood of relying on IP fragmentation for connectivity
   2. Provide best-effort optimization of MTU/MRU configuration with ability
      for administrators to override with simple configuration.

### Non-Goals

* Assume that Kubernetes view of the network is the correct view, ignoring
  other networks

## Proposal

### Overview

Regardless of whatever configuration you have, by default Cilium should be able
to successfully establish connectivity from local endpoints and the local stack
to arbitrary endpoints on any connected network. This might not provide optimal
performance depending on the networks available to the node, but in that case
we should provide visibility and configuration options so that administrators
can understand how Cilium is determining MTU/MRU, and configure Cilium in a way
that satisfies their use case.

Use the available network devices on the node to calculate a maximum
transmission unit (MTU) size and maximum receive unit (MRU) size separately in
such a way that the lowest attached unit size will be the default MTU
configured in routes[^d] for Cilium pods and the highest attached unit size
will be the default MRU configured on devices for Cilium pods.

### Automatic MTU/MRU calculation

**TODO: This has not been revised since SIG-Datapath review on 2021-03-03**

* Cilium should detect all "relevant" networks that the node is connected to.
  * By default, "relevant" should correspond to every device available on the node
     * Should ignore some interfaces
        * `cilium_vxlan`
        * `cilium_host`
        * Virtual Ethernet (veth)
        * IPVLAN
        * Docker
        * IPVS (Cilium issue report[^n])
  * To allow more user control, we should have a deterministic way to control
    which networks are being considered.
    * Specific approaches to this configuration are discussed in the "Manual
      MTU/MRU override" section below.
* For each network, determine the MTU on that network.
  * If Cilium is configured with tunneling and/or encryption enabled, also add
    new network(s) to the set of "relevant" networks to represent these
    tunnel(s) and determine their MTU.
    * Simplest option for MTU would be to take the lowest available MTU of
      "relevant" networks and subtract vxlan, ipsec header lengths.
    * More intelligent option would figure out which path the tunnel would take
      and calculate MTU based on that network.
* Select the lowest available MTU from "relevant" networks and propagate that
  through `GetRouteMTU()`.
* Select the highest available MTU (MRU really) from "relevant" networks and
  propagate that through `GetDeviceMTU()`.

### Manual MTU/MRU override

Each user you ask will provide a different response for how they want to
configure the MTU. Ideally we can optimize the automatic detection above such
that most users never have to worry about this, but when they do, this section
describes some options we have.

The new options below rely on arbitrary options provided to the Cilium agent
and interpret MTU semantics based on them. Assuming that there are multiple
options available, there would need to be a defined precedence around how the
MTU is actually calculated in the presence of multiple flags. If all must be
included, we could always add a `--mtu-mode=foo` option which the user would
also configure to disambiguate the options.

#### Existing: Configure `--mtu`

This option is already available in Cilium today, nothing new proposed in this section.

The `--mtu` flag overrides the MTU to be used across the cluster. This actually
enforces both MTU and MRU configurations to be consistent across the cluster,
without taking into account tunnel device overhead[^e]. Administrators can
configure this option if they have only a single consistent MTU across all of
their nodes and use native routing for forwarding between nodes.

#### New: Configure `--devices`

The idea here is that if the automatic MTU/MRU calculation takes into account
the wrong set of devices for determining the maximum unit size configurations,
then if the administrator does not want to explicitly configure the MTU to a
known value but instead wants to specify to Cilium which devices it should use
for determining the MTU, then the administrator can override the setting.

This could plausibly be exposed through a parameter that accepts a set of
wildcards, for instance to select `eth*` devices and `wg*` interfaces.

#### New: Configure `--primary-network-cidr`[^f]

The idea here is to allow the administrator to select[^g][^h] which network
they consider to be the "primary" network based on the CIDR and derive the MTU
from this network. In this mode, even if the primary network has a higher MTU
than another attached network, Cilium would configure the MTU based on the
primary network. This could potentially cause connectivity issues for other
attached networks, but it would allow administrators to ignore other attached
networks if they believe they are not necessary for pod connectivity (for
example, non-Kubernetes networks; management networks; docker/ipvs/etc)

* Example use case: Cilium pod traffic should only consider Kubernetes networks
* Example use case: Cilium pod traffic should not use Kubernetes networks

### Status visibility

Example status command output (terse):

```
# cilium status
....
MTU mode: auto, MTU 1450 [cilium_vxlan], MRU 1500 [eth0]
```

Example status command output (verbose):

```
# cilium status --verbose
...
MTU mode: auto[^i]
- Features: tunnel[^j]
- Devices: [ eth0, eth1 ]
- Transmit: 1450 [ cilium_vxlan ]
- Receive: 1500 [ eth0, eth1 ][^k][^l]
```

## Impacts / Key Questions

### Impact: MTU selection affects connectivity / performance

The default behaviour of Cilium will determine how optimized the network
bandwidth is for the vast majority of users. We can err on the side of
providing guaranteed connectivity to users, or attempt to optimize the network
connectivity even if it may cause connectivity issues when connecting across
other networks.

### Key question: How to pick MTU by default

#### Option 1: Use minimum MTU from available network interfaces

##### Pros
* Cilium will not put users in a situation where pod connectivity fails by
  default just because there are multiple networks connected. Cons
* Lower default MTU can translate to lower bandwidth for pod<->pod traffic in
  user environments where unrelated networks are attached

#### Option 2:  Select a primary network and use that to determine MTU

##### Pros
* Higher default bandwidth

##### Cons
* In some corner cases (particularly with UDP), two-way connectivity with a
  peer on a lower-MTU network may not work correctly.
* Increases the likelihood of using IP fragmentation on the network, which
  Cilium does not always optimally handle today.
* VM-based CRIs like Kata may generate larger frames that Cilium is unable to
  place on certain networks (particularly in the case of IPv6 with no
  in-network fragmentation).

## SIG-Datapath discussion 2021-03-03
* It would be unfortunate that this requires user input to optimize in common
  cloud scenarios. Can we figure out a way to get this to work optimally there
  out-of-the-box? If not, are we really 'solving' the overall MTU issue?
* Can we implement some datapath solution for reducing MTU
  * ICMP PTB/FragNeeded type messages in eBPF datapath
    * Lots of assumptions here
    * Kernel enablement of PMTU
    * Application support - socket configuration
  * TCP MSS clamping
    * Could work, still need UDP solution though
* Maybe we should scope the milestone "expose native networks into pods" into
  this proposal rather than having it as a future step.
  * Needs further scoping on this implementation.

## Future Milestones

### Expose native networks into pods

*TODO: Seriously consider moving this into main proposal.[^m]*

Expose individual routes for each detected network into each pod. This feature
would allow a pod to establish connectivity to remote endpoints in each network
using the optimal MTU for that particular network based on the destination. It
may also require the ability to dynamically respond to network join or leave
events in the cluster to allow recalculation of routes in each pod to take into
account the correct MTU for each destination at a given point in time.

* Concern: Service IPs. K8s does not provide Service range to us today.
   * Mitigation: Socket LB. Pick backend before routing to select MTU.

### Dynamically update MTU based on agent configuration

Today, if you reconfigure Cilium MTU and restart the agent, it will not adjust
the configuration for existing pods in the cluster. Given that modification of
MTU configuration is not a common operation, we have not implemented automatic
enforcement of new MTU configuration on existing pods. Current solutions to
this problem are to either recreate the pods affected by this issue, or use the
cilium/mtu-config tool to manually update the MTU configuration.

---

[^a]: Next steps for this CFP are to update it to account for configuring multiple routes inside Pods in order to allow Pods to take advantage of each connected network with an appropriate MTU for that network.

[^b]: I've just added this as a new goal, if we agree this is a goal then it influences which solutions we think are sufficient to satisfy this CFP.

[^c]: Sounds good, given we want to rework this I think this appears to be quite a common case that should be covered to get resolved under this cfp, imho.

[^d]: This is one of the "Key Questions" below and as of SIG-Datapath 2020-03-03 we may revise this; it does not satisfy goal #4.

[^e]: We could argue whether this is a bug; the user has explicitly configured the MTU but also this means that overriding the MTU with tunnel mode configured is bound to cause problems.

[^f]: The semantics of this flag may not be limited to MTU handling, but could encompass determining the network that Cilium should favour for sending traffic over, ie also include https://github.com/cilium/cilium/pull/15072

[^g]: I see one UX problem with this approach - in L3 networks, users would need to specify possibly lots of subnets.

[^h]: Are you saying that you don't expect a single L3 network to be used to connect all of the nodes in the cluster?

[^i]: Other modes would be the ones under the "Manual MTU/MRU override" section above.

[^j]: Could include stuff like ipsec here too. Anything where Cilium takes into account other configuration to calculate the MTU.

[^k]: Valas: CIDR-block based MTU config would be good to reflect here.

[^l]: ref: https://docs.google.com/document/d/1GLYB5-DnQmz63AiqYHubRScE9xrzAiH4XAN38LWeLUI/edit#bookmark=id.ct9eciholhfd

[^m]: nikolay.aleksandrov@isovalent.com since the prior discussions, this idea seems more and more like the right approach going forward. Would be curious to hear your thoughts.

[^n]: https://github.com/cilium/cilium/issues/14339#issuecomment-786259600

[^o]: https://github.com/cilium/cilium/issues/14339
