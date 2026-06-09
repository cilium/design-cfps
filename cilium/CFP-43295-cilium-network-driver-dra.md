# CFP-43295: Cilium Network (DRA) Driver

**SIG: SIG-Agent, SIG-Datapath**

**Begin Design Discussion:** 2025-10-17

**Cilium Release:** TBD

**Authors:** bernardo <bersoare@isovalent.com>, Fabio <fabio.falzoi@isovalent.com>

**Status:** Draft

## Summary

This CFP proposes extending cilium to support allocating dedicated (NIC)
and shared (SRIOV) network devices to workloads - leveraging the Kubernetes DRA 
framework. 

References:

[Dynamic Resource Allocation](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/)

[DRA Plugin Interface definition](https://github.com/kubernetes/kubernetes/blob/d777de7741d36d1cc465162d94f39200e299070b/staging/src/k8s.io/dynamic-resource-allocation/kubeletplugin/draplugin.go#L53-L144)

## Motivation

Abstractions do not come for free. The traditional Kubernetes networking 
model relies on a set of abstractions with the purpose of simplifying 
connectivity between applications and the rest of the world (including other workloads).
Under the hood, Cilium CNI attaches a pod to a logical fabric over which workloads
can benefit from secure network access to other workloads, services and clients. 
These attachments take the form of virtual devices (veth/netkit) that hand packets to
a custom forwarding plane (ebpf) that magically achieves connectivity between the 
endpoints. Most of this magic happens in software with varying levels of involvement
of the kernel. 
This is suitable for the vast majority of the use cases, but performance/latency sensitive 
workloads could benefit even further from having access to some underlying network hardware
(a NIC, so to speak) and shorten even further the path a packet takes between the application
and the network device itself. In such scenarios, performance (HW access) is more important 
than sophisticated connectivity and feature set. Some use cases that come to mind are network 
function (CNF/VNF) workloads, low latency data ingestion, and dpdk based applications combined with SR-IOV.

## Goals

- Cilium to be able to recognize and publish network devices through
    the Kubernetes DRA plugin API, in a way that Pods that require a given network 
    device can get scheduled on an appropriate node.
- Cilium to be able to preconfigure such devices
- Support SR-IOV VF assignment to pods

## Non-Goals

- Integrate network device (allocated to pods) connectivity with the cilium fabric. 
    It is assumed that these devices will be treated as “secondary” 
    networks and no bpf processing occurs on them.

## Proposal

### Overview

Extends the agent to register itself with the Kubernetes API as a 
DRA plugin (for publishing resources), and also with the container runtime (for configuring devices).
The Cilium Network Driver publishes, for each node, the local resources (ex: network devices) 
that match a given set of filters grouped in resource pools, allowing workloads to claim such resources. 

Upon receiving a claim request, Cilium Network Driver performs any preparation needed for the allocated device(s).
When the Pod finally starts, Cilium Network Driver performs any final configuration before assigning
the device to the Pod sandbox.

A simplified workflow can be seen below:

![Sequence](./images/CFP-43295.png)

### Solution

The Network Driver functionality in the Cilium agent is an opt-in feature.
In order to enable it, use the `networkDriver.enabled` helm flag. 
Once enabled, each node that detects a configuration assigned to it
becomes a Cilium Network Driver agent.

#### Deploying the configuration

A resource of type `CiliumNetworkDriverClusterConfig` is introduced,
and the presence of an object of this type in the Kubernetes API 
causes the cilium-operator pod to apply the desired configuration
to the cilium-agent pods for which a given configuration is suitable.

Since the Network Driver configuration for each agent
depends on node specific hardware characteristics, 
it is possible to specify `nodeSelector` to match a given 
`CiliumNetworkDriverClusterConfig` to a node or subset of nodes
by matching on the node labels.
Alternatively, a configuration with an empty `nodeSelector` matches
all nodes.

Since only one configuration is used at a time for
any given node, the cilium-operator is responsible for choosing
the right configuration for each node, and once it selects
a suitable configuration among the candidates, an object of type
`CiliumNetworkDriverNodeConfig` is created for each node, containing
the configuration specifics for the node.
If the operator detecs that more than one configuration suits a node
based on the node labels, it reports a conflict back to the 
status field of the offending `CiliumNetworkDriverClusterConfig` object
as described in the [`Conflict scenarios`](./CFP-43295-cilium-network-driver-dra.md#conflict-scenarios) section,
The example below illustrates how a cluster configuration looks like:

```
---
apiVersion: cilium.io/v1
kind: CiliumNetworkDriverClusterConfig
metadata:
  name: sriov-nodes-cluster-config
spec:
  nodeSelector:
      matchLabels: 
        sriov: true
  spec:
    driverName: "sriov.cilium.k8s.io"
    deviceManagerConfigs:
        sriov:
          ifaces:
            - ifName: enp2s0f0np0
              vfCount: 6
            - ifName: enp2s0f1np1
              vfCount: 6
```

Which in turn, will lead to the creation of `CiliumNetworkDriverNodeConfig`
objects for each of the nodes, for example:

```
---
apiVersion: cilium.io/v1
kind: CiliumNetworkDriverNodeConfig
metadata:
  name: mynode-hostname
spec:
  driverName: "sriov.cilium.k8s.io"
  deviceManagerConfigs:
      sriov:
        ifaces:
          - ifName: enp2s0f0np0
            vfCount: 6
          - ifName: enp2s0f1np1
            vfCount: 6
```

##### Conflict scenarios

The dynamic aspect of the nodeLabel selector mechanism to match configuration
to nodes might lead to cases where conflicts may arise. A conflict happens
when more than exactly one `CiliumNetworkDriverClusterConfig` object matches
a given node based on the node's labels. The operator handles conflicts
with the ultimate goal of preventing unwanted configuration updates or changes
to the agents.
This means that the operator needs to be aware of any previous decision and persist
that decision across reloads/reboots/restarts with a deterministic outcome.

#### Agent configuration

The feature introduces the concept of a Device Manager; which is the device
specific logic required to discover, select, publish and set-up devices using
the DRA and NRI frameworks. In other words, it provides extensibility by
abstracting the device specific logic from the assignment lifecycle.
Since our first use case is for assigning SR-IOV VFs to pods,
that one is the one focused throughout this document unless stated otherwise.

Under the deviceManagerConfigs section, an operator is able to control how a specific device manager is set up. 
In this context, the device manager is an abstraction of a certain type of resource. In the example below, 
you can see that we’re working with the `sriov` devicemanager - implying that we must account for extending 
the feature set by introducing device managers.

Updating the configuration at runtime is possible for the most part. Some settings, for example,
the number of sr-iov VF to be spawned, might need a full device restart - causing existing
traffic to be affected. In such cases, we ideally want to avoid applying the configuration right away.
It is up to each configuration feature to know what is safe to be changed or not, and future
contributors should be mindful of that.
On the other hand, it should be straightforward to add or remove pools and change the
grouping criteria, and these kind of changes should be allowed.
Cilium agent listens for configuration updates, and if there is the need of
any changes considered impactful, it will log an error message to signal that
an action may need to be taken.

#### Resource pools

The resource pools to be advertised by the DRA component on the driver are explicitly set by configuration. 
The Network Driver receives a structured configuration that contains parameters to match devices and group them together or apart
in ResourceSlice pools. This gives flexibility to the operator in respect to how the nodes expose their resources.
An example configuration structure with pools defined can be seen below:

```
---
apiVersion: cilium.io/v1
kind: CiliumNetworkDriverClusterConfig
metadata:
  name: cilium-network-driver-cluster-config
spec:
  spec:
    driverName: "sriov.cilium.k8s.io"
    deviceManagerConfigs:
        sriov:
          ifaces:
            - ifName: enp2s0f0np0
              vfCount: 6
            - ifName: enp2s0f1np1
              vfCount: 6
    pools:
      - name: a-side
        filter:
          pfNames:
            - enp2s0f0np0
      - name: b-side
        filter:
          pfNames:
            - enp2s0f1np1
```

With these filters, all the SR-IOV VFs whose PF kernel ifname matches `enp2s0f0np0` will be assigned
to `a-side` pool, whereas all the VFs under the PF named `enp2s0f1np1` are advertised as part of `b-side` pool.

#### Publishing and claiming resources

The Agent’s Network Driver then publishes a ResourceSlice pool named after Name, containing all the local 
devices that fulfill all the Filter conditions. Multiple pools can be defined.
Here is an example of a device member of resource pool of name `a-side` advertised as a ResourceSlice:

```
devices:
- attributes:
    deviceID:
      string: "0x1016"
    driver:
      string: mlx5_core
    kernelIfName:
      string: enp2s0f0v0
    pfName:
      string: enp2s0f0np0
    pool:
      string: a-side
    vendor:
      string: "0x15b3"
  name: enp2s0f0v0
driver: sriov.cilium.k8s.io
nodeName: c3-small-x86-01-bernardo
pool:
  generation: 1
  name: a-side
  resourceSliceCount: 1
```

Note: the set of attributes above are illustrative and may change in the upcoming iterations.

Devices can be assigned to Pods by creating pods with a ResourceClaim statement in the pod manifest.
The ResourceClaim object can be seen as the set of resources a Pod needs - influencing the Kubernetes 
scheduler decision on which node to place the pod. Only nodes that fulfill the claim requirements are eligible for scheduling.
A ResourceClaim references a DeviceClass which does the actual device matching and filtering based 
on the device attributes (advertised in the ResourceSlice) - and an operator can request multiple resources in a single 
claim by referencing multiple DeviceClass objects. The example below show how we can have device classes matching on 
certain attributes we published with our devices:

```
---
apiVersion: resource.k8s.io/v1
kind: DeviceClass
metadata:
  name: a-side.sriov.cilium.k8s.io
  namespace: kube-system
spec:
  selectors:
  - cel:
      expression: device.driver == "sriov.cilium.k8s.io" && device.attributes["sriov.cilium.k8s.io"].pool == "a-side"

---
apiVersion: resource.k8s.io/v1
kind: DeviceClass
metadata:
  name: b-side.sriov.cilium.k8s.io
  namespace: kube-system
spec:
  selectors:
  - cel:
      expression: device.driver == "sriov.cilium.k8s.io" && device.attributes["sriov.cilium.k8s.io"].pool == "b-side"
```

A ResourceClaimTemplate can be used in case several pods are expected to request similar assignments. 
The benefit here is to avoid duplicating similar requests across many Pod manifests.
An example template can be seen below - requests a pair of devices on two different pools 
(published as an attribute by the Cilium Network Driver DRA plugin, using the DeviceClass from the example above):

```
---
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: sriov
spec:
  spec:
    devices:
      requests:
      - name: a-side
        exactly:
          deviceClassName: a-side.sriov.cilium.k8s.io
      - name: b-side
        exactly:
          deviceClassName: b-side.sriov.cilium.k8s.io
```

Once a device is assigned, the Pod might require specific configuration in it. In the case of network devices, 
this configuration usually contains IP (v4/v6) addresses, routes and VLANs. These parameters are passed by the 
ResourceClaimTemplate definition as opaque configs for the request. Note that there is no parameter for passing
IP addresses directly, and an IPAM pool must be referenced if an address is to be allocated.
Here is a modified version the claim above to pass additional parameters to the Network Driver in the claim request:

```
---
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: sriov
spec:
  spec:
    devices:
      config:
        - requests:
            - a-side
          opaque:
            driver: sriov.cilium.k8s.io
            parameters:
              vlanID: 123
        - requests:
            - b-side
          opaque:
            driver: sriov.cilium.k8s.io
            parameters:
              vlan: 321
      requests:
      - name: a-side
        exactly:
          deviceClassName: a-side.sriov.cilium.k8s.io
      - name: b-side
        exactly:
          deviceClassName: b-side.sriov.cilium.k8s.io
```

When processing a PrepareResourceClaim request, the agent performs all the necessary operations on
the device and stores any information that will be needed when the pod finally starts and the device is ready 
to be configured in the pod sandbox.

Preparation steps may include reconfigure the interface mac address, adjust interface / device flags,
associate a SR-IOV device with a VLAN, among others. When the pod finally starts, the container runtime hook is called and it 
executes the last steps in the configuration within the pod namespace: configures the addresses on the interface,
bring it up, add routes to the routing table, for example.

#### CiliumResourceNetworkConfig

Alternatively, for the less trivial deployments, a requested device on a ResourceClaim may reference a CiliumResourceNetworkConfig object.
This method is suited for environments where node specific variables (such as VLAN IDs, routes, and IPAM details) are required.
`CiliumResourceNetworkConfig` allows for using a named object that abstracts away these details with the goal of making the 
`ResourceClaim` objects more generic and impose less constraints in regards to the underlying network.
The example below shows a `ResourceClaimTemplate` object that references a `CiliumResourceNetworkConfig` object - and the attributes
used to set up the allocation differ between the nodes.

```
---
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  ...
  resourceClaims:
  - name: attach-network-a
    resourceClaimTemplateName: attach-network-a
---
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: attach-network-a
spec:
  spec:
    devices:
      config:
        - requests:
            - a-side
          opaque:
            driver: sriov.cilium.k8s.io
            parameters:
              networkConfig: network-config-a
      requests:
      - name: a-side
        exactly:
          deviceClassName: a-side.sriov.cilium.k8s.io
---
apiVersion: cilium.io/v2alpha1
kind: CiliumResourceNetworkConfig
metadata:
  name: network-config-a
spec:
  - nodeSelector:
      matchLabels:
        kubernetes.io/hostname: kind-worker
    ipv4:
      address: 10.10.0.10/24
      staticRoutes:
        - destination: 10.0.0.0/8
          gateway: 10.10.0.1
  - nodeSelector:
      matchLabels:
        kubernetes.io/hostname: kind-control-plane
    ipv4:
      adress: 10.20.0.10/24
      staticRoutes:
        - destination: 10.0.0.0/8
          gateway: 10.20.0.1
```

The configuration set above results in the Pod being able to be scheduled on both kind-worker and
kind-control-plane. Depending on the node where it gets scheduled, it receives a certain ip
and with certain routes.

Note: in the example, we use static addresses for illustration purposes only. The long term plan is
to be able to reference an IPAM pool - as static addresses brings an increase in the
management overhead.

### Restrictions 
Requires Kubernetes v1.34

