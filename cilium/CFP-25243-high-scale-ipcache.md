# CFP-25243: High-Scale IPcache

**SIG: SIG-DATAPATH**

**Begin Design Discussion:** 2023-04-26

**Cilium Release:** 1.14

**Authors:** Paul Chaignon <paul@isovalent.com>

## Summary

This CFP describes a new mode for the ipcache to enable scaling to very large
clustermeshes. In this mode, the ipcache doesn't contain entries for remote
pods by default. Pod-to-pod traffic is encapsulated with the pod IP addresses,
just to carry the source identity across nodes and allow us to implement
ingress policies. Egress policy enforcement in not described in this CFP, but
it would have to rely on CIDR rules (or FQDN rules).

## Motivation

Cilium currently tracks the IP addresses of all pods in all clusters in the
ipcache map of each node. Some Cilium users however have clustermeshes with
more than one million pods where that approach doesn't scale. To support such
large clustermeshes, we need to avoid adding all remote pod IP addresses in the
ipcache.

## Goals

* Scale clustermeshes to >1M pods.
* Grow the ipcache sub-linearly with the number of pods in the cluster.
* Support ingress policies as usual.

## Non-Goals

* Support FQDN policies for egress traffic to entities inside the cluster. It
  is a lot more involved and may come in a follow-up CFP.
* Support toEndpoints policies on egress.
* Support advanced features such as the egress gateway or IPsec.
* Support tunneling mode. The physical network must be able to route pod IP
  addresses.

## Proposal

### Walkthrough

Let's walk through a pod-to-remote-pod (pod1@node1 -> pod2@node2) connection
with policies enforced on both egress and ingress:

1. Pod1 sends a SYN packet to pod2's IP.
2. Pod2's IP is looked up into a BPF map to check if encapsulation is
   necessary. See section "World CIDR CRD and Map" below for details.
2. The packet is encapsulated with "pod1 -> pod2" as the outer IP header and
   the tunnel ID is set to pod1's security identity. See section "Pod-to-pod
   encapsulation" below for details.
3. Node2 receives the encapsulated packet, retrieves the source security
   identity and decapsulates.
4. The decapsulated packet is sent to pod2's ingress policy enforcement via
   tail call and the ingress policies are enforced using the retrieved security
   identity.

### Changes to the IPcache

The goal is to reduce the size of the ipcache and in particular for the ipcache
to not grow linearly with the number of pods in the cluster.

We will keep in the ipcache:
- Special identities for entities (host, remote nodes, etc.).
- Identities for well-known identities. This is mostly to get DNS pod
  identities. If we want to later use FQDN policies on egress, we will need to
  resolve domain names via the DNS pods. We can't use the FQDN policies to
  allow connections to the DNS pods obviously so we'll have to rely on
  toEndpoints for that.

### Pod-to-pod Encapsulation

In high-scale IPcache mode, the encapsulation for pod-to-pod traffic is only
required to carry the source identity over to the destination node. However,
since Cilium doesn't know how to map pods to their node IP addresses (since
we don't have the ipcache), we can't use our usual encapsulation with node IPs.

Instead, for the inner and outer headers, we will use the same IP addresses,
the pod IP addresses. Setting the outer destination IP address is easy, we can
use `bpf_skb_set_tunnel_key` as we do today. Setting the outer source IP
address is only supported in Linux v5.19
([26101f5ab6bd](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=26101f5ab6bd)).

It would also have been possible to use the node IP address as the outer source
IP. That would however result in an assymetric connection on the physical
network. Forward packets would have IPs IP_NodeA -> IP_PodB and replies
IP_NodeB -> IP_PodA. That may cause issues with middleboxes and observability
tooling on the physical network.

On ingress, we shouldn't need to do anything new to decapsulate the traffic. We
already retrieve the identity in bpf_overlay and don't use the outer IP
addresses anyway.

### World CIDR CRD and Map

To know what traffic needs to be encapsulated, we can't rely on the ipcache or
the tunnel map (we don't have a clear podCIDR->node mapping for all IPAMs). We
will need a new CRD for the users to provide the CIDRs of destination that
don't need to be encapsulated. Everything else will be encapsulated.

This new CRD is CiliumWorldCIDRs. It will provide a list of destination CIDRs
for which no encapsulation is necessary, typically destination outside the
cluster. This list of CIDRs will be plumbed into an LPM BPF map for the
datapath to check if encapsulation is needed.

### Migration to new mode

To allow for the migration of Cilium nodes to this new mode, an intermediate
mode without any encapsulation will be added. Users will first migrate all
nodes to the decapsulation-only mode. Once that is complete, they will be able
to migrate nodes to the full high-scale IPcache mode without risking
encapsulated traffic being unrecognized.
