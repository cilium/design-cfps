# CFP-20129: Cilium-Native Multi-Homing

**SIG: SIG-Agent**

**Begin Design Discussion:** 2023-03-31

**Cilium Release:** 1.14

**Authors:** Sebastian Wicki <sebastian@isovalent.com>, Tobias Klauser <tobias@cilium.io>

## Summary

Allow Cilium to attach pods to multiple networks.

## Motivation

Users might want to connect their workload pods to multiple different
on-premise networks. Currently, Kubernetes defines no common API allowing to
define networks in addition to the cluster default pod network. This CFP is
about introducing an API in Cilium which will allow to support multiple
networks to be attached to a pod.

## Goals

* Multiple interfaces in a pod, with direct routing mode on the host netns
* IP allocation for secondary pod interface using IPAM pools (see [CFP-2022-07-20: Pools for PodCIDR IPAM][1])
* Basic policy enforcement on secondary networks (TBD)

## Non-Goals

* Compatibility with upstream [MultiNetwork KEP][2] API. However we try to keep
	as close as possible to the KEP in order to allow integrating with the
	upstream solution with minimal changes once it is in place.
* Support for the following on secondary networks:
  * K8s services
  * Other forms of IPAM (or DHCP etc)
	* BGP
	* Tunnel mode
	* Transparent encryption
  * SR-IOV

## Proposal

### Overview

TODO

### Resources

#### CRD

This CFP proposes to introduce a `CiliumPodNetwork` CRD which allows to define
networks to which pods can be attached.

The following `CiliumPodNetwork` defines a pod network `default` which maps to
the cluster default pod network. It allocates addresses from the IPAM pool
`ganymede` to the pods. It defines all traffic to routed via the (default)
gateway 1.0.0.1.

```yaml
apiVersion: "cilium.io/v2alpha1"
kind: CiliumPodNetwork
metadata:
  name: default
spec:
  ipam:
    mode: cluster-pool-v2beta2
    pool:
      name: ganymede
  routes:
  - dst: 0.0.0.0/0
    gw: 1.0.0.1
```

The only IPAM mode currently supported is `cluster-pool-v2beta2` (pooled IPAM).
Support for other IPAM modes might be added in future milestones.

Note: This CRD could be referenced by K8s `PodNetwork` CRD via ParamsRef for
Cilium-specific knobs for upstream compatibility

In addition, the following `CiliumPodNetwork` defines a secondary pod network
`jupiter`.  It allocates addresses from the IPAM pool `europa` to the pods'
secondary interface. It defines that traffic directed to the network 2.0.0.0/8
should be routed via the gateway 2.0.0.1.

```yaml
apiVersion: "cilium.io/v2alpha1"
kind: CiliumPodNetwork
metadata:
  name: jupiter
spec:
  ipam:
    mode: cluster-pool-v2beta2
    pool:
      name: europa
  routes:
  - dst: 2.0.0.0/8
    gw: 2.0.0.1
```

The IPAM pools are defined using the `CiliumIPPool` CRD as specified in
[CFP-2022-07-20: Pools for PodCIDR IPAM][1]:

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumIPPool
metadata:
  name: europa
spec:
  ipv4:
    cidrs:
      - 172.16.0.0/16
      - 172.17.0.0/16
    mask-size: 26
```

TODO: modifications to the `CiliumNode` CRD

TBD:
* Should `CiliumPodNetwork` reference `CiliumIPAMPool` or the other way around?

#### Annotations

To define which networks a pod should be attached to, the `cilium.io/networks`
annotation is used.

The following pod `juice` is annotated with the `cilium.io/networks` annotation
value `default, jupiter` defining that the pod should be attached to the
networks `default` and `jupiter`.

```yaml
kind: Pod
metadata:
  name: juice
  namespace: solar-system
  annotations:
    cilium.io/networks: default, jupiter
```

#### Config items

TBD:
* Do we want to gate this feature behind a ConfigMap flag?
* Do we need other configuration knobs?

### Architecture

One `CiliumEndpoint` is created per per network in each pod.

Consider the following pod `juice` which is attached to the `default` and the
`jupiter` networks:

```yaml
kind: Pod
metadata:
  name: juice
  namespace: solar-system
  annotations:
    cilium.io/networks: default, jupiter
status:
  podIP: 1.10.0.111
```

It's `podIP` on the cluster default network is 1.10.0.11. Note that its IP on
the secondary network is not reflected in the `Pod` resource but only in the
respective `CiliumEndpoint`. The following `CiliumEndpoint`s are created
mapping to pod `juice`:

```yaml
kind: CiliumEndpoint
metadata:
  name: juice
  namespace: solar-system
status:
  networking:
    addressing:
    - ipv4: 1.10.0.111
```

```yaml
kind: CiliumEndpoint
metadata:
  name: juice-jupiter
  namespace: solar-system
status:
  networking:
    addressing:
    - ipv4: 2.20.0.222
```

* TBD:
	* How to handle the `default` network? Is implicitly created by Cilium if the
		user doesn't define a `CiliumPodNetwork` for it?

### Routing

#### Egress from pod

For multi-homed pods, the CNI plugin will install an egress route for every
secondary interface. This ensures traffic to a secondary network leaves on the
secondary interface. The default route will route all other egress traffic to
the primary interface.

Once the traffic hits the host network namespace, egress packet routing is
defined by the rules and routes installed in the host network namespace (BPF
routing will respect those as well). We expect that there (again) will be a
route for every secondary network, steering the egress packet the secondary
native network interface (e.g. 2.2.0.0/16 dev eth2 via 2.2.0.1). The default
network will rely on the default interface.

#### Ingress to pod

Ingress to pod is expected to be handled either via endpoint routes (i.e. a
per-podIP route pointing directly to the veth pair of the pod, as we already
e.g. do in ENI mode) or via local node routes, where we will have a route for
each local (primary and secondary network) podCIDR to the `cilium_host` device.
This implies that the `cilium_host` device is shared for all networks.

Once the packet reaches the `cilium_host` device, the Cilium datapath will do an
endpoint lookup and steer the traffic to the network interface of the pod.
Since there is one endpoint per pod interface (i.e. multiple endpoints per pod
of multi-homed pods), the packet will arrive on the correct interface within
the pod based on the destination address of the packet.

### `cilium_host` IP

It seems that we will need to allocate a `cilium_host` IP for every
`CiliumPodNetwork` configured on the node. This is required for local-host to
remote-pod traffic.

Assume the local node is attached to two networks, i.e.  1.1.0.0/16 (on eth1)
and 2.2.0.0/16 (on eth2). If the local `cilium_host` IP is 1.1.100.1 and we
initiate traffic from the local host namespace to a remote pod 2.2.200.2, that
traffic leave the node on eth2. The reply packet however will arrive on
interface eth1, because the source IP of the packet was 1.1.100.1.  This is
effectively cross-network traffic. If we want host2pod traffic not cross
network boundaries, we need to have one `cilium_host` IP for each pod network.

### Masquerading

Will need to make sure traffic to the network's CIDRs are not NATed. Currently
implemented via `{ipv4,ipv6}-native-routing-cidr` for the default network, for
additional networks we will have to make use of the ip-masq-agent
https://docs.cilium.io/en/v1.13/network/concepts/masquerading/

MVP might require users to configure masquerading separately from
`CiliumPodNetwork` CRD.

## Impacts / Key Questions

_List crucial impacts and key questions. They likely require discussion and are required to understand the trade-offs of the CFP. During the lifecycle of a CFP, discussion on design aspects can be moved into this section. After reading through this section, it should be possible to understand any potentially negative or controversial impact of this CFP. It should also be possible to derive the key design questions: X vs Y._

### Impact: ... 1

_Describe crucial impacts and key questions that likely require discussion and debate._

### Key Question: ... 2

_Describe a key question_

### Option 1:

#### Pros

* ...

#### Cons

* ...

### Option 2:

#### Pros

* ...

#### Cons

* ...

## Future Milestones

* Support for IPAM modes other than `cluster-pool-v2beta2`.

[1]: https://docs.google.com/document/d/1FIyfhVegljG9anXpyA25gEYzuUGpfHo_MycZ-Wk0VeE
[2]: https://docs.google.com/document/d/17LhyXsEgjNQ0NWtvqvtgJwVqdJWreizsgAZH
