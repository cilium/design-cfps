# CFP-42904: Rule evaluation order with multiple network policies

**SIG: SIG-Policy** 

**Begin Design Discussion:** 2025-11-20

**Cilium Release:** 1.19

**Authors:** Blaz Zupan <blaz@google.com>

**Status:** Draft

## Summary

This CFP defines the order of evaluation for network policy rules when Kubernetes ClusterNetworkPolicy (formerly AdminNetworkPolicy), Kubernetes NetworkPolicy, and CiliumNetworkPolicy or CiliumClusterwideNetworkPolicy are all present in a cluster

## Motivation

ClusterNetworkPolicy is a cluster-scoped custom resource allowing administrators to define network policies that apply cluster-wide, taking precedence over namespaced network policies. It also allows the definition of a baseline tier, which specifies network policies that apply to traffic that is not matched by any other network policy ClusterNetworkPolicy assigns strict ordering to the evaluation of policy rules.

There are three levels of ordering:

### Tier

The **Admin tier** takes precedence over all other policies. Policies defined in this tier are used to set cluster-wide security rules that cannot be overridden in the other tiers. If a policy in the Admin tier renders a final decision (Accept or Deny) for a connection, evaluation stops.

The standard Kubernetes v1.NetworkPolicy resources operate at the **NetworkPolicy tier**. These policies always make a final decision for the pods they select. Pods not selected by any v1.NetworkPolicy fall through to the Baseline tier.

The **Baseline tier** provides cluster-wide default policies. Policies in the NetworkPolicy tier take precedence over the Baseline tier.

### Priority

Priority is a value from 0 to 1000 indicating the precedence of the policy within its tier. Lower priority values indicate higher precedence, meaning policies with lower values are evaluated first within the same tier. Admin tier policies always have higher precedence than NetworkPolicy or Baseline tier policies, regardless of the priority values within those tiers. If multiple policies in the same tier have the same priority and match a connection, the behavior is undefined. The implementation may choose any of the matching policies.

### Rule order

A maximum of 25 rules is allowed per direction (ingress and egress). Within a single CNP object, rules are evaluated in the order they are listed. Rules appearing earlier have higher precedence.

## Goals

* Implement ordering that conforms with the ClusterNetworkPolicy specifications
* Preserve the semantics of all existing network policies
* Make the interaction between various network policies easy to understand

## Non-Goals

* Changing the fundamental behavior of existing policies: This proposal aims to integrate CNP ordering around existing NP semantics, not alter how standard NetworkPolicies function on their own.
* Introducing policy ordering within the standard Kubernetes NetworkPolicy, CiliumNetworkPolicy or CiliumClusterwideNetworkPolicy APIs. The priority is an internal implementation detail to handle the tiering, not a proposed change to the Kubernetes APIs.
* Deprecating or replacing CiliumNetworkPolicy or CiliumClusterwideNetworkPolicy: These policy types will continue to be supported within the "NetworkPolicy" tier.

## Proposal

The interaction between ClusterNetworkPolicy and v1.NetworkPolicy is well defined in the ClusterNetworkPolicy CRD and is described above.

CiliumNetworkPolicy and CiliumClusterwideNetworkPolicy do not support explicit rule ordering. Currently, CiliumNetworkPolicy and CiliumClusterwideNetworkPolicy rules are ordered implicitly: Deny rules take precedence over Allow rules, and more specific L4/L7 rules take precedence over less specific ones. Otherwise, all rules effectively have the same priority. To maintain backward compatibility, CiliumNetworkPolicy and CiliumClusterwideNetworkPolicy will be placed within the NetworkPolicy tier.

[PR 42784](https://github.com/cilium/cilium/pull/42784) will implement the capability to explicitly order network policy rules by introducing a new *Priority* field in the internal PolicyEntry struct. Policies with lower numeric Priority values will have higher precedence. The default priority value will be 0, representing the highest precedence. To ensure predictable ordering, we will internally assign explicit priority values to all policy rules, including those that don't natively support explicit priorities.

The priority field is an *int32*, but only the first 24 bits are usable with the upper 8 bits reserved by the dataplane for mapping the ProxyPort precedence. ClusterNetworkPolicy supports a maximum of 1001 priorities (0-1000) with a maximum of 25 rules in each policy, giving us a theoretical maximum of 25025 priorities. 

There will be four numeric priority ranges: one for the admin tier, one for the regular tier (k8s NP and CNP/CCNP), one for the baseline tier and one for the explicit default allow or deny.


| Tier          | Priority range    |
| ------------- | ----------------- |
| Admin         | 0-25024           |
| NetworkPolicy | 100000            |
| Baseline      | 200000-225024     |
| Default       | 300000            |

The specific ranges are chosen to provide clear separation and ample space for future expansion:
* The ClusterNetworkPolicy spec allows for priorities 0-1000, with up to 25 rules per policy. Reserving a range of 100,000 for the Admin and Baseline tiers provides a simple mapping (as shown below) and room for growth.
* The NetworkPolicy and Default tiers do not have explicit priorities within their tiers, but a large range is reserved for consistency and internal use.
* CiliumNetworkPolicy and CiliumClusterwideNetworkPolicy will be assigned to the NetworkPolicy tier. This means that Admin tier policies will always override CiliumClusterwideNetworkPolicy rules.
* The priority numbering is purely internal to Cilium and only exists in memory within the Cilium Agent. It is thus possible to change these allocations at any time without any external impact.
* While it is possible to split the ranges on binary boundaries, there does not seem to be a compelling reason to do so.
* The priority mapping is applied independently for ingress and egress rules.
* The Default tier will only contain a single "allow all" or "deny all" fallback rule.
* The proposed priority range is well under the supported maximum (1^24 - 1).

Each ClusterNetworkPolicy rule will be mapped to a Cilium priority by multiplying the CNP priority with 25 and adding the zero-based relative order of the rules within each to the priority. For example:

| Tier          | Priority | Rule # | Calculation                 | Internal priority |
| ------------  | -------- | ------ | --------------------------- |------------------ |
| Admin         | 0        | 1      | 0 * 25 + 1 - 1              | 0                 |
| Admin         | 0        | 25     | 0 * 25 + 25 - 1             | 24                |
| Admin         | 1        | 1      | 1 * 25 + 1 - 1              | 25                |
| Admin         | 5        | 2      | 5 * 25 + 2 - 1              | 126               |
| Admin         | 12       | 7      | 12 * 25 + 7 - 1             | 306               |
| Admin         | 1000     | 25     | 1000 * 25 + 25 - 1          | 25024             |
| NetworkPolicy | N/A      | 1      | 100000                      | 100000            |
| NetworkPolicy | N/A      | 10     | 100000                      | 100000            |
| Baseline      | 0        | 1      | 200000 + 0 * 25 + 1 - 1     | 200000
| Baseline      | 1        | 1      | 200000 + 1 * 25 + 1 - 1     | 200025            |
| Baseline      | 50       | 3      | 200000 + 50 * 25 + 3 - 1    | 201252            |
| Baseline      | 1000     | 25     | 200000 + 1000 * 25 + 25 -1  | 225024            |
| Default       | N/A      | N/A    | 300000                      | 300000            |

ClusterNetworkPolicy specifications mandate that behavior of policies with the same priority is undefined. For the NetworkPolicy tier, the existing precedence rules (deny over allow) continue to be in force even though they all have the same internal priority. For simplicity Cilium will do the same for other tiers as well.

## Impacts / Key Questions

### Key Question: Should CiliumClusterwideNetworkPolicy be able to override ClusterNetworkPolicy rules?

### Option: Increase priority of CiliumClusterwideNetworkPolicy

#### Pros

* CCNP rules can override ClusterNetworkPolicy and Kubernetes NetworkPolicy rules.

#### Cons

* Backwards incompatible. Currently CCNP has the same priority as k8s NP and CilumNetworkPolicy.
* Relative priority vs ClusterNetworkPolicy is unclear. Should it be higher, lower or equal?

## Future Milestones

This CFP will enable the introduction of rule priorities, which are required to support AdminNetworkPolicy.
