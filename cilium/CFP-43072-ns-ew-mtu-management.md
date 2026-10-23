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
Specifically, we want to be able to cover the use-case of a cluster running
jumbo packets but that may still need to communicate over networks where
larger MTUs are unsafe (i.e. can lead to drops or fragmentation).

## Motivation
Cilium manages Pod endpoint MTU via route based MTU rules taking into
account Cilium overlay specified considerations like tunnel and encryption headers. 
This is controlled by the MTU manager which can either auto-detect and reconcile
this value based on what MTU relevant host device(s) can support, or this value
can be overridden via the `--mtu` flag.

This was added to deal with undesirable fragmentation and drops when using Cilium.
In particular because of the otherwise opaque cluster underlay network such as VXLAN
or Geneve (as well as encryption overheads) it is necessary for cilium to install a
default route mtu based on the highest safe MTU value accounting for these overhead(s).

This approach works well to defend against dropped or fragmented traffic by
using the kernel route MTU setting to signal to the local kernel or applications
regarding correct MSS sizing.

Where this is limited is that it only allows setting a single cluster MTU.
Taking the cannonical example where cluster is running on a network with jumbo-packets
enabled (say MTU=9000) for performance benefits. This presents the problem
that Pod traffic bound for other networks may not have a path MTU (PMTU) as big as
our in-cluster network (i.e. east-west (E/W) traffic).
In general, it is broadly accepted that 1500 is the correct "safe" MTU used for the
internet.

This document thus seeks to propose a amended approach to managing MTU in Cilium that
allows users to run a split MTU environment: in-cluster east-west (E/W) traffic, and all
other external traffic (north-south (N/S)).

## Background 

Conventionally there are a few ways to deal with this problem:

* Fragmentation: Performed by either the Kernels IP layer or intermediate routers; undesirable due
    to large performance overhead. As well instead of fragmentations packets may simply be dropped
    leading ot connection black holes.
* Path MTU discovery (PMTUD) (Ref)[https://www.rfc-editor.org/info/rfc1191/#section-2]: relies on 
    ICMP frag-needed (FN) and ICMPv6 packet-too-bug (PTB) messages
    sent back by intermediate routers that determine the packet size will exceed the networks MTU.
    This is considered a generally unreliable mechanism as it relies on all intermediate network hops
    to correctly forward the PMTU ICMP message back to the client. Any drops (i.e. due to firewall rules,
    misconfigured routers, etc) can result in black-holed traffic.
* Route MTU: Linux provides a mechanism to allow discovery of MTU as traffic is passing through the kernels
    ip layer using a FIB routing lookups. Traffic that matches a route with a route-MTU set influences: network 
    syscall errors, kernel TCP segmentation sizing and in some cases sending of PMTUD ICMP messages.
* Packet Layer Path MTU discovery (PlPMTUD) and other L4 discover approaches: General approach of using packet 
    layer (i.e. L4) constructs to discover path MTU for a connection. Linux specifically implements this for its
    TCP stack. Roughly speaking it uses a binary-search like probing approach to discover a reasonably high 
    path MTU. Downsides is that it is protocol specific and also introduces possible network connection latency
    during probing phase.

This CFP proposes extending Ciliums current flat MTU management approach into one that distinguishes between
"in-cluster" (i.e. E/W) and "external" (N/S) traffic. This solves the problem of a cluster with its own
homogenous network MTU being higher than what is considered "safe" for general internet traffic.

We also acknowledge a broader use case which will be out of scope this proposal but we keep this use
case in mind for future work. Ideally the proposed solution here would lend itself well to any such
functionaltiy.

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
configuration model; how we will define and seperate the EW and NS MTU values, and b) the
enforcement mechanism; how we realize these values in a way that allows client applications
to easily deal with path MTU issues. 

### E/W N/S MTU configuration model 

Currently Cilium exposes the `--mtu` flag to allow configuring of MTU. If set to `0` this
enables auto detection where the Agent will attempt to detect the correct MTU from the host
device. The specified/detected MTU value then has any necessary header overheads subtracted
such as VXLAN/Geneve tunnel or encryption headers. This computed MTU value that is current
set in the Pod netns default route MTU as well as endpoint BPF programs (i.e. for cases
like nodeport geneve DSR redirect to remote backend node). For the remainder of this document
this computed MTU value is synonymous with our E/W MTU value; auto-detection improvements
are out of scope (see [CFP-14339](CFP-14339-cilium-mtu-model.md)).

This realized E/W value will become our base "default" MTU to be installed as the device 
and Pod container default route MTU (no change from current approach). Instead we propose
adding a MTU restriction system on top of this that will signal to applications/system
processes as well as the kernel what the correct PMTUD is for N/S traffic.

Cilium already reconciles pod endpoint MTU in response to host netlink route events; this
proposal follows the same model for any E/W MTU changes. The E/W MTU is also loaded into all TC/XDP
BPF programs via the `CONFIG(device_mtu)` runtime variable (as well as the `#MTU` environment
variable), ensuring in-cluster traffic such as Geneve DSR redirects to remote backend nodes
use the correct MTU. This is unchanged by this proposal.

Additionally we propose the following flags:

* `--enable-socket-ns-mtu (default: false)`: enables the functionality proposed in 
    "Enforcement Mechanism 1" (@todo: add link to section?). Requires socketLB to be enabled
    and `--ns-mtu` to not be `0`.
* `--ns-mtu (default: 1500)`: Specifies what N/S MTU should be used for N/S MTU enforcement.
    If zero the N/S MTU enforcement is disabled.

These will both enable and configure our N/S MTU management. The specific implementation plan will 
follow. At high level this provides the following functionality: allows configuring of Cilium E/W
above a "internet-safe" MTU (1500) by providing signalling to both Pod endpoint and host processes
of PMTU restrictions for out-of-cluster traffic. 

Specificaly; we are targetting the use case of a cluster running a network with jumbo-packets
enabled (i.e. mtu=9000) and having Cilium also configure itself with `--mtu=9000`. In such case
external "N/S" traffic should safely have it's MTU restricted to avoid issues with drops, IP
fragmentation and black holes while also allowing workloads to make use of the full cluster 
network for performance reasons.

### MTU enforcement mechanism

Excluding fragmentation, the Kernel network stack has two mechanismsof dealing with packets which 
will exceed *known* MTU limits; pre-configured FIB route MTU and PMTUD "discovered" 
values stored seperately.
When a packet traverses the kernels IP layer we perform the FIB match as well as lookup 
any known PTMU's - the best known path MTU is the minimum of these. 
In a TCP socket this allows the kernel to automatically segment the TCP data in such 
a way that transmitted packet will fit this MTU. For UDP sockets traffic, if dont-fragment 
(DF=1) and the socket PMTU option is set (`IP_PMTUDISC_DO`) then individual `sendmsg4()` 
syscalls will return an error for the application to decide on how to handle it (in many 
cases protocols built on top of UDP can use this to manage segmentation).

Using these mechanisms we can have Cilium help manage implied path MTUs to aid
applications to avoid MTU or fragmentation issues as well as maximizing network
utilization.

Although we currently rely on route-MTU for enforcement, this document proposes solutions
that rely on solutions that can take advantage of Ciliums ip identities to distinguish
between N/S and E/W traffic.

It is important to note; these MTU values are not themselves the network path MTU.
MTU is fundamentally a property of a network path, resulting from intermediate networks
having physical hardware limits, as well as the possibility of things like network tunnels
and encryption which introduce unexpected header overheads to existing traffic. 

#### Proposed Approach: Hybrid approach using mechanisms 1. and 2. (described below)

Mechanism 1 (PMTUD) is our primary approach: it can be applied at both the Pod endpoint TC
and the host namespace device TC, has no dependency on socketLB, and covers all traffic types.
However, PMTUD is purely reactive — the first violating packet is dropped before the kernel
PMTU cache is updated, causing a TCP retransmit delay. Additionally, PMTUD cache entries are
per-destination-IP and expire after `net.ipv4.route.mtu_expires` (default 600s), so each new
external destination requires its own drop-and-recover cycle.

Mechanism 2 (socket hooks) addresses these limitations specifically for Pod netns TCP: by
marking sockets at connect time, the kernel selects the correct route MTU before the first
packet is sent, MSS is correctly negotiated at connection establishment and never renegotiated,
and there are no PMTUD cache expiry concerns. However, it requires socketLB to be enabled and
is not practical for the host namespace due to routing table complexity.

We therefore propose the following hybrid approach:

* [PMTUD (Mechanism 1)](#enforcement-mechanism-1-send-pmtud-messages-from-tc-back-to-pod):
    Primary mechanism. Covers all traffic types for both Pod and host netns.
* [Socket level routing hooks (Mechanism 2)](#enforcement-mechanism-2-via-socket-level-connect4sendmsg-hooks):
    Optimization for Pod netns TCP. Establishes correct MSS at connect time, avoiding the
    drop-and-recover cycle for the common TCP case. Gated behind `--enable-socket-ns-mtu`.

#### Enforcement Mechanism 1. Send PMTUD messages from TC back to Pod

One mechanism we already employ in Cilium is crafting and sending back synthetic
PMTU messages from Cilium's datapath.
Path MTU Discovery (PMTUD) is specified in [RFC-1191](https://www.rfc-editor.org/info/rfc1191/#section-2).
It provides a IP layer approach to MTU discovery whereby intermediate routers notify peers regarding 
MTU violations using ICMP DF messages.
When consumed by the host kernel, these messages are stored as path-MTU values
keyed on the socket connection tuple.

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
of DF/PTB packet being dropped along the return path (the return path is all cilium
managed local links/veths). As this PMTU message is expected to arrive back at 
the Pod namespace the kernel network stack will cache the emitted MTU value.
Essentially this proposes that Cilium take on the role of sending PMTUD messages
to both Pod and host namespace stack.

One downside of using PMTUD messaging for this is that the underlying mechanism is purely reactive.
This means that under normal circumstances we can only be alerted to a violating packet after it
has been sent.
According to RFC-1191, following a PMTUD message the offending packet is be dropped. In the kernel
TCP implementation this notifies that the tcp packet was dropped and needs to be transmitted.

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

We will add a similar hook to the host namespace device for egress.

Pros:
* Can be added to both the Pod endpoint and host device TC without interfering with
    host level routing/rules etc.
* Uses external endpoint identity from ipcache to determine if traffic is N/S.
    This avoids the difficult task of computing and duplicating external/internal CIDR ranges.
* Lends itself well to the general network MTU case mentioned in the introduction.
    A more general approach would combine the identity based checking with a LPM network MTU
    table that matches network CIDRs.

Cons:
* Must be set up as fail open; PMTUD messages can only lower known PMTUD on the Pod
    netns kernel. Therefore Pod inner default route MTU must be the EW value (high).
* PMTUD can only react: PMTUD messages are sent as a response to violating packet being
    sent. In the Kernel TCP stack a PMTUD matching the socket will result in TCP re-transmit.
    This may introduce latency into connections that rely on the PMTUD mechanism for recovery.
* PMTUD is also on specific endpoint IPs not CIDR ranges; this means that every
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

#### Enforcement Mechanism 2. Via socket level connect4/sendmsg hooks 

Cilium currently installs kfunc hooks for the `connect` and `sendmsg` syscalls to implement
service LB DNAT at the socket layer (prior to INGRESS TC). 
This mechanism would add another step to these hooks, following socketLB DNAT, we would
lookup remote address identities to classify traffic as either E/W or N/S. 
Within the hook; N/S traffic would have its socket `fwmark` be set to `0x40`. Then during
ip route matching an `ip rule` would direct this traffic at an alternate route table `100`
which mirrors the Pods route table, except with a route MTU set using the N/S value.

* bpf_sock.c (connect and sendmsg hooks): following socketLB DNAT we lookup remote endpoint
    against ipcache to retrieve identity. For external "N/S" identities we perform `bpf_setsockopt()`
    to set the fwmark of the socket to be `0x40`.
* On Pod endpoint `CNI ADD` (as well as reconciled later by Cilium agent): we install the following:

```bash
ip rule add pref 1000 fwmark 0x40 lookup 100
ip route add 10.244.1.52 dev eth0 scope link table 100
ip route add default via 10.244.1.52 dev eth0 mtu 1500 table 100
```

```
        ┌───────────────┐
        │  ip rule eval │
        │  (pref order) │
        └───────┬───────┘
                │
                ▼
       ┌─────────────────────┐
       │   fwmark == 0x40?   │
       └───┬─────────────┬───┘
           │             │
          YES            NO
           │             │
           ▼             ▼
  ┌─────────────────┐  ┌─────────────────┐
  │ table 100       │  │   main table    │
  │ ─────────────── │  │ ─────────────── │
  │ default         │  │ default         │
  │ via 10.244.1.52 │  │ via 10.244.1.52 │
  │ dev eth0        │  │ dev eth0        │
  │ mtu 1500 (N/S)  │  │ mtu 9000 (E/W)  │
  └────────┬────────┘  └────────┬────────┘
           │                    │
           └─────────┬──────────┘
                     │
                     ▼
                   eth0
```

TCP/`connect()` case:

* Client Pod opens socket via connect syscall with specified external endpoint tuple.
* Connect hook is invoked, socket tuple is DNAT-ed.
* Destination endpoint addr is looked up in ipcache map (LPM).
* If endpoint is *not* a known cluster identity we set sock opts with `fwmark=0x40`.
* Client sends data over socket:
    * Packet is sent with `fwmark=0x40` in skb.
    * Packet is matched against FIB and hits ip rule for `fwmark == 0x40`; rule instructs
        to use route table 100.
    * Packet is matched against route table 100; hits default route `0.0.0.0/0` with route
        route MTU=1500.

The reason we choose to have the E/W MTU on the default table/route is that this is what
accepted listener (i.e. server) connections will use to negotiate their end of TCP MSS.
By default we want this to be the "high" E/W MTU as otherwise in-cluster connections
would get clamped down by the server Pod using the N/S value.
For inbound connections from external endpoints this is fine since MSS is still clamped
down by the minimum of the client endpoints MTU.

We opt to mark the socket on the negative E/W case: failing to lower the MTU is considered
more harmful than a false positive in which an artificially low value is used.

UDP/Connect case: Same as TCP/Connect case; only difference is how kernel handles subsequent
    sendmsg len violations.

UDP/Connectionless case:
* UDP socket is created 
* Client sends data via sendmsg
* sendmsg hook is invoked; sendmsg dst is DNAT-ed. 
* Message destination endpoint addr is looked up in ipcache map.
* Message is sent as-is, using default route (mtu=9000).
* If endpoint is *not* a known cluster identity we set sock opts with fwmark=0x40.
* Subsequent message is sent; socket is marked and ip routing rule applies so
    we match on route table 100 which has MTU=1500.

Pros:
* Allows for using external endpoint identity from ipcache to determine if traffic is N/S.
    This avoids complex Pod/Host ns routing setups.
* Efficient on the most common TCP socket use case: fwmark is installed immediately on the
    connect hook — no waiting for PMTUD message/retransmits.
    Only overhead incurred is the additional LPM lookup for external endpoint identity.

Cons:
* Requires socketLB feature to be enabled (default=`true`) for MTU management to work.
* On UDP connectionless sockets the first message sent may black-hole silently as we depend on the
    sendmsg syscall to mark the socket (this is also true of Mechanism 1 as we cannot know the
    correct PMTU for a packet on an unbound UDP socket until it is sent).
    * Otherwise, UDP sockets that use connect to "connect" to a specific external endpoint see the
        same benefits as the TCP case.

### Upgrade/downgrade considerations 

Because with `--enable-ns-mtu=false` our fallback behavior is the same as existing
versions that have the `--mtu` functionality; downgrading is simple as it simply drops
the NS MTU lowering (assuming no configuration changes on the downgrade).
For the upgrade path, this is also unaffected for the same reasons.

### Implementation plan

#### Phase 1: Add E/W N/S configuration model and implement PMTUD-based enforcement mechanism

We don't need to add both enforcement mechanisms at once. In fact we can gain
much of the benefit of this feature by implementing the configuration approach
and PMTUD messaging mechanism.

#### Phase 2: Add socket level enforcement mechanism

As a follow up, we add the socket layer enforcement mechansim as well as flag to enabled/disable
this functionality.
