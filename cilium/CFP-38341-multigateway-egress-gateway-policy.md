# CFP-38341: Multigateway support for Egress Gateway

**SIG: SIG-DATAPATH**

**Begin Design Discussion:** 2025-03-26

**Cilium Release:** TBD

**Authors:** Carlos Abad <carlosab@google.com>

**Status:** Implementable

## Summary

This CFP describes a modification for the Egress Gateway to optionally support multiple 
gateways on the same Cilium Egress Gateway Policy. When this feature is used Cilium will 
automatically distribute the source endpoints between the multiple gateways. 
The distribution will be static, and Cilium will try to assign them in a way that will 
balance the number of endpoints per gateway on a best-effort basis.

## Motivation

Currently, if a user wants to split a group of endpoints between different egress IPs or gateways, 
they have to group them manually under separate labels, and then create separate Cilium Egress 
Gateway Policies for each label.

In that case the user is responsible for attaching the label for a specific Egress Gateway to each 
new endpoint, and for manually balancing the number of endpoints per Egress Gateway.

With this feature, Cilium will automatically distribute the endpoints among the gateways, which 
will make distributing endpoints across multiple egress IPs much easier to create and maintain.

## Goals

* Automatically distribute the source endpoints from a single Cilium Egress Gateway Policy among 
several gateways in said policy.
* Only assign the endpoints to gateways when the policy is added or modified, or new endpoints are 
added or removed.

## Non-Goals

* Reassign the endpoints already assigned to a gateway based on traffic or any other run-time 
metrics. 
* Minimize the number of reassignments when the list of egress gateways or endpoints change.
* Change the curent Egress Gateway datapath behaviour (reconnections, etc).

## Proposal

### Overview

From the API perspective, this proposal will extend the Cilium Egress Gateway Policy to 
include a new list of Egress Gateways.

From the control plane perspective (golang), it will update the code path that transforms 
Cilium Egress Gateway Policies to entries in the Egress Policy Map, so it distributes the 
endpoints covered by the policy among the multiple egress gateways in the policy.

From the data path side (ebpf), nothing will change.

### API changes

The 
[CiliumEgressGatewayPolicySpec field](https://github.com/cilium/cilium/blob/03a2d94b4ada323c639512a817a3e78fa5f2833d/pkg/k8s/apis/cilium.io/v2/cegp_types.go#L46C6-L46C35) 
from the CiliumEgressGatewayPolicy will be updated with a new field: `egressGateways`. 
This new field will contain a list of EgressGateway structs. Like in 
[similar changes in the Kubernetes API](https://github.com/kubernetes/api/blob/fc83166ea9db777b32244a3cacec783fa065ab50/core/v1/types.go#L6069), 
the first entry will contain the gateway node from the `egressGateway` field.

```
type CiliumEgressGatewayPolicySpec struct {
  // Egress represents a list of rules by which egress traffic is
  // filtered from the source pods.
  Selectors []EgressRule `json:"selectors"`

  // DestinationCIDRs is a list of destination CIDRs for destination IP addresses.
  // If a destination IP matches any one CIDR, it will be selected.
  DestinationCIDRs []IPv4CIDR `json:"destinationCIDRs"`

  // ExcludedCIDRs is a list of destination CIDRs that will be excluded
  // from the egress gateway redirection and SNAT logic.
  // Should be a subset of destinationCIDRs otherwise it will not have any
  // effect.
  //
  // +kubebuilder:validation:Optional
  ExcludedCIDRs []IPv4CIDR `json:"excludedCIDRs"`

  // EgressGateway is the gateway node responsible for SNATing traffic.
  EgressGateway *EgressGateway `json:"egressGateway"`

  // Optional list of gateway nodes responsible for SNATing traffic. The first entry will
  // contain the gateway node from the EgressGateway field.
  //
  // +kubebuilder:validation:Optional
  EgressGateways []EgressGateway `json:"egressGateways"`
}
```

### Control plane changes

First the [PolicyConfig struct](https://github.com/cilium/cilium/blob/396d6b0b187089121c16faaf10ea6fca4a677f2e/pkg/egressgateway/policy.go#L55) in the 
Egress Gateway will be updated to hold the list of gateways. 
[ParseCEGP()](https://github.com/cilium/cilium/blob/396d6b0b187089121c16faaf10ea6fca4a677f2e/pkg/egressgateway/policy.go#L222C6-L222C16) 
will also be upgraded to parse the new field into the struct.

Then [updateEgressRules()](https://github.com/cilium/cilium/blob/396d6b0b187089121c16faaf10ea6fca4a677f2e/pkg/egressgateway/manager.go#L579) 
will be upgraded to distribute the endpoint IPs across the gateways in the 
`PolicyConfig struct` and write the resulting assignments into the 
[policyMap](https://github.com/cilium/cilium/blob/396d6b0b187089121c16faaf10ea6fca4a677f2e/pkg/egressgateway/manager.go#L616C73-L616C82).

To consistently generate the same configuration regardless of the order in which the 
endpoint IPs are received, or the order in which the egress gateways are specified in the 
CEGP, both lists will be ordered lexicographically before a round-robin algorithm assigns 
the endpoints to each gateway. The algorithm will start by sorting the lists of endpoint 
IPs and gateway nodes lexicographically to ensure consistency across nodes. Then it will 
assign the first endpoint IP to the first gateway node. Then it will assign the next 
endpoint IP to the second gateway node, and so on. Everytime it reaches the end of the 
list of gateway nodes, it will restart from the beginning.
