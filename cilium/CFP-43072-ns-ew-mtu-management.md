# CFP-43072: Cilium EW NS MTU

**SIG:** SIG-Datapath

**Sharing:** Public

**Begin Design Discussion:** 2026-10-27

**Cilium Release:** 1.21

**Authors:** Tom Hadlaw <tom.hadlaw@isovalent.com>

**Status:** Implementable

---

## Summary

Current versions of Cilium (v1.20) have functionality to discover the network
Maximum Transmission Unit (MTU) and enforce that for Pod workloads via kernel
route-MTU settings.

This document proposes a new approach to MTU management that builds on top
of the existing functionality.
Specifically, we want to be able to solve the use-case of a cluster running
jumbo packets but that may still need to communicate over networks where
larger MTUs are unsafe (i.e. can lead to drops or fragmentation).

## Motivation
Cilium manages Pod endpoint MTU via route based MTU rules taking into
account Cilium overlay specific considerations like tunnel and encryption headers.
This is controlled by the MTU manager which can either auto-detect and reconcile
this value based on what MTU relevant host device(s) can support, or it can be 
overridden via the `--mtu` flag.

This was added to deal with undesirable fragmentation and drops when using Cilium.
In particular because of the otherwise opaque cluster underlay network such as VXLAN
or Geneve (as well as encryption overheads) it is necessary for cilium to install a
default route mtu based on the highest safe MTU value accounting for these overhead(s).

This approach works well to defend against dropped or fragmented traffic by
using the kernel route MTU setting to signal to the local kernel or applications
regarding correct MSS sizing.

This is limited as it only allows setting a single cluster MTU.
Taking the canonical example where the cluster is running on a network with jumbo-packets
enabled (say MTU=9000) for performance benefits. This presents the problem
that Pod traffic bound for other networks may not have a path MTU (PMTU) as big as
our in-cluster network (i.e. east-west (E/W) traffic).
In general, it is broadly accepted that 1500 is the correct "safe" MTU used for the
internet ([RFC-4821](https://www.rfc-editor.org/info/rfc4821)).

This document seeks to propose an amended approach to managing MTU in Cilium that
allows users to run a split MTU environment: in-cluster east-west (E/W) traffic, and all
other external traffic (north-south (N/S)).

## Background

Conventionally there are a few ways to deal with the problem of differing network MTUs:

* Fragmentation: Performed by either the Kernels IP layer or intermediate routers; undesirable due
    to large performance overhead. As well, instead of fragmentation, packets may simply be dropped
    leading to connection black holes.
* Path MTU discovery (PMTUD) [Ref](https://www.rfc-editor.org/info/rfc1191/#section-2): relies on
    ICMP frag-needed (FN) and ICMPv6 packet-too-big (PTB) messages
    sent or forwarded back by intermediate routers that determine the packet size will exceed the networks MTU.
    This is considered a generally unreliable mechanism as it relies on all intermediate network hops
    to correctly forward the PMTU ICMP message back to the client. Any drops (i.e. due to firewall rules,
    misconfigured routers, etc) can result in black-holed traffic.
* Route MTU: Linux provides a mechanism to allow discovery of MTU as traffic is passing through the kernel's
    IP layer using a FIB routing lookup. Traffic that matches a route with a route-MTU set influences: network
    syscall errors, kernel TCP segmentation sizing and in some cases sending of PMTUD ICMP messages.
* Link MTU: MTU value configured on individual links, following routing decisions the link MTU is used as the
    backstop MTU value if no route or PMTUD derived path-MTU was discovered.
* Packet Layer Path MTU discovery (PlPMTUD) and other L4 discovery approaches: General approach of using packet
    layer (i.e. L4) constructs to discover path MTU for a connection. Linux specifically implements this for
    TCP stack. Roughly speaking it uses a binary-search like probing approach to discover a reasonably high
    path MTU. Downsides is that it is protocol specific and also introduces possible network connection latency
    during probing phase.

This CFP proposes extending Cilium's current flat MTU management approach into one that distinguishes between
"in-cluster" (i.e. E/W) and "external" (N/S) traffic. This solves the problem of a cluster with its own
homogenous network MTU being higher than what is considered "safe" for general internet traffic.

We also acknowledge a broader use case which will be out of scope of this proposal but we keep this use
case in mind for future work. Ideally the proposed solution here would lend itself well to any such
functionality.

## Goals

* Propose a E/W N/S model of configuring Cilium MTU management.
* Design MTU enforcement mechanism that can reliably apply this split domain MTU model.

## Non Goals
* Propose general (i.e. CRD based) approach to managing network MTUs with Cilium.
* Implement BPF based TCP MSS clamping mechanism to aid with TCP MTU negotiation.
* Fix all places where unexpected fragmentation may still occur in Cilium (in the past
    we've struggled to plug every hole where an unexpected header overhead or link
    MTU may cause fragmentation or drops).
* Make any changes to how to auto-detect the correct E/W value from host interfaces (i.e
  questions related to [CFP-14339](CFP-14339-cilium-mtu-model.md)). This CFP takes the
  approach used by `--mtu=0` as-is. Any potential future changes there should not affect
  the major components of this design.

## Proposal

This CFP has two parts that together form the E/W N/S MTU management feature: a) the
configuration model; how we will define and separate the EW and NS MTU values, and b) the
enforcement mechanism; how we realize these values in a way that allows client applications
to easily deal with path MTU issues.

### E/W N/S MTU configuration model

Currently Cilium exposes the `--mtu` flag to allow configuring of MTU. If set to `0` this
enables auto detection where the Agent will attempt to detect the correct MTU from the host
device. The specified/detected MTU value then has any necessary header overheads subtracted
such as VXLAN/Geneve tunnel or encryption headers. This computed MTU value that is currently
set as the Pod netns default route MTU as well as in endpoint BPF programs; it is used only for
cases like NodePort Geneve DSR redirect to remote backend nodes.
For the remainder of this document this computed MTU value is synonymous with our E/W MTU 
value; auto-detection improvements are out of scope 
(see: [CFP-14339](CFP-14339-cilium-mtu-model.md)).

This realized E/W value will become our base "default" MTU to be installed as the device
and Pod container default route MTU (no change from current approach). 

Given this, we propose a new MTU restriction approach that signal to applications/system 
processes as well as the kernel what the correct PMTU is for N/S traffic.
Namely, this approach will leverage PMTUD messaging - effectively providing a way for Ciliums
datapath to signal to client sockets regarding path MTU violations.

To accommodate this functionality we propose the following flag:

* `--mtu-north-south (default: 0)`: Specifies the N/S MTU for PMTUD enforcement via ICMP
    fragmentation-needed. If zero (default), N/S MTU enforcement is disabled and behavior
    is identical to current Cilium MTU management.

This flag enables and configures N/S MTU management. The specific implementation plan will
follow. At high level this provides the following functionality: allows configuring of Cilium E/W
above a "internet-safe" MTU (1500) by providing PMTUD signalling to Pod endpoints for
out-of-cluster traffic.

We are seeking to solve the use case of a cluster running a network with jumbo-packets
enabled (i.e. mtu=9000) and having Cilium also configure itself with `--mtu=9000`. In such case
external "N/S" traffic should safely have its MTU restricted to avoid issues with drops, IP
fragmentation and black holes while also allowing workloads to make use of the full cluster
network for performance reasons.
This document will henceforth use this as a canonical example when describing the proposed
feature set: real world use cases will likely differ from this specific example.

Cilium already reconciles pod endpoint MTU in response to host netlink route events; this
proposal follows the same model for any E/W MTU changes. The E/W MTU is also loaded into all TC/XDP
BPF programs via the `CONFIG(device_mtu)` runtime variable (as well as the `#MTU` environment
variable), ensuring in-cluster traffic such as Geneve DSR redirects to remote backend nodes
use the correct MTU. This is unchanged by this proposal.

### MTU enforcement mechanism

Excluding fragmentation, the kernel network stack has two (network-layer) mechanisms of 
dealing with packets which will exceed *known* MTU limits; pre-configured FIB route MTU 
and PMTUD "discovered" values stored separately.

When a packet traverses the kernel's IP layer we perform the FIB match as well as look up
any known PMTUs - the best known path MTU is the minimum of these.
In a TCP socket this allows the kernel to automatically segment the TCP data in such
a way that transmitted packets will fit this MTU. For UDP socket traffic, if dont-fragment
(DF=1) and the socket PMTU option is set (`IP_PMTUDISC_DO`) then individual `sendmsg4()`
syscalls will return an error for the application to decide on how to handle it (in many
cases protocols built on top of UDP can use this to manage segmentation).

Using these mechanisms we can have Cilium help manage implied path MTUs to aid
applications to avoid MTU or fragmentation issues as well as maximizing network
utilization.

Although we currently rely on route-MTU for enforcement, this document proposes solutions
that can take advantage of Cilium's IP identities to distinguish between N/S and E/W traffic.

It is important to note; these MTU values are not themselves the network path MTU.
MTU is fundamentally a property of a network path, resulting from intermediate networks
having physical hardware limits, as well as the possibility of things like network tunnels
and encryption which introduce unexpected header overheads to existing traffic.

#### PMTUD Enforcement Mechanism: Send PMTUD messages from TC back to Pod

One mechanism we already employ in Cilium is crafting and sending back synthetic
PMTU messages from Cilium's datapath.
Path MTU Discovery (PMTUD) is specified in [RFC-1191](https://www.rfc-editor.org/info/rfc1191/#section-2).
It provides a IP layer approach to MTU discovery whereby intermediate routers notify peers regarding
MTU violations using ICMP DF messages.
When consumed by the host kernel, these messages are stored as path-MTU values keyed on the socket connection tuple.

Cilium already has code infrastructure to generate correct icmp4 replies by
overwriting the current packet context (see: `generate_icmp4_reply`).
In fact, the nodeport DSR code path already uses this to notify external clients
of MTU violations caused by opaque tunnel overheads.

We would check packet lengths (or rather GSO segment length) on post DNAT packets
on the TC egress program. Looking up the identity of the destination endpoint we classify
traffic on whether it is N/S or E/W. On the N/S case we check the packet length and
see if it exceeds the configured N/S MTU. If the packet length exceeds the N/S MTU we
drop the packet and respond with ICMP frag-needed (ICMPv6 Packet-too-big in the case
of ipv6).

For the special case that traffic is classified as in-cluster bound but to a NodePort
with DSR enabled, we would also subtract the DSR path overhead. Effectively performing
Cilium's existing pmtu-discovery functionality (this feature performs a similar PMTUD
messaging from on NodePort ingress to account for additional DSR path overhead).
As pmtu-discovery is related to the E/W MTU (i.e. `--mtu` value) this would still be
toggled by the `enable-pmtu-discovery` flag as this is independent from N/S considerations.
External NodePort PMTUD will still require that `pmtuDiscovery.enabled` be set.
In the case that such traffic is classified as N/S, such as if Node IP not in the ipcache
is used, then we simply fall back to the N/S MTU which is lower than E/W.

The following function describes how we classify between N/S and E/W:

```c
static __always_inline bool
endpoint_is_external(const struct remote_endpoint_info *info)
{
    __u32 identity = info->sec_identity;
    return identity == UNKNOWN_ID ||                    /* assume world, restrict to N/S MTU */
            !identity_is_cluster(identity) ||            /* see: bpf/lib/identity.h */
            info->flag_remote_cluster ||                 /* remote-cluster nodes*/
            (!identity_is_local(identity) && identity >= 100 &&
            extract_cluster_id_from_identity(identity) != CONFIG(cluster_id)); /* remote-cluster pods */
}

```

Unlike the general case of PMTUD, using it in this context avoids any pitfalls
of DF/PTB packet being dropped along the return path (the return path is all Cilium-managed local links/veths). As this PMTU message is expected to arrive back at
the Pod namespace the kernel network stack will cache the emitted MTU value.
Essentially this proposes that Cilium take on the role of sending PMTUD messages
to both Pod and host namespace stack.

One downside of using PMTUD messaging for this is that the underlying mechanism is purely reactive.
This means that under normal circumstances we can only be alerted to a violating packet after it
has been sent.
According to RFC-1191, following a PMTUD message the offending packet is to be dropped. In the kernel
TCP implementation this notifies that the TCP packet was dropped and needs to be retransmitted.

```
          ┌─────────────────────────────┐
          │         Pod netns           │
          │                             │
          │   App ──► eth0              │
          └──────────────┬──────────────┘
                         │
                         ▼
               bpf_lxc: TC ingress
          ┌───────────────────────────────────┐
          │       cil_from_container          │
          │  ───────────────────────────      │
          │  per-packet LB (DNAT)             │
          │  ct_lookup4                       │
          │  policy_can_egress4               │
          │  ipv4_needs_frag                  │ <- Classify traffic based on external or not (i.e. N/S)
          │    generate_icmp_pmtud            │      If traffic is N/S and wire len exceeds N/S MTU call
          └───────────────────────────────────┘      generate_icmp_pmtud: ICMP Unreachable Frag-Needed.
                                                     If necessary; fixup ICMP src addr to be svc VIP.
```

For the lxc ingress side, the PMTUD redirects directly back to the inner Pod device. This happens after
both service DNAT (both socket LB or in TC), and policy enforcement. Therefore the lookup we perform is
on the post svc LB translated packet and blocked packets will be dropped before the PMTUD check/redirect. 
LXC ingress has a host firewall check but that also happens after initial policy lookup. The PMTUD
will happen before any host policy checks, meaning that the PMTUD may be initially delivered for a
host endpoint that will be dropped by the firewall.

The LXC ingress side is irrelevant as the generated PMTUD messages are always redirected back into
the Pod inner device.

##### Host enforcement

Unlike the Pod LXC case, host namespace enforcement will need to be done on the netdev egress.
This means PMTUD rewritten packets will be redirected back to the same device (i.e. netdev ifindex).
Because the kernel pmtud cache update is performed on the ingress ip-layer in kernel function `ip_rt_update_pmtu`;
we need to redirect the PMTUD message to the ingress, treating the ICMP PMTUD message like an incoming packet.

Host datapath, on ingress, already has host-firewall exceptions for ICMP 'unreachable' frag-needed messages
when the pmtudDiscovery feature is enabled ('true' by default).

For the proposed north-south enforcement, when this is disabled, we will also allow such ICMP PMTUD messages
to bypass the firewall.

```
          ┌─────────────────────────────┐
          │         Host netns          │
          │                             │
          │   App ──► eth0              │
          └──────────────┬──────────────┘
                         │
                         ▼
               bpf_host: netdev egress 
          ┌───────────────────────────────────┐
          │       cil_to_netdev               │
          │  ───────────────────────────      │
          │  generate_icmp_pmtud              │
          │    ctx_redirect(ctx-ifindex,      │
          │    BPF_INGRESS)                   │
          └──────────────┬────────────────────┘ 
                         │
                         ▼
               bpf_host: netdev ingress 
          ┌───────────────────────────────────┐
          │       cil_from_netdev             │
          │  ───────────────────────────      │
          │  generate_icmp_pmtud              │
          │    ctx_redirect(ctx-ifindex,      │
          │    BPF_INGRESS)                   │
          └───────────────────────────────────┘ 
```

##### Pros/Cons/Tradeoffs

Pros:
* Can be added to both the Pod endpoint and host device TC without interfering with
    host level routing/rules etc.
* Uses external endpoint identity from ipcache to determine if traffic is N/S.
    This avoids the difficult task of computing and duplicating external/internal CIDR ranges.
    Unlike possible route MTU based approaches; this means much cleaner integration with existing
    Cilium modes as well as avoiding the need to manage user host namespace ip routes.
* Lends itself well to the general network MTU case mentioned in the introduction.
    A more general approach would combine the identity based checking with a LPM network MTU
    table that matches network CIDRs. In the future we can adapt such use cases by also matching
    on LPM map based "route-MTU" policies (we leave that as future work).

Cons:
* Must be set up as fail open; PMTUD messages can only lower known PMTUD on the Pod
    netns kernel. Therefore Pod inner default route MTU must be the EW value (high).
* PMTUD can only react: PMTUD messages are sent as a response to violating packet being
    sent. In the Kernel TCP stack a PMTUD matching the socket will result in TCP re-transmit.
    This may introduce latency into connections that rely on the PMTUD mechanism for recovery.
* PMTUD is also on specific endpoint IPs, not CIDR ranges; this means that every
    subsequent TCP socket connecting to a different host will also need to wait
    for the PMTUD and retransmit.
* Finally, cached PMTUD entries are only held for mtu_expires=600s so even on established
    connections users may experience a blip while the correct MTU is re-learned.
    The sysctl `net.ipv4.route.mtu_expires` can be used to adjust this timeout per the use case.
* BPF generated ICMP messages are not currently rate-limited. Given the broadening
    of usage of such message we may need to add rate limiting of such messages by
    destination.
* The PMTUD caching mechanism in the kernel uses a bounded hash table. A workload
    connecting to a large set of distinct remote endpoints may cause the hash structure
    to perform poorly.
* For host based enforcement, we will likely need to perform a remote host ipcache lookup for
    each outbound packet adding some slight overhead to such connections. This is likely 
    mitigated by there being much less host traffic being processed.

### Upgrade/downgrade considerations

Because the semantics of `--mtu` don't change, when upgrading there will be no functional
difference unless `--mtu-north-south` is set away from the default of 0 (disabled).
If the user enabled north-south enforcement during or after the upgrade then the 
north-south PMTUD functionality will stop working upon downgrade. All else equal, any
endpoints datapath will continue to work the same.

The same applies to host namespace networking.

#### Testing

As it is key that local generated PMTUD messages are never dropped on their way back
to the Pod, we will want to add additional checks to the cilium e2e connectivity
tests "no-unexpected-drops" check to flag any dropped PMTUD messages.
In general, under most CI runtimes, we shouldn't expect to see these messages
otherwise.

Otherwise implementation will focus heavily on unit testing, leaving e2e
as a final sanity check for functionality. We will write bpf unit tests
as generic headers so we can re-run them under a comprehensive set of configs.

Primary BPF testing coverage cases:
* from_container:
    * Packet is correctly checked for MTU against wire size (i.e. simulate GSO packets).
        * Packets dropped by policy should not be rewritten.
        * Host bound packets are still rewritten before host-firewall policy.
    * Offending packets are rewritten and redirected correctly as PMTUD messages.
        * Using scapy framework, check PMTUD payload for correctness ensuring Pod
            kernel does not drop the incoming packet.
* to_netdev:
    * Packet is correctly checked for MTU against wire size (i.e. simulate GSO packets).
    * Offending packets are rewritten and redirected correctly as PMTUD messages to from_netdev.

#### Observability and operational features

Datapath metrics:

* Counter: `cilium_north_south_pmtud_message_total`: counts the number of
    generated north-south PMTUD messages generated and sent. This is also
    the number of drops due to packets exceeding n/s MTU.
    This is useful for detecting workloads frequently exceeding N/S MTU
    possibly due to misconfiguration or incorrect MTU handling (ex. such
    as for UDP sockets).

Control-plane metrics:

* Gauge: `cilium_north_south_mtu`: current derived north south MTU size.
    If north-south mtu enforcement is disabled due to the configured value
    being below the realized `--mtu` value (possibly auto-detected) then
    this will show '0'.

* Gauge: `cilium_east_west_mtu`: current Cilium derived internal MTU size.
    This is the MTU applied to the Pod inner default route after subtracting
    overheads.
    Because Cilium expects link MTUs to be heterogeneous this can double
    as a way to easily detect misconfigured cluster networks or nodes.
