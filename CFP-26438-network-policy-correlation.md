# CFP-26438: Network Policy Correlation

**SIG: SIG-Network**

**Begin Design Discussion:**

**Cilium Release:** 1.15

**Authors:** [Mark St. John](mailto:markstjohn@google.com)

**Status:** Released Cilium 1.17

## Summary

Correlate forwarded flows to the set of policies that allow that flow and expose the set of policies via hubble.

## Motivation

Improve observability of policy verdicts by mapping forwarded flows to configured policies. This will display the
most specific matched policy for a flow, performing correlation on behalf of a user.

For multi-tenant clusters (isolated by namespace), this would additionally provide context for policy enforcement
specified at the cluster level – i.e `CiliumClusterwideNetworkPolicy` (`CCNP`).

## Goals

* Correlate most specific policy for a flow
* Correlate policy for flows with `FORWARDED` verdicts
* Correlate [deny policies](https://docs.cilium.io/en/latest/security/policy/language/#deny-policies) for flows with [DropReason_POLICY_DENY](https://github.com/cilium/cilium/blob/c6f3f6868ab494f34ab131facb8ec5e6e2f5b15b/api/v1/flow/flow.pb.go#L549)
flow verdict's DropReasonDesc
* Correlate for node policies (Cilium network policies using a `nodeSelector`)
* Expose correlated policy in Flow struct

## Non-Goals

* Correlate all policies matched by the flow.

## Proposal

For policy verdict events, populate one of the corresponding fields on the Flow struct:

* repeated [Policy](https://github.com/cilium/cilium/blob/c6f3f6868ab494f34ab131facb8ec5e6e2f5b15b/api/v1/flow/flow.proto#L462)
`EgressAllowedBy`: policies corresponding to allows egress traffic
* repeated Policy `EgressDeniedBy`: policies corresponding to denied egress traffic
* repeated Policy `IngressAllowedBy`: policies corresponding to allowed ingress traffic
* repeated Policy `IngressDeniedBy`: policies corresponding to allowed ingress traffic

Making the fields repeated allows for the flow to be correlated against multiple policies.

## Implementation

1. Extract the endpoint ID of the subject of the network policy rule
   * The source for egress traffic
   * The destination for ingress traffic

1. Using an [EndpointGetter](https://github.com/cilium/cilium/blob/51ca065a1187e1ec8bdf875dda7fa2d9b13580a2/pkg/hubble/parser/getters/getters.go#L27),
fetch the endpoint information using [GetEndpointInfoByID](https://github.com/cilium/cilium/blob/51ca065a1187e1ec8bdf875dda7fa2d9b13580a2/pkg/hubble/parser/getters/getters.go#L31)

1. Using the monitoring API [matchType](https://github.com/cilium/cilium/blob/c6f3f6868ab494f34ab131facb8ec5e6e2f5b15b/pkg/monitor/datapath_policy.go#L105),
use the most specific [Key](https://github.com/cilium/cilium/blob/51ca065a1187e1ec8bdf875dda7fa2d9b13580a2/pkg/policy/types/types.go#L20)
in [GetRealizedPolicyRuleLabelsForKey](https://github.com/cilium/cilium/blob/c6f3f6868ab494f34ab131facb8ec5e6e2f5b15b/pkg/endpoint/policy.go#L961)

1. Parse the labels on the derived label set to extract:

* Kind (e.g. `NetworkPolicy`, `CiliumNetworkPolicy`, etc.)
* Name
* [optional] Namespace (only present for namespaced resources)

## Invocation

Populating the flow with the policy verdict may be performed in one of several locations.

### Perform during decode

Perform correlation during events during L3/L4 flow parsers [Decode](https://github.com/cilium/cilium/blob/882d77d98d0fc1a4918e8575bbf069552849c4a0/pkg/hubble/parser/threefour/parser.go#L94):

```c++
func (p *Parser) Decode(data []byte, decoded *pb.Flow) error {
  ...
	if p.endpointGetter != nil {
		correlation.CorrelatePolicy(p.endpointGetter, decoded)
	}
  ...
```

#### Pros

* Correlated policy result on the flow may be used by downstream `OnDecodedFlow` functions

## Future Milestones

1. Flag-gate functionality to reduce resource utilization.
1. Correlate all policies matched by the flow.
1. Support for `AdminNetworkPolicy` (`ANP`).
1. For [Compact](https://pkg.go.dev/github.com/cilium/hubble/pkg/printer#Compact) output (default for `hubble observe`) add a new flag (`--print-correlated-policies`) to the observe subcommand to optionally include and format the corresponding policy field in the output.
