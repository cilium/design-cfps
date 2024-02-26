# CFP-26438: Network Policy Correlation

**SIG: SIG-Network**

**Begin Design Discussion:**

**Cilium Release:** 1.X

**Authors:** [Mark St. John](mailto:markstjohn@google.com)


## Summary

Correlate forwarded flows to the set of policies that allow that flow and expose the set of policies via hubble.


## Motivation

Improve observability of policy verdicts by mapping forwarded flows to configured policies. This will display matched policies for a flow, performing correlation on behalf of a user.

For multi-tenant clusters (isolated by namespace), this would additionally provide context for policy enforcement specified at the cluster level – i.e `CiliumClusterwideNetworkPolicy` (`CCNP`) and later `AdminNetworkPolicy` (`ANP`).


## Goals

* Correlate policies for flows with `FORWARDED` verdicts
* Correlate for node policies
* Expose correlated policies in Flow struct


## Non-Goals



* Correlate [deny policies](https://docs.cilium.io/en/latest/security/policy/language/#deny-policies) for flows with `DROPPED` verdicts


## Proposal

For policy verdict events, populate the (new)  `CorrelatedPolicies` field on the Flow struct:

For [Compact](https://pkg.go.dev/github.com/cilium/hubble/pkg/printer#Compact) output (default for `hubble observe`) a new flag (`--print-correlated-policies`) will be added to the observe subcommand to optionally include and format the correlated policy field in the output:


### Enablement

Configuring the correlation intent should be configurable to users. That way, users can control when correlation is performed.

The feature will be guarded per-agent via an enablement flag:

For CRD-based options, the CRD will always be installed by the operator. The flag will only be used by the agents.


### Option 1: Enable via cilium agent flag

Cilium agent flag `--enable-correlate-policies` will control whether to instantiate a policy correlator. This flag will be defined in the `cilium-config` map (effectively making it a cluster-scoped configuration)


#### Pros



* No additional watches
* Fewer dependencies


#### Cons



* Agent restart required to reconcile correlation intent.
* Not multi-tenant friendly
* Extending configurable behavior requires additional flags


### Option 2: Enable via cluster-scoped CRD

Cilium agent will watch a cluster-scoped singleton resource and dynamically update correlation intent. This is identical to the GKE approach for [network policy logging](https://cloud.google.com/kubernetes-engine/docs/how-to/network-policy-logging#configuring_network_policy_logging).


#### Pros



* No agent restart required to reconcile correlation intent
* Can install a default resource


#### Cons



* Creates an extra watch per agent
* Not multi-tenant friendly


### Option 3: Enable via namespaced CRD

Similar to Option 1, but more granular as the CRD is namespaced. One major difference is that no default policy will be installed per namespace.


#### Pros

* No agent restart required to reconcile correlation intent
* Multi-tenant operators can better control policy correlation for their namespace(s)


#### Cons

* More watch events per agent
* No default resource
    * this proposal does not specify a namespace reconciler to create the resource per-namespace


## Implementation

TODO(): document the GKE implementation to be used in upstream OSS.


## Invocation

Populating the flow with the policy verdict may be performed in one of several locations.


### Option 1: Perform during flow parsing (preferred)

Perform correlation for [MessageTypePolicyVerdict](https://sourcegraph.com/github.com/cilium/cilium@6d4b2f7/-/blob/pkg/hubble/parser/threefour/parser.go?L134-141) events during flow decoding:

[https://sourcegraph.com/github.com/cilium/cilium@6d4b2f7/-/blob/pkg/hubble/parser/threefour/parser.go?L134-141](https://sourcegraph.com/github.com/cilium/cilium@6d4b2f7/-/blob/pkg/hubble/parser/threefour/parser.go?L134-141)

```c++
case monitorAPI.MessageTypePolicyVerdict:
	pvn = &monitor.PolicyVerdictNotify{}
	if err := binary.Read(bytes.NewReader(data), byteorder.Native, pvn); err != nil {
		return fmt.Errorf("failed to parse policy verdict: %v", err)
	}
	eventSubType = pvn.SubType
	packetOffset = monitor.PolicyVerdictNotifyLen
	authType = flow.AuthType(pvn.GetAuthType())
	if correlator != nil {
		if err := correlator.Correlate(ctx, decoded); err != nil {
			return fmt.Errorf("failed to correlate policy: %v", err)
}
	}
```
Convention in the parser is to check for `nil` rather than check flag value, so reusing that pattern.

#### Pros

* Contained within the switch statement for the event type, so is only processed for events of the appropriate type
* Correlated policy result on the flow may be used by downstream `OnDecodedFlow` functions


#### Cons

* An additional check is performed per policy-verdict
* OnDecodedFlow functions cannot rely on correlation field data


### Option 2: Inject as OnDecodedFlow function

Perform correlation for flows as an observer [OnDecodedFlow](https://sourcegraph.com/github.com/cilium/cilium@6d4b2f7092d634f67c6192c85143051c49846245/-/blob/pkg/hubble/observer/observeroption/option.go) option:

[https://sourcegraph.com/github.com/cilium/cilium@6d4b2f7092d634f67c6192c85143051c49846245/-/blob/daemon/cmd/hubble.go?L124-130](https://sourcegraph.com/github.com/cilium/cilium@6d4b2f7092d634f67c6192c85143051c49846245/-/blob/daemon/cmd/hubble.go?L124-130)

This approach is less efficient when policy correlation is enabled as it evaluates all flows.

```c++
if option.Config.EnablePolicyCorrelation {
  	observerOpts = append(
observerOpts,
observeroption.WithOnDecodedFlowFunc(func(ctx context.Context, flow *flowpb.Flow) (bool, error) {
  			if flow.PolicyMatchType == 0 {
				return false, nil
			}
			return false, correlator.Correlate(ctx, decoded)
		}),
	)

}

```

#### Pros

* If disabled, this does not add any per-flow processing


#### Cons

* Function needs to evaluate all flows to filter for policy verdict events
* Other OnDecodedFlow functions cannot rely on correlation field data as it is dependent on the ordering


## Future Milestones


### Deny policy support

CNP and CCNP support [deny policies](http://CiliumClusterwideNetworkPolicy), and ANP will likely support a [deny action](https://github.com/kubernetes-sigs/network-policy-api/blob/master/apis/v1alpha1/adminnetworkpolicy_types.go#L182) as well. As this initial proposal does not seek to implement deny action support, this is being deferred to a future milestone.
