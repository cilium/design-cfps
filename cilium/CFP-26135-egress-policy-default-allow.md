# CFP-26135: Egress policy default allow


## Meta

**SIG: SIG-Policy**

**Sharing: Public**

**Begin Design Discussion: 2023-06-12**

**End Design Discussion:**

**Cilium Release: 1.15**

**Authors:** tamanoha@microsoft.com

**Issue:** [https://github.com/cilium/cilium/issues/26135](https://github.com/cilium/cilium/issues/26135)


## Summary

The current implementation of Cilium lacks a mechanism to restrict egress traffic to specific endpoints without blocking all traffic. When the egress policy is enabled for an endpoint, it effectively goes into a default deny state for egress traffic. Even if an "allow-all" policy is enforced to counteract this behavior, it can introduce conflicts when other network policies are defined with restricted access. This proposal aims to solve this issue and enhance cilium functionality.


## Motivation

In cloud provider environments, it is often necessary to enforce firewall policies to restrict access to certain infrastructure components. Historically, before the emergence of eBPF, firewall rules were enforced using IPTables. Cilium primarily utilizes eBPF for policy enforcement. In eBPF host routing mode, Cilium bypasses IPtable rules and rules can be enforced only via eBPF. Also in certain scenarios (eg: L7), cilium prepends some IPTable rules by default which conflicts with cloud provider specific rules . Disabling the "prepend-iptables-chains" option is also not a viable solution as disabling prepend means Cilium can no longer guarantee that its policies are enforced.

This proposal aims to bring the management of cloud provider specific policies within Cilium. The primary goal of this proposal is to integrate the cloud provider-specific policies into the Cilium framework. By doing so, we aim to address potential conflicts and other concerns associated with managing these policies across multiple components.


## Goals

* _Block egress traffic to certain endpoints without blocking all_
* _Behavior of existing network policy or cilium network policy should not be affected_


## Proposal


### Overview**

The proposal aims to address the issue by configuring a default allow behavior for this special egress deny traffic rule. It also ensures that the behavior of existing network policies or Cilium network policies is not regressed.


### Proposal Content


#### CommandLine Flag

The decision to pass input as command line was taken based on what is more appropriate for this scenario. There is a section below that describes the reason for going with this option.

A new command line flag, `--deny-tuples=[ip:port:protocol]` could be added. Cloud providers would customize this cilium config to specify apriori that traffic from Cilium-managed Pods towards specific address+port+protocol combinations should be blocked. This flag would effectively act independently of all of the NP, CNP, CCNP resources and enforce a deny for such tuples regardless of the user-defined policy. These deny rules can be plumbed either via iptables or bpf policy rules. IPtables rules would not work with ebpf host routing mode. This proposal considers to solve this via bpf policy rules as it will work for both legacy host routing and ebpf host routing modes.

The deny rules from cilium config are sanitized and converted to cilium egress deny policy rules. Then these rules should get added to policy map of all endpoints. Several options are considered to implement this and the following section would talk about each option

Pros:

* Simpler semantics
* No need to change the existing CRD specifications

Cons:

* Not as generic as the alternate proposals
* Not all policies would be defined in k8s resources - more difficult to debug.
* Need to update cilium config and restart cilium to update/add/delete policies.

Option 3 is preferred as it aligns with existing code flow and does not affect existing network policies schema or behavior in any way.


#### Option 1

Cilium Agent would create EgressDeny rule  and does policyAdd. PolicyAdd will add the rules to repository and triggers policyreaction event to trigger identity allocation for labels specified in the policy. Cilium by default goes to deny mode if any policy added. To prevent this, this rule can be created with a specific label. If a rule contains this specific label, then cilium can set a field say “adminEgressEnabled” which never force default deny for the rest of traffic. Default deny will be enforced if any of network policy specified irrespective of adminEgressEnabled field set or not. It would not regress any existing scenarios and conforms to the Examples section of 1st proposal.


```
Labels labels.LabelArray `json:"labels,omitempty"`

type Label struct {
    Key   string `json:"key"`
    Value string `json:"value,omitempty"`
    // Source can be one of the above values (e.g.: LabelSourceContainer).
    //
    // +kubebuilder:validation:Optional
    Source string `json:"source"`
}
```

In above, Source can be set to something unique “adminEgressEnabled” and can use that to detect if rule added via regular network policy or via this command line flag.

Cons:

* These labels are exposed to users and user can also create CiliumNetworkPolicy with same label and it would cause confusion


#### Option 2

Not to add egress deny rules to policy repo instead synthesize a new deny rule and add it to matching rules list when policy rules are applied to an endpoint, ie before calling `matchingRuleList`.`resolveL4EgressPolicy()`. Would need to exclude the host endpoint from this logic, otherwise applications local to the node would also be restricted from accessing the peers defined by the flag. Identity allocation for cidr range has to be triggered manually.

Pros:

* No need for the special logic to tweak default deny behavior

Cons:

* Identity allocation of peer cidr range is not triggered automatically and require a manual trigger for each cidr range in those egress deny rules.
* Not align with other network policy rules by not adding to policy repo.


#### Option 3

This option is similar to option 1 except that instead of creating unique label in api.Rule can extend ruleMetadata in api:rule structure to add a new field named source. When evaluating matching policy rules for an endpoint, can differentiate cloud provider policy rules from other based on source field. If matching rules contains source only from “CloudProvider” then default allow will be enforced. If there exists alteast one matching rule with source different from “CloudProvider” then default deny will be enforced.

This new field “Source” can be initialized by extending `AddListLocked(rules api.Rules)`

Api through `policyAdd` call . Once egress deny rules are created based on config input, cilium will trigger policyAdd for these rules to add it to repo and to trigger identity allocation for cidr specified in those rules. policyAdd api contains `AddOptions`

as a parameter in addition to api.Rules.  The Source field in AddOptions can be leveraged and set to “`CloudProvider`” if EgressDeny rule is created via commandLine option. For rules created via NP or CNP, the source is set to “`CustomResource`“. This option is not exposed to user and user don’t have control over it.

```
type rule struct {
    api.Rule
    metadata *ruleMetadata
}

type ruleMetadata struct {
    Mutex lock.Mutex
    Source source.Source
    IdentitySelected map[identity.NumericIdentity]bool
}

// AddOptions are options which can be passed to PolicyAdd
type AddOptions struct {
    Replace bool
    ReplaceWithLabels labels.LabelArray
    Generated bool
    // The source of this policy, one of api, fqdn or k8s
    Source source.Source
    ProcessingStartTime time.Time
    Resource ipcacheTypes.ResourceID
}

CloudProvider Source = "CloudProvider"
```

Pros

* Doesn’t change existing api.Rule structure or interpret based on label values
* Exposing this field in ruleMetadata makes the user doesn’t have control over this field.
* Align with flow of other network policy rules by adding to policy repository. No need to trigger idenitty allocation for cidr manually.


#### Examples

A few examples on what the actions will be with network policies defined along with this commandLine option.

<table>
  <tr>
   <td><strong>CloudProviderRule</strong>
   </td>
   <td><strong>Other Network policies</strong>
   </td>
   <td><strong>Action</strong>
   </td>
  </tr>
  <tr>
   <td>–deny-tuples=[“169.254.128.5;80;;]
   </td>
   <td>
   </td>
   <td>Egress - Default Allow for all. Block access to 169.254.128.5:80 for all
<p>
Ingress - Default Allow for all
   </td>
  </tr>
  <tr>
   <td>–deny-tuples=[“169.254.128.5;80;tcp]
   </td>
   <td>apiVersion: "cilium.io/v2"
<p>
kind: CiliumNetworkPolicy
<p>
metadata:
<p>
  name: "l3-ingress"
<p>
spec:
<p>
  endpointSelector:
<p>
    matchLabels:
<p>
      role: backend
<p>
  ingress:
<p>
  - fromEndpoints:
<p>
    - matchLabels:
<p>
        role: frontend
   </td>
   <td>Egress - Default Allow for all. Block access to 169.254.128.5:80 TCP for all
<p>
Ingress - Default Allow for all except for pods matching “role: backend” which allow only from pods having label “role: frontend”
   </td>
  </tr>
  <tr>
   <td>–deny-tuples=[“169.254.128.5;80;tcp]
   </td>
   <td>apiVersion: "cilium.io/v2"
<p>
kind: CiliumNetworkPolicy
<p>
metadata:
<p>
  name: "l3-egress"
<p>
spec:
<p>
  endpointSelector:
<p>
    matchLabels:
<p>
      role: backend
<p>
  egress:
<p>
  - toEndpoints:
<p>
    - matchLabels:
<p>
        role: frontend
   </td>
   <td>Egress - Allow for all except for pods with label “role: backend”. For those pods, egress allowed only to pods with “role: frontend”. Block access to 169.254.128.5:80 for all
<p>
Ingress - Default Allow for all
   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>apiVersion: "cilium.io/v2"
<p>
kind: CiliumNetworkPolicy
<p>
metadata:
<p>
  name: "l3-egress"
<p>
spec:
<p>
  endpointSelector:
<p>
    matchLabels:
<p>
      role: backend
<p>
  egress:
<p>
  - toEndpoints:
<p>
    - matchLabels:
<p>
        role: frontend
   </td>
   <td>Egress - Allow for all except for pods with label “role: backend”. For those pods, egress allowed only to pods with “role: frontend”.
<p>
Ingress - Default Allow for all
   </td>
  </tr>
</table>

#### Why commandLine Flag Option not CRDs?

Two approaches are being considered to pass input for CloudProvider policies: Custom Resource Definitions (CRD) and Cilium configuration (command-line).

Passing input via CRD: This approach requires addition of new field in api:Rule structure to differentiate cloud provider policy from other network policies. Cilium uses same structure for all Network Policies. However, a potential concern with this method is that using the same structure for all network policies might confuse customers (as it’s not supported with CNP or NP) and complicate the RBAC model when managing these policies.

Passing input via Cilium configuration (command-line): Cloud provider policies doesn’t require all features present in other network policies. Additionally, since these policies are not expected to undergo frequent updates, handling them through the Cilium configuration offers a reasonable and manageable approach.


## Other Proposals

### CCNP CRD with new field

The proposal suggests introducing a new field called "defaultAllowEgress" in the ClusterwideCiliumNetworkPolicy (CCNP). This field aims to enforce a default allow instead of the default deny for egress traffic within the cluster. By default, the defaultAllowEgress field is disabled, indicating that the default deny is in effect. The scope of the defaultAllowEgress field is at the policy level, meaning it applies to the entire policy, even if there are multiple rules defined within the same policy. Default deny for egress traffic will take precedence if egress enabled in atleast one of Network Policy or Cilium Network Policy or CCNP that does not have the defaultAllowEgress flag set.

When creating or updating a policy, Cilium triggers the regeneration of the associated endpoint and its policies. It selects all rules that match the endpoint’s identity and in that process, it identifies if egress or ingress policy is enabled. It adds a default allow rule for egress traffic with identity 0 when the egress policy is not enabled. It skips adding this rule when egress policy is enabled. With the proposed change, Cilium will add a new field say  “DefaultAllowEgress” and will be set only if there exists CCNP with defaultAllowEgress set. If there are any other network policies or CCNP without that field, then it will be disabled. With this change, Cilium will add default allow rule for egress traffic if it satisfies any of these two conditions:

1. The egress policy is not enabled for the endpoint.
2. The egress policy is enabled and the DefaultAllowEgress field is set.

```
apiVersion: "cilium.io/v2"

kind: CiliumClusterwideNetworkPolicy

metadata:

  name: "block-azure-wireserver"

spec:

  endpointSelector: {}

  egressDeny:

  - toCIDR:

    - 168.63.129.16/32

    toPorts:

    - ports:

      - port: "80"

        protocol: "TCP"

  defaultAllowEgress: true
```

#### Pros

* Offer lot of features and aligns with other policies
* Easy to update, add or delete policies
* Code changes are simple and doesn’t touch many components
* New CRD is not required.


#### Cons

* Updating existing CRD to add new field creates new variation to existing policy
* Deviate from the default deny behavior defined by the Kubernetes Network Policy standard
* Egress rules are not functional if customer creates CCNP with both egress rules (not egressDeny) and defaultEgressAllow set in the same policy.


### IPTable Approach

The Proposal is to add a new config and new CRD. If config “enforceCloudProviderPolicy” is defined, then cilium will watch on this new CRD on startup before starting serving any new endpoint requests. Cilium will read cloudprovider specific policy via CRD and invoke installing the iptable rules. Today cilium adds a rule on top of the FORWARD chain to redirect all traffic to a CILIUM_FORWARD chain. So any rule that comes after this would not be executed. The suggestion is to add a new chain which enforces cloud provider specific rules and it should be placed on top of the FORWARD chain. This will ensure that the rules in the new chain takes precedence over rules in the CILIUM_FORWARD chain and infrastructure policy will remain enforced.

Pros:

1. Code changes are simple and don’t modify existing scenarios.
2. Add once for all pods and no changes after that.

Cons:

1. Doesn’t work if bpf host routing is enabled. If bpf routing enabled, then all iptable rules are getting skipped. Should make sure this flag is enabled only if legacy-host-routing and install-iptable-rules is set.
2. Managing new CRD and its used only with iptables may confuse users.


### Separate Policy map and CRD

A separate CRD and  policy map will be created to handle policies which require restriction to certain endpoints without blocking all traffic. This policy map will be processed before the existing network policy map, allowing for more fine-grained control over policy enforcement.

The new policy map will not have a default deny or allow behaviour, meaning that it will only process rules explicitly defined within it.  Within the BPF code responsible for enforcing egress policies, a step will be added to check the new policy map before processing the regular network policy map. If the rules in the new policy map are not matched, Cilium will fallback to processing the rules in the regular network policy map.

More similar to approach specified here: [https://github.com/cilium/cilium/issues/7111#issuecomment-558864657](https://github.com/cilium/cilium/issues/7111#issuecomment-558864657)

Pros:



1. Supports both legacy and bpf host routing

Cons:



1. Considerable code change. Defining and processing new policy map touches multiple components
2. Addition of new CRD and managing it
3. Deviate from the default deny behavior defined by the Kubernetes Network Policy standard


### AdminNetworkPolicy

Kubernetes SIG-network is working on a new type of policy that has the ability to express more complex policies, and this does not come with the default semantics that selected endpoints are isolated by default.

Links:

* [https://github.com/kubernetes/enhancements/issues/2091](https://github.com/kubernetes/enhancements/issues/2091)
* [https://github.com/kubernetes/enhancements/tree/master/keps/sig-network/2091-admin-network-policy](https://github.com/kubernetes/enhancements/tree/master/keps/sig-network/2091-admin-network-policy)
* [https://github.com/cilium/cilium/issues/23380](https://github.com/cilium/cilium/issues/23380)

Pros:

* Kubernetes community is pushing towards this path

Cons:

* Complicated to implement
