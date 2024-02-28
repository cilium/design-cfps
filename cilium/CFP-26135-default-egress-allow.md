# CFP-26135: Block Endpoints with default egress allow

**SIG: SIG-Policy**

**Begin Design Discussion:** 2023-06-12

**Cilium Release:** 1.15

**Authors:** Tamilmani Manoharan <tamanoha@microsoft.com>

## Summary

The current implementation of Cilium lacks a mechanism to restrict egress traffic to specific endpoints without blocking all traffic. When the egress policy is enabled for an endpoint, it effectively goes into a default deny state for egress traffic. This proposal talks about options for adding default egress allow while blocking egress to specific endpoints.


## Motivation

In cloud provider environments, it is often necessary to enforce firewall policies to restrict access to certain infrastructure components. Historically, before the emergence of eBPF, firewall rules were enforced using IPTables. Cilium primarily utilizes eBPF for policy enforcement. In eBPF host routing mode, Cilium bypasses IPtable rules and rules can be enforced only via eBPF. Also in certain scenarios (eg: L7), cilium prepends some IPTable rules by default which conflicts with cloud provider specific rules. Disabling the "prepend-iptables-chains" option is also not a viable solution as disabling prepend means Cilium can no longer guarantee that its policies are enforced.

This proposal aims to bring the management of cloud provider specific policies within Cilium. The primary goal of this proposal is to integrate the cloud provider-specific policies into the Cilium framework. By doing so, we aim to address potential conflicts and other concerns associated with managing these policies across multiple components.

## Goals

- Block egress traffic to certain endpoints without blocking all
- Behavior of existing network policy or cilium network policy should not be affected

## Non Goals
 - Ingress Restriction without blocking all
 - Applying this policy for specific endpoints
 - Complete Network Policy support with all features

## Proposal

### Overview
Allow cloud providers to block egress traffic to certain endpoints without blocking all. This proposal considers to solve this via bpf policy rules as it will work for both legacy host routing and ebpf host routing modes.

### Option 1 (CommandLine Flag)

Add a new command line flag, --egress-deny-tuples=[ip:port:protocol] as stringSlice. Cloud providers would customize this cilium config to specify apriori that traffic from Cilium-managed Pods towards specific address+port+protocol combinations should be blocked. This flag would effectively act independently of all of the NP, CNP, CCNP resources and enforce a deny for such tuples regardless of the user-defined policy. The deny tuples from cilium config are sanitized and converted to cilium egress deny policy rules. Then these rules should get added to policy map of all endpoints. 

#### Adding new field in ruleMetadata
Based on config received via command line, Cilium Agent would convert it to EgressDeny rule and does policyAdd. PolicyAdd will add the rules to repository and triggers policyreaction event to trigger identity allocation for labels specified in the policy. Cilium by default goes to deny mode if any policy added. To prevent this, this rule can be created with this flag `DefaultAllow` enabled. If a rule has this flag enabled, then cilium will never enforce default deny for the rest of traffic unless if there any egress k8s network policy or cilium network policy rule is enforced. By default, this flag is disabled and it would not regress existing network policy scenarios. 

`api.Rule` can extend `ruleMetadata` in `api:rule` structure to add a new field named `DefaultAllow`. This field is internal and not exposed to user. When evaluating matching policy rules for an endpoint, based on this flag cilium can differentiate this deny rule from other network policy rules. If all matching rules contains `DefaultAllow` to true then default allow will be enforced. If any one of matching rule contains `DefaultAllow` to false, then default deny will be enforced.

`getMatchingRules` function inaddition to ingressMatch and egressMatch, it would also return `adminEgressMatch`.  adminEgressMatch is enabled only if there `DefaultAllow` is set for the rule and there is no other egressMatch without defaultAllow set. 

`Resource` field in `policy.AddOptions` used to indicate the source of policy rule. A new entry of `ResourceKind` named `ResourceKindDaemonConfig` can be added to indicate the source of this rule is via cilium config.

#### Examples
##### Example 1

**Cloud Provider deny tuples**  
-–egress-deny-tuples=[“169.254.128.5;80;;]

**Network Policy/Cilium Network Policy**  
None

**Action**  
Egress - Default Allow for all. Block access to 169.254.128.5:80 for all  
Ingress - Default Allow for all

##### Example 2

**Cloud Provider deny tuples**  
-–egress-deny-tuples=[“169.254.128.5;80;;]

**Network Policy/Cilium Network Policy**  
```
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "l3-egress"
spec:
  endpointSelector:
    matchLabels:
      role: backend
  egress:
  - toEndpoints:
    - matchLabels:
        role: frontend
```
**Action**  
Egress - Allow for all except for pods with label “role: backend”. For those pods, egress allowed only to pods with “role: frontend”. Block access to 169.254.128.5:80 for all  
Ingress - Default Allow for all

### Alternate Options
#### Option 2 (CCNP CRD with new field)
Add a new field named `defaultAllowEgress` in the `ClusterwideCiliumNetworkPolicy` (CCNP) resource is required to differentiate cloud provider policy from other network policies. This field aims to enforce a default allow instead of the default deny for egress traffic within the cluster. By default, the defaultAllowEgress field is disabled, indicating that the default deny is in effect. The scope of the defaultAllowEgress field is at the policy level, meaning it applies to the entire policy, even if there are multiple rules defined within the same policy. Default deny for egress traffic will take precedence if egress enabled in at least one of Network Policy or Cilium Network Policy that does not have the defaultAllowEgress flag set. 
When creating or updating a policy, Cilium triggers the regeneration of the associated endpoint and its policies. It selects all rules that match the endpoint’s identity and in that process, it identifies if egress or ingress policy is enabled. It adds a default allow rule for egress traffic with identity 0 (allow all) when the egress policy is not enabled. It skips adding this rule when egress policy is enabled. With the proposed change, Cilium will add a new field named “DefaultAllowEgress” and will be set only if there exists CCNP with defaultAllowEgress set. If there are any other network policies or CCNP without that field, then it will be disabled. With this change, Cilium will add default allow rule for egress traffic if it satisfies any of these two conditions:

- The egress policy is not enabled for the endpoint.
- The egress policy is enabled and the DefaultAllowEgress field is set.

```
apiVersion: "cilium.io/v2"
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: "block-azure-wireserver"
spec:
  endpointSelector: {}
  egressDeny:
  - toCIDR:
    - 169.254.128.5
    toPorts:
    - ports:
      - port: "80"
        protocol: "TCP"
  defaultAllowEgress: true
```

Cloudprovider will create CiliumClusterwideNetworkPolicy with `defaultAllowEgress` which indicates block specified endpoint wihtout blocking all

**Action**  
Egress - Allow all except traffic destined to 169.254.128.5 and port 80
Ingress - Allow all

#### Option 3 (Separate Policy map and CRD)

A separate CRD and policy map will be created to handle policies which require restriction to certain endpoints without blocking all traffic. This policy map will be processed before the existing network policy map, allowing for more fine-grained control over policy enforcement. 

The new policy map will not have a default deny or allow behaviour, meaning that it will only process rules explicitly defined within it.  Within the BPF code responsible for enforcing egress policies, a step will be added to check the new policy map before processing the regular network policy map. If the rules in the new policy map are not matched, Cilium will fallback to processing the rules in the regular network policy map.

More similar to approach specified here: https://github.com/cilium/cilium/issues/7111#issuecomment-558864657 

#### Option 4 (AdminNetworkPolicy)

Kubernetes SIG-network is working on a new type of policy(AdminNetworkPolicy) that has the ability to express more complex policies, and this does not come with the default semantics that selected endpoints are isolated by default. Developers cannot override these rules by creating NetworkPolicies that applies to the same workloads as the AdminNetworkPolicy does.

##### Links
https://github.com/kubernetes/enhancements/issues/2091
https://github.com/kubernetes/enhancements/tree/master/keps/sig-network/2091-admin-network-policy 
https://github.com/cilium/cilium/issues/23380


 **Option 1 is preferred option for following reasons.**

- Doesn't change semantics of any existing CRDs
- Users or cluster admin cannot accidentaly override this via any network policy
- Simpler semantics and not exposed to users by design

#### FAQs

##### Why commandLine Flag Option not CRDs?
- The potential concern is that cluster admin is expected to have control over all `CiliumClusterwideNetworkPolicy` except for this cloud provider specific policy. This will complicate the RBAC model between cloud admin and cluster admin when managing these policies.
- Cilium uses same `api:Rule` for NetworkPolicy and CiliumNetworkPolicy CRDs as well. Since this feature is supported only with `CiliumClusterWideNetworkPolicy`, adding a new field might confuse customers when using other Network Policies.

##### Why not iptable rules instead of eBPF?
cilium adds a rule on top of the FORWARD chain to redirect all traffic to a CILIUM_FORWARD chain. So any Cloud Provider specific firewall rules placed after cilium rules doesn't get executed. Disabling the "prepend-iptables-chains" option is also not a viable solution as disabling prepend means Cilium can no longer guarantee that its policies are enforced. Also in eBPF host routing mode, Cilium bypasses IPtable rules and rules can be enforced only via bpf policies. 

## Testing

### Validation of deny tuples input
The tests will validates `--egress-deny-tuples` input and test if data is provided in right format which is <ip;port;protocol>. Not all fields are mandatory and test will valdiate for all possible valid inputs and return error if input data is invalid.

### Cloud Provider policy with no other network policies
The test deploys cloudprovider policy rule and validates if cloudprovider policy rule was added to repo and policy map state contains deny rules and default allow rule.

### Cloud Provider policy with Egress NetworkPolicy 
The test deploys cloudprovider policy rule and validates if cloudprovider policy rule was added to repo. It then adds egress rule and validates if policy map state contains deny rule as specified in cloudprovider rule and should not contain default allow rule for the same identity to which egress rule is enforced.

### Cloud Provider policy with Ingress CiliumNetworkPolicy 
The test deploys cloudprovider policy rule and validates if cloudprovider policy rule was added to repo. It then adds ingress rule and validates if policy map state contains deny rule as specified in cloudprovider rule and should contain default allow rule for the same identity to which ingress rule is enforced.

### Cilium connectivity test
The e2e functionality of cloudprovider policies can be validated with `cilium connectivity test`


## Impacts / Key Questions

### Cloud provider network policy not defined as k8s resource
Cloud provider configure deny tuples to restrict traffic to certain endpoints wihtout blocking all. Cilium Agent would create Egress Deny policy converting these tuples but it would not be a kubernetes resource as like other network policies. Listing network policy resources in kubernetes cluster would not show up this network policy.

### Key Question: Which option should be taken to implement the feature?

### Option 1 (CommandLine Flag)

#### Pros
 * Simpler semantics
 * No need to change any existing CRD specifications

#### Cons
 * Not all policies would be defined as k8s resources
 * Need to update cilium config and restart cilium agent to update/add/delete policies.

### Option 2 (CCNP with new field):

#### Pros
* Offer lot of features and aligns with other policies
* Easy to update, add or delete policies on fly without restarting cilium agent

#### Cons
* Updating existing CRD to add new field creates new variation to existing policy
* Deviate from the default deny behavior defined by the Kubernetes Network Policy standard
* Egress rules are not functional if customer creates CCNP with both egress rules (not egressDeny) and defaultEgressAllow set in the same policy. 
* Field exposed to user. Anyone can create CCNP with new field. There is no RBAC control over this new field to be configured only by cloud provider.

### Option 3 (Separate Policy Map and CRD)

#### Pros:
* No change to existing CRD

#### Cons:
* Considerable code change. Defining and processing new policy map touches multiple components
* Addition of new CRD and managing it
* Deviate from the default deny behavior defined by the Kubernetes Network Policy standard

### Option 4 (AdminNetworkPolicy)

#### Pros:
* Kubernetes community is looking towards this path

#### Cons:
* Complicate to implement
* Longer term

## Future Milestones

_List things that this CFP will enable but that are out of scope for now. This can help understand the greater impact of a proposal without requiring to extend the scope of a CFP unnecessarily._

### Deferred Milestone 1

_Description of deferred milestone_

### Deferred Milestone 2
