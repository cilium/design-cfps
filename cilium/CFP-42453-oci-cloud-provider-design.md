# CFP-42453: OCI Cloud Provider Design

**SIG: SIG-COMMUNITY**

**Begin Design Discussion:** 2025-10-15

**Cilium Release:** X.XX

**Authors:** Trung Nguyen <trung.tn.nguyen@oracle.com>

**Status:** Draft

## Summary

This document provides details for integration with OCI Oracle Kubernetes Engine to implement a Cilium Direct Routing solution.

## Motivation

Many customers are requesting support for Cilium on OCI. This feature proposal will provide guidance on various implementation options.

## Goals

* Discuss possible solutions for integrating OCI with Cilium IPAM solutions for Direct Routing
* Determine short term solutions for providing an integration with Cilium
* Determine long term solutions for providing an integration with Cilium where users can specify which VNIC a pod can route out of (see examples below for details)

## Non-Goals

* Timelines of integrations

## Proposal

### Background

OCI Oracle Kubernetes Engine (OKE) provides controls for the user to determine how many VNICs should be attached to a node and which VNIC a pod should route out of.

*Note: The existing OCI OKE solution uses a pre-attach model, where the Cloud Controller Manager preallocates all VNICs/IPs and does not perform any detaches (detaches happen upon node termination), and then the CNI only sets up OS level network routing to the pods*

#### Example 1

A node has 1 VNIC with 256 IPs attached to it. Host and Pod traffic will route out of this VNIC.

```
+--------------------------------------+
|         Kubernetes Node              |
|                                      |
|   +--------+                         |
|   | VNIC 1 |                         |
|   +--------+                         |
|       |                              |
|   +--------+                         |
|   |  Pod A |                         |
|   +--------+                         |
|                                      |
|  Pod A routes traffic via VNIC1      |
+--------------------------------------+
```

#### Example 2

A node has 3 VNIC. Each VNIC has 256 IPs attached. Host traffic routes out of the Primary VNIC (via the default route). And a user can use multus to specify that Pod B can route out of VNIC 2 or VNIC 3 depending on host routing rules.

```
+--------------------------------------------------+
|               Kubernetes Node                    |
|                                                  |
|   +--------+   +--------+   +--------+           |
|   | VNIC 1 |   | VNIC 2 |   | VNIC 3 |           |
|   +--------+   +--------+   +--------+           |
|       |            |            |                |
|       |        +------+         |                |
|       |        |Pod B |----------                |
| (Node Traffic) +------+                          |
|                                                  |
| - Pod B can route traffic via VNIC 2 or VNIC 3   |
| - Node-level traffic exits via VNIC 1            |
+--------------------------------------------------+
```

### Overview

Cilium documents several existing [IPAM solutions](https://docs.cilium.io/en/stable/network/kubernetes/ipam/):

* Out-of-tree solution that attaches an IP CIDR block and sets `v1.node.spec.podCIDR`. Cilium's "kubernetes" IPAM configuration knows how to process `v1.node.spec.podCIDR`
* In-tree solution that extends the Cilium controller to make OCI calls to attach VNICs/IPs and populate IPAM
* Out-of-tree solution that populates the Cilium IPAM Custom Resource
* Out-of-tree Delegated IPAM binary that is compatible with Cilium

#### Kubernetes IPAM solution

There will be a component outside the scope of Cilium (e.g. a leader elected component like Cloud Controller Manager or a per-node daemonset) that attaches a CIDR block to the Primary VNIC of the node and populates the `v1.Node.spec.podCIDR` field.

Cilium has a built-in `kubernetes` IPAM solution, which provides a simple, cloud agnostic solution (implementation-wise, the solution just populates the v1.Node object, so it does not make any assumptions about what CNI is being used).

However, only 1 CIDR block is attachable (`v1.Node.spec.podCIDRs` does not allow you to attach multiple blocks, except to have one IPv4 block and IPv6 block). This solution will not be compatible with requirements for using multiple VNICs (OCI has a basic (non-Cilium) offering that supports separating node traffic from pod traffic onto separate VNICs).

Pros:
- Very Simple
- All Cloud Provider changes are CNI Agnostic

Cons:
- Only one CIDR block can be attached
- Will not fit the multi-VNIC model

#### In-tree Extending the Cilium Operator

The CiliumNode CRD will be updated to have Oracle related fields and the Cilium Operator will be extended to have access to the OCI-Go-SDK. It will perform the VNIC/IP attaches and the CNI IPAM will be updated to know how to route out of specific VNICs.

Pros:
- can support a multi-VNIC model
- requires change in a single component (Cilium Operator)

Cons:
- requires changes/coordination between Cloud Provider and Cilium

#### Out-of-tree solution to populate Cilium IPAM Custom Resource

*Note: This is similar to the previous solution. However, since OKE already has a process to attach VNICs/IPs, we can use this existing functionality.*

The Cilium Agent will generate the Custom Resource object (via `--auto-create-cilium-node-resource`), and a component outside the scope of Cilium (e.g. the Cloud Controller Manager) will attach IP Addresses and populate the Custom Resource objects.

*Note: This solution will require a CNI change to add the option to choose which VNIC to route out of*

Pros:
- with CNI changes, this solution can support a multi-VNIC model

Cons:
- requires changes in two separate components (Cloud Controller Manager and CNI)

#### Out-of-tree Delegated IPAM binary

OCI OKE already has a built-in process (in the Cloud Controller Manager) to attach the appropriate VNICs/IPs. OCI OKE's existing CNI IPAM plugin has the capability to choose a VNIC to route out of and would be modified to become compatible with Cilium as a delegated IPAM plugin.

Pros:
- can support a multi-VNIC model
- requires change in a single component (CNI binary)

Cons:
- requires changes/coordination between Cloud Provider and Cilium to test IPAM

## Impacts / Key Questions

_List crucial impacts and key questions. They likely require discussion and are required to understand the trade-offs of the CFP. During the lifecycle of a CFP, discussion on design aspects can be moved into this section. After reading through this section, it should be possible to understand any potentially negative or controversial impact of this CFP. It should also be possible to derive the key design questions: X vs Y._

### Impact: Integration with OCI

The Kubernetes IPAM solution requires the least effort to integrate with Cilium while also providing a Kubernetes Agnostic solution. All other solutions require additional maintenance or coordination to implement.

### Key Question: Cilium Testing

> How does Cilium perform CNI testing? What solution will require the least amount of maintenance/coordination?

Cloud providers are responsible for their own solutions.

### Key Question: Cloud Provider Support Model

> What does the support model look like for Cloud Providers for integrated pieces, like the Cilium Operator?

The team suggests not implementing in-tree solutions into the Cilium Operator because of the required coordination.

### Key Question: Multi-VNIC Solution

> Is there any negative impact from implementing a Multi-VNIC solution?

Cilium does not have strong support for Multi-VNIC solutions right now.

## Future Milestones

## Single-VNIC Solution

As a first pass, OCI will implement the Kubernetes IPAM solution.

Ultimately, the simplicity of the solution along with the CNI agnostic behavior made this the most attractive short term solution.

### Multi-VNIC Solution

Long term, OKE will reevalute multi-NIC solutions, where users can route pods out of VNICs, when Dynamic Resource Allocation (DRA) provides better support.

[DRANet](https://github.com/google/dranet) is a possible future solution (it would need to incorporate [consumable capacity](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/#consumable-capacity)).

[Node Resource Interface](https://github.com/containerd/nri) is another possible alternative.