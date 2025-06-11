# CFP-39646: Decouple internal policy engine from external API

**SIG: SIG-Policy** 

**Begin Design Discussion:** 2025-05-20

**Cilium Release:** 1.19

**Authors:** Blaz Zupan <blaz@google.com>

**Status:** Implementable

## Summary

This CFP proposes a new internal PolicyEntry structure to decouple Cilium's network policy representation from its public API. This aims to simplify the PolicyRepository, support existing features, and enable future rule prioritization, which is necessary for AdminNetworkPolicy.

## Motivation

The current internal representation of network policies relies on the public api.Rule structure, making it challenging to introduce new features without altering the public API.

## Goals

* Separate internal policy representation from the external API.
* Maintain a single, flat list of rules in the policy repository.
* Ensure full support for existing policy engine features.
* Enable future rule prioritization.

## Non-Goals

* New features are not included in this CFP; rule prioritization, in particular, is out of scope.

## Proposal

We are proposing to overhaul the internal storage of network policies within the PolicyRepository. This means each rule will become a PolicyEntry struct, and the complete policy will be a flat list of pointers to these structs. Every kind of network policy, including Kubernetes, Cilium, and Admin Network Policy, would then translate its public API into this standardized internal representation.

```go
type PolicyEntry struct {
        // DefaultDeny is true if affected subjects should have non-selected traffic denied
        DefaultDeny bool

        // Deny is true if this rule should deny traffic
        Deny bool

        // Ingress is true if rule should affect ingress traffic, false otherwise
        Ingress bool

        // EndpointSelector specifies the endpoint that this rule applies to
        EndpointSelector api.EndpointSelector

        // L3 specifies the source/destination endpoints or all endpoints if empty
        L3 api.EndpointSelectorSlice

        // L4 specifies the source/destination port rules or none if empty
        L4 api.PortRules

        // Authentication specifies the cryptographic authentication required for the traffic to be allowed
        Authentication *api.Authentication

        // Labels stores optional metadata
        Labels labels.LabelArray
}

// PolicyEntries is a slice of pointers to PolicyEntry
type PolicyEntries []*PolicyEntry
```

## Impacts / Key Questions

### Key Question: Reusing certain Cilium Network Policy API structs

We are proposing to keep using certain Cilium Network Policy API structs, like api.PortRules, within the new PolicyEntry structure. An alternative, using a slice of api.PortProtocol and a pointer to PerSelectorPolicy for policy definition, was evaluated. However, this would necessitate moving PerSelectorPolicy out of pkg/policy, causing a widespread impact. Therefore, we have decided to stick with api.PortRules. 

### Option: Use api.PortProtocol and PerSelectorPolicy to represent port rules

#### Pros

* Significantly simplified internal representation of port rules.

#### Cons

* Implementing this change would require moving PerSelectorPolicy outside of pkg/policy (likely to pkg/policy/types), which has broad impact.

## Future Milestones

This CFP will enable the introduction of rule priorities, which are required to support AdminNetworkPolicy.
